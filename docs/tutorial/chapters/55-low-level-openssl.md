# 55. low-level-openssl

对应示例：`examples/low-level-openssl`

本章目标：使用 OpenSSL 手写 TLS accept loop，并把 `tokio_openssl::SslStream` 交给 Hyper/Axum。

这是第三个低层 TLS 示例。  
结构和前两章相同，TLS 库换成 OpenSSL。

## 这个小项目在做什么

链路：

```text
TcpListener
-> OpenSSL SslAcceptor
-> tokio_openssl::SslStream
-> TokioIo
-> Hyper
-> Axum Router
```

## 关键依赖

- `openssl`
- `tokio-openssl`
- `hyper`
- `hyper-util`
- `tower`
- `axum`

## 第一步：创建 OpenSSL acceptor

源码：

````rust
let mut tls_builder = SslAcceptor::mozilla_modern_v5(SslMethod::tls()).unwrap();
````

`mozilla_modern_v5` 使用较现代的 TLS 配置模板。

## 第二步：加载证书和私钥

源码：

````rust
tls_builder
    .set_certificate_file(path_to_cert, SslFiletype::PEM)
    .unwrap();

tls_builder
    .set_private_key_file(path_to_key, SslFiletype::PEM)
    .unwrap();

tls_builder.check_private_key().unwrap();
````

最后 `check_private_key` 确认证书和私钥匹配。

## 第三步：每个 TCP 连接执行 TLS 握手

源码：

````rust
let ssl = Ssl::new(tls_acceptor.context()).unwrap();
let mut tls_stream = SslStream::new(ssl, cnx).unwrap();
if let Err(err) = SslStream::accept(Pin::new(&mut tls_stream)).await {
    error!("error during tls handshake connection from {}: {}", addr, err);
    return;
}
````

OpenSSL 这里需要显式创建 `Ssl` 和 `SslStream`。  
然后 `accept(...).await` 完成 TLS 握手。

## 第四步：交给 Hyper/Axum

握手成功后和前两章一样：

````rust
let stream = TokioIo::new(tls_stream);

let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
    tower_service.clone().call(request)
});

hyper_util::server::conn::auto::Builder::new(TokioExecutor::new())
    .serve_connection_with_upgrades(stream, hyper_service)
    .await
````

## 函数职责速查

- `main`：创建 OpenSSL acceptor，accept TCP，TLS 握手，交给 Hyper。
- `handler`：返回 Hello。

## 带中文注释的手写版

完整 `src/main.rs`：

````rust
//! Run with
//!
//! ```not_rust
//! cargo run -p example-low-level-openssl
//! ```

use axum::{http::Request, routing::get, Router};
use hyper::body::Incoming;
use hyper_util::rt::{TokioExecutor, TokioIo};
use openssl::ssl::{Ssl, SslAcceptor, SslFiletype, SslMethod};
use std::{path::PathBuf, pin::Pin};
use tokio::net::TcpListener;
use tokio_openssl::SslStream;
use tower::Service;
use tracing::{error, info, warn};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // 初始化日志。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 使用 OpenSSL 提供的现代 TLS 配置模板。
    let mut tls_builder = SslAcceptor::mozilla_modern_v5(SslMethod::tls()).unwrap();

    // 加载服务端证书。
    tls_builder
        .set_certificate_file(
            PathBuf::from(env!("CARGO_MANIFEST_DIR"))
                .join("self_signed_certs")
                .join("cert.pem"),
            SslFiletype::PEM,
        )
        .unwrap();

    // 加载服务端私钥。
    tls_builder
        .set_private_key_file(
            PathBuf::from(env!("CARGO_MANIFEST_DIR"))
                .join("self_signed_certs")
                .join("key.pem"),
            SslFiletype::PEM,
        )
        .unwrap();

    // 确认证书和私钥匹配。
    tls_builder.check_private_key().unwrap();

    let tls_acceptor = tls_builder.build();

    let bind = "[::1]:3000";
    let tcp_listener = TcpListener::bind(bind).await.unwrap();
    info!("HTTPS server listening on {bind}. To contact curl -k https://localhost:3000");

    let app = Router::new().route("/", get(handler));

    loop {
        let tower_service = app.clone();
        let tls_acceptor = tls_acceptor.clone();

        // 先接收普通 TCP 连接。
        let (cnx, addr) = tcp_listener.accept().await.unwrap();

        tokio::spawn(async move {
            // 为当前连接创建 OpenSSL SslStream。
            let ssl = Ssl::new(tls_acceptor.context()).unwrap();
            let mut tls_stream = SslStream::new(ssl, cnx).unwrap();

            // 执行 TLS 握手。tokio-openssl 这里需要 Pin。
            if let Err(err) = SslStream::accept(Pin::new(&mut tls_stream)).await {
                error!(
                    "error during tls handshake connection from {}: {}",
                    addr, err
                );
                return;
            }

            // TLS stream 适配成 Hyper IO。
            let stream = TokioIo::new(tls_stream);

            // Hyper service 调用 Axum Router。
            let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
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
````

## 运行和验证

````bash
cargo run -p example-low-level-openssl
curl -k https://localhost:3000
````

如果本机缺少 OpenSSL 开发库，编译可能失败。  
这属于本地依赖环境问题，不是 Axum handler 问题。

## 常见卡点

### 1. OpenSSL 版本和系统有关吗？

有关。  
OpenSSL crate 可能依赖本机 OpenSSL 或构建配置。

### 2. 为什么要 Pin？

`tokio_openssl::SslStream::accept` 需要 pinned stream。  
所以源码使用：

````rust
Pin::new(&mut tls_stream)
````

### 3. 和 rustls 版本怎么选？

优先根据项目依赖、部署环境和合规要求选择。  
很多纯 Rust 项目会偏向 rustls，有些环境会要求 OpenSSL。

## 手写任务

1. 创建 `SslAcceptor`。
2. 加载证书和私钥。
3. 检查私钥匹配。
4. accept TCP。
5. 创建 `SslStream` 并 TLS handshake。
6. 交给 Hyper/Axum。

## 本章真正要记住什么

OpenSSL 低层接入链路是：

```text
TCP -> OpenSSL TLS handshake -> TokioIo -> Hyper -> Axum
```

三种低层 TLS 示例的共同点是：TLS 只负责把 TCP stream 变成安全 stream，后面的 HTTP 和 Axum 路由模型不变。

### 三个库怎么选（53/54/55 总结）

这是低层 TLS 三部曲的选型总结表：

| 维度 | rustls（53 章） | native-tls（54 章） | openssl（55 章） |
| --- | --- | --- | --- |
| **依赖** | 纯 Rust，无 C 依赖 | 系统库（macOS Secure Transport / Linux OpenSSL / Windows SChannel） | 系统的 OpenSSL 库 |
| **跨平台行为** | 一致（同一套代码） | 各平台不同（依赖系统 TLS 实现） | 一致（都是 OpenSSL） |
| **ALPN / HTTP/2** | 原生支持 | 支持不一致，跨平台行为难保证 | 支持 |
| **证书来源** | 默认无 root store，需 `rustls-native-certs` 或 `webpki-roots` | 系统信任库 | 系统信任库 |
| **部署体积** | 小（适合容器） | 中 | 中（需带 OpenSSL） |
| **合规场景** | 通用 | 通用 | 金融/合规有时强制 OpenSSL |

**选型建议**：

- **新项目、容器化部署**：优先 **rustls**。纯 Rust 无 C 依赖，编译简单，跨平台一致，体积小。
- **需要和系统 TLS 深度集成**：用 **native-tls**（macOS 会用 Secure Transport，行为最"原生"）。
- **合规要求强制 OpenSSL、或已有 OpenSSL 证书/配置**：用 **openssl**。
- **需要注意**：native-tls 的 ALPN 支持跨平台不一致，如果你依赖 HTTP/2，优先用 rustls。

三个库的 API 各不同，但接入 Axum 的模式完全一样（accept loop → TLS → `TokioIo` → Hyper → Router）。理解了这个模式，换库只是换 TLS 握手那一段代码。

## 源码对照

- `examples/low-level-openssl/src/main.rs`
- `examples/low-level-openssl/self_signed_certs/cert.pem`
- `examples/low-level-openssl/self_signed_certs/key.pem`
- `examples/low-level-openssl/Cargo.toml`
