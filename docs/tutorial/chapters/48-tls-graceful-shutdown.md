# 48. tls-graceful-shutdown

对应示例：`examples/tls-graceful-shutdown`

本章目标：把 TLS 服务、HTTP 到 HTTPS 重定向、优雅关闭组合起来，理解 `axum_server::Handle` 和 `with_graceful_shutdown` 分别如何关闭两个服务。

它把第 46 章 graceful shutdown 和第 47 章 TLS rustls 合在一起。后面第 49-55 章会讲更底层的 hyper 接入、Unix domain socket、代理和低层 TLS。

## 这个小项目在做什么

应用启动两个服务：

```text
HTTPS 服务：127.0.0.1:3000
HTTP  跳转：127.0.0.1:7878
```

同时支持优雅关闭：

```text
Ctrl+C 或 SIGTERM
-> shutdown_signal 触发
-> HTTPS 服务通过 axum_server::Handle 优雅关闭
-> HTTP redirect 服务通过 with_graceful_shutdown 优雅关闭
```

## 先理解为什么需要两个关闭方式

本章有两个 server：

```text
axum_server::bind_rustls(...) -> HTTPS server
axum::serve(...)              -> HTTP redirect server
```

它们的关闭方式不同：

```text
HTTPS server 用 axum_server::Handle
HTTP redirect server 用 with_graceful_shutdown(signal)
```

同一个终止信号要同时影响两个服务。

## 文件和依赖

这个 example 有四类文件：

1. `examples/tls-graceful-shutdown/Cargo.toml`：声明 axum-server tls-rustls。
2. `examples/tls-graceful-shutdown/src/main.rs`：实现 TLS、重定向和优雅关闭。
3. `examples/tls-graceful-shutdown/self_signed_certs/cert.pem`：示例证书。
4. `examples/tls-graceful-shutdown/self_signed_certs/key.pem`：示例私钥。

关键依赖：

- `axum-server`：TLS server 和 `Handle`。
- `axum`：Router、Redirect。
- `tokio::signal`：监听 Ctrl+C/SIGTERM。
- `tracing`：输出关闭日志。

## 第一步：创建端口和 Handle

源码：

````rust
let ports = Ports {
    http: 7878,
    https: 3000,
};

let handle = axum_server::Handle::new();
let shutdown_future = shutdown_signal(handle.clone());
````

`Handle` 是控制 `axum_server` 的句柄。  
后面收到关闭信号时，会调用：

````rust
handle.graceful_shutdown(...)
````

因为 `shutdown_signal` 需要持有 handle，所以这里 clone 一份传进去。

## 第二步：把 shutdown future 给 HTTP 重定向服务

源码：

````rust
tokio::spawn(redirect_http_to_https(ports, shutdown_future));
````

HTTP 重定向服务在后台任务里运行。

它收到同一个 `shutdown_future` 后，会这样关闭：

````rust
axum::serve(listener, redirect.into_make_service())
    .with_graceful_shutdown(signal)
    .await;
````

所以当 `shutdown_signal` 完成时，HTTP redirect server 也会优雅关闭。

## 第三步：加载 TLS 证书

源码：

````rust
let config = RustlsConfig::from_pem_file(
    PathBuf::from(env!("CARGO_MANIFEST_DIR"))
        .join("self_signed_certs")
        .join("cert.pem"),
    PathBuf::from(env!("CARGO_MANIFEST_DIR"))
        .join("self_signed_certs")
        .join("key.pem"),
)
.await
.unwrap();
````

和上一章一样，加载证书和私钥。

这些是教学用自签名证书。  
生产环境必须换成正式证书和安全管理的私钥。

## 第四步：HTTPS 服务绑定 Handle

源码：

````rust
axum_server::bind_rustls(addr, config)
    .handle(handle)
    .serve(app.into_make_service())
    .await
    .unwrap();
````

关键是：

````rust
.handle(handle)
````

它把前面创建的 `axum_server::Handle` 绑定到 TLS server。

这样 `shutdown_signal` 里调用：

````rust
handle.graceful_shutdown(Some(Duration::from_secs(10)));
````

就能通知 HTTPS server 优雅关闭。

## 第五步：shutdown_signal

源码：

````rust
async fn shutdown_signal(handle: axum_server::Handle<SocketAddr>) {
    let ctrl_c = async {
        signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install signal handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }

    tracing::info!("Received termination signal shutting down");
    handle.graceful_shutdown(Some(Duration::from_secs(10)));
}
````

这和第 46 章类似，监听：

```text
Ctrl+C
SIGTERM
```

区别是信号到来后，它不只是返回，而是主动调用：

```text
handle.graceful_shutdown(...)
```

`Some(Duration::from_secs(10))` 表示最多给 10 秒做优雅关闭。

## 第六步：HTTP redirect server 也优雅关闭

源码：

````rust
async fn redirect_http_to_https<F>(ports: Ports, signal: F)
where
    F: Future<Output = ()> + Send + 'static,
{
    ...
    axum::serve(listener, redirect.into_make_service())
        .with_graceful_shutdown(signal)
        .await;
}
````

这里把 `signal` 作为泛型 future 传进来。  
它就是前面创建的：

````rust
shutdown_signal(handle.clone())
````

所以 HTTP 服务和 HTTPS 服务共享同一个关闭触发源。

## 第七步：重定向逻辑和上一章相同

源码：

````rust
fn make_https(uri: Uri, https_port: u16) -> Result<Uri, BoxError> {
    let mut parts = uri.into_parts();

    parts.scheme = Some(axum::http::uri::Scheme::HTTPS);
    parts.authority = Some(format!("localhost:{https_port}").parse()?);

    if parts.path_and_query.is_none() {
        parts.path_and_query = Some("/".parse().unwrap());
    }

    Ok(Uri::from_parts(parts)?)
}
````

把 HTTP 请求改写到 HTTPS 地址。

## 函数职责速查

- `main`：初始化日志，创建 `Handle` 和 shutdown future，启动 HTTP redirect server，加载 TLS，启动 HTTPS server。
- `shutdown_signal`：监听 Ctrl+C/SIGTERM，并通过 handle 关闭 TLS server。
- `handler`：返回 `Hello, World!`。
- `redirect_http_to_https`：启动 HTTP redirect server，并使用 `with_graceful_shutdown`。
- `make_https`：把 URI 改成 HTTPS。


## 带中文注释的手写版

完整 `src/main.rs`：

````rust
//! TLS + 优雅关闭示例。
//!
//! Run with
//!
//! ```not_rust
//! cargo run -p example-tls-graceful-shutdown
//! ```

use axum::{
    handler::HandlerWithoutStateExt,
    http::{StatusCode, Uri},
    response::Redirect,
    routing::get,
    BoxError, Router,
};
use axum_server::tls_rustls::RustlsConfig;
use std::{future::Future, net::SocketAddr, path::PathBuf, time::Duration};
use tokio::signal;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[derive(Clone, Copy)]
struct Ports {
    http: u16,
    https: u16,
}

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

    let ports = Ports {
        http: 7878,
        https: 3000,
    };

    // HTTPS 服务使用 axum_server::Handle 做优雅关闭。
    let handle = axum_server::Handle::new();
    let shutdown_future = shutdown_signal(handle.clone());

    // HTTP 重定向服务使用同一个 shutdown future。
    tokio::spawn(redirect_http_to_https(ports, shutdown_future));

    // 加载 TLS 证书和私钥。
    let config = RustlsConfig::from_pem_file(
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("cert.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("key.pem"),
    )
    .await
    .unwrap();

    let app = Router::new().route("/", get(handler));

    // HTTPS 服务绑定 handle，这样 shutdown_signal 能通知它关闭。
    let addr = SocketAddr::from(([127, 0, 0, 1], ports.https));
    tracing::debug!("listening on {addr}");
    axum_server::bind_rustls(addr, config)
        .handle(handle)
        .serve(app.into_make_service())
        .await
        .unwrap();
}

async fn shutdown_signal(handle: axum_server::Handle<SocketAddr>) {
    let ctrl_c = async {
        signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install signal handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }

    tracing::info!("Received termination signal shutting down");
    // 给 TLS server 最多 10 秒完成优雅关闭。
    handle.graceful_shutdown(Some(Duration::from_secs(10)));
}

async fn handler() -> &'static str {
    "Hello, World!"
}

async fn redirect_http_to_https<F>(ports: Ports, signal: F)
where
    F: Future<Output = ()> + Send + 'static,
{
    fn make_https(uri: Uri, https_port: u16) -> Result<Uri, BoxError> {
        let mut parts = uri.into_parts();

        parts.scheme = Some(axum::http::uri::Scheme::HTTPS);
        parts.authority = Some(format!("localhost:{https_port}").parse()?);

        if parts.path_and_query.is_none() {
            parts.path_and_query = Some("/".parse().unwrap());
        }

        Ok(Uri::from_parts(parts)?)
    }

    let redirect = move |uri: Uri| async move {
        match make_https(uri, ports.https) {
            Ok(uri) => Ok(Redirect::permanent(uri.to_string())),
            Err(error) => {
                tracing::warn!(%error, "failed to convert URI to HTTPS");
                Err(StatusCode::BAD_REQUEST)
            }
        }
    };

    let addr = SocketAddr::from(([127, 0, 0, 1], ports.http));
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    tracing::debug!("listening on {addr}");
    axum::serve(listener, redirect.into_make_service())
        .with_graceful_shutdown(signal)
        .await
        .unwrap();
}
````

## 运行和验证

运行：

````bash
cargo run -p example-tls-graceful-shutdown
````

访问 HTTPS：

````bash
curl -k https://localhost:3000/
````

验证 HTTP 重定向：

````bash
curl -i http://localhost:7878/
````

测试关闭：

```text
运行服务
发起请求
按 Ctrl+C
观察日志中 Received termination signal shutting down
```

## 常见卡点

### 1. 为什么 HTTPS server 不用 with_graceful_shutdown？

这个示例用的是 `axum_server::bind_rustls`。  
它通过 `axum_server::Handle` 控制 graceful shutdown。

### 2. 为什么 HTTP redirect server 还能用 with_graceful_shutdown？

HTTP redirect server 用的是普通 `axum::serve`。  
所以继续用第 46 章的方式。

### 3. 10 秒是什么意思？

````rust
handle.graceful_shutdown(Some(Duration::from_secs(10)))
````

表示给 TLS server 最多 10 秒完成优雅关闭。  
源码注释里也提到这和 Docker 强制关闭等待时间有关。

### 4. 证书能直接用于生产吗？

不能。  
示例证书只用于教学。

## 手写任务

1. 先完成第 47 章的 TLS 服务。
2. 创建 `axum_server::Handle::new()`。
3. 写 `shutdown_signal(handle)`。
4. 在信号到来后调用 `handle.graceful_shutdown(...)`。
5. TLS server 上加 `.handle(handle)`。
6. HTTP redirect server 使用 `with_graceful_shutdown(signal)`。

## 本章真正要记住什么

本章是组合题：

```text
TLS server -> axum_server::Handle graceful_shutdown
HTTP redirect server -> with_graceful_shutdown
同一个 shutdown signal 触发两个服务收尾
```

到这里，你已经见过 Axum 从最小 Hello World 到认证、数据库、实时通信、测试、TLS 和优雅关闭的完整后端学习路径。

## 源码对照

本章手写版对应源码：

- `examples/tls-graceful-shutdown/src/main.rs`
- `examples/tls-graceful-shutdown/self_signed_certs/cert.pem`
- `examples/tls-graceful-shutdown/self_signed_certs/key.pem`
- `examples/tls-graceful-shutdown/Cargo.toml`
