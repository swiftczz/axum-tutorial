# 15. reqwest-response

对应示例：`examples/reqwest-response`

本章目标：理解如何把外部 HTTP 响应转成 Axum 响应，并用流式 body 转发数据。

这一章开始接近“代理”类场景：你的 Axum 服务收到请求后，再用 `reqwest` 去请求另一个 HTTP 服务，然后把对方的响应转发给客户端。

## 这个小项目在做什么

这个 example 在同一个 Axum 服务里提供两个接口：

```text
GET /stream
-> 慢慢产生 0、1、2、3、4 这几段数据

GET /
-> 用 reqwest 请求 http://127.0.0.1:3000/stream
-> 把 reqwest 的响应状态码、headers、body 转成 Axum Response
```

请求主线是：

```text
客户端 GET /
-> stream_reqwest_response
-> reqwest Client 请求本服务的 /stream
-> 拿到 reqwest::Response
-> 构造 axum::response::Response
-> 用 Body::from_stream 转发字节流
```

这个 example 的重点不是业务数据，而是响应转换：

```text
reqwest::Response -> axum::Response
```

## 先理解为什么要流式转发

如果外部服务返回很大的响应，最直接但不理想的做法是：

```text
先把外部响应完整读进内存
再一次性返回给客户端
```

这样会占用更多内存，也会让客户端等更久。

流式转发的思路是：

```text
外部响应每到一块 bytes
-> Axum 就往客户端发一块
```

这适合：

- 大文件下载。
- 代理接口。
- SSE 或长响应。
- 需要边接收边转发的场景。

本章示例用 `/stream` 每秒产出一小段数据，方便观察流式效果。

## 文件和依赖

这个 example 有两个文件：

1. `examples/reqwest-response/Cargo.toml`：声明 Axum、reqwest、tokio-stream、tower-http 等依赖。
2. `examples/reqwest-response/src/main.rs`：实现 reqwest client state、流式响应和响应转换。

关键依赖：

- `axum`：提供 `State`、`Body`、`Response`、`IntoResponse`。
- `reqwest`：作为 HTTP 客户端请求外部服务。
- `tokio-stream`：创建节流 stream。
- `tower-http`：提供 trace layer，观察 body chunk。
- `tokio`、`tracing`、`tracing-subscriber`：运行时和日志。

`reqwest` 依赖启用了 `stream` feature：

````toml
reqwest = { version = "0.12", default-features = false, features = ["stream"] }
````

因为本章要用：

````rust
reqwest_response.bytes_stream()
````

## 第一步：创建 reqwest Client 并放入 state

源码：

````rust
let client = Client::new();

let app = Router::new()
    .route("/", get(stream_reqwest_response))
    .route("/stream", get(stream_some_data))
    .with_state(client);
````

`Client` 是 reqwest 的 HTTP 客户端。  
它适合复用，不应该每个请求都重新创建。

所以这里用：

```text
with_state(client)
```

把它放进 Axum 应用状态里。

handler 里通过：

````rust
State(client): State<Client>
````

把它取出来使用。

这是本教程第一次正式使用 `State<T>`，先记住：

```text
State 用来把应用共享对象传给 handler。
```

## 第二步：注册两个接口和 trace layer

源码：

````rust
let app = Router::new()
    .route("/", get(stream_reqwest_response))
    .route("/stream", get(stream_some_data))
    .layer(TraceLayer::new_for_http().on_body_chunk(
        |chunk: &Bytes, _latency: Duration, _span: &Span| {
            tracing::debug!("streaming {} bytes", chunk.len());
        },
    ))
    .with_state(client);
````

两个接口：

| 路径 | handler | 作用 |
| --- | --- | --- |
| `/` | `stream_reqwest_response` | 请求 `/stream` 并转发响应 |
| `/stream` | `stream_some_data` | 产生慢速流式数据 |

`TraceLayer` 这里加了一个 `on_body_chunk` 回调。  
每当响应 body 发出一个 chunk，就打印：

```text
streaming N bytes
```

这方便观察流式传输，而不是等全部数据准备好后一次性返回。

## 第三步：用 reqwest 请求 /stream

源码：

````rust
async fn stream_reqwest_response(State(client): State<Client>) -> Response {
    let reqwest_response = match client.get("http://127.0.0.1:3000/stream").send().await {
        Ok(res) => res,
        Err(err) => {
            tracing::error!(%err, "request failed");
            return (StatusCode::BAD_REQUEST, Body::empty()).into_response();
        }
    };

    ...
}
````

这里 handler 先从 state 里拿到 `Client`，然后发起 HTTP 请求。

如果请求失败，返回：

```text
400 Bad Request
empty body
```

示例里请求的是同一个服务的 `/stream`。  
真实项目中也可以是其他服务：

```text
http://internal-service/users
https://api.example.com/data
```

## 第四步：复制状态码和响应头

源码：

````rust
let mut response_builder = Response::builder().status(reqwest_response.status());
*response_builder.headers_mut().unwrap() = reqwest_response.headers().clone();
````

`reqwest_response.status()` 是外部响应的状态码。  
`reqwest_response.headers()` 是外部响应头。

这两行的意思是：

```text
外部服务返回什么状态码，Axum 就返回什么状态码。
外部服务返回什么 headers，Axum 就先复制一份。
```

真实代理场景里要谨慎转发 headers。  
有些 header 可能不应该原样转发，比如 hop-by-hop headers。  
这个 example 为了演示核心思路，直接 clone。

## 第五步：把 reqwest bytes_stream 变成 Axum Body

源码：

````rust
response_builder
    .body(Body::from_stream(reqwest_response.bytes_stream()))
    .unwrap()
````

这是本章最关键的一行。

`reqwest_response.bytes_stream()` 会产出一段段 `Bytes`。  
`Body::from_stream(...)` 把这个 stream 包装成 Axum 可以返回的 response body。

所以数据流向是：

```text
reqwest response body stream
-> Body::from_stream
-> axum Response body
-> 客户端
```

这里没有把外部响应完整读进内存。

## 第六步：生成一个慢速 stream

源码：

````rust
async fn stream_some_data() -> Body {
    let stream = tokio_stream::iter(0..5)
        .throttle(Duration::from_secs(1))
        .map(|n| n.to_string())
        .map(Ok::<_, Infallible>);
    Body::from_stream(stream)
}
````

这段产生 5 个数据：

```text
0
1
2
3
4
```

`throttle(Duration::from_secs(1))` 让它每秒产出一个。  
`map(|n| n.to_string())` 把数字变成字符串。  
`map(Ok::<_, Infallible>)` 把每个 item 包成 `Result`，表示这个 stream 不会失败。

最后用 `Body::from_stream(stream)` 作为响应 body。

## 函数职责速查

- `main`：初始化日志，创建 reqwest `Client`，注册路由、trace layer 和 state，启动服务。
- `stream_reqwest_response`：请求 `/stream`，把 reqwest 响应转成 Axum 响应。
- `stream_some_data`：生成一个慢速 body stream，方便观察流式转发。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-reqwest-response
//! ```

// 引入 Body、Bytes、State、状态码、响应转换、GET 路由和 Router。
use axum::{
    body::{Body, Bytes},
    extract::State,
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::get,
    Router,
};
// reqwest HTTP 客户端。
use reqwest::Client;
// Infallible 表示不会失败，Duration 表示时间间隔。
use std::{convert::Infallible, time::Duration};
// StreamExt 提供 throttle/map 等 stream 组合方法。
use tokio_stream::StreamExt;
// TraceLayer 用来观察 HTTP 请求和 body chunk。
use tower_http::trace::TraceLayer;
// Span 用在 on_body_chunk 回调参数里。
use tracing::Span;
// tracing_subscriber 用来初始化日志。
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 初始化日志系统。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 创建 reqwest client。Client 适合复用，所以放入 app state。
    let client = Client::new();

    // 注册两个路由，并加上 trace layer 和 state。
    let app = Router::new()
        .route("/", get(stream_reqwest_response))
        .route("/stream", get(stream_some_data))
        // 每发送一个 body chunk，就打印 chunk 大小。
        .layer(TraceLayer::new_for_http().on_body_chunk(
            |chunk: &Bytes, _latency: Duration, _span: &Span| {
                tracing::debug!("streaming {} bytes", chunk.len());
            },
        ))
        // 把 reqwest client 放进应用状态。
        .with_state(client);

    // 绑定本机 3000 端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 输出监听地址。
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动 HTTP 服务。
    axum::serve(listener, app).await;
}

// GET / 的 handler：请求 /stream，并把 reqwest 响应转成 Axum 响应。
async fn stream_reqwest_response(State(client): State<Client>) -> Response {
    // 用 reqwest 请求同一个服务里的 /stream。
    let reqwest_response = match client.get("http://127.0.0.1:3000/stream").send().await {
        Ok(res) => res,
        Err(err) => {
            // 请求失败时记录错误，并返回 400 空 body。
            tracing::error!(%err, "request failed");
            return (StatusCode::BAD_REQUEST, Body::empty()).into_response();
        }
    };

    // 创建 Axum response builder，并复用 reqwest 响应状态码。
    let mut response_builder = Response::builder().status(reqwest_response.status());
    // 复制 reqwest 响应 headers。
    *response_builder.headers_mut().unwrap() = reqwest_response.headers().clone();

    // 把 reqwest 的 bytes_stream 转成 Axum Body。
    response_builder
        .body(Body::from_stream(reqwest_response.bytes_stream()))
        // 这里 body 构造简单，所以 unwrap 可以接受。
        .unwrap()
}

// GET /stream 的 handler：每秒产生一个数字字符串。
async fn stream_some_data() -> Body {
    let stream = tokio_stream::iter(0..5)
        // 每秒产出一个 item。
        .throttle(Duration::from_secs(1))
        // 把数字转成字符串。
        .map(|n| n.to_string())
        // Body::from_stream 需要 Result item；这里用 Infallible 表示不会失败。
        .map(Ok::<_, Infallible>);

    // 把 stream 转成响应 body。
    Body::from_stream(stream)
}
````

## 运行和验证

运行前先确认 `examples/reqwest-response/Cargo.toml` 里的 `axum = { path = "../../axum" }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-reqwest-response
````

直接访问流：

````bash
curl -i http://127.0.0.1:3000/stream
````

访问转发接口：

````bash
curl -i http://127.0.0.1:3000/
````

两个请求都应该逐步收到 `0` 到 `4`。  
服务端日志里应该能看到每个 body chunk 的大小。

常见卡点：

- `Client` 应该复用，所以放进 `State<Client>`。
- `Body::from_stream` 可以把外部字节流变成 Axum 响应体。
- 示例直接复制 headers，真实代理要谨慎处理某些不该转发的 header。
- `/` 会请求同一个服务的 `/stream`，所以服务必须已经启动并监听 3000。

## 手写任务

完成本章代码后，试着做三个小改动：

1. 把 `/stream` 的 `0..5` 改成 `0..10`，观察响应时间变化。
2. 把 `throttle` 从 1 秒改成 200 毫秒，观察日志里的 chunk 输出频率。
3. 给 `/stream` 增加一个自定义 header，再观察 `/` 是否也转发了这个 header。

## 本章真正要记住什么

- Axum 服务也可以作为 HTTP 客户端请求其他服务。
- `State<T>` 用来把可复用对象传给 handler。
- `reqwest::Response` 不能直接作为 Axum response 返回，需要手动转换。
- 状态码、headers 和 body 都可以从 reqwest response 转到 Axum response。
- `Body::from_stream` 是流式转发的关键。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/reqwest-response/Cargo.toml`
- `examples/reqwest-response/src/main.rs`
