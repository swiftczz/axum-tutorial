# 15. reqwest-response

对应示例：`examples/reqwest-response`

这章把 axum 的**流式响应**和 **reqwest 客户端**组合起来：用 reqwest 调下游接口，把下游响应**原样流式转发**给客户端。常见场景：反向代理、API gateway、下载中转。

分 3 步：先写流式响应基础（`Body::from_stream`），再写 reqwest 客户端转 response，最后用 `TraceLayer` 观察流式 chunk。

相比前面章节新引入：**`Body::from_stream`（从任意 Stream 造 body）、`reqwest::Client` 作为 State、`reqwest_response.bytes_stream()`、`TraceLayer::on_body_chunk`**。

## Cargo.toml

````toml
[package]
name = "example-reqwest-response"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
reqwest = { version = "0.12", default-features = false, features = ["rustls-tls"] }
tokio = { version = "1.0", features = ["full"] }
tokio-stream = "0.1"
tower-http = { version = "0.6", features = ["trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：流式响应基础——`Body::from_stream`

前面章节 handler 返回的 `Json`/`String`/`Html` 都是一次性 buffer 好的响应体。这章要返回**流式**响应体——数据一段段产生，不一次缓冲。先写一个最简的流式 handler：每秒产一个数字。

````rust
use axum::{
    body::Body,
    routing::get,
    Router,
};
use std::{convert::Infallible, time::Duration};
use tokio_stream::StreamExt;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app = Router::new().route("/stream", get(stream_some_data));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn stream_some_data() -> Body {
    let stream = tokio_stream::iter(0..5)
        .throttle(Duration::from_secs(1))          // 每秒一个
        .map(|n| n.to_string())
        .map(Ok::<_, Infallible>);                 // Body::from_stream 要 Stream<Item = Result<_,_>>
    Body::from_stream(stream)
}
````

验证：

````bash
cd examples
cargo run -p example-reqwest-response
````

````bash
curl http://127.0.0.1:3000/stream
# 看到 0 1 2 3 4 每秒一个字符流过来（不是一次性出来）
````

> **新面孔：`Body::from_stream`**
>
> 把任意 `Stream<Item = Result<Bytes, E>>` 转成 axum 响应体。axum 一段段 poll 这个 stream，每 yield 一段就发给客户端——**不缓冲**，适合大文件、实时数据、流式代理。
>
> 注意 stream 的 Item 必须是 `Result`——错误类型可以是 `Infallible`（永不失败）或具体错误类型。

> **新面孔：`tokio_stream::StreamExt::throttle`**
>
> 给 stream 加节流——每 `Duration` 才放一个 item 通过。这章用来模拟"数据源慢慢产生数据"，验证流式行为。

> **新面孔：`tokio_stream::iter` + `map(Ok)`**
>
> 把迭代器转成 stream，再 `map(Ok)` 把每个 item 包成 `Result<_, Infallible>`——因为 `Body::from_stream` 要求 `Stream<Item = Result<_, _>>`。

---

## 第二步：用 reqwest 调下游接口，把响应流转给客户端

加一个 `/` handler：用 reqwest 调本机 `/stream`（第一步那个），把 reqwest 的响应体**原样流式转发**回客户端。

````rust
use axum::{
    body::Body,
    extract::State,
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::get,
    Router,
};
use reqwest::Client;

# #[tokio::main]
# async fn main() {
#     // ...
    let client = Client::new();

    let app = Router::new()
        .route("/", get(stream_reqwest_response))
        .route("/stream", get(stream_some_data))
        .with_state(client);
#     // ...
# }

async fn stream_reqwest_response(State(client): State<Client>) -> Response {
    // 调下游接口
    let reqwest_response = match client.get("http://127.0.0.1:3000/stream").send().await {
        Ok(res) => res,
        Err(err) => {
            tracing::error!(%err, "request failed");
            return (StatusCode::BAD_REQUEST, Body::empty()).into_response();
        }
    };

    // 复制下游的状态码和 headers
    let mut response_builder = Response::builder().status(reqwest_response.status());
    *response_builder.headers_mut().unwrap() = reqwest_response.headers().clone();

    // 把下游的响应体作为流转给客户端
    response_builder
        .body(Body::from_stream(reqwest_response.bytes_stream()))
        .unwrap()
}
````

验证：

````bash
curl http://127.0.0.1:3000/
# 同样看到 0 1 2 3 4 每秒一个，但这次是从 /stream 流式转过来的
````

> **新面孔：`reqwest::Client` 作为 State**
>
> reqwest 是 HTTP 客户端库（第 39 章 OAuth 也用过）。`Client::new()` 创建实例（内部有连接池，**必须复用**，不要每次请求 `new` 一次）。
>
> 作为 State 注入：`Router::with_state(client)` + `State(client): State<Client>` 取出。和 ch18 依赖注入同构。

> **新面孔：`reqwest_response.bytes_stream()`**
>
> reqwest 的响应体是流式的，`.bytes_stream()` 返回 `Stream<Item = Result<Bytes, reqwest::Error>>`。直接传给 `Body::from_stream` 完成流式转发——**下游一段段产，我们一段段转发给客户端，不缓冲**。

> **新面孔：复制 Response 头和状态码**
>
> `Response::builder().status(reqwest_response.status())` 复制状态码，`*headers_mut() = reqwest_response.headers().clone()` 复制 headers。这样转发响应保留所有元数据（content-type、缓存头等）。

### 为什么这是反向代理的核心

```text
客户端 ──请求──> axum server ──reqwest──> 下游 API
                  ↑                          │
                  └──流式转发响应<──────────┘
```

下游响应体**不被完整缓冲**——一段段产一段段转发。下游慢/响应大也不爆内存。生产场景：API gateway、CDN origin、跨域代理。

---

## 第三步：用 `TraceLayer::on_body_chunk` 观察流式行为

加 `TraceLayer` 观察每个 chunk 流过——这是验证流式行为的最佳方式（看到日志一段段出就说明流式生效）。

````rust
use axum::body::Bytes;
use tower_http::trace::TraceLayer;
use tracing::Span;

# #[tokio::main]
# async fn main() {
#     // ...
    let app = Router::new()
        .route("/", get(stream_reqwest_response))
        .route("/stream", get(stream_some_data))
        .layer(
            TraceLayer::new_for_http().on_body_chunk(
                |chunk: &Bytes, _latency: Duration, _span: &Span| {
                    tracing::debug!("streaming {} bytes", chunk.len());
                },
            ),
        )
        .with_state(client);
#     // ...
# }
````

> **新面孔：`TraceLayer::on_body_chunk`**
>
> Tower-http 的 TraceLayer 提供多个钩子（`on_request`/`on_response`/`on_failure`/`on_body_chunk`/`on_eos`）。`on_body_chunk` 在每个响应体 chunk 发出时调用——这章用它打日志验证"流式"。
>
> 详细 TraceLayer 配置见第 22 章 tracing-aka-logging。

验证（看日志）：

````bash
RUST_LOG=example_reqwest_response=debug,tower_http=debug cargo run -p example-reqwest-response
# 另一个终端
curl http://127.0.0.1:3000/
````

服务端日志每隔 1 秒看到一行 `streaming 1 bytes`（数字长度），共 5 行——证明响应是流式发出的，不是一次性的。

---

## 完整代码

````rust
use axum::{
    body::{Body, Bytes},
    extract::State,
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::get,
    Router,
};
use reqwest::Client;
use std::{convert::Infallible, time::Duration};
use tokio_stream::StreamExt;
use tower_http::trace::TraceLayer;
use tracing::Span;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let client = Client::new();

    let app = Router::new()
        .route("/", get(stream_reqwest_response))
        .route("/stream", get(stream_some_data))
        // Add some logging so we can see the streams going through
        .layer(TraceLayer::new_for_http().on_body_chunk(
            |chunk: &Bytes, _latency: Duration, _span: &Span| {
                tracing::debug!("streaming {} bytes", chunk.len());
            },
        ))
        .with_state(client);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn stream_reqwest_response(State(client): State<Client>) -> Response {
    let reqwest_response = match client.get("http://127.0.0.1:3000/stream").send().await {
        Ok(res) => res,
        Err(err) => {
            tracing::error!(%err, "request failed");
            return (StatusCode::BAD_REQUEST, Body::empty()).into_response();
        }
    };

    let mut response_builder = Response::builder().status(reqwest_response.status());
    *response_builder.headers_mut().unwrap() = reqwest_response.headers().clone();
    response_builder
        .body(Body::from_stream(reqwest_response.bytes_stream()))
        // This unwrap is fine because the body is empty here
        .unwrap()
}

async fn stream_some_data() -> Body {
    let stream = tokio_stream::iter(0..5)
        .throttle(Duration::from_secs(1))
        .map(|n| n.to_string())
        .map(Ok::<_, Infallible>);
    Body::from_stream(stream)
}
````

## 运行

````bash
cd examples
cargo run -p example-reqwest-response
````

````bash
curl http://127.0.0.1:3000/stream    # 直接流式（每秒 1 字符）
curl http://127.0.0.1:3000/          # 通过 reqwest 中转的流式（同样每秒）
````

服务端日志每隔 1 秒看到 `streaming N bytes`，证明流式生效。

## 解读

### 缓冲响应 vs 流式响应

| 维度 | 缓冲响应（前面章节） | 流式响应（这章） |
| --- | --- | --- |
| 数据产出 | 一次性算完整 | 一段段产 |
| 内存占用 | 整个 body 进内存 | 一 chunk 大小 |
| 首字节时间（TTFB） | 慢（要算完） | 快（数据来了就发） |
| 适用场景 | JSON/HTML 响应 | 大文件、实时数据、代理 |

### `Body` 的多种来源

```text
Body::empty()                   空响应
Body::from_string(s)            字符串
Body::from_bytes(b)             bytes
Body::from_stream(stream)       任意 Stream<Item = Result<Bytes, _>>
```

`Body::from_stream` 是最通用的——其他都是它的特化。

## 常见问题

**为什么要流式转发？** 不流式的话下游响应整段进 axum 内存再发给客户端——下游大文件就 OOM。流式一段段中转，内存恒定。

**`reqwest::Client` 能复用吗？** 应该复用——内部有连接池，每次 `Client::new()` 会建新池、新 DNS、新 TLS context，慢且浪费。

**`on_body_chunk` 性能影响大吗？** 不大——只在每个 chunk 调一次闭包，tracing span 也是延迟构造的。生产可保留。

## 手写任务

1. 改 `stream_some_data` 成无限流（`tokio_stream::iter` 换成 `tokio::time::interval`），观察客户端长时间收到数据。
2. 转发 handler 加上错误重试（下游 5xx 重试一次）。
3. 加个聚合 handler：同时调多个下游接口，把所有响应拼成一个流发回。
4. 加 `Content-Type: text/event-stream`，把 `/stream` 变成 SSE 接口（参考 ch40）。

## 小结

这章用 3 步讲了流式响应 + reqwest 转发：

1. **流式基础**：`Body::from_stream(stream)` 把任意 Stream 转成响应体，数据一段段产不发。
2. **reqwest 转发**：`reqwest::Client` 调下游，`.bytes_stream()` 拿流式 body，直接 `Body::from_stream` 中转——反向代理的核心模式。
3. **观察流式**：`TraceLayer::on_body_chunk` 在每个 chunk 打日志，验证流式行为。

核心：**`Body::from_stream` 是 axum 流式响应的通用接口**——reqwest 响应、文件流、SSE、WebSocket 都通过它接入。reqwest 转发是反向代理最简实现。

## 源码对照

- `examples/reqwest-response/Cargo.toml`
- `examples/reqwest-response/src/main.rs`
