# 46. graceful-shutdown

对应示例：`examples/graceful-shutdown`

本章目标：理解优雅关闭，学会用 `with_graceful_shutdown` 监听 Ctrl+C/SIGTERM，并用 `TimeoutLayer` 避免请求永远阻塞关闭。

后端服务停止时，不应该总是“立刻杀掉”。  
更好的方式是：

```text
停止接收新请求
等待正在处理的请求完成
然后退出进程
```

这就是 graceful shutdown。

## 这个小项目在做什么

应用有两个路由：

```text
GET /slow    -> sleep 5 秒后完成
GET /forever -> 永远 pending
```

服务启动后：

```text
Ctrl+C 或 SIGTERM
-> shutdown_signal 完成
-> axum::serve 开始优雅关闭
-> 等待正在处理的请求完成
-> TimeoutLayer 防止 /forever 永远挂住
```

## 为什么需要优雅关闭

如果服务直接退出，可能会导致：

```text
正在处理的请求被中断
客户端收到连接错误
写数据库操作做了一半
日志和指标没有刷完
```

优雅关闭让服务有机会收尾。

但也不能无限等。  
所以本章加了超时层。

## 文件和依赖

这个 example 有两个文件：

1. `examples/graceful-shutdown/Cargo.toml`：声明 Axum tracing、Tokio、tower-http timeout/trace。
2. `examples/graceful-shutdown/src/main.rs`：实现 slow/forever 路由、timeout、shutdown signal。

关键依赖：

- `axum`：提供 Router 和 `with_graceful_shutdown`。
- `tokio::signal`：监听 Ctrl+C 和 Unix SIGTERM。
- `tower-http`：提供 `TimeoutLayer` 和 `TraceLayer`。
- `tokio::time`：sleep。

## 第一步：构造两个测试路由

源码：

````rust
let app = Router::new()
    .route("/slow", get(|| sleep(Duration::from_secs(5))))
    .route("/forever", get(std::future::pending::<()>))
    .layer((TraceLayer::new_for_http(), TimeoutLayer::with_status_code(...)));
````

`/slow` 用来模拟慢请求：

```text
5 秒后完成
```

`/forever` 用来模拟永远不结束的请求：

```text
永远 pending
```

这两个路由能帮助你观察关闭行为。

## 第二步：加 TraceLayer 和 TimeoutLayer

源码：

````rust
.layer((
    TraceLayer::new_for_http(),
    TimeoutLayer::with_status_code(StatusCode::REQUEST_TIMEOUT, Duration::from_secs(10)),
))
````

`TraceLayer` 用于日志。  
`TimeoutLayer` 表示请求最多执行 10 秒。

如果请求超过 10 秒，返回：

```text
408 Request Timeout
```

这对 graceful shutdown 很重要。  
因为优雅关闭会等已有请求完成，如果某个请求永远不完成，服务也会一直等。

## 第三步：with_graceful_shutdown

源码：

````rust
axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())
    .await;
````

`shutdown_signal()` 是一个 future。  
当这个 future 完成时，服务开始优雅关闭。

可以理解成：

```text
server 正常运行
shutdown_signal 等待 Ctrl+C/SIGTERM
信号来了
server 开始关闭
```

## 第四步：监听 Ctrl+C

源码：

````rust
let ctrl_c = async {
    signal::ctrl_c()
        .await
        .expect("failed to install Ctrl+C handler");
};
````

这会等待用户在终端按：

```text
Ctrl+C
```

开发环境最常见。

## 第五步：监听 SIGTERM

源码：

````rust
#[cfg(unix)]
let terminate = async {
    signal::unix::signal(signal::unix::SignalKind::terminate())
        .expect("failed to install signal handler")
        .recv()
        .await;
};

#[cfg(not(unix))]
let terminate = std::future::pending::<()>();
````

Unix 系统上监听：

```text
SIGTERM
```

这在容器、systemd、部署平台里很常见。

非 Unix 平台没有这段能力，所以用 `pending()` 占位。

## 第六步：tokio::select 等任意信号

源码：

````rust
tokio::select! {
    _ = ctrl_c => {},
    _ = terminate => {},
}
````

Ctrl+C 或 SIGTERM 任意一个先发生，`shutdown_signal()` 就结束。

然后 `with_graceful_shutdown` 开始关闭服务。

## 函数职责速查

- `main`：初始化日志，创建带 timeout 的 Router，启动带 graceful shutdown 的服务。
- `shutdown_signal`：等待 Ctrl+C 或 SIGTERM。


## 带中文注释的手写版

完整 `src/main.rs`：

````rust
//! 优雅关闭示例。
//!
//! Run with
//!
//! ```not_rust
//! cargo run -p example-graceful-shutdown
//! kill or ctrl-c
//! ```

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
    // 初始化 tracing 日志。
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

    // 创建普通 Axum app。
    let app = Router::new()
        // 模拟慢请求：5 秒后完成。
        .route("/slow", get(|| sleep(Duration::from_secs(5))))
        // 模拟永远不会完成的请求。
        .route("/forever", get(std::future::pending::<()>))
        .layer((
            TraceLayer::new_for_http(),
            // 优雅关闭会等待已有请求完成，所以要加超时，避免 forever 请求永远挂住。
            TimeoutLayer::with_status_code(StatusCode::REQUEST_TIMEOUT, Duration::from_secs(10)),
        ));

    // 创建 TcpListener。
    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

    // 带 graceful shutdown 启动服务。
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}

async fn shutdown_signal() {
    // 本地开发常见：Ctrl+C。
    let ctrl_c = async {
        signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
    };

    // Unix 部署常见：SIGTERM。
    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install signal handler")
            .recv()
            .await;
    };

    // 非 Unix 平台没有上面的 SIGTERM 处理，使用永远 pending 的 future 占位。
    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    // 任意一个信号到来，就触发 graceful shutdown。
    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
````

## 运行和验证

运行：

````bash
cargo run -p example-graceful-shutdown
````

请求慢接口：

````bash
curl http://127.0.0.1:3000/slow
````

在另一个终端按 Ctrl+C 停止服务，观察服务是否等待请求完成。

请求永远 pending 的接口：

````bash
curl -i http://127.0.0.1:3000/forever
````

它会被 `TimeoutLayer` 在约 10 秒后变成 408。

## 常见卡点

### 1. 优雅关闭是不是立刻退出？

不是。  
它会停止接新请求，并等待已有请求完成。

### 2. 为什么还需要 TimeoutLayer？

如果已有请求永远不完成，优雅关闭也会一直等。  
TimeoutLayer 给请求一个上限。

### 3. Ctrl+C 和 SIGTERM 有什么区别？

Ctrl+C 常用于本地开发。  
SIGTERM 常用于部署平台通知进程退出。

## 手写任务

1. 写一个 `/slow` 路由，sleep 5 秒。
2. 写 `shutdown_signal()`，监听 Ctrl+C。
3. 给 `axum::serve` 加 `with_graceful_shutdown`。
4. 再监听 Unix SIGTERM。
5. 加 `TimeoutLayer`，处理永远 pending 的请求。

## 本章真正要记住什么

优雅关闭的核心是：

```text
shutdown signal future 完成
server 停止接新请求
等待已有请求结束
请求必须有超时保护
```

在 Axum 中就是：

````rust
axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())
    .await;
````

## 源码对照

本章手写版对应源码：

- `examples/graceful-shutdown/src/main.rs`
- `examples/graceful-shutdown/Cargo.toml`
