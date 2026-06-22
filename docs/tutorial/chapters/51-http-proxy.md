# 51. http-proxy

对应示例：`examples/http-proxy`

理解正向 HTTP 代理的基本工作方式,尤其是 HTTPS 代理常用的 `CONNECT` 方法和 TCP tunnel。正向代理:客户端 → 代理 → 目标网站。

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

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

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
                println!("Failed to serve connection: {err:?}");
            }
        });
    }
}

async fn proxy(req: Request) -> Result<Response, hyper::Error> {
    tracing::trace!(?req);

    if let Some(host_addr) = req.uri().authority().map(|auth| auth.to_string()) {
        tokio::task::spawn(async move {
            match hyper::upgrade::on(req).await {
                Ok(upgraded) => {
                    if let Err(e) = tunnel(upgraded, host_addr).await {
                        tracing::warn!("server io error: {}", e);
                    };
                }
                Err(e) => tracing::warn!("upgrade error: {}", e),
            }
        });

        Ok(Response::new(Body::empty()))
    } else {
        tracing::warn!("CONNECT host is not socket addr: {:?}", req.uri());
        Ok((StatusCode::BAD_REQUEST, "CONNECT must be to a socket address").into_response())
    }
}

async fn tunnel(upgraded: Upgraded, addr: String) -> std::io::Result<()> {
    let mut server = TcpStream::connect(addr).await?;
    let mut upgraded = TokioIo::new(upgraded);

    let (from_client, from_server) =
        tokio::io::copy_bidirectional(&mut upgraded, &mut server).await?;

    tracing::debug!(
        "client wrote {} bytes and received {} bytes",
        from_client,
        from_server
    );

    Ok(())
}
````

## 运行

````bash
cd examples
cargo run -p example-http-proxy
````

另一终端通过代理访问 HTTPS:

````bash
curl -v -x "127.0.0.1:3000" https://tokio.rs
````

普通请求:

````bash
curl http://127.0.0.1:3000/
# Hello, World!
````

## 解读

### CONNECT 是什么

访问 HTTPS 网站时代理通常看不到 HTTP 内容。客户端先发 `CONNECT tokio.rs:443 HTTP/1.1`,意思是"请代理帮我打通到 tokio.rs:443 的 TCP 隧道,后面的 TLS 流量我自己和目标网站协商"。代理只负责转发字节,**不解密 HTTPS**。

### 本示例只做 CONNECT 隧道,不做明文 HTTP 转发

要点清常见误解:本示例**只处理 CONNECT**(HTTPS 隧道),非 CONNECT 请求直接交给本地 Router 返 Hello World,**没有转发**。

真正的"明文 HTTP 正向代理"(客户端发 `GET http://example.com/`,代理转发到 example.com)本示例没实现。如果要做明文代理转发,必须处理 **hop-by-hop headers**(RFC 7230 §6.1):

```text
Connection、Keep-Alive、Proxy-Authenticate、Proxy-Authorization、
TE、Trailer、Transfer-Encoding、Upgrade
```

这些 header 只在**相邻两跳**有效(客户端↔代理、代理↔目标),不能原样透传。比如客户端发 `Connection: close` 给代理,转发给目标会让目标误以为是"客户端要关闭"导致连接行为错乱。

本示例是 CONNECT 字节隧道(代理看不到 HTTP 内容),**不需要处理 hop-by-hop header**。但明文代理转发必须处理它们。第 52 章反向代理也有同样要求(`hyper-util` client 自动处理了一部分)。

> CONNECT 方法定义在 RFC 7231 §4.3.6。生产正向代理通常还带 `407 Proxy Authentication Required` 认证、ACL,本示例都省略。

### 按方法分流(Tower service)

````rust
let tower_service = tower::service_fn(move |req: Request<_>| {
    let router_svc = router_svc.clone();
    let req = req.map(Body::new);   // Hyper Incoming body 转 axum Body
    async move {
        if req.method() == Method::CONNECT {
            proxy(req).await              // CONNECT → proxy
        } else {
            router_svc.oneshot(req).await // 其他 → axum Router
        }
    }
});
````

`req.map(Body::new)` 把 Hyper `Incoming` body 转成 axum `Body`。然后用 `service_fn` 桥接成 Hyper service(同第 49 章),手写 accept loop。

### `serve_connection().with_upgrades()`

````rust
http1::Builder::new()
    .serve_connection(io, hyper_service)
    .with_upgrades()    // CONNECT 需要 upgrade 到底层 TCP tunnel
    .await
````

`with_upgrades()` 很关键——CONNECT 要升级到底层连接,没它 `hyper::upgrade::on(req)` 无法工作。

### `proxy` 处理 CONNECT

````rust
async fn proxy(req: Request) -> Result<Response, hyper::Error> {
    if let Some(host_addr) = req.uri().authority().map(|auth| auth.to_string()) {
        tokio::task::spawn(async move {
            match hyper::upgrade::on(req).await {       // 等待 upgrade
                Ok(upgraded) => { let _ = tunnel(upgraded, host_addr).await; }
                Err(e) => tracing::warn!("upgrade error: {}", e),
            }
        });
        Ok(Response::new(Body::empty()))               // 先返空响应表示 CONNECT 建立
    } else {
        Ok((StatusCode::BAD_REQUEST, "CONNECT must be to a socket address").into_response())
    }
}
````

CONNECT 目标在 URI authority(如 `tokio.rs:443`)。spawn 后台任务等待 upgrade,成功后进 `tunnel`;**先返回空响应**表示 CONNECT 建立(协议要求)。没目标地址返回 400。

### `tunnel` 双向转发

````rust
async fn tunnel(upgraded: Upgraded, addr: String) -> std::io::Result<()> {
    let mut server = TcpStream::connect(addr).await?;       // 连目标服务器
    let mut upgraded = TokioIo::new(upgraded);
    let (from_client, from_server) =
        tokio::io::copy_bidirectional(&mut upgraded, &mut server).await?;
    ...
}
````

代理隧道:客户端连接 ↔ 目标服务器连接。`copy_bidirectional` 同时两边数据拷贝:客户端字节 → 目标,目标字节 → 客户端。

## 常见问题

**这是反向代理吗?** 不是,是正向代理,客户端显式配置 `-x 127.0.0.1:3000`。

**代理会解密 HTTPS 吗?** 不会。CONNECT 只打通 TCP tunnel,TLS 在客户端和目标网站之间进行。

**为什么需要 `with_upgrades`?** CONNECT 要升级到底层连接,Hyper server 必须启用 upgrades。

## 手写任务

1. 手写 Hyper accept loop。
2. 普通请求交给 axum Router。
3. CONNECT 请求交给 `proxy`。
4. `hyper::upgrade::on(req)` 获取升级连接。
5. `TcpStream::connect` 连目标地址。
6. `copy_bidirectional` 转发字节。

## 小结

- HTTP 正向代理处理 HTTPS 的核心:`CONNECT host:port` → Hyper upgrade → 连接目标 → `copy_bidirectional` 转发字节。代理转发的是字节流,不理解里面的 HTTPS 内容。
- 本示例只做 CONNECT 隧道,不做明文 HTTP 转发——明文代理转发必须处理 hop-by-hop headers(`Connection`/`Keep-Alive`/`Proxy-*`/`TE`/`Trailer`/`Transfer-Encoding`/`Upgrade`),不能原样透传。
- `serve_connection().with_upgrades()` 是关键,CONNECT 需 upgrade 到底层 TCP tunnel。
- `proxy` 先返空响应表示 CONNECT 建立,后台 spawn 任务等待 upgrade 并建 tunnel。
- `tokio::io::copy_bidirectional` 实现双向字节转发,代理不解密 HTTPS。

## 源码对照

- `examples/http-proxy/Cargo.toml`
- `examples/http-proxy/src/main.rs`
