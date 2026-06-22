# 54. low-level-native-tls

对应示例：`examples/low-level-native-tls`

本章目标：使用 `tokio-native-tls` 手写 TLS accept loop，并把 TLS stream 交给 Hyper/Axum。

这一章和 low-level-rustls 结构几乎一样。  
差异在 TLS 实现换成了 native-tls。

## 这个小项目在做什么

链路：

```text
TcpListener
-> native-tls accept
-> TokioIo
-> Hyper service_fn
-> Axum Router
```

访问：

````bash
curl -k https://localhost:3000
````

返回：

```text
Hello, World!
```

## native-tls 是什么

`native-tls` 会使用平台 TLS 实现。  
不同系统底层可能不同。

这和 rustls 不同：

```text
rustls      -> Rust TLS 实现
native-tls  -> 平台 TLS 实现
```

## 关键依赖

- `tokio-native-tls`
- `hyper`
- `hyper-util`
- `tower-service`
- `axum`

## 第一步：读取 PEM 并创建 NativeTlsAcceptor

源码：

````rust
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

它读取 cert 和 key，然后创建 identity。  
同时设置最低 TLS 版本为 TLS 1.2。

## 第二步：转换成 Tokio TlsAcceptor

源码：

````rust
let tls_acceptor = native_tls_acceptor(...);
let tls_acceptor = TlsAcceptor::from(tls_acceptor);
````

第一步是 native-tls acceptor。  
第二步包装成 Tokio 可 await 的 acceptor。

## 第三步：accept TCP 并做 TLS handshake

源码：

````rust
let (cnx, addr) = tcp_listener.accept().await.unwrap();

tokio::spawn(async move {
    let Ok(stream) = tls_acceptor.accept(cnx).await else {
        error!("error during tls handshake connection from {}", addr);
        return;
    };
    ...
});
````

握手成功后，后面就和 rustls 版本一样。

## 函数职责速查

- `main`：创建 native TLS acceptor，accept TCP，TLS 握手，交给 Hyper。
- `native_tls_acceptor`：从 PEM 创建 native-tls acceptor。
- `handler`：返回 Hello。

## 带中文注释的手写版

完整 `src/main.rs`：

````rust
//! Run with
//!
//! ```not_rust
//! cargo run -p example-low-level-native-tls
//! ```

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
    // 初始化日志。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "example_low_level_rustls=debug".into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // native_tls_acceptor 读取 PEM 证书和私钥，创建 native-tls acceptor。
    let tls_acceptor = native_tls_acceptor(
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("key.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("cert.pem"),
    );

    // 包成 tokio-native-tls 的 TlsAcceptor，之后就可以 .await TLS 握手。
    let tls_acceptor = TlsAcceptor::from(tls_acceptor);
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
            // 再通过 native-tls 完成 TLS 握手。
            let Ok(stream) = tls_acceptor.accept(cnx).await else {
                error!("error during tls handshake connection from {}", addr);
                return;
            };

            // TLS stream 适配成 Hyper IO。
            let stream = TokioIo::new(stream);

            // Hyper service 调用 Axum Router。
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
    // native-tls 这里从 PEM 文件读取证书和私钥。
    let key_pem = std::fs::read_to_string(&key_file).unwrap();
    let cert_pem = std::fs::read_to_string(&cert_file).unwrap();

    // Identity 表示服务端 TLS 身份，由证书和私钥组成。
    let id = Identity::from_pkcs8(cert_pem.as_bytes(), key_pem.as_bytes()).unwrap();
    NativeTlsAcceptor::builder(id)
        // 限制最低 TLS 版本，避免使用过旧协议。
        .min_protocol_version(Some(Protocol::Tlsv12))
        .build()
        .unwrap()
}
````

## 运行和验证

````bash
cargo run -p example-low-level-native-tls
curl -k https://localhost:3000
````

## 常见卡点

### 1. native-tls 和 rustls 选哪个？

rustls 更纯 Rust。  
native-tls 更依赖系统 TLS 能力。项目选择取决于部署环境和依赖要求。

### 2. 为什么要设置 TLS 1.2？

示例限制最低协议版本，避免过旧 TLS 版本。

## 手写任务

1. 读取 cert/key。
2. `Identity::from_pkcs8`。
3. 构造 native-tls acceptor。
4. 包成 tokio-native-tls acceptor。
5. TLS handshake 后交给 Hyper。

## 本章真正要记住什么

native-tls 版本的链路是：

```text
TCP -> native-tls -> TokioIo -> Hyper -> Axum
```

## 源码对照

- `examples/low-level-native-tls/src/main.rs`
- `examples/low-level-native-tls/self_signed_certs/cert.pem`
- `examples/low-level-native-tls/self_signed_certs/key.pem`
- `examples/low-level-native-tls/Cargo.toml`
