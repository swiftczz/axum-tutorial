# 47. tls-graceful-shutdown

对应示例：`examples/tls-graceful-shutdown`

把第 45 章 graceful shutdown 和第 46 章 TLS rustls 合在一起。启动 HTTPS + HTTP 重定向两个服务,理解 `axum_server::Handle` 和 `with_graceful_shutdown` 分别如何关闭两个服务,同一个终止信号同时影响两者。



相比前面章节新引入：**`axum_server::Handle`（不同于 `with_graceful_shutdown`）、共享 `shutdown_signal` future**。

## Cargo.toml

````toml
[package]
name = "example-tls-graceful-shutdown"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
axum-extra = "0.12"
axum-server = { version = "0.8", features = ["tls-rustls"] }
tokio = { version = "1", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 完整代码

````rust
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
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let ports = Ports { http: 7878, https: 3000 };

    let handle = axum_server::Handle::new();
    let shutdown_future = shutdown_signal(handle.clone());

    tokio::spawn(redirect_http_to_https(ports, shutdown_future));

    let config = RustlsConfig::from_pem_file(
        PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("self_signed_certs").join("cert.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("self_signed_certs").join("key.pem"),
    )
    .await
    .unwrap();

    let app = Router::new().route("/", get(handler));

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
        signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install signal handler")
            .recv().await;
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

## 运行

````bash
cd examples
cargo run -p example-tls-graceful-shutdown
````

访问 HTTPS:

````bash
curl -k https://localhost:3000/
````

验证 HTTP 重定向:

````bash
curl -i http://localhost:7878/
````

测试关闭:运行服务 → 发起请求 → 按 Ctrl+C → 观察日志 `Received termination signal shutting down`。

## 解读

### 两个 server,两种关闭方式

本章有两个 server,关闭方式不同:

```text
HTTPS server(axum_server::bind_rustls)→ axum_server::Handle 的 graceful_shutdown
HTTP redirect server(axum::serve)    → with_graceful_shutdown(signal)
```

同一个终止信号要同时影响两个服务。

### `axum_server::Handle` 控制 HTTPS server

````rust
let handle = axum_server::Handle::new();
let shutdown_future = shutdown_signal(handle.clone());   // clone 给 shutdown_signal

axum_server::bind_rustls(addr, config)
    .handle(handle)                                       // 绑定到 HTTPS server
    .serve(app.into_make_service()).await.unwrap();
````

`Handle` 是控制 `axum_server` 的句柄。`.handle(handle)` 把它绑定到 TLS server,这样 `shutdown_signal` 里调用 `handle.graceful_shutdown(...)` 就能通知 HTTPS server 关闭。

### `shutdown_signal` 收到信号后主动调 handle

````rust
async fn shutdown_signal(handle: axum_server::Handle<SocketAddr>) {
    // ...监听 Ctrl+C / SIGTERM(同第 45 章)
    tokio::select! { _ = ctrl_c => {}, _ = terminate => {}, }

    tracing::info!("Received termination signal shutting down");
    handle.graceful_shutdown(Some(Duration::from_secs(10)));   // 主动触发 HTTPS 关闭
}
````

和第 45 章区别:信号到来后不只返回,而是主动调 `handle.graceful_shutdown(Some(Duration::from_secs(10)))`——给 TLS server 最多 10 秒完成优雅关闭(源码注释提到这和 Docker 强制关闭等待时间有关)。

### HTTP redirect server 用 `with_graceful_shutdown`

````rust
async fn redirect_http_to_https<F>(ports: Ports, signal: F)
where F: Future<Output = ()> + Send + 'static,
{
    // ...make_https + redirect(同第 46 章)
    axum::serve(listener, redirect.into_make_service())
        .with_graceful_shutdown(signal)    // 把 shutdown_future 传进来
        .await.unwrap();
}

// main 里:
let shutdown_future = shutdown_signal(handle.clone());   // 同一个 future
tokio::spawn(redirect_http_to_https(ports, shutdown_future));
````

HTTP redirect server 用普通 `axum::serve` 所以继续用第 45 章的 `with_graceful_shutdown(signal)`。`signal` 是泛型 future(`Future<Output = ()> + Send + 'static`),就是前面创建的 `shutdown_signal(handle.clone())`——**HTTP 服务和 HTTPS 服务共享同一个关闭触发源**。

### 共享关闭源的机制

```text
shutdown_signal(handle.clone()) 返回一个 future
  → 这个 future 同时:
     1. 被传给 redirect_http_to_https 作为 with_graceful_shutdown 的 signal
     2. 内部持有 handle clone,信号来时调 handle.graceful_shutdown() 触发 HTTPS 关闭
  → 一个 Ctrl+C / SIGTERM 同时关闭两个服务
```

### 重定向逻辑同第 46 章

`make_https` 把 HTTP URI 改成 HTTPS(scheme→https、authority→localhost:3000、path/query 保留),`Redirect::permanent` 永久重定向。

## 常见问题

**为什么 HTTPS server 不用 `with_graceful_shutdown`?** 这章用 `axum_server::bind_rustls`,它通过 `axum_server::Handle` 控制 graceful shutdown。

**为什么 HTTP redirect server 还能用 `with_graceful_shutdown`?** HTTP redirect server 用普通 `axum::serve`,所以继续用第 45 章方式。

**10 秒是什么?** `handle.graceful_shutdown(Some(Duration::from_secs(10)))` 给 TLS server 最多 10 秒完成优雅关闭,和 Docker 强制关闭等待时间有关。

**证书能用于生产?** 不能,示例证书只用于教学。

## 手写任务

1. 先完成第 46 章的 TLS 服务。
2. 创建 `axum_server::Handle::new()`。
3. 写 `shutdown_signal(handle)`。
4. 信号来后调 `handle.graceful_shutdown(...)`。
5. TLS server 加 `.handle(handle)`。
6. HTTP redirect server 用 `with_graceful_shutdown(signal)`。

## 小结

- 本章是组合题:TLS server 用 `axum_server::Handle::graceful_shutdown`,HTTP redirect server 用 `with_graceful_shutdown`,同一个 `shutdown_signal` future 触发两个服务收尾。
- `axum_server::Handle` 控制 `axum_server` 启动的服务(TLS);`.handle(handle)` 绑定,`handle.graceful_shutdown(Some(duration))` 触发关闭并给上限。
- 普通服务(`axum::serve`)继续用 `with_graceful_shutdown(signal)`;`shutdown_signal` 同时持有 handle clone,信号来时既让自己 future 完成(触发 HTTP redirect 关闭)又调 handle.graceful_shutdown(触发 HTTPS 关闭)。
- 到这里你已经见过 axum 从 Hello World 到认证/数据库/实时通信/测试/TLS/优雅关闭的完整后端学习路径。

## 源码对照

- `examples/tls-graceful-shutdown/Cargo.toml`
- `examples/tls-graceful-shutdown/src/main.rs`
- `examples/tls-graceful-shutdown/self_signed_certs/cert.pem`
- `examples/tls-graceful-shutdown/self_signed_certs/key.pem`
