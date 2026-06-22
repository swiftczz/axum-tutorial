# 55. low-level-openssl

对应示例：`examples/low-level-openssl`

第三个低层 TLS 示例,结构和前两章相同,TLS 库换成 OpenSSL。用 OpenSSL 手写 TLS accept loop,把 `tokio_openssl::SslStream` 交给 Hyper/Axum。

> **本章只讲和 rustls/native-tls 的差异**。完整 accept loop + TLS + Hyper 桥接链路见第 53 章——TokioIo 适配、service_fn 桥接、auto::Builder 等都一样。



相比前面章节新引入：**`SslAcceptor::mozilla_modern_v5`、`SslStream::accept(Pin::new(...))`、OpenSSL**。

## Cargo.toml

````toml
[package]
name = "example-low-level-openssl"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
hyper = { version = "1.0.0", features = ["full"] }
hyper-util = { version = "0.1" }
openssl = "0.10"
tokio = { version = "1", features = ["full"] }
tokio-openssl = "0.6"
tower = { version = "0.5.2", features = ["make"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

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

    tls_builder.check_private_key().unwrap();

    let tls_acceptor = tls_builder.build();

    let bind = "[::1]:3000";
    let tcp_listener = TcpListener::bind(bind).await.unwrap();
    info!("HTTPS server listening on {bind}. To contact curl -k https://localhost:3000");

    let app = Router::new().route("/", get(handler));

    loop {
        let tower_service = app.clone();
        let tls_acceptor = tls_acceptor.clone();

        let (cnx, addr) = tcp_listener.accept().await.unwrap();

        tokio::spawn(async move {
            let ssl = Ssl::new(tls_acceptor.context()).unwrap();
            let mut tls_stream = SslStream::new(ssl, cnx).unwrap();

            if let Err(err) = SslStream::accept(Pin::new(&mut tls_stream)).await {
                error!("error during tls handshake connection from {}: {}", addr, err);
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
curl -k https://localhost:3000
````

本机缺少 OpenSSL 开发库编译会失败——这是本地依赖环境问题,不是 axum handler 问题。

## 解读

### 和 rustls/native-tls 的三个差异

**差异 1:用 `SslAcceptor::mozilla_modern_v5` 模板**

````rust
let mut tls_builder = SslAcceptor::mozilla_modern_v5(SslMethod::tls()).unwrap();
````

`mozilla_modern_v5` 是 OpenSSL 提供的现代 TLS 配置模板(rustls 用 `ServerConfig::builder()` 从零构建,native-tls 用 `NativeTlsAcceptor::builder(id)`)。

**差异 2:分别加载证书和私钥 + `check_private_key`**

````rust
tls_builder.set_certificate_file(cert_path, SslFiletype::PEM).unwrap();
tls_builder.set_private_key_file(key_path, SslFiletype::PEM).unwrap();
tls_builder.check_private_key().unwrap();   // 确认证书和私钥匹配
````

OpenSSL 分别加载证书和私钥文件,然后 `check_private_key` 确认匹配(rustls 用 `with_single_cert(certs, key)` 一起传,native-tls 用 `Identity::from_pkcs8` 合成)。

**差异 3:每连接显式创建 `Ssl` + `SslStream` + `Pin`**

````rust
let ssl = Ssl::new(tls_acceptor.context()).unwrap();
let mut tls_stream = SslStream::new(ssl, cnx).unwrap();
if let Err(err) = SslStream::accept(Pin::new(&mut tls_stream)).await { ... }
````

OpenSSL 需为每个连接显式创建 `Ssl` 和 `SslStream`,然后 `SslStream::accept(Pin::new(&mut tls_stream)).await` 完成 TLS 握手。**`Pin::new` 是因为 `tokio_openssl::SslStream::accept` 需要 pinned stream**(见第 8 章对 `Pin` 的解释——自引用/状态机不能随便移动)。

之后 `TokioIo::new(tls_stream)` + `service_fn` + `auto::Builder` 和前两章完全一样。

## 低层 TLS 三部曲选型总结(53/54/55)

| 维度 | rustls(53) | native-tls(54) | openssl(55) |
| --- | --- | --- | --- |
| **依赖** | 纯 Rust,无 C 依赖 | 系统库(Secure Transport/OpenSSL/SChannel) | 系统的 OpenSSL 库 |
| **跨平台行为** | 一致(同一套代码) | 各平台不同(依赖系统 TLS 实现) | 一致(都是 OpenSSL) |
| **ALPN / HTTP/2** | 原生支持 | 支持不一致,跨平台难保证 | 支持 |
| **证书来源** | 默认无 root store,需 `rustls-native-certs`/`webpki-roots` | 系统信任库 | 系统信任库 |
| **部署体积** | 小(适合容器) | 中 | 中(需带 OpenSSL) |
| **合规场景** | 通用 | 通用 | 金融/合规有时强制 OpenSSL |

**选型建议:**

- **新项目、容器化部署** → **rustls**。纯 Rust 无 C 依赖,编译简单,跨平台一致,体积小。
- **需要和系统 TLS 深度集成** → **native-tls**(macOS 用 Secure Transport,行为最"原生")。
- **合规要求强制 OpenSSL、或已有 OpenSSL 证书/配置** → **openssl**。
- **需 HTTP/2 注意**:native-tls ALPN 支持跨平台不一致,优先 rustls。

三个库 API 各不同,但**接入 axum 的模式完全一样**(accept loop → TLS → `TokioIo` → Hyper → Router)。理解了这个模式,换库只是换 TLS 握手那段代码。

## 常见问题

**OpenSSL 版本和系统有关吗?** 有关,openssl crate 依赖本机 OpenSSL 或构建配置。

**为什么用 `Pin`?** `tokio_openssl::SslStream::accept` 需 pinned stream,所以 `Pin::new(&mut tls_stream)`。

**和 rustls 怎么选?** 按项目依赖、部署环境、合规要求选。纯 Rust 项目偏向 rustls,有些环境要求 OpenSSL。

## 手写任务

1. 创建 `SslAcceptor::mozilla_modern_v5`。
2. 加载证书和私钥。
3. `check_private_key` 检查匹配。
4. accept TCP。
5. 创建 `SslStream` 并 TLS handshake(用 `Pin`)。
6. 交给 Hyper/Axum(同第 53 章)。

## 小结

- OpenSSL 低层接入链路:`TCP → OpenSSL TLS handshake → TokioIo → Hyper → axum`,和前两章结构相同,差异只在 TLS 库 API。
- 三个差异:`mozilla_modern_v5` 模板(对比 rustls/native-tls 的 builder)、分别加载证书和私钥 + `check_private_key`、每连接显式创建 `Ssl` + `SslStream` + `Pin::new`。
- **三种低层 TLS 示例的共同点**:TLS 只负责把 TCP stream 变成安全 stream,后面的 HTTP 和 axum 路由模型不变;接入模式都是 accept loop → TLS → `TokioIo` → Hyper → Router。
- 选型:新项目/容器优先 rustls(纯 Rust、跨平台一致、体积小);需系统 TLS 深度集成用 native-tls;合规强制 OpenSSL 或已有 OpenSSL 配置用 openssl。

## 源码对照

- `examples/low-level-openssl/Cargo.toml`
- `examples/low-level-openssl/src/main.rs`
- `examples/low-level-openssl/self_signed_certs/cert.pem`
- `examples/low-level-openssl/self_signed_certs/key.pem`
