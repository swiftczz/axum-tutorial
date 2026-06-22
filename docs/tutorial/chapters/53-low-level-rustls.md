# 53. low-level-rustls

对应示例：`examples/low-level-rustls`

本章目标：不用 `axum-server::bind_rustls`，手写 TCP accept loop，并用 `tokio-rustls` 完成 TLS 握手后交给 Hyper/Axum。

第 47 章用的是高层封装：

````rust
axum_server::bind_rustls(addr, config)
````

这一章看低层流程。

## 先理解：为什么需要低层 TLS 接入？

大多数项目用第 47 章的 `axum_server::bind_rustls` 就够了。需要手动写低层 TLS 的场景通常是：

```text
- 需要 SNI（一个端口多张证书，根据域名动态选）
- 需要 mTLS（双向证书认证，客户端也要出示证书）
- 需要在 TLS 握手前/后做 IO 拦截（如自定义协议协商）
- 需要把 TLS 和非 TLS 端口复用
- 需要接入 systemd socket activation
- 想完全控制连接生命周期（自定义超时、限流等）
```

第 53/54/55 章是"低层 TLS 三部曲"，分别用 rustls、native-tls、openssl 三个库实现**同一件事**。本章（rustls）是三部曲的基础，会讲透完整的 accept loop + TLS + Hyper 桥接链路；第 54、55 章只讲各库的差异，其余流程引用本章。

理解本章后，第 49 章（serve-with-hyper）和第 51 章（http-proxy）里出现的 `TokioIo`、`service_fn`、accept loop 也都会串起来——它们用的是同一套底层模式。

## 这个小项目在做什么

请求主线：

```text
TcpListener 接收 TCP 连接
-> TlsAcceptor 执行 rustls TLS handshake
-> TokioIo 包装 TLS stream
-> Hyper service_fn 调用 Axum Router
-> Hyper serve_connection_with_upgrades
```

访问：

````bash
curl -k https://localhost:3000
````

返回：

```text
Hello, World!
```

## 文件和依赖

主要文件：

- `examples/low-level-rustls/src/main.rs`
- `examples/low-level-rustls/self_signed_certs/cert.pem`
- `examples/low-level-rustls/self_signed_certs/key.pem`
- `examples/low-level-rustls/Cargo.toml`

关键依赖：

- `tokio-rustls`：TLS acceptor。
- `hyper-util`：TokioIo、Hyper server builder。
- `tower-service`：调用 Axum Router。

## 第一步：读取证书并创建 ServerConfig

源码：

````rust
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

这里手动读取：

```text
private key
certificate chain
```

并配置 ALPN：

```text
h2
http/1.1
```

ALPN 让客户端和服务端协商使用 HTTP/2 或 HTTP/1.1。

## 第二步：创建 TlsAcceptor

源码：

````rust
let rustls_config = rustls_server_config(...);
let tls_acceptor = TlsAcceptor::from(rustls_config);
````

`TlsAcceptor` 用来把普通 TCP 连接升级成 TLS 连接。

## 第三步：accept TCP 后做 TLS handshake

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

如果 TLS 握手失败，直接结束这个连接。

## 第四步：交给 Hyper 处理 HTTP

源码：

````rust
let stream = TokioIo::new(stream);

let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
    tower_service.clone().call(request)
});

hyper_util::server::conn::auto::Builder::new(TokioExecutor::new())
    .serve_connection_with_upgrades(stream, hyper_service)
    .await
````

TLS 完成后，里面跑的仍然是 HTTP。  
Hyper 负责解析 HTTP，Axum Router 负责路由。

### 为什么要 `TokioIo::new(stream)`？

这一行是低层接入的关键，初学者最容易忽略。`TokioIo` 解决的问题是：**Hyper 和 Tokio 用的是两套不同的 async IO trait**。

```text
Tokio 的 TcpStream / rustls 的 TlsStream  →  实现 tokio::io::AsyncRead + AsyncWrite
Hyper 的 serve_connection                 →  要求实现 hyper::rt::Read + Write（IO trait）
```

两套 trait 长得像但不兼容，不能直接传。`TokioIo` 是 hyper-util 提供的**适配器**，把实现了 tokio IO trait 的对象包装成 Hyper 能用的 IO。

记住这个规律：**只要在低层代码里把 tokio 的 IO 交给 hyper，都要包一层 `TokioIo`**。第 49 章（serve-with-hyper）、第 51 章（http-proxy）也用了它，原因完全相同。

### `service_fn` 在做什么？

```text
Tower Service（Axum Router）：call(&mut self, req) -> Future
Hyper Service：              call(&self, req) -> Future
```

两者的 `call` 方法签名不同（`&mut self` vs `&self`）。`service_fn` 把一个闭包包装成 Hyper 能调用的 service，闭包内部 clone 一份 Router 再用 Tower 的 `call`。这也是为什么第 49、51 章都用同样的模式。

### `auto::Builder` 自动协商 HTTP/1 和 HTTP/2

`auto::Builder` 会根据 TLS ALPN 协商结果（第一步设的 `alpn_protocols`），自动选择走 HTTP/2 还是 HTTP/1.1。这就是为什么第一步必须设置 ALPN——否则 `auto::Builder` 无法知道用哪个协议，浏览器请求 HTTP/2 会失败。

## 函数职责速查

- `main`：初始化日志，创建 rustls config 和 acceptor，手写 accept loop。
- `handler`：返回 Hello。
- `rustls_server_config`：读取 PEM key/cert，构建 rustls `ServerConfig`。

## 带中文注释的手写版

完整 `src/main.rs`：

````rust
//! Run with
//!
//! ```not_rust
//! cargo run -p example-low-level-rustls
//! ```

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
    // 初始化日志，便于看到 TLS 握手和连接处理错误。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 从示例目录读取自签名证书和私钥，构造 rustls ServerConfig。
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

        // 第一步：先接收普通 TCP 连接。
        let (cnx, addr) = tcp_listener.accept().await.unwrap();

        tokio::spawn(async move {
            // 第二步：在 TCP 连接上执行 TLS 握手，成功后得到 TLS stream。
            let Ok(stream) = tls_acceptor.accept(cnx).await else {
                error!("error during tls handshake connection from {}", addr);
                return;
            };

            // 第三步：把 Tokio TLS stream 适配成 Hyper 能使用的 IO。
            let stream = TokioIo::new(stream);

            // 第四步：把 Axum Router 这个 Tower service 桥接成 Hyper service。
            let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
                // Router 总是 ready。Hyper 的 Service 用 &self，Tower 的 call 需要 &mut self，
                // 所以每次请求 clone 一份 Router。
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
    // 读取 PEM 私钥。
    let key = PrivateKeyDer::from_pem_file(key).unwrap();

    // 读取 PEM 证书链。
    let certs = CertificateDer::pem_file_iter(cert)
        .unwrap()
        .map(|cert| cert.unwrap())
        .collect();

    let mut config = ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(certs, key)
        .expect("bad certificate/key");

    // ALPN 用于在 TLS 握手阶段协商 HTTP/2 或 HTTP/1.1。
    config.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1".to_vec()];

    Arc::new(config)
}
````

## 运行和验证

运行：

````bash
cargo run -p example-low-level-rustls
````

请求：

````bash
curl -k https://localhost:3000
````

`-k` 用于接受自签名证书。

## 常见卡点

### 1. 平时要用低层写法吗？

通常不用。  
优先用 `axum-server::bind_rustls`。

### 2. ALPN 为什么重要？

它让 TLS 握手阶段协商 HTTP/2 或 HTTP/1.1。

### 3. 证书能用于生产吗？

不能。示例证书只用于教学。

## 手写任务

1. 读取 key.pem 和 cert.pem。
2. 创建 rustls `ServerConfig`。
3. 创建 `TlsAcceptor`。
4. accept TCP。
5. TLS handshake。
6. 用 Hyper 处理 TLS stream。

## 本章真正要记住什么

低层 rustls 接入链路是：

```text
TCP -> rustls TLS -> TokioIo -> Hyper -> Axum Router
```

## 源码对照

- `examples/low-level-rustls/src/main.rs`
- `examples/low-level-rustls/self_signed_certs/cert.pem`
- `examples/low-level-rustls/self_signed_certs/key.pem`
- `examples/low-level-rustls/Cargo.toml`
