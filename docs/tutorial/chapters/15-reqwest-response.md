# 15. reqwest-response

对应示例：`examples/reqwest-response`

这一章接近"代理"场景:axum 服务收到请求后,用 `reqwest` 请求另一个 HTTP 服务,再把对方的响应**流式转发**给客户端。重点是响应转换 `reqwest::Response → axum::Response`,以及第一次正式使用 `State<T>`。



相比前面章节新引入：**`reqwest::Client` 放入 `State`、`Body::from_stream` 流式转发**。

## Cargo.toml

````toml
[package]
name = "example-reqwest-response"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
reqwest = { version = "0.12", default-features = false, features = ["stream"] }
tokio = { version = "1.0", features = ["full"] }
tokio-stream = "0.1"
tower-http = { version = "0.6.1", features = ["trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

`reqwest` 启用 `stream` feature 才能用 `bytes_stream()`。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 关键概念

> **新面孔：`reqwest::Client` + `State<Client>`**
>
> HTTP 客户端适合复用（内部维护连接池），放进 `State` 让所有 handler 共享。`State(client): State<Client>` 是解构语法。

> **新面孔：`Body::from_stream`**
>
> 流式转发的关键：reqwest 每到一块 bytes，axum 就往客户端发一块，不把整个响应读进内存。


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

    axum::serve(listener, app).await;
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

直接访问流:

````bash
curl -i http://127.0.0.1:3000/stream
````

访问转发接口:

````bash
curl -i http://127.0.0.1:3000/
````

两个请求都逐步收到 `0` 到 `4`,服务端日志能看到每个 body chunk 大小。

## 解读

### 两个接口

| 路径 | handler | 作用 |
| --- | --- | --- |
| `/` | `stream_reqwest_response` | 请求 `/stream` 并转发响应 |
| `/stream` | `stream_some_data` | 每秒产生一个数字,模拟慢速流 |

### 第一次用 `State<T>`

````rust
let client = Client::new();
let app = Router::new()
    .route("/", get(stream_reqwest_response))
    ...
    .with_state(client);
````

`reqwest::Client` 适合复用(内部维护连接池),不应该每个请求重建。用 `.with_state(client)` 放进应用状态,handler 通过 `State(client): State<Client>` 取出。

### `State(client): State<Client>` 是解构语法

第一次看到会疑惑为什么 `State` 出现两次。这是 Rust 函数参数模式匹配,不是 axum 特殊语法。等价写法:

````rust
// 写法 A(常见):参数里直接解构
async fn handler(State(client): State<Client>) { ... }

// 写法 B:显式取
async fn handler(state: State<Client>) {
    let client = state.0;  // State 是元组结构体,.0 取出里面的 Client
    ...
}
````

`State<T>` 是 axum 定义的元组结构体(`struct State<T>(pub T)`),包住你的共享对象。写 `State(client): State<Client>` 就是说"这个参数类型是 `State<Client>`,把它拆开,绑定到 `client`"。后面第 16 章开始会大量看到这种写法。

### 响应转换三步

````rust
let mut response_builder = Response::builder().status(reqwest_response.status());
*response_builder.headers_mut().unwrap() = reqwest_response.headers().clone();

response_builder
    .body(Body::from_stream(reqwest_response.bytes_stream()))
    .unwrap()
````

1. 复制状态码:外部返回什么状态码,axum 就返回什么。
2. 复制 headers:外部返回什么 headers,axum 先复制一份。真实代理要谨慎转发某些 header(比如 hop-by-hop headers),example 为了演示直接 clone。
3. **`Body::from_stream(reqwest_response.bytes_stream())`**:这是流式转发的关键。reqwest 每到一块 bytes,axum 就往客户端发一块,没有把整个响应读进内存。

数据流向:

```text
reqwest response body stream
  → Body::from_stream
  → axum Response body
  → 客户端
```

适合大文件下载、代理接口、SSE 或长响应。

### `TraceLayer::on_body_chunk`

````rust
.layer(TraceLayer::new_for_http().on_body_chunk(
    |chunk: &Bytes, _latency: Duration, _span: &Span| {
        tracing::debug!("streaming {} bytes", chunk.len());
    },
))
````

每发送一个 body chunk 就打印大小,方便观察流式传输。

### `stream_some_data` 模拟慢速流

````rust
let stream = tokio_stream::iter(0..5)
    .throttle(Duration::from_secs(1))  // 每秒产出一个
    .map(|n| n.to_string())
    .map(Ok::<_, Infallible>);          // Body::from_stream 要 Result item
````

`Infallible` 表示这个 stream 不会失败。

## 手写任务

跑通后做三个小改动:

1. 把 `/stream` 的 `0..5` 改成 `0..10`,观察响应时间变化。
2. 把 `throttle` 从 1 秒改成 200 毫秒,观察日志里 chunk 输出频率。
3. 给 `/stream` 增加一个自定义 header,观察 `/` 是否也转发了这个 header。

## 小结

- axum 服务也能作为 HTTP 客户端请求其他服务。
- `State<T>` 用来把可复用对象(如 reqwest Client)传给 handler,`State(client): State<Client>` 是解构语法。
- `reqwest::Response` 不能直接作为 axum response,要手动转换状态码、headers、body。
- `Body::from_stream` 是流式转发的关键,外部每到一块 bytes 就转发一块,不读进内存。

## 源码对照

- `examples/reqwest-response/Cargo.toml`
- `examples/reqwest-response/src/main.rs`
