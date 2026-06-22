# 54. low-level-native-tls

对应示例：`examples/low-level-native-tls`

**前置**：第 53 章（low-level TLS 骨架）

> ch53-55 三章骨架完全相同，只换 TLS backend。选型对比见[第 53 章 "Low-level TLS backend 选型"](./53-low-level-rustls.md#low-level-tls-backend-选型)。本章用 native-tls（系统 OpenSSL/SChannel），适合 FIPS 合规或需要系统证书的场景。

第 53 章用 rustls 做 low-level HTTPS server。这章换 **native-tls** backend——用**操作系统自带的 TLS 实现**（Linux/macOS 用 OpenSSL，Windows 用 SChannel）。代码骨架和 ch53 完全一样，只换 TLS 库。

分 2 步：先建 native-tls `TlsAcceptor`（Identity 方式配置），再写手动 TCP+TLS 循环（同 ch53）。

相比前面章节新引入：**`tokio-native-tls` crate、`native_tls::Identity`（PKCS#12 包含 cert+key）、`Protocol` 设置最小 TLS 版本**。

## Cargo.toml

````toml
[package]
name = "example-low-level-native-tls"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
hyper = "1.0.0"
hyper-util = { version = "0.1", features = ["server", "tokio"] }
native-tls = "0.2"
tokio = { version = "1.0", features = ["full"] }
tokio-native-tls = "0.3"
tower-service = "0.3"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

本章和 ch53 骨架完全相同，只换 TLS backend：用 `native-tls`（系统 OpenSSL/SChannel，Linux/macOS 自动用系统库）+ `tokio-native-tls`（异步包装）。`hyper`/`hyper-util`/`tower-service` 同 ch53，见那章的依赖说明。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：native-tls `TlsAcceptor`

native-tls 用 `Identity` 表示"证书+私钥"组合（PKCS#12 格式或 PEM）。这步从 PEM 文件加载并构造 `TlsAcceptor`。

````rust
use std::path::PathBuf;
use tokio_native_tls::{
    native_tls::{Identity, Protocol, TlsAcceptor as NativeTlsAcceptor},
    TlsAcceptor,
};

fn native_tls_acceptor(key_file: PathBuf, cert_file: PathBuf) -> NativeTlsAcceptor {
    let key_pem = std::fs::read_to_string(&key_file).unwrap();
    let cert_pem = std::fs::read_to_string(&cert_file).unwrap();

    // 从 PEM 创建 Identity（cert + key 打包）
    let id = Identity::from_pkcs8(cert_pem.as_bytes(), key_pem.as_bytes()).unwrap();
    NativeTlsAcceptor::builder(id)
        // 强制 TLS 1.2+（弃用旧版本）
        .min_protocol_version(Some(Protocol::Tlsv12))
        .build()
        .unwrap()
}

// 包成 tokio 版 TlsAcceptor
# let tls_acceptor = native_tls_acceptor(/* ... */);
# let tls_acceptor = TlsAcceptor::from(tls_acceptor);
````

> **新面孔：`native_tls::Identity`**
>
> native-tls 用 `Identity` 表示证书+私钥组合（PKCS#12 是它的本命格式，但也能 `from_pkcs8` 从 PEM 加载）。`Identity::from_pkcs8(cert_pem, key_pem)` 把两个 PEM 合成一个 Identity。
>
> 和 rustls 的"分别加载 cert 和 key"不同——native-tls 因为底层是 OpenSSL/SChannel，更倾向 PKCS#12 打包格式。

> **新面孔：`Protocol::Tlsv12`（最小版本）**
>
> `.min_protocol_version(Some(Protocol::Tlsv12))` 强制 TLS 1.2 以上。TLS 1.0/1.1 有已知漏洞（POODLE、BEAST），现代服务必须禁用。
>
> native-tls 默认版本取决于系统，显式设最小版本更安全。

> **新面孔：`tokio_native_tls::TlsAcceptor`**
>
> native-tls 是同步的，`tokio-native-tls` 把它包装成异步。`TlsAcceptor::from(native_acceptor)` 把 `native_tls::TlsAcceptor` 转成 `tokio_native_tls::TlsAcceptor`，提供 `.accept(stream).await` 异步握手。

---

## 第二步：手动 TCP+TLS 循环（同 ch53）

这步和 ch53 第二、三步完全一样，只是 `TlsAcceptor` 来自 native-tls。骨架：手动 accept TCP → spawn task → TLS 握手 → hyper 服务。

````rust
use axum::{routing::get, Router};
use hyper::body::Incoming;
use hyper_util::rt::{TokioExecutor, TokioIo};
use hyper_util::server;
use tokio::net::TcpListener;
use tower_service::Service;
use tracing::info;

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(/* tracing 初始化 */)
        .with(tracing_subscriber::fmt::layer())
        .init();

    let tls_acceptor = native_tls_acceptor(/* ... */);
    let tls_acceptor = TlsAcceptor::from(tls_acceptor);

    let bind = "[::1]:3000";
    let tcp_listener = TcpListener::bind(bind).await.unwrap();
    info!("HTTPS server listening on {bind}. To contact curl -k https://localhost:3000");
    let app = Router::new().route("/", get(handler));

    loop {
        let tower_service = app.clone();
        let tls_acceptor = tls_acceptor.clone();

        let (cnx, addr) = tcp_listener.accept().await.unwrap();

        tokio::spawn(async move {
            // TLS 握手
            let Ok(stream) = tls_acceptor.accept(cnx).await else {
                tracing::error!("error during tls handshake connection from {}", addr);
                return;
            };

            let stream = TokioIo::new(stream);

            let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
                tower_service.clone().call(request)
            });

            let ret = hyper_util::server::conn::auto::Builder::new(TokioExecutor::new())
                .serve_connection_with_upgrades(stream, hyper_service)
                .await;

            if let Err(err) = ret {
                tracing::warn!("error serving connection from {addr}: {err}");
            }
        });
    }
}

async fn handler() -> &'static str {
    "Hello, World!"
}
````

**详细解释见第 53 章 low-level-rustls**——这步骨架完全一样：`TokioIo` 适配、`service_fn` 包 Router、`auto::Builder::serve_connection_with_upgrades`。

---

## 完整代码

````rust
use axum::{extract::Request, routing::get, Router};
use hyper::body::Incoming;
use hyper_util::rt::{TokioExecutor, TokioIo};
use std::path::PathBuf;
use tokio::net::TcpListener;
use tokio_native_tls::{
    native_tls::{Identity, Protocol, TlsAcceptor as NativeTlsAcceptor},
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
                .unwrap_or_else(|_| "example_low_level_rustls=debug".into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let tls_acceptor = native_tls_acceptor(
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("key.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("cert.pem"),
    );

    let tls_acceptor = TlsAcceptor::from(tls_acceptor);
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
                warn!("error serving connection from {addr}: {err}");
            }
        });
    }
}

async fn handler() -> &'static str {
    "Hello, World!"
}

fn native_tls_acceptor(key_file: PathBuf, cert_file: PathBuf) -> NativeTlsAcceptor {
    let key_pem = std::fs::read_to_string(&key_file).unwrap();
    let cert_pem = std::fs::read_to_string(&cert_file).unwrap();

    let id = Identity::from_pkcs8(cert_pem.as_bytes(), key_pem.as_bytes()).unwrap();
    NativeTlsAcceptor::builder(id)
        // let's be modern
        .min_protocol_version(Some(Protocol::Tlsv12))
        .build()
        .unwrap()
}
````

## 运行

````bash
cd examples
cargo run -p example-low-level-native-tls

curl -k https://localhost:3000/
# Hello, World!
````

## 解读

### ch53 rustls vs ch54 native-tls 对比

| 维度 | rustls（ch53） | native-tls（这章） |
| --- | --- | --- |
| 实现 | Rust 自带 | 用系统 OpenSSL/SChannel |
| 配置 API | `ServerConfig::builder()` | `Identity` + `TlsAcceptor::builder(id)` |
| 证书加载 | 分别加载 cert/key | PKCS#12 / from_pkcs8 打包 |
| 异步包装 | `tokio-rustls` | `tokio-native-tls` |
| 后续 TCP 循环 | 一样 | 一样 |
| 优点 | 无 C 依赖、内存安全 | 兼容性好、系统证书 |
| 缺点 | 算法少（缺一些老算法） | C 依赖、平台差异 |

### 选哪个

- **新项目优先 rustls**——纯 Rust 实现无 C 依赖，编译简单，内存安全
- **必须用系统证书 / FIPS 合规** 选 native-tls
- **必须用 OpenSSL 特定功能**（如特定 cipher）选 ch55 的 openssl 版本

## 常见问题

**native-tls 在 macOS 编译需要什么？** macOS 自带 LibreSSL（OpenSSL 兼容），native-tls 自动用。如果系统没有 OpenSSL，`brew install openssl` + 设 `OPENSSL_DIR`。

**`Identity::from_pkcs8` 和 `from_pkcs12` 区别？** PKCS#8 是 PEM 格式的私钥（现代），PKCS#12 是二进制打包格式（旧式，需要密码）。`from_pkcs8` 接受两个 PEM 文件，`from_pkcs12` 接受一个 .p12 文件 + 密码。

**为什么 `min_protocol_version(Some(Protocol::Tlsv12))`？** TLS 1.0/1.1 已弃用（有 POODLE/BEAST 等漏洞）。强制 TLS 1.2+ 是现代标准。

## 手写任务

1. 改 `min_protocol_version` 为 `Tlsv13`，强制 TLS 1.3（用 curl `--tlsv1.3` 测）。
2. 加客户端证书认证（mTLS）：`.request_cert(true).add_root_certificate(ca)`。
3. 对比同样的代码用 rustls 版（ch53），看差异。

## 小结

这章用 2 步讲了 native-tls 版 low-level HTTPS server：

1. **`Identity` + `TlsAcceptor`**：从 PEM 加载 cert+key 成 Identity，构造 `native_tls::TlsAcceptor`，包成 `tokio_native_tls::TlsAcceptor`。
2. **手动 TCP+TLS 循环**：和 ch53 完全一样，只是 TLS 库换了。

核心：**ch53/54/55 三章骨架完全相同**，只是 TLS backend 不同。native-tls 用系统 OpenSSL，适合需要 FIPS/系统证书的场景；新项目优先 rustls（ch53）。

## 源码对照

- `examples/low-level-native-tls/Cargo.toml`
- `examples/low-level-native-tls/src/main.rs`
- `examples/low-level-native-tls/self_signed_certs/cert.pem`
- `examples/low-level-native-tls/self_signed_certs/key.pem`
