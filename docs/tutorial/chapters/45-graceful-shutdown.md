# 45. graceful-shutdown

对应示例：`examples/graceful-shutdown`

后端服务停止时不应总是"立刻杀掉"。更好的方式:停止接收新请求 → 等正在处理的请求完成 → 退出进程。这就是 graceful shutdown。本章用 `with_graceful_shutdown` 监听 Ctrl+C/SIGTERM,用 `TimeoutLayer` 避免请求永远阻塞关闭。



相比前面章节新引入：**`with_graceful_shutdown`、`shutdown_signal`（Ctrl+C + SIGTERM）、`TimeoutLayer`**。

## Cargo.toml

````toml
[package]
name = "example-graceful-shutdown"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["tracing"] }
tokio = { version = "1.0", features = ["full"] }
tower-http = { version = "0.6", features = ["timeout", "trace"] }
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 完整代码

````rust
use std::time::Duration;

use axum::{http::StatusCode, routing::get, Router};
use tokio::net::TcpListener;
use tokio::signal;
use tokio::time::sleep;
use tower_http::timeout::TimeoutLayer;
use tower_http::trace::TraceLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug,axum=trace", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer().without_time())
        .init();

    let app = Router::new()
        .route("/slow", get(|| sleep(Duration::from_secs(5))))
        .route("/forever", get(std::future::pending::<()>))
        .layer((
            TraceLayer::new_for_http(),
            TimeoutLayer::with_status_code(StatusCode::REQUEST_TIMEOUT, Duration::from_secs(10)),
        ));

    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}

async fn shutdown_signal() {
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
}
````

## 运行

````bash
cd examples
cargo run -p example-graceful-shutdown
````

请求慢接口:

````bash
curl http://127.0.0.1:3000/slow
````

另一终端按 Ctrl+C,观察服务是否等请求完成。

请求永远 pending 的接口:

````bash
curl -i http://127.0.0.1:3000/forever
````

会被 `TimeoutLayer` 约 10 秒后变成 408。

## 解读

### 为什么需要优雅关闭

直接退出可能导致:正在处理的请求被中断、客户端收连接错误、写数据库操作做一半、日志和指标没刷完。优雅关闭让服务有机会收尾。但也不能无限等,所以加超时层。

### 两个测试路由

````rust
.route("/slow", get(|| sleep(Duration::from_secs(5))))           // 5 秒后完成
.route("/forever", get(std::future::pending::<()>))              // 永远 pending
````

帮你观察关闭行为——`/slow` 应在优雅关闭期间完成,`/forever` 永远不完成需靠超时。

### `TimeoutLayer` ——优雅关闭的必要补充

````rust
.layer((
    TraceLayer::new_for_http(),
    TimeoutLayer::with_status_code(StatusCode::REQUEST_TIMEOUT, Duration::from_secs(10)),
))
````

请求超过 10 秒返回 `408 Request Timeout`。**这对 graceful shutdown 很重要**——优雅关闭会等已有请求完成,如果某请求永远不完成(如 `/forever`),服务会一直等。TimeoutLayer 给请求一个上限,保证关闭最终能完成。

### `with_graceful_shutdown`

````rust
axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())
    .await;
````

`shutdown_signal()` 是一个 future,future 完成时服务开始优雅关闭:

```text
server 正常运行
  → shutdown_signal 等 Ctrl+C/SIGTERM
  → 信号来了
  → server 停止接新请求,等待已有请求完成(受 TimeoutLayer 保护)
```

### 监听 Ctrl+C + SIGTERM

````rust
async fn shutdown_signal() {
    let ctrl_c = async { signal::ctrl_c().await... };            // 本地开发常见

    #[cfg(unix)]
    let terminate = async { signal::unix::signal(SignalKind::terminate())....recv().await; };  // 容器/systemd 常见

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();                 // 非 Unix 占位

    tokio::select! {                                              // 任一信号触发
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
````

- **Ctrl+C**:开发环境最常见。
- **SIGTERM**:容器/systemd/部署平台通知进程退出常见,只 Unix 有,非 Unix 用 `pending()` 占位。
- `tokio::select!`:任一信号先发生 `shutdown_signal()` 就结束。

## 常见问题

**优雅关闭立刻退出吗?** 不是,停止接新请求并等已有请求完成。

**为什么还需 TimeoutLayer?** 已有请求永远不完成时优雅关闭会一直等,TimeoutLayer 给上限保证关闭最终完成。

**Ctrl+C 和 SIGTERM 区别?** Ctrl+C 本地开发常见,SIGTERM 部署平台通知进程退出常见。

## 手写任务

1. 写 `/slow` 路由 sleep 5 秒。
2. 写 `shutdown_signal()` 监听 Ctrl+C。
3. 给 `axum::serve` 加 `with_graceful_shutdown`。
4. 监听 Unix SIGTERM。
5. 加 `TimeoutLayer` 处理永远 pending 的请求。

## 小结

- 优雅关闭核心:`shutdown_signal` future 完成 → server 停止接新请求 → 等已有请求结束 → **请求必须有超时保护**(否则永远 pending 的请求会挂住关闭)。
- `with_graceful_shutdown(shutdown_signal())` 是 axum 的标准写法。
- `shutdown_signal` 用 `tokio::select!` 等任一信号:Ctrl+C(本地)或 SIGTERM(Unix 部署,非 Unix 用 `pending()` 占位)。
- `TimeoutLayer` 是优雅关闭的必要补充,防止 `/forever` 这类请求让关闭永远不完成。

## 源码对照

- `examples/graceful-shutdown/Cargo.toml`
- `examples/graceful-shutdown/src/main.rs`
