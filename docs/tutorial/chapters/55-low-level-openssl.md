# 55. low-level-openssl

对应示例：`examples/low-level-openssl`

**前置**：第 53 章（low-level TLS 骨架）

> ch53-55 三章骨架完全相同，只换 TLS backend。选型对比见[第 53 章 "Low-level TLS backend 选型"](./53-low-level-rustls.md#low-level-tls-backend-选型)。本章用 openssl（直接绑 OpenSSL C 库），适合需要 OpenSSL 特定功能（如 OCSP stapling、特定 cipher 控制）的场景。

第 53 章 rustls、第 54 章 native-tls 之后，这章用 **OpenSSL 直接版本**——`openssl` crate 直接绑定 OpenSSL C 库，比 native-tls 暴露更多底层 API。代码骨架还是和 ch53 一样，只是 TLS 库换。

分 2 步：先建 `SslAcceptor`（mozilla_modern_v5 预设），再写手动 TCP+TLS 循环。

相比前面章节新引入：**`openssl` crate（直接 OpenSSL 绑定）、`SslAcceptor::mozilla_modern_v5`（预设安全配置）、`SslStream::accept`（手动握手）**。

## Cargo.toml

````toml
[package]
name = "example-low-level-openssl"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
hyper = "1.0.0"
hyper-util = { version = "0.1", features = ["server", "tokio"] }
openssl = "0.10"
tokio = { version = "1.0", features = ["full"] }
tokio-openssl = "0.6.1"
tower = "0.5"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

本章和 ch53/54 骨架相同，只换 TLS backend：用 `openssl`（直接绑 OpenSSL C 库，API 最全）+ `tokio-openssl`（异步包装，注意握手要 `Pin`）。`hyper`/`hyper-util`/`tower` 同 ch53，见那章的依赖说明。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：`SslAcceptor` 用 mozilla_modern 预设

OpenSSL 的 `SslAcceptor` 提供 `mozilla_modern_v5` / `mozilla_intermediate_v5` 等预设——直接套用 Mozilla 推荐的安全配置（cipher suite、协议版本、curve 等）。

````rust
use openssl::ssl::{SslAcceptor, SslFiletype, SslMethod};
use std::path::PathBuf;

# #[tokio::main]
# async fn main() {
    // mozilla_modern_v5：Mozilla 推荐的现代配置（TLS 1.2+1.3，强 cipher）
    let mut tls_builder = SslAcceptor::mozilla_modern_v5(SslMethod::tls()).unwrap();

    tls_builder
        .set_certificate_file(
            PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("self_signed_certs").join("cert.pem"),
            SslFiletype::PEM,
        )
        .unwrap();

    tls_builder
        .set_private_key_file(
            PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("self_signed_certs").join("key.pem"),
            SslFiletype::PEM,
        )
        .unwrap();

    // 验证私钥和证书匹配
    tls_builder.check_private_key().unwrap();

    let tls_acceptor = tls_builder.build();
#     // ...
# }
````

> **新面孔：`SslAcceptor::mozilla_modern_v5`**
>
> OpenSSL 推荐用 Mozilla 预设而不是手配置 cipher——Mozilla 持续更新推荐配置，`mozilla_modern_v5` 是当前最严格的（TLS 1.2+1.3，禁用弱算法）。
>
> 三档预设：
> - `mozilla_modern_v5`：最严（TLS 1.2+1.3），现代浏览器
> - `mozilla_intermediate_v5`：中（TLS 1.0+1.2+1.3），兼容老客户端
> - 手动：自己配 cipher，**不推荐**（容易出错）

> **新面孔：`set_certificate_file` / `set_private_key_file`**
>
> OpenSSL 分别加载 cert 和 key（不像 native-tls 打包成 Identity）。`SslFiletype::PEM` 指定 PEM 格式（也支持 DER）。
>
> `check_private_key()` 验证私钥和证书匹配（防止配错）。

> **新面孔：`SslMethod::tls()`**
>
> 表示支持 TLS（不是固定 1.2 或 1.3，由 cipher suite 决定）。OpenSSL 1.1+ 都支持。

---

## 第二步：手动 TCP+TLS 循环（含手动 `SslStream::accept`）

这步和 ch53/54 骨架相同，但 OpenSSL 的握手比 rustls/native-tls 多一层手动——要先 `Ssl::new` + `SslStream::new` 创建 stream，再 `SslStream::accept` 握手。

````rust
use axum::{http::Request, routing::get, Router};
use hyper::body::Incoming;
use hyper_util::rt::{TokioExecutor, TokioIo};
use openssl::ssl::Ssl;
use std::pin::Pin;
use tokio::net::TcpListener;
use tokio_openssl::SslStream;
use tower::Service;
use tracing::info;

# #[tokio::main]
# async fn main() {
#     // ... tls_acceptor 已 build ...
    let bind = "[::1]:3000";
    let tcp_listener = TcpListener::bind(bind).await.unwrap();
    info!("HTTPS server listening on {bind}. To contact curl -k https://localhost:3000");
    let app = Router::new().route("/", get(handler));

    loop {
        let tower_service = app.clone();
        let tls_acceptor = tls_acceptor.clone();

        let (cnx, addr) = tcp_listener.accept().await.unwrap();

        tokio::spawn(async move {
            // OpenSSL 多一步：先 Ssl::new 创建 context，再 SslStream::new 包 TCP
            let ssl = Ssl::new(tls_acceptor.context()).unwrap();
            let mut tls_stream = SslStream::new(ssl, cnx).unwrap();

            // 手动握手
            if let Err(err) = SslStream::accept(Pin::new(&mut tls_stream)).await {
                tracing::error!("error during tls handshake connection from {}: {}", addr, err);
                return;
            }

            let stream = TokioIo::new(tls_stream);

            let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
                tower_service.clone().call(request)
            });

            let ret = hyper_util::server::conn::auto::Builder::new(TokioExecutor::new())
                .serve_connection_with_upgrades(stream, hyper_service)
                .await;

            if let Err(err) = ret {
                tracing::warn!("error serving connection from {}: {}", addr, err);
            }
        });
    }
# }

async fn handler() -> &'static str {
    "Hello, World!"
}
````

> **新面孔：`Ssl::new` + `SslStream::new` + `SslStream::accept`**
>
> OpenSSL 三步握手（比 rustls/native-tls 多几步）：
>
> 1. `Ssl::new(tls_acceptor.context())`：从 acceptor 的 context 创建单连接的 `Ssl` 对象
> 2. `SslStream::new(ssl, tcp_stream)`：把 Ssl 和 TCP 流绑成 `SslStream`
> 3. `SslStream::accept(Pin::new(&mut tls_stream)).await`：异步执行 TLS 握手
>
> `Pin::new(&mut tls_stream)` 是因为 `accept` 要求 `Pin<&mut Self>`（OpenSSL async API 要求 self 被 pin 住保证内存地址稳定）。

> **新面孔：`Pin`**
>
> Rust 的指针封装，保证指向的对象不会被 move（地址稳定）。OpenSSL 的 async API 用自引用结构，必须 pin。`tokio_openssl::SslStream::accept(Pin<&mut Self>)` 强制 pin。
>
> 普通业务代码很少直接用 Pin，OpenSSL 这种 C 绑定库才需要。

---

## 完整代码

````rust
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
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let mut tls_builder = SslAcceptor::mozilla_modern_v5(SslMethod::tls()).unwrap();

    tls_builder
        .set_certificate_file(
            PathBuf::from(env!("CARGO_MANIFEST_DIR"))
                .join("self_signed_certs")
                .join("cert.pem"),
            SslFiletype::PEM,
        )
        .unwrap();

    tls_builder
        .set_private_key_file(
            PathBuf::from(env!("CARGO_MANIFEST_DIR"))
                .join("self_signed_certs")
                .join("key.pem"),
            SslFiletype::PEM,
        )
        .unwrap();

    tls_builder.check_private_key().unwrap();

    let tls_acceptor = tls_builder.build();

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
            let ssl = Ssl::new(tls_acceptor.context()).unwrap();
            let mut tls_stream = SslStream::new(ssl, cnx).unwrap();
            if let Err(err) = SslStream::accept(Pin::new(&mut tls_stream)).await {
                error!(
                    "error during tls handshake connection from {}: {}",
                    addr, err
                );
                return;
            }

            // Hyper has its own `AsyncRead` and `AsyncWrite` traits and doesn't use tokio.
            // `TokioIo` converts between them.
            let stream = TokioIo::new(tls_stream);

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
````

## 运行

````bash
cd examples
cargo run -p example-low-level-openssl

curl -k https://localhost:3000/
# Hello, World!
````

## 解读

### ch53/54/55 三章对比

| 维度 | rustls（ch53） | native-tls（ch54） | openssl（这章） |
| --- | --- | --- | --- |
| 实现 | 纯 Rust | 系统 OpenSSL/SChannel | 直接 OpenSSL C |
| 握手 API | `TlsAcceptor::accept(stream)` | `TlsAcceptor::accept(stream)` | `Ssl::new` + `SslStream::accept(Pin)` |
| 配置 | `ServerConfig::builder()` | `Identity` + builder | `SslAcceptor::mozilla_modern_v5` |
| Pin 要求 | 不需要 | 不需要 | **需要** |
| 编译依赖 | 无 | 系统 OpenSSL | OpenSSL dev lib |
| 适合场景 | 默认选择 | FIPS/系统证书 | 需要特定 OpenSSL 功能 |

### 为什么有这章

OpenSSL 直接绑定比 native-tls 暴露更多底层 API（如手动 ALPN、特定 cipher 控制、自定义 verify callback）。绝大多数场景用 rustls（ch53）或 native-tls（ch54）就够了，这章为特殊需求保留。

### 选哪个

- **新项目优先 rustls**（ch53）—— 纯 Rust 无 C 依赖
- **需要系统证书/FIPS** 选 native-tls（ch54）
- **需要 OpenSSL 特定功能**（如 OCSP stapling、特定 cipher 控制）选 openssl（这章）

## 常见问题

**为什么 OpenSSL 握手要 `Pin::new`？** OpenSSL 用自引用结构（C 风格），async API 要求对象地址稳定，必须 pin。rustls/native-tls 在内部处理了 pin，外部 API 不暴露。

**编译报错 "could not find openssl"** 装 OpenSSL dev：Ubuntu `apt install libssl-dev`，macOS `brew install openssl@3 && export OPENSSL_DIR=...`。

**`mozilla_modern_v5` 真的安全吗？** Mozilla 官方维护的安全预设，持续更新。比手配 cipher 安全（人手配容易遗漏）。

## 手写任务

1. 对比 ch53 rustls 版本，把 `SslStream::accept(Pin::new(...))` 换成 rustls 的 `tls_acceptor.accept(stream)`，看简化了多少。
2. 加 ALPN：`tls_builder.set_alpn_protos(b"\x02h2\x08http/1.1")`。
3. 改用 `mozilla_intermediate_v5` 预设，对比 cipher suite 差异。

## 小结

这章用 2 步讲了 OpenSSL 版 low-level HTTPS server：

1. **`SslAcceptor::mozilla_modern_v5`**：Mozilla 安全预设 + `set_certificate_file` + `set_private_key_file` + `check_private_key`。
2. **手动握手（含 Pin）**：`Ssl::new` + `SslStream::new` + `SslStream::accept(Pin::new(...))`，比 rustls/native-tls 多几步。

核心：ch53/54/55 三章骨架完全相同，TLS backend 不同。OpenSSL 版 API 最繁琐但暴露最多底层控制；新项目优先 rustls。

## 源码对照

- `examples/low-level-openssl/Cargo.toml`
- `examples/low-level-openssl/src/main.rs`
- `examples/low-level-openssl/self_signed_certs/cert.pem`
- `examples/low-level-openssl/self_signed_certs/key.pem`
