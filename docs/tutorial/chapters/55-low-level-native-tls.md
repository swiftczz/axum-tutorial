# 55. low-level-native-tls

对应示例：`examples/low-level-native-tls`

和第 54 章 low-level-rustls 结构几乎一样,差异在 TLS 实现换成 native-tls(平台 TLS 实现)。用 `tokio-native-tls` 手写 TLS accept loop,把 TLS stream 交给 Hyper/Axum。

> **本章只讲和 rustls 的差异**。完整 accept loop + TLS + Hyper 桥接链路见第 54 章——TokioIo 适配、service_fn 桥接、auto::Builder 协商等都一样,这里不重复。

## Cargo.toml

````toml
[package]
name = "example-low-level-native-tls"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
hyper = { version = "1.0.0", features = ["full"] }
hyper-util = { version = "0.1" }
tokio = { version = "1", features = ["full"] }
tokio-native-tls = "0.3.1"
tower-service = "0.3.2"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

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
        PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("self_signed_certs").join("key.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("self_signed_certs").join("cert.pem"),
    );

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
            let Ok(stream) = tls_acceptor.accept(cnx).await else {
                error!("error during tls handshake connection from {}", addr);
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
        .min_protocol_version(Some(Protocol::Tlsv12))
        .build()
        .unwrap()
}
````

## 运行

````bash
cd examples
cargo run -p example-low-level-native-tls
curl -k https://localhost:3000
````

## 解读

### native-tls vs rustls

```text
rustls     → 纯 Rust TLS 实现
native-tls → 平台 TLS 实现(macOS Secure Transport / Windows SChannel / Linux OpenSSL)
```

不同系统底层不同。选哪个取决于部署环境和依赖要求——rustls 更纯 Rust 易交叉编译,native-tls 依赖系统 TLS 能力。

### 和 rustls 的两个差异

**差异 1:证书读取用 `Identity::from_pkcs8`**

````rust
fn native_tls_acceptor(key_file: PathBuf, cert_file: PathBuf) -> NativeTlsAcceptor {
    let key_pem = std::fs::read_to_string(&key_file).unwrap();
    let cert_pem = std::fs::read_to_string(&cert_file).unwrap();

    let id = Identity::from_pkcs8(cert_pem.as_bytes(), key_pem.as_bytes()).unwrap();
    NativeTlsAcceptor::builder(id)
        .min_protocol_version(Some(Protocol::Tlsv12))   // 限制最低 TLS 版本避免过旧协议
        .build()
        .unwrap()
}
````

native-tls 把证书和私钥合成一个 `Identity`(rustls 分别传 certs 和 key)。`.min_protocol_version(Some(Protocol::Tlsv12))` 设最低 TLS 1.2。

**差异 2:acceptor 包一层 tokio 包装**

````rust
let tls_acceptor = native_tls_acceptor(...);     // native-tls acceptor(同步 API)
let tls_acceptor = TlsAcceptor::from(tls_acceptor);   // 包成 tokio-native-tls 可 await 的
````

native-tls 是同步 API,`tokio_native_tls::TlsAcceptor::from` 包装成可 `.await` 的。之后 accept TCP、TLS handshake、`TokioIo`、`service_fn`、`auto::Builder` 都和第 54 章一样。

### ⚠️ native-tls 没有显式 ALPN

第 54 章 rustls 显式设 `config.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1"]`。native-tls 在不同平台对 ALPN 支持不一致(取决于平台 TLS 实现),所以本章没显式设——这通常意味着 HTTP/2 协商不如 rustls 可靠。如果需要稳定的 HTTP/2 over TLS,优先用 rustls。

## 常见问题

**native-tls 和 rustls 选哪个?** rustls 更纯 Rust 易交叉编译,native-tls 依赖系统 TLS。项目选择取决于部署环境和依赖要求。

**为什么设 TLS 1.2?** 限制最低协议版本避免过旧 TLS。

**ALPN 怎么办?** native-tls 不同平台 ALPN 支持不一致,本章没显式设;需稳定 HTTP/2 over TLS 优先用 rustls。

## 手写任务

1. 读取 cert/key。
2. `Identity::from_pkcs8`。
3. 构造 native-tls acceptor 设最低 TLS 1.2。
4. 包成 tokio-native-tls acceptor。
5. TLS handshake 后交给 Hyper(同第 54 章)。

## 小结

- native-tls 版链路:`TCP → native-tls → TokioIo → Hyper → axum`,和 rustls 版结构完全一样,差异只在 TLS 库。
- 两个差异:`Identity::from_pkcs8`(证书+私钥合成 Identity,rustls 分别传)+ `TlsAcceptor::from`(native-tls 同步 API 包成 tokio 可 await)。
- native-tls 用平台 TLS 实现(macOS Secure Transport/Windows SChannel/Linux OpenSSL),rustls 纯 Rust;选哪个看部署环境和依赖要求。
- native-tls 不同平台 ALPN 支持不一致,需稳定 HTTP/2 over TLS 优先 rustls。

## 源码对照

- `examples/low-level-native-tls/Cargo.toml`
- `examples/low-level-native-tls/src/main.rs`
- `examples/low-level-native-tls/self_signed_certs/cert.pem`
- `examples/low-level-native-tls/self_signed_certs/key.pem`
