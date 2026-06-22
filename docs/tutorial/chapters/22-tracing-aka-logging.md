# 22. tracing-aka-logging

对应示例：`examples/tracing-aka-logging`

业务功能很简单(`GET /` 返回 Hello World),但给 Router 加了完整的 HTTP trace 配置。这一章重点不是 handler,而是 `TraceLayer`:request span、matched path 和各类 trace 回调。从这章开始进入中间件、日志、跨域、压缩和指标阶段。



相比前面章节新引入：**`TraceLayer` 的 5 个回调（on_request/on_response/on_body_chunk/on_eos/on_failure）、`MatchedPath`、`EnvFilter`**。

## Cargo.toml

````toml
[package]
name = "example-tracing-aka-logging"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["tracing"] }
tokio = { version = "1.0", features = ["full"] }
tower-http = { version = "0.6.1", features = ["trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 完整代码

````rust
use axum::{
    body::Bytes,
    extract::MatchedPath,
    http::{HeaderMap, Request},
    response::{Html, Response},
    routing::get,
    Router,
};
use std::time::Duration;
use tokio::net::TcpListener;
use tower_http::{classify::ServerErrorsFailureClass, trace::TraceLayer};
use tracing::{info_span, Span};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!(
                    "{}=debug,tower_http=debug,axum::rejection=trace",
                    env!("CARGO_CRATE_NAME")
                )
                .into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app = Router::new()
        .route("/", get(handler))
        .layer(
            TraceLayer::new_for_http()
                .make_span_with(|request: &Request<_>| {
                    let matched_path = request
                        .extensions()
                        .get::<MatchedPath>()
                        .map(MatchedPath::as_str);

                    info_span!(
                        "http_request",
                        method = ?request.method(),
                        matched_path,
                        some_other_field = tracing::field::Empty,
                    )
                })
                .on_request(|_request: &Request<_>, _span: &Span| {
                    tracing::debug!("started processing request")
                })
                .on_response(|_response: &Response, _latency: Duration, _span: &Span| {
                    tracing::debug!("finished processing request")
                })
                .on_body_chunk(|_chunk: &Bytes, _latency: Duration, _span: &Span| {
                    tracing::debug!("sending body chunk")
                })
                .on_eos(
                    |_trailers: Option<&HeaderMap>, _stream_duration: Duration, _span: &Span| {
                        tracing::debug!("stream closed")
                    },
                )
                .on_failure(
                    |_error: ServerErrorsFailureClass, _latency: Duration, _span: &Span| {
                        tracing::error!("something went wrong")
                    },
                ),
        );

    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app).await;
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
````

## 运行

````bash
cd examples
cargo run -p example-tracing-aka-logging
````

访问:

````bash
curl -i http://127.0.0.1:3000/
````

终端看到类似日志:

````text
started processing request
finished processing request
sending body chunk
stream closed
````

## 解读

### tracing vs `println!`

`println!` 只打印文本。`tracing` 记录**结构化字段**(`method=GET`、`matched_path="/"`、`latency=...`、`status=...`),而且有 **span**——"一次请求的日志上下文",把同一次请求的多条日志关联起来:

```text
request span
  -> on_request log
  -> handler log
  -> on_response log
```

### tracing 三层架构

| 概念 | 是什么 | 谁产生它 |
| --- | --- | --- |
| **span** | 一段代码的"上下文范围",可嵌套 | 你的代码 `info_span!(...)` 或 TraceLayer 自动创建 |
| **event** | 一个具体日志事件(一行日志) | 你的代码 `info!`/`debug!`/`error!` |
| **subscriber** | 接收 span/event 并决定怎么处理(打印/过滤/上报) | `tracing_subscriber` 在 `main` 开头初始化 |

**span 和 event 本身不产生输出,只有 subscriber 才会"打印"它们**。这解释了一个常见困惑:**为什么代码里写了 `info!("hello")` 却看不到输出?** 因为没初始化 subscriber,event 没人接收就被丢弃。所以 `main` 开头这段是必须的:

````rust
tracing_subscriber::registry()
    .with(...)                              // 过滤层:决定哪些 event 要处理
    .with(tracing_subscriber::fmt::layer()) // 格式化层:把 event 变成终端文本
    .init();                                // 注册成全局 subscriber
````

`init()` 之后全局有一个 subscriber,所有 `info!`/`debug!`/TraceLayer 创建的 span 都流向它。

### 初始化配置

````rust
tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
    format!("{}=debug,tower_http=debug,axum::rejection=trace", env!("CARGO_CRATE_NAME"))
})
````

日志级别:当前 crate `debug`、`tower_http` `debug`、`axum::rejection` `trace`。`axum::rejection=trace` 很有用——axum 内置 extractor 失败时会在这个 target 输出更详细的 rejection 日志。

### `make_span_with` 创建请求 span

````rust
.make_span_with(|request: &Request<_>| {
    let matched_path = request
        .extensions()
        .get::<MatchedPath>()
        .map(MatchedPath::as_str);

    info_span!(
        "http_request",
        method = ?request.method(),
        matched_path,
        some_other_field = tracing::field::Empty,
    )
})
````

为每个请求创建 span,记录 method、matched_path、预留字段。后面 `on_request`/`on_response` 的 event 都带上这个 span 信息。

**为什么用 `MatchedPath` 而不是 `request.uri()`?** 真实请求 `/users/123` 和 `/users/456` 的 uri 不同,但 `MatchedPath` 统一是 `/users/{id}`——更适合日志聚合和指标统计。

### 五个回调

| 回调 | 触发时机 | 常见用途 |
| --- | --- | --- |
| `on_request` | 请求刚进入服务 | 记录请求开始 |
| `on_response` | 响应准备完成(可拿 latency) | 记录耗时和状态码 |
| `on_body_chunk` | 每发送一个 body chunk | streaming/大响应观察 |
| `on_eos` | body stream 结束 | 记录流结束 |
| `on_failure` | TraceLayer 判断请求失败 | 记录错误分类、latency |

`on_failure` 的错误类型是 `ServerErrorsFailureClass`,主要用于服务端错误分类。

### 动态记录 span 字段

````rust
some_other_field = tracing::field::Empty  // 先声明为空
// 后续回调里填值:
span.record("some_other_field", "some value");
````

真实项目常用:请求开始时还不知道 user_id,认证 middleware 解析完后再写入 span。

## 手写任务

跑通后做三个小改动:

1. 在 `on_response` 里打印 `_latency`。
2. 把 `some_other_field` 改成实际字段,例如固定字符串 `"demo"`。
3. 新增 `/users/{id}` 路由,观察 `MatchedPath` 和真实 URI 的区别。

## 小结

- 后端服务应用结构化日志(tracing),不只靠 `println!`。
- tracing 三层:span(上下文)、event(日志事件)、subscriber(决定怎么处理);没初始化 subscriber,event 就被丢弃。
- `TraceLayer` 为 HTTP 请求自动记录生命周期事件(on_request/on_response/on_body_chunk/on_eos/on_failure)。
- `MatchedPath` 是路由模板(`/users/{id}`),比真实 URI 更适合聚合统计。
- 日志级别通过 `EnvFilter` 控制;`axum::rejection=trace` 能显示 extractor rejection 详细日志。

## 源码对照

- `examples/tracing-aka-logging/Cargo.toml`
- `examples/tracing-aka-logging/src/main.rs`
