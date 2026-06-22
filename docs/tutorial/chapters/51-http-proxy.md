# 51. http-proxy

对应示例：`examples/http-proxy`

本章目标：理解正向 HTTP 代理的基本工作方式，尤其是 HTTPS 代理常用的 `CONNECT` 方法和 TCP tunnel。

代理分很多种。  
本章是正向代理：

```text
客户端 -> 代理 -> 目标网站
```

运行示例后，可以这样请求：

````bash
curl -v -x "127.0.0.1:3000" https://tokio.rs
````

## 这个小项目在做什么

服务监听：

```text
127.0.0.1:3000
```

如果请求是普通 HTTP：

```text
GET /
```

就交给 Axum Router 返回：

```text
Hello, World!
```

如果请求方法是：

```text
CONNECT
```

就进入代理逻辑：

```text
读取目标 host:port
-> 等待 Hyper upgrade
-> TcpStream::connect(host:port)
-> copy_bidirectional 在客户端和目标之间双向转发字节
```

## 先理解 CONNECT

访问 HTTPS 网站时，代理通常看不到里面的 HTTP 内容。  
客户端会先发：

```text
CONNECT tokio.rs:443 HTTP/1.1
```

意思是：

```text
请代理帮我打通到 tokio.rs:443 的 TCP 隧道
后面的 TLS 流量我自己和目标网站协商
```

代理只负责转发字节，不解密 HTTPS 内容。

### 本示例只做 CONNECT 隧道，不做明文 HTTP 转发

要点清一个常见误解：本示例**只处理 CONNECT**（HTTPS 隧道），非 CONNECT 请求直接交给本地 Router 返回 Hello World，**没有转发**。

真正的"明文 HTTP 正向代理"（客户端发 `GET http://example.com/`，代理转发到 example.com）本示例没实现。如果你要做明文代理转发，必须处理一组特殊 header——**hop-by-hop headers**（RFC 7230 §6.1）：

```text
Connection
Keep-Alive
Proxy-Authenticate
Proxy-Authorization
TE
Trailer
Transfer-Encoding
Upgrade
```

这些 header 只在**相邻两跳**之间有效（客户端↔代理、代理↔目标），不能原样透传。比如客户端发 `Connection: close` 给代理，如果你把这个 header 转发给目标服务器，目标会误以为是"客户端要关闭"，导致连接行为错乱。

本示例因为是 CONNECT 字节隧道（代理看不到 HTTP 内容），**不需要处理 hop-by-hop header**。但你要知道明文代理转发必须处理它们。第 52 章反向代理也有同样的要求（虽然 `hyper-util` client 自动处理了一部分）。

> 顺带一提：CONNECT 方法定义在 RFC 7231 §4.3.6，想深入了解可以查阅。生产正向代理通常还会带 `407 Proxy Authentication Required` 认证、访问控制列表等，本示例都省略了。

## 文件和依赖

这个 example 有两个文件：

1. `examples/http-proxy/Cargo.toml`
2. `examples/http-proxy/src/main.rs`

关键依赖：

- `hyper`：低层 HTTP/1 连接和 upgrade。
- `hyper-util::TokioIo`：Tokio IO 与 Hyper IO 适配。
- `tower`：桥接 service。
- `tokio::io::copy_bidirectional`：双向拷贝隧道数据。
- `axum`：普通非 CONNECT 请求仍可交给 Router。

## 第一步：构造 Tower service

源码：

````rust
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
````

这段按方法分流：

```text
CONNECT -> proxy
其他请求 -> Axum Router
```

`req.map(Body::new)` 把 Hyper `Incoming` body 转成 Axum `Body`。

## 第二步：桥接成 Hyper service

源码：

````rust
let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
    tower_service.clone().call(request)
});
````

Hyper 低层 server 需要 Hyper service。  
这里把 Tower service 包起来给 Hyper 调用。

## 第三步：手写 accept loop

源码：

````rust
let listener = TcpListener::bind(addr).await.unwrap();
loop {
    let (stream, _) = listener.accept().await.unwrap();
    let io = TokioIo::new(stream);
    let hyper_service = hyper_service.clone();
    tokio::task::spawn(async move {
        ...
    });
}
````

每个 TCP 连接独立 spawn。  
这是代理服务的基本并发模型。

## 第四步：serve_connection with_upgrades

源码：

````rust
http1::Builder::new()
    .preserve_header_case(true)
    .title_case_headers(true)
    .serve_connection(io, hyper_service)
    .with_upgrades()
    .await
````

`with_upgrades()` 很关键。  
CONNECT 需要升级到底层 TCP tunnel。

如果没有它，`hyper::upgrade::on(req)` 无法正常工作。

## 第五步：proxy 处理 CONNECT

源码：

````rust
async fn proxy(req: Request) -> Result<Response, hyper::Error> {
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
        Ok((StatusCode::BAD_REQUEST, "CONNECT must be to a socket address").into_response())
    }
}
````

核心逻辑：

1. 从 URI authority 取目标地址。
2. spawn 后台任务等待 upgrade。
3. upgrade 成功后进入 `tunnel`。
4. 先返回空响应，表示 CONNECT 建立。

如果没有目标地址，返回 400。

## 第六步：tunnel 双向转发

源码：

````rust
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

这就是代理隧道：

```text
客户端连接 <-> 目标服务器连接
```

`copy_bidirectional` 同时做两边数据拷贝：

```text
客户端发来的字节 -> 目标服务器
目标服务器返回的字节 -> 客户端
```

## 函数职责速查

- `main`：初始化日志，构造 Router 和 proxy service，手写 Hyper accept loop。
- `proxy`：处理 CONNECT 请求，拿目标地址，等待 upgrade，并启动 tunnel。
- `tunnel`：连接目标服务器，并双向转发字节。

## 带中文注释的手写版

完整 `src/main.rs`：

````rust
//! Run with
//!
//! ```not_rust
//! $ cargo run -p example-http-proxy
//! ```
//!
//! In another terminal:
//!
//! ```not_rust
//! $ curl -v -x "127.0.0.1:3000" https://tokio.rs
//! ```
//!
//! Example is based on <https://github.com/hyperium/hyper/blob/master/examples/http_proxy.rs>

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
    // 初始化日志。代理代码里会打印 CONNECT、upgrade、tunnel 的信息。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=trace,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 普通 HTTP 请求仍然可以交给 Axum Router。
    let router_svc = Router::new().route("/", get(|| async { "Hello, World!" }));

    // 这一层 Tower service 用来按请求方法分流。
    let tower_service = tower::service_fn(move |req: Request<_>| {
        let router_svc = router_svc.clone();

        // Hyper server 给进来的是 Incoming body，这里转成 Axum Body。
        let req = req.map(Body::new);

        async move {
            if req.method() == Method::CONNECT {
                // HTTPS 正向代理的核心：CONNECT 建 tunnel。
                proxy(req).await
            } else {
                // 非 CONNECT 请求继续走普通 Axum 路由。
                router_svc.oneshot(req).await.map_err(|err| match err {})
            }
        }
    });

    // Hyper 低层 server 需要 hyper::Service，所以把 Tower service 包一层。
    let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
        tower_service.clone().call(request)
    });

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {}", addr);

    let listener = TcpListener::bind(addr).await.unwrap();
    loop {
        // 手写 accept loop：每个 TCP 连接由一个任务处理。
        let (stream, _) = listener.accept().await.unwrap();
        let io = TokioIo::new(stream);
        let hyper_service = hyper_service.clone();

        tokio::task::spawn(async move {
            if let Err(err) = http1::Builder::new()
                // 这两个设置保留示例代理对 header 大小写的处理方式。
                .preserve_header_case(true)
                .title_case_headers(true)
                .serve_connection(io, hyper_service)
                // CONNECT 需要 upgrade 到底层连接，所以这里必须开启 upgrades。
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

    // CONNECT 的目标通常在 URI authority 中，例如 tokio.rs:443。
    if let Some(host_addr) = req.uri().authority().map(|auth| auth.to_string()) {
        tokio::task::spawn(async move {
            // 等待 Hyper upgrade 成功，拿到底层连接。
            match hyper::upgrade::on(req).await {
                Ok(upgraded) => {
                    if let Err(e) = tunnel(upgraded, host_addr).await {
                        tracing::warn!("server io error: {}", e);
                    };
                }
                Err(e) => tracing::warn!("upgrade error: {}", e),
            }
        });

        // 返回空响应，表示 CONNECT 可以建立。
        Ok(Response::new(Body::empty()))
    } else {
        tracing::warn!("CONNECT host is not socket addr: {:?}", req.uri());
        Ok((
            StatusCode::BAD_REQUEST,
            "CONNECT must be to a socket address",
        )
            .into_response())
    }
}

async fn tunnel(upgraded: Upgraded, addr: String) -> std::io::Result<()> {
    // 连接目标服务器。
    let mut server = TcpStream::connect(addr).await?;
    // 把 upgraded 连接适配成 Tokio IO。
    let mut upgraded = TokioIo::new(upgraded);

    // 双向拷贝数据。
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

## 运行和验证

运行：

````bash
cargo run -p example-http-proxy
````

另一个终端：

````bash
curl -v -x "127.0.0.1:3000" https://tokio.rs
````

普通请求：

````bash
curl http://127.0.0.1:3000/
````

应返回：

```text
Hello, World!
```

## 常见卡点

### 1. 这是反向代理吗？

不是。  
这是正向代理，客户端显式配置 `-x 127.0.0.1:3000`。

### 2. 代理会解密 HTTPS 吗？

不会。  
CONNECT 只是打通 TCP tunnel，TLS 仍然在客户端和目标网站之间进行。

### 3. 为什么需要 with_upgrades？

CONNECT 要升级到底层连接。  
Hyper server 必须启用 upgrades。

## 手写任务

1. 手写 Hyper accept loop。
2. 普通请求交给 Axum Router。
3. CONNECT 请求交给 `proxy`。
4. 用 `hyper::upgrade::on(req)` 获取升级连接。
5. `TcpStream::connect` 连接目标地址。
6. `copy_bidirectional` 转发字节。

## 本章真正要记住什么

HTTP 正向代理处理 HTTPS 的核心是：

```text
CONNECT host:port
Hyper upgrade
连接目标 host:port
copy_bidirectional
```

代理此时转发的是字节流，不理解里面的 HTTPS 内容。

## 源码对照

本章手写版对应源码：

- `examples/http-proxy/src/main.rs`
- `examples/http-proxy/Cargo.toml`
