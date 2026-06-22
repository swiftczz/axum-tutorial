# 51. http-proxy

对应示例：`examples/http-proxy`

理解正向 HTTP 代理的基本工作方式，尤其是 HTTPS 代理常用的 `CONNECT` 方法和 TCP tunnel。正向代理：客户端 → 代理 → 目标网站。

相比前面章节新引入：`tower::service_fn`（桥接 Tower/Hyper service）、`hyper::upgrade::on`（协议升级）、`tokio::io::copy_bidirectional`（双向字节转发）。

分 3 步搭。

## Cargo.toml

````toml
[package]
name = "example-http-proxy"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
hyper = { version = "1", features = ["full"] }
hyper-util = "0.1.1"
tokio = { version = "1.0", features = ["full"] }
tower = { version = "0.5.2", features = ["make", "util"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

本章是 HTTP 代理，新增 `hyper` + `hyper-util`（用 hyper client 转发请求到下游）+ `tower` 启用 `make`（`MakeService` trait，把单个 service 转成可不断生产 service 的工厂）和 `util`（`ServiceExt` 等扩展）。对比第 52 章 reverse-proxy（更简单的代理）和第 15 章 reqwest 转发（高层 API）。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：理解 CONNECT 和 accept loop

访问 HTTPS 网站时，代理看不到里面的 HTTP 内容。客户端先发 `CONNECT tokio.rs:443`，意思是"请代理帮我打通到 tokio.rs:443 的 TCP 隧道，后面的 TLS 流量我自己和目标网站协商"。代理只负责转发字节，不解密 HTTPS。

这章需要手写 Hyper accept loop（同 ch49 serve-with-hyper），按请求方法分流：CONNECT 走代理，其他走 axum Router。

````rust
use axum::{
    body::Body,
    extract::Request,
    http::{Method, StatusCode},
    response::{IntoResponse, Response},
    routing::get,
    Router,
};
use hyper::body::Incoming;
use hyper_util::rt::TokioIo;
use std::net::SocketAddr;
use tokio::net::TcpListener;
use tower::Service;
use tower::ServiceExt;

#[tokio::main]
async fn main() {
    let router_svc = Router::new().route("/", get(|| async { "Hello, World!" }));

    // 按方法分流：CONNECT → proxy，其他 → axum Router
    let tower_service = tower::service_fn(move |req: Request<_>| {
        let router_svc = router_svc.clone();
        let req = req.map(Body::new);
        async move {
            if req.method() == Method::CONNECT {
                proxy(req).await
            } else {
                router_svc.oneshot(req).await.map_err(|err| match err {})
            }
        }
    });

    // 桥接成 Hyper service
    let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
        tower_service.clone().call(request)
    });

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    let listener = TcpListener::bind(addr).await.unwrap();
    loop {
        let (stream, _) = listener.accept().await.unwrap();
        let io = TokioIo::new(stream);
        let hyper_service = hyper_service.clone();
        tokio::task::spawn(async move {
            if let Err(err) = http1::Builder::new()
                .serve_connection(io, hyper_service)
                .with_upgrades()
                .await
            {
                eprintln!("Failed to serve connection: {err:?}");
            }
        });
    }
}
````

> **新面孔：`tower::service_fn` + 按方法分流**
>
> `service_fn` 把闭包包装成 Tower service。这里按 `req.method()` 分流：CONNECT 走代理逻辑，其他请求走 axum Router。
>
> `req.map(Body::new)` 把 Hyper 的 `Incoming` body 转成 axum 的 `Body`——因为 axum Router 需要自己的 body 类型。

---

## 第二步：proxy 处理 CONNECT——tunnel 隧道

CONNECT 请求的目标地址在 URI authority 里（如 `tokio.rs:443`）。先返回空响应表示隧道建立，后台 spawn 任务做双向转发。

````rust
use hyper::upgrade::Upgraded;

async fn proxy(req: Request) -> Result<Response, hyper::Error> {
    // 从 URI authority 取目标地址
    if let Some(host_addr) = req.uri().authority().map(|auth| auth.to_string()) {
        tokio::task::spawn(async move {
            // 等待 Hyper upgrade 成功，拿到底层连接
            match hyper::upgrade::on(req).await {
                Ok(upgraded) => {
                    if let Err(e) = tunnel(upgraded, host_addr).await {
                        eprintln!("server io error: {}", e);
                    };
                }
                Err(e) => eprintln!("upgrade error: {}", e),
            }
        });
        // 先返回空响应，表示 CONNECT 隧道可以建立
        Ok(Response::new(Body::empty()))
    } else {
        Ok((StatusCode::BAD_REQUEST, "CONNECT must be to a socket address").into_response())
    }
}
````

> **新面孔：`hyper::upgrade::on` + 先返空响应**
>
> CONNECT 需要"协议升级"——把 HTTP 连接升级成裸 TCP 隧道。`hyper::upgrade::on(req)` 等待升级完成，拿到底层连接 `Upgraded`。
>
> 协议要求服务端**先返回空响应**（`Response::new(Body::empty())`）表示隧道建立，然后后台才开始转发。所以 `tunnel` 在 `tokio::spawn` 里跑。

---

## 第三步：tunnel 双向转发

拿到升级后的连接后，连接目标服务器，双向拷贝字节：

````rust
async fn tunnel(upgraded: Upgraded, addr: String) -> std::io::Result<()> {
    // 连接目标服务器
    let mut server = tokio::net::TcpStream::connect(addr).await?;
    let mut upgraded = TokioIo::new(upgraded);

    // 双向拷贝：客户端 ↔ 目标服务器
    let (from_client, from_server) =
        tokio::io::copy_bidirectional(&mut upgraded, &mut server).await?;

    eprintln!("client wrote {} bytes and received {} bytes", from_client, from_server);
    Ok(())
}
````

> **新面孔：`tokio::io::copy_bidirectional`**
>
> 同时在两个方向拷贝字节：客户端 → 目标、目标 → 客户端。代理不解密 HTTPS，只转发字节流。
>
> 注意需要 `with_upgrades()`（第一步的 `serve_connection`），否则 `hyper::upgrade::on` 无法工作。

---

## 完整代码

````rust
use axum::{
    body::Body,
    extract::Request,
    http::{Method, StatusCode},
    response::{IntoResponse, Response},
    routing::get,
    Router,
};
use hyper::body::Incoming;
use hyper::server::conn::http1;
use hyper::upgrade::Upgraded;
use hyper_util::rt::TokioIo;
use std::net::SocketAddr;
use tokio::net::{TcpListener, TcpStream};
use tower::Service;
use tower::ServiceExt;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=trace,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let router_svc = Router::new().route("/", get(|| async { "Hello, World!" }));

    let tower_service = tower::service_fn(move |req: Request<_>| {
        let router_svc = router_svc.clone();
        let req = req.map(Body::new);
        async move {
            if req.method() == Method::CONNECT {
                proxy(req).await
            } else {
                router_svc.oneshot(req).await.map_err(|err| match err {})
            }
        }
    });

    let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
        tower_service.clone().call(request)
    });

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {}", addr);

    let listener = TcpListener::bind(addr).await.unwrap();
    loop {
        let (stream, _) = listener.accept().await.unwrap();
        let io = TokioIo::new(stream);
        let hyper_service = hyper_service.clone();

        tokio::task::spawn(async move {
            if let Err(err) = http1::Builder::new()
                .preserve_header_case(true)
                .title_case_headers(true)
                .serve_connection(io, hyper_service)
                .with_upgrades()
                .await
            {
                eprintln!("Failed to serve connection: {err:?}");
            }
        });
    }
}

async fn proxy(req: Request) -> Result<Response, hyper::Error> {
    if let Some(host_addr) = req.uri().authority().map(|auth| auth.to_string()) {
        tokio::task::spawn(async move {
            match hyper::upgrade::on(req).await {
                Ok(upgraded) => {
                    if let Err(e) = tunnel(upgraded, host_addr).await {
                        eprintln!("server io error: {}", e);
                    };
                }
                Err(e) => eprintln!("upgrade error: {}", e),
            }
        });

        Ok(Response::new(Body::empty()))
    } else {
        eprintln!("CONNECT host is not socket addr: {:?}", req.uri());
        Ok((StatusCode::BAD_REQUEST, "CONNECT must be to a socket address").into_response())
    }
}

async fn tunnel(upgraded: Upgraded, addr: String) -> std::io::Result<()> {
    let mut server = TcpStream::connect(addr).await?;
    let mut upgraded = TokioIo::new(upgraded);

    let (from_client, from_server) =
        tokio::io::copy_bidirectional(&mut upgraded, &mut server).await?;

    eprintln!("client wrote {} bytes and received {} bytes", from_client, from_server);

    Ok(())
}
````

## 运行

````bash
cd examples
cargo run -p example-http-proxy
````

通过代理访问 HTTPS：

````bash
curl -v -x "127.0.0.1:3000" https://tokio.rs
````

普通请求：

````bash
curl http://127.0.0.1:3000/
# Hello, World!
````

## 手写任务

1. 手写 Hyper accept loop。
2. 普通请求交给 axum Router。
3. CONNECT 请求交给 `proxy`。
4. `hyper::upgrade::on(req)` 获取升级连接。
5. `TcpStream::connect` 连接目标地址。
6. `copy_bidirectional` 转发字节。

## 小结

这章分 3 步搭了 HTTP 正向代理：

1. **CONNECT + accept loop**：`service_fn` 按方法分流，`with_upgrades()` 支持 CONNECT 升级。
2. **proxy 处理**：从 URI authority 取目标地址，`hyper::upgrade::on` 等待升级，先返空响应。
3. **tunnel 双向转发**：`copy_bidirectional` 同时两个方向拷贝字节，代理不解密 HTTPS。

核心模式：`CONNECT host:port` → upgrade → 连接目标 → `copy_bidirectional`。

## 源码对照

- `examples/http-proxy/Cargo.toml`
- `examples/http-proxy/src/main.rs`
