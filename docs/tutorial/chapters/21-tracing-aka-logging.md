# 21. tracing-aka-logging

对应示例：`examples/tracing-aka-logging`

本章目标：使用 `TraceLayer` 做结构化 HTTP 日志，理解 request span、matched path 和各类 trace 回调。

从这一章开始进入中间件、日志、跨域、压缩和指标阶段。  
日志是后端服务上线后定位问题的基础能力。

## 这个小项目在做什么

这个 example 业务功能很简单：

```text
GET / -> <h1>Hello, World!</h1>
```

但它给 Router 加了非常完整的 HTTP trace 配置：

- 请求开始时打印日志。
- 响应完成时打印日志。
- body chunk 发送时打印日志。
- stream 结束时打印日志。
- 请求失败时打印错误日志。
- 给每个请求创建 span，并记录 method 和 matched path。

请求主线是：

```text
客户端 GET /
-> TraceLayer 创建 http_request span
-> on_request 打印 started processing request
-> handler 返回 HTML
-> on_response 打印 finished processing request
-> body 发送时触发 on_body_chunk
-> stream 结束时触发 on_eos
```

## 先理解 tracing 和普通 println 的区别

`println!` 只是打印文本。  
`tracing` 更适合后端服务，因为它能记录结构化字段：

```text
method=GET
matched_path="/"
latency=...
status=...
```

而且 tracing 有 span。  
span 可以理解成"一次请求的日志上下文"：

```text
request span
  -> on_request log
  -> handler log
  -> on_response log
```

这样你能知道哪些日志属于同一个请求。

### tracing 的三层架构（建立心智模型）

tracing 里有三个核心概念，初学者经常分不清。理清它们，后面的代码就不会是黑盒：

| 概念 | 是什么 | 谁产生它 |
| --- | --- | --- |
| **span** | 一段代码的"上下文范围"，可以嵌套 | 你的代码用 `info_span!(...)` 创建，或 TraceLayer 自动创建 |
| **event** | 一个具体的日志事件（就是一行日志） | 你的代码用 `info!(...)`、`debug!(...)`、`error!(...)` 产生 |
| **subscriber** | 接收 span/event 并决定怎么处理（打印、过滤、上报） | `tracing_subscriber` 在 `main` 开头初始化 |

关键点：**span 和 event 本身不产生输出，只有 subscriber 才会"打印"它们**。

这就解释了一个常见困惑：**为什么代码里写了 `info!("hello")` 却看不到输出？** 因为如果没有初始化 subscriber，event 就没人接收，直接被丢弃。所以 `main` 开头的这段是必须的：

````rust
tracing_subscriber::registry()
    .with(...)  // 过滤层：决定哪些 event 要处理
    .with(tracing_subscriber::fmt::layer())  // 格式化层：把 event 变成终端文本
    .init();    // 注册成全局 subscriber
````

`init()` 之后，全局就有一个 subscriber 了。之后所有 `info!`、`debug!`、TraceLayer 创建的 span，都会流向这个 subscriber。

span 的作用是给 event 提供"上下文"。当你在 span 里发一个 event，subscriber 知道这个 event 属于哪个请求，可以把它和同一个请求的其他 event 关联起来。这就是为什么 TraceLayer 要 `make_span_with` 先创建 span——后面所有 `on_request`、`on_response` 产生的 event 都会带上这个 span 的信息（method、path 等）。

## 文件和依赖

这个 example 有两个文件：

1. `examples/tracing-aka-logging/Cargo.toml`：声明 Axum tracing feature、tower-http trace、tracing。
2. `examples/tracing-aka-logging/src/main.rs`：实现 Router、TraceLayer 和各类回调。

关键依赖：

- `axum`：提供 `Router`、`MatchedPath`、`Html`。
- `tower-http`：提供 `TraceLayer`。
- `tracing`：创建 span 和输出事件。
- `tracing-subscriber`：初始化日志订阅器。
- `tokio`：异步运行时。

`Cargo.toml` 里启用了 Axum tracing feature：

````toml
axum = { path = "../../axum", features = ["tracing"] }
````

## 第一步：初始化 tracing

源码：

````rust
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
````

这段配置日志级别：

- 当前 crate：`debug`
- `tower_http`：`debug`
- `axum::rejection`：`trace`

`axum::rejection=trace` 很有用。  
Axum 内置 extractor 失败时，会在这个 target 下输出更详细的 rejection 日志。

## 第二步：创建基础路由

源码：

````rust
let app = Router::new()
    .route("/", get(handler))
    .layer(...);
````

业务 handler 很简单：

````rust
async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
````

这一章的重点不是 handler，而是 `.layer(TraceLayer...)`。

## 第三步：为请求创建 span

核心源码：

````rust
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
````

`make_span_with` 会为每个请求创建一个 span。

这里记录了：

- `method`：HTTP 方法，例如 GET。
- `matched_path`：匹配到的路由模板，例如 `/users/{id}`。
- `some_other_field`：先留空，后续回调可以填值。

为什么用 `MatchedPath`，而不是直接用 `request.uri()`？

如果真实请求是：

```text
/users/123
/users/456
```

`request.uri()` 会得到不同值。  
`MatchedPath` 会得到统一模板：

```text
/users/{id}
```

这更适合日志聚合和指标统计。

## 第四步：on_request 和 on_response

源码：

````rust
.on_request(|_request: &Request<_>, _span: &Span| {
    tracing::debug!("started processing request")
})
.on_response(|_response: &Response, _latency: Duration, _span: &Span| {
    tracing::debug!("finished processing request")
})
````

这两个回调最常用：

- `on_request`：请求刚进入服务时调用。
- `on_response`：响应准备完成时调用。

`on_response` 里能拿到 `_latency`，真实项目通常会记录请求耗时和状态码。

## 第五步：on_body_chunk 和 on_eos

源码：

````rust
.on_body_chunk(|_chunk: &Bytes, _latency: Duration, _span: &Span| {
    tracing::debug!("sending body chunk")
})
.on_eos(
    |_trailers: Option<&HeaderMap>, _stream_duration: Duration, _span: &Span| {
        tracing::debug!("stream closed")
    },
)
````

这两个和响应 body 流有关：

- `on_body_chunk`：每发送一个 body chunk 调用一次。
- `on_eos`：body stream 结束时调用。

对于普通小响应，这两个可能不明显。  
对于 streaming、下载、大响应，它们会更有用。

## 第六步：on_failure

源码：

````rust
.on_failure(
    |_error: ServerErrorsFailureClass, _latency: Duration, _span: &Span| {
        tracing::error!("something went wrong")
    },
)
````

`on_failure` 在 TraceLayer 判断请求失败时调用。  
这里的错误类型是 `ServerErrorsFailureClass`，主要用于服务端错误分类。

真实项目里可以在这里记录：

- 错误分类。
- latency。
- span 上的请求字段。

## 第七步：动态记录 span 字段

源码注释里提到：

````rust
// _span.record("some_other_field", value)
````

这表示 span 字段可以先声明为空：

````rust
some_other_field = tracing::field::Empty
````

后续某个回调里再填值：

````rust
span.record("some_other_field", "some value");
````

这在真实项目里很常见，比如请求开始时还不知道 user_id，认证 middleware 解析完后再把 user_id 写入 span。

## 函数职责速查

- `main`：初始化 tracing，注册路由和 TraceLayer，启动服务。
- `handler`：返回 Hello World HTML。
- `make_span_with`：为每个请求创建结构化 span。
- `on_request`：请求开始时记录日志。
- `on_response`：响应完成时记录日志。
- `on_body_chunk`：发送 body chunk 时记录日志。
- `on_eos`：stream 结束时记录日志。
- `on_failure`：请求失败时记录错误日志。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-tracing-aka-logging
//! ```

// 引入 Bytes、MatchedPath、HeaderMap、Request、Html、Response、GET 路由和 Router。
use axum::{
    body::Bytes,
    extract::MatchedPath,
    http::{HeaderMap, Request},
    response::{Html, Response},
    routing::get,
    Router,
};
// Duration 用于记录耗时。
use std::time::Duration;
// TcpListener 绑定端口。
use tokio::net::TcpListener;
// TraceLayer 提供 HTTP trace，ServerErrorsFailureClass 表示失败分类。
use tower_http::{classify::ServerErrorsFailureClass, trace::TraceLayer};
// info_span 创建 span，Span 是当前请求日志上下文。
use tracing::{info_span, Span};
// tracing_subscriber 初始化日志。
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 初始化 tracing 日志系统。
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

    // 注册路由，并添加 TraceLayer。
    let app = Router::new()
        .route("/", get(handler))
        .layer(
            TraceLayer::new_for_http()
                // 为每个请求创建 span。
                .make_span_with(|request: &Request<_>| {
                    // MatchedPath 是路由模板，例如 /users/{id}。
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
                // 请求开始时调用。
                .on_request(|_request: &Request<_>, _span: &Span| {
                    tracing::debug!("started processing request")
                })
                // 响应完成时调用。
                .on_response(|_response: &Response, _latency: Duration, _span: &Span| {
                    tracing::debug!("finished processing request")
                })
                // 每发送一个 body chunk 时调用。
                .on_body_chunk(|_chunk: &Bytes, _latency: Duration, _span: &Span| {
                    tracing::debug!("sending body chunk")
                })
                // body stream 结束时调用。
                .on_eos(
                    |_trailers: Option<&HeaderMap>, _stream_duration: Duration, _span: &Span| {
                        tracing::debug!("stream closed")
                    },
                )
                // 请求失败时调用。
                .on_failure(
                    |_error: ServerErrorsFailureClass, _latency: Duration, _span: &Span| {
                        tracing::error!("something went wrong")
                    },
                ),
        );

    // 绑定本机 3000 端口。
    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动 HTTP 服务。
    axum::serve(listener, app).await;
}

// GET / 的 handler。
async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
````

## 运行和验证

运行前先确认 `examples/tracing-aka-logging/Cargo.toml` 里的 `axum = { path = "../../axum", features = ["tracing"] }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-tracing-aka-logging
````

访问：

````bash
curl -i http://127.0.0.1:3000/
````

你应该在终端看到类似日志：

````text
started processing request
finished processing request
sending body chunk
stream closed
````

常见卡点：

- `TraceLayer` 来自 `tower-http`，不是 Axum 本体。
- `MatchedPath` 是路由模板，适合日志聚合。
- `request.uri()` 是真实 URI，可能包含具体 id 和 query。
- `axum::rejection=trace` 可以显示内置 extractor rejection 的更多日志。

## 手写任务

完成本章代码后，试着做三个小改动：

1. 在 `on_response` 里打印 `_latency`。
2. 把 `some_other_field` 改成实际字段，例如固定字符串 `"demo"`。
3. 新增一个 `/users/{id}` 路由，观察 `MatchedPath` 和真实 URI 的区别。

## 本章真正要记住什么

- 后端服务应该使用结构化日志，而不是只靠 `println!`。
- `TraceLayer` 能为 HTTP 请求自动记录关键生命周期事件。
- span 可以把一次请求的多条日志关联起来。
- `MatchedPath` 比真实 URI 更适合聚合统计。
- 日志级别可以通过 `EnvFilter` 控制。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/tracing-aka-logging/Cargo.toml`
- `examples/tracing-aka-logging/src/main.rs`
