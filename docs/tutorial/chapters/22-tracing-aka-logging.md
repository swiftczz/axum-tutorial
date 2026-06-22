# 22. tracing-aka-logging

对应示例：`examples/tracing-aka-logging`

前面章节用 `println!` 调试，但生产环境需要**结构化日志**——带 timestamp、level、span（请求作用域）、字段。`tracing` crate 是 Rust 生态的事实标准。这章用 `tower-http` 的 `TraceLayer` 给 axum app 加完整的 HTTP 请求日志。

分 2 步：先初始化 tracing subscriber（log sink），再加 `TraceLayer` 自定义日志格式（span、request、response、body chunk、eos、failure 各种钩子）。

相比前面章节新引入：**`tracing` crate、`tracing_subscriber`（EnvFilter + fmt layer）、`tower_http::trace::TraceLayer`、`info_span!`、`MatchedPath` 用于日志**。

## Cargo.toml

````toml
[package]
name = "example-tracing-aka-logging"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }
tower-http = { version = "0.6", features = ["trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

`tower-http` 启用 `trace` feature 才能用 `TraceLayer`。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：初始化 tracing subscriber

tracing 是 façade——日志事件要送到具体 sink（stdout、文件、journald）需要装 subscriber。这步用最常见的 `tracing_subscriber` 的 fmt layer（人类可读 stdout 输出）+ EnvFilter（按环境变量控制级别）。

````rust
use axum::{routing::get, Router};
use tokio::net::TcpListener;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                // axum 内置 extractor 的 rejection 日志在 axum::rejection target，默认 TRACE
                format!(
                    "{}=debug,tower_http=debug,axum::rejection=trace",
                    env!("CARGO_CRATE_NAME")
                )
                .into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app = Router::new().route("/", get(handler));

    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> &'static str {
    "Hello, World!"
}
````

验证：

````bash
cd examples
RUST_LOG=debug cargo run -p example-tracing-aka-logging
````

````bash
curl http://127.0.0.1:3000/
````

看到日志 `listening on 127.0.0.1:3000`。

> **新面孔：`tracing_subscriber::registry`**
>
> tracing 的 subscriber 安装入口。`registry()` 是基础，`.with(...)` 链式添加 layer（每个 layer 是一个 sink 或处理逻辑）。`.init()` 设为全局默认 subscriber。
>
> 没装 subscriber 的话所有 tracing 事件被丢弃（noop）——前面章节没装 subscriber 所以 `tracing::debug!` 不输出。

> **新面孔：`EnvFilter`**
>
> 按环境变量 `RUST_LOG` 控制日志级别的过滤器。格式 `target=level`：
>
> - `debug`：全局 debug 级别
> - `my_crate=debug,tower_http=debug`：分别给不同 target 设级别
> - `axum::rejection=trace`：axum 内置 extractor 的 rejection（解析失败）日志默认 TRACE 级别，要看的话必须设 `trace`
>
> `EnvFilter::try_from_default_env()` 读 `RUST_LOG` 环境变量，没设则 fallback 到 closure 里的默认值。

> **新面孔：`fmt::layer`**
>
> 把 tracing 事件渲染成人类可读文本写到 stdout。生产环境通常加 JSON formatter（如 `tracing_subscriber::fmt::format::Json`）方便 ELK/Loki 解析。

这步 app 还没加 TraceLayer，所以 `curl` 请求不会产生请求级日志。下一步加。

---

## 第二步：`TraceLayer` 给请求加完整日志

加 `TraceLayer`——tower-http 提供的 HTTP 请求日志 layer，提供多个钩子（make_span_with / on_request / on_response / on_body_chunk / on_eos / on_failure）覆盖请求生命周期的每个阶段。

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
use tower_http::{classify::ServerErrorsFailureClass, trace::TraceLayer};
use tracing::{info_span, Span};

# #[tokio::main]
# async fn main() {
#     // ... tracing 初始化 ...
    let app = Router::new()
        .route("/", get(handler))
        .layer(
            TraceLayer::new_for_http()
                // 1. 请求来时创建 span（带 method + matched_path）
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
                // 2. 请求开始
                .on_request(|_request: &Request<_>, _span: &Span| {
                    tracing::debug!("started processing request")
                })
                // 3. 响应发出
                .on_response(|_response: &Response, _latency: Duration, _span: &Span| {
                    tracing::debug!("finished processing request")
                })
                // 4. 每个 body chunk
                .on_body_chunk(|_chunk: &Bytes, _latency: Duration, _span: &Span| {
                    tracing::debug!("sending body chunk")
                })
                // 5. 流结束
                .on_eos(
                    |_trailers: Option<&HeaderMap>, _stream_duration: Duration, _span: &Span| {
                        tracing::debug!("stream closed")
                    },
                )
                // 6. 失败（5xx）
                .on_failure(
                    |_error: ServerErrorsFailureClass, _latency: Duration, _span: &Span| {
                        tracing::error!("something went wrong")
                    },
                ),
        );
#     // ...
# }

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
````

验证：

````bash
RUST_LOG=example_tracing_aka_logging=debug,tower_http=debug cargo run -p example-tracing-aka-logging
curl http://127.0.0.1:3000/
````

服务端日志现在有：

```text
DEBUG started processing request
DEBUG finished processing request
DEBUG sending body chunk
```

> **新面孔：`TraceLayer::new_for_http`**
>
> tower-http 提供的 HTTP 请求日志 layer。默认配置就有不错的日志（method、uri、status、latency），这章演示怎么用闭包自定义每个阶段。
>
> 不带任何配置直接 `.layer(TraceLayer::new_for_http())` 就能产生基本请求日志——这章的闭包是高级定制。

> **新面孔：`make_span_with`**
>
> 每个请求来时调一次，创建一个 `Span`（tracing 的请求作用域）。后续所有日志事件都附在这个 span 下，便于关联。
>
> 这章 span 包含 `method` 和 `matched_path` 两个字段。`matched_path` 从 `request.extensions().get::<MatchedPath>()` 取——这是 axum 路由匹配后塞进去的（参考第 27 章 metrics）。用 `matched_path` 而非 `request.uri().path()` 因为后者带具体值（如 `/users/123`），前者是模板（`/users/{id}`）——日志聚合更友好。

> **新面孔：六个生命周期钩子**
>
> TraceLayer 提供六个钩子覆盖请求生命周期：
>
> 1. `make_span_with`：请求来时创建 span（**最先调**）
> 2. `on_request`：开始处理
> 3. `on_response`：响应头发出（**handler 返回后**）
> 4. `on_body_chunk`：每个响应 body chunk
> 5. `on_eos`：流结束（streaming 响应）
> 6. `on_failure`：5xx 错误
>
> 每个钩子都有 `_span: &Span` 参数，可以 `.record("field", value)` 给 span 补字段——比如 `on_response` 时记录 `latency`、`status`。

> **新面孔：`info_span!` 宏**
>
> 创建一个 INFO 级别的 span。语法 `info_span!("name", field1 = value, field2 = Empty)`：
> - 第一个参数是 span 名称
> - 后续 `key = value` 是 span 字段
> - `tracing::field::Empty` 表示先占位，后续用 `span.record("field", value)` 填值

---

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
                // axum logs rejections from built-in extractors with the `axum::rejection`
                // target, at `TRACE` level. `axum::rejection=trace` enables showing those events
                format!(
                    "{}=debug,tower_http=debug,axum::rejection=trace",
                    env!("CARGO_CRATE_NAME")
                )
                .into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // build our application with a route
    let app = Router::new()
        .route("/", get(handler))
        // `TraceLayer` is provided by tower-http so you have to add that as a dependency.
        // It provides good defaults but is also very customizable.
        //
        // See https://docs.rs/tower-http/0.1.1/tower_http/trace/index.html for more details.
        //
        // If you want to customize the behavior using closures here is how.
        .layer(
            TraceLayer::new_for_http()
                .make_span_with(|request: &Request<_>| {
                    // Log the matched route's path (with placeholders not filled in).
                    // Use request.uri() or OriginalUri if you want the real path.
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
                    // You can use `_span.record("some_other_field", value)` in one of these
                    // closures to attach a value to the initially empty field in the info_span
                    // created above.
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

    // run it
    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
````

## 运行

````bash
cd examples
RUST_LOG=example_tracing_aka_logging=debug,tower_http=debug cargo run -p example-tracing-aka-logging
````

````bash
curl http://127.0.0.1:3000/
````

服务端日志：

```text
DEBUG started processing request
DEBUG finished processing request
DEBUG sending body chunk
```

## 解读

### tracing 的三个核心概念

```text
Event（事件）：单条日志（tracing::info! / debug! / error!）
Span（跨度）：带作用域的事件组（info_span!），多个 event 关联到一个 span
Subscriber：收集 event/span 的 sink（stdout、文件、Jaeger）
```

Span 是 tracing 比传统日志强大的原因——一条请求对应一个 span，里面所有日志都关联。聚合后能按 span 看一条请求的完整生命周期。

### 为什么用 `matched_path` 而非 `request.uri().path()`

```text
matched_path:  /users/{id}     # 路由模板，每次一样
uri().path():  /users/123      # 实际路径，每次不同
```

日志聚合时按 `/users/{id}` 聚合能看到"所有查用户接口的 QPS"；按 `/users/123`/`/users/456` 聚合就分散了。和第 27 章 metrics 用 `MatchedPath` 同理。

## 常见问题

**为什么默认看不到 rejection 日志？** axum 内置 extractor 的 rejection（如 `Json` 解析失败）在 `axum::rejection` target，默认 TRACE 级别。要看的话 `EnvFilter` 加 `axum::rejection=trace`。

**生产用什么 sink？** JSON formatter + 输出到 stdout，由 docker/k8s 收集进 ELK/Loki/Datadog。`tracing_subscriber::fmt().json().finish()`。

**tracing 比 log 好在哪？** tracing 有 span 概念（结构化作用域），log 只是字符串。tracing 是 log 的超集（兼容 log facade）。

## 手写任务

1. 在 `on_response` 闭包里用 `_span.record("status", response.status().as_u16())` 把状态码记进 span。
2. 在 `on_failure` 区分 4xx 和 5xx，4xx 不算 failure（默认 5xx 才算）。
3. 加 JSON formatter：`.with(tracing_subscriber::fmt::layer().json())`。
4. 加 request id（ch24）：每个请求唯一 ID 记进 span，日志能按 ID 关联。

## 小结

这章用 2 步讲了 axum 的结构化日志：

1. **初始化 subscriber**：`tracing_subscriber::registry() + EnvFilter + fmt::layer()`，没装 subscriber 所有 tracing 事件被丢弃。
2. **TraceLayer**：tower-http 提供的请求日志 layer，6 个生命周期钩子（make_span_with / on_request / on_response / on_body_chunk / on_eos / on_failure）覆盖完整请求生命周期。

核心：**tracing 是 façade**，subscriber 决定 sink；**Span 是请求作用域**，关联同一条请求的所有日志；**`matched_path` 比 `uri().path()` 适合日志聚合**（模板 vs 具体值）。

## 源码对照

- `examples/tracing-aka-logging/Cargo.toml`
- `examples/tracing-aka-logging/src/main.rs`
