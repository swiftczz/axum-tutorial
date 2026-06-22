# 47. tls-graceful-shutdown

对应示例：`examples/tls-graceful-shutdown`

**前置**：第 45 章（graceful shutdown）、第 46 章（TLS rustls）

把第 45 章 graceful shutdown 和第 46 章 TLS rustls 合在一起。启动 HTTPS + HTTP 重定向两个服务，理解 `axum_server::Handle` 和 `with_graceful_shutdown` 分别如何关闭两个服务，同一个终止信号同时影响两者。

分 3 步：先启动带 Handle 的 HTTPS server，再加 HTTP→HTTPS 重定向服务，最后用共享 shutdown future 同时关闭两者。

相比前面章节新引入：**`axum_server::Handle`（不同于 `with_graceful_shutdown`）、`handle.graceful_shutdown(Some(duration))`、共享 `shutdown_signal` future 同时关闭两个服务**。

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

---

## 第一步：HTTPS server + `axum_server::Handle`

第 46 章用 `axum_server::bind_rustls` 启动 HTTPS，但没说怎么控制它优雅关闭。这步加一个 `axum_server::Handle`：它是 `axum_server` 的控制句柄，能从外部触发 TLS server 关闭。

````rust
use axum::{routing::get, Router};
use axum_server::tls_rustls::RustlsConfig;
use std::{net::SocketAddr, path::PathBuf, time::Duration};
use tokio::signal;
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

    // 控制句柄：外部能通过它触发 graceful shutdown
    let handle = axum_server::Handle::new();

    // 证书（第 46 章）
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

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {addr}");
    axum_server::bind_rustls(addr, config)
        .handle(handle)                              // 关键：绑定 handle
        .serve(app.into_make_service())
        .await
        .unwrap();
}

async fn handler() -> &'static str {
    "Hello, World!"
}
````

验证：

````bash
cd examples
cargo run -p example-tls-graceful-shutdown
````

````bash
curl -k https://localhost:3000/
````

返回 `Hello, World!`。这步 server 能跑，但还没接 shutdown 信号——按 Ctrl+C 会立即退出。下一步加信号监听。

> **新面孔：`axum_server::Handle`**
>
> `axum_server::Handle` 是控制 `axum_server` 启动的服务的句柄（区别于第 45 章普通 `axum::serve` 的 `with_graceful_shutdown`）。
>
> 用法：
> 1. `Handle::new()` 创建句柄
> 2. `.handle(handle.clone())` 绑定到 server builder
> 3. 在别处调 `handle.graceful_shutdown(Some(duration))` 触发优雅关闭
>
> Handle 可 clone，多线程共享——这正是后面"一个信号关两个服务"的基础。

---

## 第二步：`shutdown_signal` 监听信号 + 主动调 handle

第 45 章的 `shutdown_signal` 只是 future 完成后让 `with_graceful_shutdown` 自动关。这章 HTTPS server 用 Handle，**信号来时要主动调 `handle.graceful_shutdown(...)`**。

````rust
# use axum_server::Handle;
# use std::{net::SocketAddr, time::Duration};
# use tokio::signal;

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
    // 主动触发 HTTPS server graceful shutdown，最多等 10 秒
    handle.graceful_shutdown(Some(Duration::from_secs(10)));
}

# #[tokio::main]
# async fn main() {
#     // ...
#     let handle = axum_server::Handle::new();
#     // shutdown_signal 持有 handle 的 clone
#     let shutdown_future = shutdown_signal(handle.clone());
#     // shutdown_future 这里还没用到，下一步用
#     axum_server::bind_rustls(addr, config)
#         .handle(handle)
#         .serve(app.into_make_service())
#         .await.unwrap();
# }
````

> **新面孔：`handle.graceful_shutdown(Some(duration))`**
>
> 和第 45 章 `with_graceful_shutdown` 的区别：
> - `with_graceful_shutdown(signal)`：signal 完成 → server 自动开始关
> - `handle.graceful_shutdown(Some(duration))`：**主动调用**触发关闭，`duration` 给一个上限（超过强制关）
>
> 这里 `Duration::from_secs(10)` 和 Docker 默认强制关闭等待时间对齐——Docker stop 后等 10 秒就 SIGKILL。

注意：`shutdown_signal(handle.clone())` 返回一个 future，但这个 future 内部**既监听信号又调 handle**——所以同一个 future 完成时，HTTPS server 就被关闭了。

---

## 第三步：加 HTTP→HTTPS 重定向服务，共享同一个 shutdown future

现在加第二个服务（HTTP → HTTPS 永久重定向），用普通 `axum::serve` 启动，关它用第 45 章的 `with_graceful_shutdown`。关键：**把同一个 `shutdown_future` 传给重定向服务**——这样 Ctrl+C 时，一个 future 同时关两个服务。

````rust
use axum::{
    handler::HandlerWithoutStateExt,
    http::{StatusCode, Uri},
    response::Redirect,
    BoxError,
};
use std::future::Future;

#[derive(Clone, Copy)]
struct Ports {
    http: u16,
    https: u16,
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
        .with_graceful_shutdown(signal)   // 共享同一个 shutdown_future
        .await
        .unwrap();
}

# #[tokio::main]
# async fn main() {
#     let ports = Ports { http: 7878, https: 3000 };
#     let handle = axum_server::Handle::new();
#     let shutdown_future = shutdown_signal(handle.clone());   // 同一个 future
#
#     // 重定向服务拿到 shutdown_future，spawn 后台跑
#     tokio::spawn(redirect_http_to_https(ports, shutdown_future));
#
#     // HTTPS 主服务用 handle 控制
#     // axum_server::bind_rustls(addr, config).handle(handle)...
# }
````

> **新面孔：`HandlerWithoutStateExt::into_make_service`**
>
> `redirect` 是一个闭包 handler，但 `axum::serve` 第二个参数需要 `MakeService`（能不断生产 service 的工厂）。`into_make_service()` 把单个 handler 转成这种工厂——等价于 `Router::new().route("/", get(redirect)).into_make_service()`，但更简洁。
>
> 这里没经过 Router 是因为重定向服务只有一个根路径，不需要路由匹配。

> **新面孔：共享 shutdown future 同时关两个服务**
>
> 关键机制（这是本章核心）：
>
> ```text
> shutdown_signal(handle.clone()) 返回的 future F 同时承担两个职责：
>   1. 被 redirect 服务用 .with_graceful_shutdown(F) 等待 → F 完成时 redirect 服务开始关闭
>   2. 内部持有 handle clone，F 内部收到信号时主动调 handle.graceful_shutdown() → HTTPS 服务开始关闭
>
> 所以一个 Ctrl+C / SIGTERM：
>   - F 完成 → redirect 服务关闭（via with_graceful_shutdown）
>   - F 内部调 handle → HTTPS 服务关闭（via Handle）
> ```

验证：

````bash
curl -k https://localhost:3000/     # HTTPS 主服务
curl -i http://localhost:7878/      # HTTP 被重定向到 HTTPS
````

测试关闭：发起请求 → 按 Ctrl+C → 观察日志 `Received termination signal shutting down` → 等待最多 10 秒后两个服务都退出。

---

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

    let ports = Ports {
        http: 7878,
        https: 3000,
    };

    //Create a handle for our TLS server so the shutdown signal can all shutdown
    let handle = axum_server::Handle::new();
    //save the future for easy shutting down of redirect server
    let shutdown_future = shutdown_signal(handle.clone());

    // optional: spawn a second server to redirect http requests to this server
    tokio::spawn(redirect_http_to_https(ports, shutdown_future));

    // configure certificate and private key used by https
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

    // run https server
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
    handle.graceful_shutdown(Some(Duration::from_secs(10))); // 10 secs is how long docker will wait
                                                             // to force shutdown
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

访问 HTTPS：

````bash
curl -k https://localhost:3000/
````

验证 HTTP 重定向：

````bash
curl -i http://localhost:7878/
````

测试关闭：运行服务 → 发起请求 → 按 Ctrl+C → 观察日志 `Received termination signal shutting down`，最多等 10 秒两个服务都退出。

## 解读

### 两个 server，两种关闭方式

```text
HTTPS server (axum_server::bind_rustls) → axum_server::Handle 的 graceful_shutdown
HTTP redirect server (axum::serve)      → with_graceful_shutdown(signal)
```

同一个终止信号要同时影响两个服务——靠 `shutdown_signal` 这个 future 同时承担两个职责。

## 常见问题

**为什么 HTTPS server 不用 `with_graceful_shutdown`？** 这章用 `axum_server::bind_rustls`，它通过 `axum_server::Handle` 控制 graceful shutdown。

**为什么 HTTP redirect server 还能用 `with_graceful_shutdown`？** HTTP redirect server 用普通 `axum::serve`，所以继续用第 45 章方式。

**10 秒是什么？** `handle.graceful_shutdown(Some(Duration::from_secs(10)))` 给 TLS server 最多 10 秒完成优雅关闭，和 Docker 强制关闭等待时间对齐。

**证书能用于生产？** 不能，示例证书只用于教学。

## 手写任务

1. 先完成第 46 章的 TLS 服务。
2. 创建 `axum_server::Handle::new()`。
3. 写 `shutdown_signal(handle)`。
4. 信号来后调 `handle.graceful_shutdown(...)`。
5. TLS server 加 `.handle(handle)`。
6. HTTP redirect server 用 `with_graceful_shutdown(signal)`。

## 小结

这章用 3 步把 TLS + 优雅关闭组合起来：

1. **HTTPS server + Handle**：`axum_server::bind_rustls(...).handle(handle)`，外部可通过 handle 控制关闭。
2. **shutdown_signal**：信号来时**主动调** `handle.graceful_shutdown(Some(10s))`，返回的 future 还能被别处 await。
3. **HTTP redirect 共享 future**：同一个 `shutdown_future` 传给 `with_graceful_shutdown`，一个 Ctrl+C 同时关两个服务。

核心：**一个 future 同时承担两个职责**——对 redirect 服务它是"完成即关闭"的 signal；对 HTTPS 服务它的内部调用了 `handle.graceful_shutdown()`。

到这里你已经见过 axum 从 Hello World 到认证/数据库/实时通信/测试/TLS/优雅关闭的完整后端学习路径。

## 源码对照

- `examples/tls-graceful-shutdown/Cargo.toml`
- `examples/tls-graceful-shutdown/src/main.rs`
- `examples/tls-graceful-shutdown/self_signed_certs/cert.pem`
- `examples/tls-graceful-shutdown/self_signed_certs/key.pem`
