# 53. low-level-rustls

对应示例：`examples/low-level-rustls`

第 46 章用 `axum-server::bind_rustls` 一行启动 HTTPS server，那是 high-level API。这章走 **low-level** 路径：**手动接管 TCP listener → 在每条连接上做 TLS 握手 → 把 TLS 流交给 hyper 服务**。理解这层就理解了 axum server 内部到底是怎么跑的。

分 3 步：先建 rustls 配置（证书/私钥），再写手动 TLS listener 循环（核心难点），最后用 hyper 的 `serve_connection_with_upgrades` 服务单条连接。

相比前面章节新引入：**`TlsAcceptor`、`ServerConfig`、`tokio_rustls::pki_types`、`hyper_util::server::conn::auto::Builder`、`TokioIo` 适配器、`hyper::service::service_fn` 把 Router 包成 hyper Service**。

## Cargo.toml

````toml
[package]
name = "example-low-level-rustls"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
hyper = "1.0.0"
hyper-util = { version = "0.1", features = ["tokio", "server-auto"] }
rustls = { version = "0.23", default-features = false, features = ["ring"] }
tokio = { version = "1.0", features = ["full"] }
tokio-rustls = "0.26"
tower-service = "0.3"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：rustls `ServerConfig`（证书 + 私钥 + ALPN）

任何 TLS server 都要配置：证书（公钥，给客户端验证）+ 私钥（解密）+ ALPN（协商 HTTP/2 还是 HTTP/1.1）。这步写 `rustls_server_config` 函数返回 `Arc<ServerConfig>`。

````rust
use std::{
    path::{Path, PathBuf},
    sync::Arc,
};
use tokio_rustls::{
    rustls::pki_types::{pem::PemObject, CertificateDer, PrivateKeyDer},
    rustls::ServerConfig,
};

fn rustls_server_config(key: impl AsRef<Path>, cert: impl AsRef<Path>) -> Arc<ServerConfig> {
    let key = PrivateKeyDer::from_pem_file(key).unwrap();

    let certs = CertificateDer::pem_file_iter(cert)
        .unwrap()
        .map(|cert| cert.unwrap())
        .collect();

    let mut config = ServerConfig::builder()
        .with_no_client_auth()              // 不要求客户端证书（单向 TLS）
        .with_single_cert(certs, key)       // 配置证书和私钥
        .expect("bad certificate/key");

    // ALPN：协商协议，http/1.1 和 h2 都支持
    config.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1".to_vec()];

    Arc::new(config)
}
````

> **新面孔：`ServerConfig`**
>
> rustls 的服务端配置，builder 模式：
> - `ServerConfig::builder()` 开始构建
> - `.with_no_client_auth()` 不要求客户端证书（普通 HTTPS 都这样，单向认证）
> - `.with_single_cert(certs, key)` 配置服务器证书链和私钥
>
> `Arc<ServerConfig>` 是共享的不可变句柄——后续多个连接共用一份配置。

> **新面孔：`pki_types::pem`**
>
> rustls 0.23 引入的 PEM 解析 API：
> - `PrivateKeyDer::from_pem_file(path)` 从 PEM 文件读私钥
> - `CertificateDer::pem_file_iter(path)` 迭代读多张证书（证书链：服务器证书 + 中间证书）
>
> 之前用 `rustls-pemfile` crate，现在 rustls 自带，少一个依赖。

> **新面孔：ALPN（`alpn_protocols`）**
>
> Application-Layer Protocol Negotiation，TLS 握手时协商上层协议：
> - `b"http/1.1"` 协商 HTTP/1.1
> - `b"h2"` 协商 HTTP/2
>
> 浏览器和 curl 都用 ALPN 决定走哪个协议。配置 ALPN 后同一个 TLS 端口能同时支持 HTTP/1.1 和 HTTP/2，由 hyper-util 的 auto Builder 自动选择。

---

## 第二步：手动 TCP listener + TLS 握手循环（核心难点）

这步是本章核心：手动接管 TCP listener，每条连接 spawn 一个 task 做 TLS 握手，再把 TLS 流交给 hyper。axum 的 `axum::serve` 内部就是这么做的，这章把它展开。

````rust
use axum::{routing::get, Router};
use tokio::net::TcpListener;
use tokio_rustls::TlsAcceptor;
use tracing::info;

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(/* tracing 初始化 */)
        .with(tracing_subscriber::fmt::layer())
        .init();

    let rustls_config = rustls_server_config(
        PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("self_signed_certs").join("key.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("self_signed_certs").join("cert.pem"),
    );

    let tls_acceptor = TlsAcceptor::from(rustls_config);
    let bind = "[::1]:3000";
    let tcp_listener = TcpListener::bind(bind).await.unwrap();
    info!("HTTPS server listening on {bind}. To contact curl -k https://localhost:3000");
    let app = Router::new().route("/", get(handler));

    loop {
        let tower_service = app.clone();
        let tls_acceptor = tls_acceptor.clone();

        // 接受新 TCP 连接
        let (cnx, addr) = tcp_listener.accept().await.unwrap();

        tokio::spawn(async move {
            // 在这条 TCP 连接上做 TLS 握手
            let Ok(stream) = tls_acceptor.accept(cnx).await else {
                tracing::error!("error during tls handshake connection from {}", addr);
                return;
            };

            // stream 是 TLS 加密流（解密后可读 HTTP）
            // 下一步用 hyper 服务这个 stream
            todo!("交给 hyper 服务连接")
        });
    }
}

async fn handler() -> &'static str {
    "Hello, World!"
}

# fn rustls_server_config(key: impl AsRef<Path>, cert: impl AsRef<Path>) -> Arc<ServerConfig> { /* ... */ }
````

> **新面孔：`TlsAcceptor`**
>
> rustls 的"接受端"：把 `Arc<ServerConfig>` 包装成 `TlsAcceptor`（可 clone 共享）。`tls_acceptor.accept(tcp_stream).await` 在 TCP 流上做 TLS 握手，返回加密流（`TlsStream`）。
>
> 握手失败（如客户端不信任证书）返回 Err，需要处理。这章直接 log 错误后 return。

> **新面孔：手动 TCP 循环 + spawn task**
>
> 这是 low-level HTTP server 的标准骨架：
>
> ```text
> loop {
>     let (cnx, addr) = tcp_listener.accept().await;   // 等新连接
>     tokio::spawn(async move {
>         // 在这条连接上做所有事（握手、服务、清理）
>     });
> }
> ```
>
> 每条连接独立 spawn task，互不阻塞。`axum::serve` 内部就是这么实现的——这章把它显式展开。

---

## 第三步：用 hyper `serve_connection_with_upgrades` 服务单条连接

最后把 TLS 流交给 hyper：`TokioIo` 适配 IO trait，`hyper::service::service_fn` 把 Router 包成 hyper Service，`auto::Builder` 自动选 HTTP/1.1 或 HTTP/2。

````rust
use axum::extract::Request;
use hyper::body::Incoming;
use hyper_util::rt::{TokioExecutor, TokioIo};
use tower_service::Service;
use tracing::warn;

# loop {
#     let tower_service = app.clone();
#     let tls_acceptor = tls_acceptor.clone();
#     let (cnx, addr) = tcp_listener.accept().await.unwrap();
#     tokio::spawn(async move {
#         let Ok(stream) = tls_acceptor.accept(cnx).await else { /* ... */ return; };

        // 1. TokioIo 把 tokio IO 适配成 hyper IO
        let stream = TokioIo::new(stream);

        // 2. service_fn 把 Router 包成 hyper Service
        let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
            // clone 是必须的：hyper 的 Service 用 &self，tower 的 Service 用 &mut self
            // Router 总是 ready，所以不用 poll_ready
            tower_service.clone().call(request)
        });

        // 3. auto::Builder 服务这条连接，自动选 h1 或 h2
        let ret = hyper_util::server::conn::auto::Builder::new(TokioExecutor::new())
            .serve_connection_with_upgrades(stream, hyper_service)
            .await;

        if let Err(err) = ret {
            warn!("error serving connection from {}: {}", addr, err);
        }
#     });
# }
````

> **新面孔：`TokioIo` 适配器**
>
> tokio 的 IO 实现 `tokio::io::{AsyncRead, AsyncWrite}`，hyper 要求 `hyper::rt::{Read, Write}`——两套独立 trait。`TokioIo::new(stream)` 把 tokio IO 适配成 hyper IO。
>
> 第 50 章 UDS 已经见过它。所有 hyper low-level 代码都要用 `TokioIo` 包装。

> **新面孔：`hyper::service::service_fn`**
>
> hyper 不直接用 `tower::Service`，它有自己的 `hyper::service::Service` trait（接口略不同）。`service_fn(closure)` 把闭包包成 hyper Service。
>
> 闭包内部 `tower_service.clone().call(request)` 调用 axum 的 Router。clone 是必须的：hyper 用 `&self`，tower 用 `&mut self`——每次 clone 一个新 Router 处理这条连接。Router 的 clone 很便宜（内部 `Arc`）。

> **新面孔：`hyper_util::server::conn::auto::Builder`**
>
> hyper-util 提供的"自动协议选择"连接 server：
> - `Builder::new(TokioExecutor::new())` 创建 builder（`TokioExecutor` 是 hyper 用的 task spawner）
> - `.serve_connection_with_upgrades(io, service)` 服务一条连接，支持 HTTP upgrade（如 WebSocket）
> - **auto**：根据 ALPN 自动选 HTTP/1.1 还是 HTTP/2
>
> 这就是 `axum::serve` 内部最终调用的东西。

---

## 完整代码

````rust
use axum::{extract::Request, routing::get, Router};
use hyper::body::Incoming;
use hyper_util::rt::{TokioExecutor, TokioIo};
use std::{
    path::{Path, PathBuf},
    sync::Arc,
};
use tokio::net::TcpListener;
use tokio_rustls::{
    rustls::pki_types::{pem::PemObject, CertificateDer, PrivateKeyDer},
    rustls::ServerConfig,
    TlsAcceptor,
};
use tower_service::Service;
use tracing::{error, info, warn};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let rustls_config = rustls_server_config(
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("key.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("cert.pem"),
    );

    let tls_acceptor = TlsAcceptor::from(rustls_config);
    let bind = "[::1]:3000";
    let tcp_listener = TcpListener::bind(bind).await.unwrap();
    info!("HTTPS server listening on {bind}. To contact curl -k https://localhost:3000");
    let app = Router::new().route("/", get(handler));

    loop {
        let tower_service = app.clone();
        let tls_acceptor = tls_acceptor.clone();

        // Wait for new tcp connection
        let (cnx, addr) = tcp_listener.accept().await.unwrap();

        tokio::spawn(async move {
            // Wait for tls handshake to happen
            let Ok(stream) = tls_acceptor.accept(cnx).await else {
                error!("error during tls handshake connection from {}", addr);
                return;
            };

            // Hyper has its own `AsyncRead` and `AsyncWrite` traits and doesn't use tokio.
            // `TokioIo` converts between them.
            let stream = TokioIo::new(stream);

            // Hyper also has its own `Service` trait and doesn't use tower. We can use
            // `hyper::service::service_fn` to create a hyper `Service` that calls our app through
            // `tower::Service::call`.
            let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
                // We have to clone `tower_service` because hyper's `Service` uses `&self` whereas
                // tower's `Service` requires `&mut self`.
                //
                // We don't need to call `poll_ready` since `Router` is always ready.
                tower_service.clone().call(request)
            });

            let ret = hyper_util::server::conn::auto::Builder::new(TokioExecutor::new())
                .serve_connection_with_upgrades(stream, hyper_service)
                .await;

            if let Err(err) = ret {
                warn!("error serving connection from {}: {}", addr, err);
            }
        });
    }
}

async fn handler() -> &'static str {
    "Hello, World!"
}

fn rustls_server_config(key: impl AsRef<Path>, cert: impl AsRef<Path>) -> Arc<ServerConfig> {
    let key = PrivateKeyDer::from_pem_file(key).unwrap();

    let certs = CertificateDer::pem_file_iter(cert)
        .unwrap()
        .map(|cert| cert.unwrap())
        .collect();

    let mut config = ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(certs, key)
        .expect("bad certificate/key");

    config.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1".to_vec()];

    Arc::new(config)
}
````

## 运行

````bash
cd examples
cargo run -p example-low-level-rustls
````

````bash
curl -k https://localhost:3000/
# 返回 "Hello, World!"
````

`-k` 忽略自签名证书警告（生产环境用受信任 CA 签发的证书）。

## 解读

### 三层关系：axum → hyper → TCP/TLS

```text
axum Router      (高层：路由、handler、extractor)
   ↓
hyper            (中层：HTTP/1.1 + HTTP/2 协议解析)
   ↓
TLS 流           (rustls：加解密)
   ↓
TCP 连接         (tokio：传输)
```

`axum::serve` 把这三层都封装好了。这章手动展开 TLS 层和 hyper 层——理解它就理解了 axum server 全栈。

### 为什么手动写

99% 场景用 `axum::serve` 或 `axum_server::bind_rustls` 就够了。手动写 low-level 的场景：

- 自定义 TLS 配置（如客户端证书认证 `with_client_cert_verifier`）
- 自定义传输层（如 QUIC、Unix Socket + TLS）
- 学习理解原理
- axum 还没封装的新协议

## 常见问题

**为什么用 `loop` 不用 `tokio::spawn` 一开始就并发？** `accept` 是单点的（同一时刻只能接受一条连接）。`spawn` 在握手后开——每条连接独立 task。

**`serve_connection_with_upgrades` 的 upgrades 是什么？** 支持 HTTP upgrade 机制，主要是 WebSocket（HTTP/1.1 的 Connection: Upgrade）。`-with_upgrades` 后缀表示支持 WebSocket。

**为什么需要 `TokioExecutor`？** hyper 内部要 spawn task（解析流、写响应），`TokioExecutor` 告诉它用 tokio runtime spawn。其他 runtime（async-std、smol）要提供各自的 executor。

## 手写任务

1. 改 `with_no_client_auth()` 为 `with_client_cert_verifier(...)`，要求客户端证书（双向 TLS/mTLS）。
2. 改 ALPN 顺序为 `["http/1.1", "h2"]`，强制 HTTP/1.1 优先，对比效果。
3. 加 graceful shutdown：用 `tokio::select!` 同时等 Ctrl+C 和 `serve_connection`，收到信号后停。

## 小结

这章用 3 步展开了 HTTPS server 的内部：

1. **rustls ServerConfig**：证书/私钥 + ALPN，`with_no_client_auth` + `with_single_cert`。
2. **手动 TCP + TLS 循环**：`TcpListener::accept` + `TlsAcceptor::accept` + `tokio::spawn`，每条连接独立 task。
3. **hyper 服务连接**：`TokioIo` 适配 IO + `service_fn` 包 Router + `auto::Builder::serve_connection_with_upgrades` 自动选协议。

核心理解：**axum → hyper → TLS → TCP** 四层栈，这章展开中间两层。low-level API 不是用来日常写，是用来理解原理、做高度定制。

## 源码对照

- `examples/low-level-rustls/Cargo.toml`
- `examples/low-level-rustls/src/main.rs`
- `examples/low-level-rustls/self_signed_certs/cert.pem`
- `examples/low-level-rustls/self_signed_certs/key.pem`
