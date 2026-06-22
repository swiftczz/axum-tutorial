# 45. graceful-shutdown

对应示例：`examples/graceful-shutdown`

生产环境部署/重启时，直接 kill 进程会**中断正在处理的请求**。graceful shutdown 让进程收到信号后：**停止接新请求 → 等已接收请求完成 → 然后退出**。配合超时机制避免某个慢请求永远卡住。

分 3 步：先启动普通 server（看直接 kill 的问题），再加 `with_graceful_shutdown` + 信号监听，最后加 `TimeoutLayer` 防止慢请求无限阻塞。

相比前面章节新引入：**`axum::serve(...).with_graceful_shutdown(future)`、`tokio::signal::ctrl_c`、`signal::unix::SignalKind::terminate`（SIGTERM）、`tower_http::timeout::TimeoutLayer`**。

## Cargo.toml

````toml
[package]
name = "example-graceful-shutdown"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }
tower-http = { version = "0.6", features = ["trace", "timeout"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

本章相比前面章节新增 `tower-http` 的 `timeout` feature（提供 `TimeoutLayer`，给每个请求设上限，防止慢请求卡死 graceful shutdown）。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：先看没有 graceful shutdown 会怎样

普通 `axum::serve(listener, app).await` 直接 kill 会立即中断处理中的请求。

````rust
use axum::{routing::get, Router};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    let app = Router::new().route("/slow", get(|| async {
        tokio::time::sleep(std::time::Duration::from_secs(5)).await;
        "done"
    }));

    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
````

测试：

````bash
cargo run &
curl http://localhost:3000/slow &   # 5 秒后返回 "done"
kill %1                              # 立即 kill
# curl 那边连接断开，请求被中断
````

生产环境这是大问题——比如正在写数据库的请求被中断导致数据不一致。下一步加 graceful shutdown。

---

## 第二步：`with_graceful_shutdown` + 信号监听

加 `shutdown_signal` 函数监听 Ctrl+C / SIGTERM，传给 `with_graceful_shutdown`。收到信号后 axum 停止接新连接，等已接收请求完成。

````rust
use axum::{routing::get, Router};
use tokio::net::TcpListener;
use tokio::signal;
use tower_http::trace::TraceLayer;

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(/* tracing 初始化 */)
        .with(tracing_subscriber::fmt::layer().without_time())
        .init();

    let app = Router::new()
        .route("/slow", get(|| async {
            tokio::time::sleep(std::time::Duration::from_secs(5)).await;
            "done"
        }))
        .layer(TraceLayer::new_for_http());

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

验证：

````bash
cd examples
cargo run -p example-graceful-shutdown &
curl http://localhost:3000/slow &   # 5 秒后返回
kill -TERM %1                       # 发 SIGTERM
# curl 仍然能拿到响应（server 等它完成）
# 然后 server 退出
````

> **新面孔：`with_graceful_shutdown(future)`**
>
> `axum::serve(listener, app)` 返回 `Serve` future，`.with_graceful_shutdown(signal_future)` 给它绑一个信号 future。signal_future 完成时 axum 开始 graceful shutdown：
>
> 1. 停止接新 TCP 连接
> 2. 等所有正在处理的请求完成
> 3. 全部完成后退出
>
> 所以 signal_future 必须**返回时触发 shutdown**——这章 `shutdown_signal` 用 `tokio::select!` 等 Ctrl+C 或 SIGTERM，等到后函数返回，axum 收到 future 完成就开始 shutdown。

> **新面孔：`tokio::signal::ctrl_c`**
>
> 监听 SIGINT（Ctrl+C）。`signal::ctrl_c().await` 完成 Ctrl+C 按下时返回。生产部署也要监听这个，开发者按 Ctrl+C 停服务。

> **新面孔：`signal::unix::SignalKind::terminate`（SIGTERM）**
>
> 监听 SIGTERM——容器编排（k8s/docker）默认发这个信号停止容器。**生产必须监听 SIGTERM**，否则 k8s stop 后等 grace period 才发 SIGKILL 强杀。
>
> `#[cfg(unix)]` 包起来因为 SIGTERM 是 Unix 独有，Windows 用 `ctrl_c` 即可。

> **新面孔：`tokio::select!` 等多个信号**
>
> ```text
> tokio::select! {
>     _ = ctrl_c => {},        // 等 Ctrl+C
>     _ = terminate => {},     // 或 SIGTERM
> }
> ```
>
> 哪个先到都触发 shutdown。生产环境 Ctrl+C 和 SIGTERM 都要处理，所以 select 两个。

---

## 第三步：加 `TimeoutLayer` 防止慢请求卡死

如果某个 handler 是 `std::future::pending`（永远不返回），graceful shutdown 会**永远等下去**。加 `TimeoutLayer` 给每个请求设上限，超时直接返回。

````rust
use std::time::Duration;
use axum::http::StatusCode;
use tower_http::timeout::TimeoutLayer;

# #[tokio::main]
# async fn main() {
#     // ...
    let app = Router::new()
        .route("/slow", get(|| sleep(Duration::from_secs(5))))
        .route("/forever", get(std::future::pending::<()>))  // 永远不返回
        .layer((
            TraceLayer::new_for_http(),
            // 关键：graceful shutdown 等正在处理的请求，加超时防止永远卡
            TimeoutLayer::with_status_code(StatusCode::REQUEST_TIMEOUT, Duration::from_secs(10)),
        ));
#     // ...
# }
````

验证：

````bash
cargo run -p example-graceful-shutdown &
curl http://localhost:3000/forever &  # 这个永远不返回
sleep 2
kill -TERM %1
# 最多 10 秒后 /forever 超时返回 408，server 退出
````

> **新面孔：`TimeoutLayer`**
>
> tower-http 提供的请求超时 layer。`TimeoutLayer::with_status_code(StatusCode::REQUEST_TIMEOUT, Duration::from_secs(10))` 给每个请求设 10 秒上限，超时返回 408 Request Timeout。
>
> 这章用它配合 graceful shutdown——保证 shutdown 最多等 10 秒（最慢的请求超时），不会永远卡。

> **新面孔：layer tuple（`.layer((A, B))`）**
>
> axum 支持把多个 layer 作为 tuple 传入：`.layer((TraceLayer, TimeoutLayer))`。等价于 `.layer(TraceLayer).layer(TimeoutLayer)`，但类型推断更友好（layer 顺序无关时方便）。

### `std::future::pending` 是什么

`std::future::pending::<()>()` 创建一个**永远不 ready** 的 future——用于模拟"卡死的请求"。生产环境真实场景：handler 调下游服务超时配置不当、死循环、锁等待死锁等。

---

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
    // Enable tracing.
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!(
                    "{}=debug,tower_http=debug,axum=trace",
                    env!("CARGO_CRATE_NAME")
                )
                .into()
            }),
        )
        .with(tracing_subscriber::fmt::layer().without_time())
        .init();

    // Create a regular axum app.
    let app = Router::new()
        .route("/slow", get(|| sleep(Duration::from_secs(5))))
        .route("/forever", get(std::future::pending::<()>))
        .layer((
            TraceLayer::new_for_http(),
            // Graceful shutdown will wait for outstanding requests to complete. Add a timeout so
            // requests don't hang forever.
            TimeoutLayer::with_status_code(StatusCode::REQUEST_TIMEOUT, Duration::from_secs(10)),
        ));

    // Create a `TcpListener` using tokio.
    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

    // Run the server with graceful shutdown
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await;
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

测试 graceful shutdown：

````bash
# 终端 1：发个慢请求
curl http://localhost:3000/slow &
# 终端 2：发 SIGTERM
kill -TERM <pid>
# 终端 1 的 curl 仍然能拿到 "done"（server 等它完成），然后 server 退出
````

## 解读

### graceful shutdown 三阶段

```text
正常运行
  ↓ SIGTERM/SIGINT
停止接新连接（新 TCP 请求被拒绝）
  ↓
等正在处理的请求完成
  ↓ 所有请求完成（或 TimeoutLayer 触发）
进程退出
```

### 容器编排（k8s）的 shutdown 流程

```text
k8s 删除 Pod
  ↓
发 SIGTERM 给主进程
  ↓ （同时开始 grace period 倒计时，默认 30s）
主进程 graceful shutdown
  ↓
进程退出，Pod 删除
```

如果 30 秒内没退出，k8s 发 SIGKILL 强杀。所以 `TimeoutLayer` 的超时要**小于 k8s grace period**，否则可能被强杀（参考 ch47 TLS graceful shutdown 的 10 秒设定）。

## 常见问题

**为什么不光监听 Ctrl+C？** 生产环境在容器里没人按 Ctrl+C，k8s/docker 发 SIGTERM。所以必须监听 SIGTERM。Ctrl+C 是开发用的。

**为什么不直接 `tokio::signal::ctrl_c().await`？** 那个 future 完成 Ctrl+C 后才返回，但 SIGTERM 不会被它捕获。必须分别监听 SIGINT（ctrl_c）和 SIGTERM（unix signal）。

**TimeoutLayer 的超时设多少？** 取决于业务最长请求时间。一般设比 grace period 略小（如 k8s 30s grace，TimeoutLayer 设 25s 留 5 秒余量）。

## 手写任务

1. 改 `TimeoutLayer` 超时为 30 秒，看 `/forever` 等多久返回 408。
2. 加日志：graceful shutdown 开始/完成时打 tracing log。
3. 加连接数监控：发 SIGTERM 时正在处理几个连接，打日志。
4. 测试 SIGINT（Ctrl+C）和 SIGTERM 都能触发。

## 小结

这章用 3 步讲了 graceful shutdown：

1. **问题**：直接 kill 中断处理中的请求，导致数据不一致。
2. **`with_graceful_shutdown`**：传 signal future，完成后停止接新请求、等已接收请求完成。
3. **`TimeoutLayer`**：给每个请求设上限，防止慢请求永远卡 shutdown。

核心：**生产必须 graceful shutdown**，监听 SIGTERM（容器）和 SIGINT（开发），配合 TimeoutLayer 保证有界退出。

## 源码对照

- `examples/graceful-shutdown/Cargo.toml`
- `examples/graceful-shutdown/src/main.rs`
