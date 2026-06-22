# 53. low-level-rustls

对应示例：`examples/low-level-rustls`

不用 `axum-server::bind_rustls`(第 46 章高层封装),手写 TCP accept loop,用 `tokio-rustls` 完成 TLS 握手后交给 Hyper/Axum。这是"低层 TLS 三部曲"(53 rustls / 54 native-tls / 55 openssl)的基础,讲透完整 accept loop + TLS + Hyper 桥接链路。



相比前面章节新引入：**低层 rustls accept loop、`ServerConfig` + ALPN、`TokioIo` + `service_fn` + `auto::Builder`**。

## Cargo.toml

````toml
[package]
name = "example-low-level-rustls"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
hyper = { version = "1.0.0", features = ["full"] }
hyper-util = { version = "0.1", features = ["http2"] }
tokio = { version = "1", features = ["full"] }
tokio-rustls = "0.26"
tower-service = "0.3.2"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 关键概念

> **新面孔：低层 rustls accept loop**
>
> `ServerConfig` + ALPN（`alpn_protocols`）→ `TlsAcceptor::accept` 握手 → `TokioIo` + `service_fn` + `auto::Builder`。`auto::Builder` 根据 ALPN 协商结果选 h2/http1.1。


## 完整代码

````rust
use axum::{extract::Request, routing::get, Router};
use hyper::body::Incoming;
use hyper_util::rt::{TokioExecutor, TokioIo};
use std::{path::{Path, PathBuf}, sync::Arc};
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
curl -k https://localhost:3000    # -k 接受自签名证书
````

## 解读

### 为什么需要低层 TLS 接入

大多数项目用第 46 章 `axum_server::bind_rustls` 就够。需要手动写低层 TLS 的场景:

- SNI(一个端口多张证书,根据域名动态选)
- mTLS(双向证书认证,客户端也要出示证书)
- TLS 握手前/后做 IO 拦截(自定义协议协商)
- TLS 和非 TLS 端口复用
- systemd socket activation
- 完全控制连接生命周期(自定义超时/限流)

53/54/55 是"低层 TLS 三部曲",分别用 rustls/native-tls/openssl 实现**同一件事**。本章(rustls)是基础,讲透 accept loop + TLS + Hyper 桥接链路;54/55 章只讲各库差异。理解后第 49/51 章的 `TokioIo`/`service_fn`/accept loop 也会串起来——同一套底层模式。

### 接入链路

```text
TcpListener 接收 TCP 连接
  → TlsAcceptor 执行 rustls TLS handshake
  → TokioIo 包装 TLS stream
  → Hyper service_fn 调用 axum Router
  → Hyper serve_connection_with_upgrades
```

### 读取证书 + 创建 ServerConfig

````rust
fn rustls_server_config(key: impl AsRef<Path>, cert: impl AsRef<Path>) -> Arc<ServerConfig> {
    let key = PrivateKeyDer::from_pem_file(key).unwrap();           // PEM 私钥
    let certs: Vec<_> = CertificateDer::pem_file_iter(cert)...;     // PEM 证书链

    let mut config = ServerConfig::builder()
        .with_no_client_auth()                                       // 不要求客户端证书
        .with_single_cert(certs, key)
        .expect("bad certificate/key");

    config.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1".to_vec()];   // ALPN 协商 h2/http1.1
    Arc::new(config)
}
````

**ALPN 必须设**——`auto::Builder` 根据 ALPN 协商结果选 HTTP/2 还是 HTTP/1.1。不设的话浏览器请求 HTTP/2 会失败(见第 46 章 ALPN 解释)。

### accept TCP + TLS handshake

````rust
let (cnx, addr) = tcp_listener.accept().await.unwrap();    // 普通 TCP

tokio::spawn(async move {
    let Ok(stream) = tls_acceptor.accept(cnx).await else {  // TLS 握手
        error!("error during tls handshake connection from {}", addr);
        return;
    };
    ...
});
````

`TlsAcceptor::accept` 把普通 TCP 连接升级成 TLS 连接。握手失败直接结束这个连接。

### 交给 Hyper(三个关键点)

````rust
let stream = TokioIo::new(stream);   // 1. TokioIo 适配 IO trait

let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
    tower_service.clone().call(request)    // 2. service_fn 桥接 Tower/Hyper service
});

hyper_util::server::conn::auto::Builder::new(TokioExecutor::new())   // 3. auto Builder
    .serve_connection_with_upgrades(stream, hyper_service).await;
````

**为什么 `TokioIo::new(stream)`?** Hyper 和 Tokio 用两套不同 async IO trait:

```text
Tokio 的 TcpStream / rustls 的 TlsStream → 实现 tokio::io::AsyncRead + AsyncWrite
Hyper 的 serve_connection                → 要求实现 hyper::rt::Read + Write(IO trait)
```

两套 trait 长得像但不兼容,不能直接传。`TokioIo` 是 hyper-util 提供的**适配器**,把 tokio IO 包装成 Hyper 能用的 IO。**规律:只要低层代码把 tokio IO 交给 hyper,都要包一层 `TokioIo`**(第 49/51 章也一样)。

**`service_fn` 在做什么?** Tower Service 和 Hyper Service 的 `call` 签名不同(`&mut self` vs `&self`):

```text
Tower Service(Axum Router):call(&mut self, req) -> Future
Hyper Service:              call(&self, req) -> Future
```

`service_fn` 把闭包包装成 Hyper 能调用的 service,闭包内部 clone Router 再用 Tower 的 `call`。第 49/51 章用同样模式。

**`auto::Builder`**:根据 TLS ALPN 协商结果(第一步设的 `alpn_protocols`)自动选 HTTP/2 还是 HTTP/1.1。这就是为什么第一步必须设 ALPN。

## 常见问题

**平时用低层写法吗?** 通常不用,优先 `axum-server::bind_rustls`。

**ALPN 为什么重要?** 让 TLS 握手阶段协商 HTTP/2 或 HTTP/1.1,`auto::Builder` 据此选协议。

**证书能用于生产?** 不能,示例证书只用于教学。

## 手写任务

1. 读取 key.pem 和 cert.pem。
2. 创建 rustls `ServerConfig`(设 ALPN)。
3. 创建 `TlsAcceptor`。
4. accept TCP。
5. TLS handshake。
6. `TokioIo` + `service_fn` + `auto::Builder` 用 Hyper 处理 TLS stream。

## 小结

- 低层 rustls 接入链路:`TCP → rustls TLS → TokioIo → Hyper → axum Router`。
- 大多数项目用 `axum-server::bind_rustls` 够;需 SNI/mTLS/IO 拦截/端口复用/socket activation/完全控制连接生命周期才用低层。
- 三个关键点:`TokioIo::new` 适配 tokio/hyper 两套 IO trait;`service_fn` 桥接 Tower/Hyper Service 不同 call 签名;`auto::Builder` 根据 ALPN 协商结果选 h2/http1.1。
- **必须设 `alpn_protocols`**(h2+http/1.1),否则 `auto::Builder` 不知用哪个协议,浏览器 HTTP/2 失败。
- 这是第 54/55 章(native-tls/openssl)的基础,也是第 49/51 章底层模式的串联。

## 源码对照

- `examples/low-level-rustls/Cargo.toml`
- `examples/low-level-rustls/src/main.rs`
- `examples/low-level-rustls/self_signed_certs/cert.pem`
- `examples/low-level-rustls/self_signed_certs/key.pem`
