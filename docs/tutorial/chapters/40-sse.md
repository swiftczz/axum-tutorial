# 40. sse

对应示例：`examples/sse`

WebSocket（ch41）是双向通信。**Server-Sent Events（SSE）** 是单向的——server 向 client 推流式事件，client 不能回消息。比 WebSocket 简单：基于 HTTP、不需要 upgrade、自带断线重连。适合**实时通知、消息推送、流式响应**（如 ChatGPT 的逐字输出）。

分 2 步：先写最简 SSE handler（每秒推一个 "hi!"），再加 `keep_alive` 防止代理超时断连。

相比前面章节新引入：**`Sse<Stream>` 响应类型、`Event` 数据结构、`stream::repeat_with` + `throttle` 造流、`KeepAlive` 心跳**。

## Cargo.toml

````toml
[package]
name = "example-sse"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["macros"] }
axum-extra = { version = "0.12", features = ["typed-header"] }
futures-util = { version = "0.3", default-features = false, features = ["std"] }
headers = "0.4"
tokio = { version = "1.0", features = ["full"] }
tokio-stream = "0.1"
tower-http = { version = "0.6", features = ["fs", "trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

[dev-dependencies]
eventsource-stream = "0.2"
reqwest = { version = "0.12", default-features = false, features = ["rustls-tls"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：最简 SSE——每秒推一个事件

SSE handler 返回 `Sse<Stream<Item = Result<Event, _>>>`——一个能不断 yield `Event` 的 stream。axum 把 stream 里的每个 event 编码成 SSE 格式（`data: xxx\n\n`）发给客户端。

````rust
use axum::{
    response::sse::{Event, Sse},
    routing::get,
    Router,
};
use axum_extra::TypedHeader;
use futures_util::stream::{self, Stream};
use std::{convert::Infallible, time::Duration};
use tokio_stream::StreamExt as _;

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(/* tracing 初始化 */)
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app = Router::new().route("/sse", get(sse_handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn sse_handler(
    TypedHeader(user_agent): TypedHeader<headers::UserAgent>,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    println!("`{}` connected", user_agent.as_str());

    // 一个每秒 yield Event 的 stream
    let stream = stream::repeat_with(|| Event::default().data("hi!"))
        .map(Ok)
        .throttle(Duration::from_secs(1));

    Sse::new(stream)
}
````

验证：

````bash
cd examples
cargo run -p example-sse

curl -N http://127.0.0.1:3000/sse
# data: hi!
#
# data: hi!
#
# data: hi!
# ...（每秒一行）
````

`-N` 禁用 curl 缓冲，立刻看到流式输出。

> **新面孔：`Sse<Stream>`**
>
> axum 的 SSE 响应类型。`Sse::new(stream)` 包一个 `Stream<Item = Result<Event, _>>`，axum 自动把每个 event 编码成 SSE 文本格式（`data: <content>\n\n`）发给客户端。
>
> SSE 的 `Content-Type: text/event-stream` 是 axum 自动设的。

> **新面孔：`Event`**
>
> SSE 事件数据结构。`Event::default().data("hi!")` 创建一个带 data 字段的事件。其他字段：
> - `.event("login")`：事件类型（client 用 `addEventListener("login", ...)`）
> - `.id("123")`：事件 ID（断线重连用 `Last-Event-ID` header 续传）
> - `.retry(5000)`：client 重连间隔（毫秒）

> **新面孔：`stream::repeat_with` + `throttle`**
>
> `futures_util::stream` 提供的 stream 工厂。`repeat_with(closure)` 无限调闭包产 item；`.throttle(duration)` 每 duration 才放一个 item 通过。
>
> 这章组合成"每秒产一个 Event"的无限 stream。生产环境换成真实数据源（如 `tokio::sync::broadcast` 接收端、数据库变更流等）。

> **新面孔：`TypedHeader<headers::UserAgent>`**
>
> 类型安全提取 User-Agent 头。`headers::UserAgent` 来自 `headers` crate，`TypedHeader<UserAgent>` 自动解析和验证。和 ch39 OAuth 用的 `TypedHeader<headers::Cookie>` 同款机制。

---

## 第二步：加 `KeepAlive` 防止代理超时断连

很多反向代理（Nginx 默认 60s）会**关闭空闲连接**——如果 SSE 一段时间没事件，代理以为连接死了就关掉。`KeepAlive` 周期发心跳注释（`: keep-alive-text\n\n`），保持连接"看起来活跃"。

````rust
use axum::response::sse::KeepAlive;

async fn sse_handler(
    TypedHeader(user_agent): TypedHeader<headers::UserAgent>,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    println!("`{}` connected", user_agent.as_str());

    let stream = stream::repeat_with(|| Event::default().data("hi!"))
        .map(Ok)
        .throttle(Duration::from_secs(1));

    Sse::new(stream).keep_alive(
        KeepAlive::new()
            .interval(Duration::from_secs(1))
            .text("keep-alive-text"),
    )
}
````

> **新面孔：`KeepAlive` 心跳**
>
> `Sse::new(stream).keep_alive(...)` 周期发心跳。源 stream 没事件时，`KeepAlive` 自动发一个注释（`: keep-alive-text\n\n`，以 `:` 开头是 SSE 注释客户端忽略）。代理看到流量就不会断。
>
> - `.interval(Duration)`：心跳间隔
> - `.text(...)`：注释内容
>
> 不加 keep_alive 的话，如果业务事件稀疏（如每小时一次），连接可能在事件之间被代理关闭。

### 测试 SSE

测试 SSE 不能用普通 reqwest（要解析 SSE 流）。用 `eventsource-stream` crate 把 reqwest 的 bytes_stream 转成 SSE event stream：

````rust
#[cfg(test)]
mod tests {
    use super::*;
    use eventsource_stream::Eventsource;
    use tokio_stream::StreamExt;

    #[tokio::test]
    async fn integration_test() {
        // 启动 app 在随机端口
        let listener = tokio::net::TcpListener::bind("127.0.0.1:0").await.unwrap();
        let port = listener.local_addr().unwrap().port();
        tokio::spawn(async { axum::serve(listener, app()).await.unwrap(); });

        let mut event_stream = reqwest::Client::new()
            .get(format!("http://127.0.0.1:{port}/sse"))
            .header("User-Agent", "test")
            .send().await.unwrap()
            .bytes_stream()
            .eventsource()
            .take(1);  // 只取第一个事件

        let event_data: Vec<String> = ...;
        assert!(event_data[0] == "hi!");
    }
}
````

---

## 完整代码

````rust
use axum::{
    response::sse::{Event, Sse},
    routing::get,
    Router,
};
use axum_extra::TypedHeader;
use futures_util::stream::{self, Stream};
use std::{convert::Infallible, path::PathBuf, time::Duration};
use tokio_stream::StreamExt as _;
use tower_http::{services::ServeDir, trace::TraceLayer};
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

    // build our application
    let app = app();

    // run it
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}

fn app() -> Router {
    let assets_dir = PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("assets");
    let static_files_service = ServeDir::new(assets_dir).append_index_html_on_directories(true);
    // build our application with a route
    Router::new()
        .fallback_service(static_files_service)
        .route("/sse", get(sse_handler))
        .layer(TraceLayer::new_for_http())
}

async fn sse_handler(
    TypedHeader(user_agent): TypedHeader<headers::UserAgent>,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    println!("`{}` connected", user_agent.as_str());

    // A `Stream` that repeats an event every second
    //
    // You can also create streams from tokio channels using the wrappers in
    // https://docs.rs/tokio-stream
    let stream = stream::repeat_with(|| Event::default().data("hi!"))
        .map(Ok)
        .throttle(Duration::from_secs(1));

    Sse::new(stream).keep_alive(
        axum::response::sse::KeepAlive::new()
            .interval(Duration::from_secs(1))
            .text("keep-alive-text"),
    )
}
````

## 运行

````bash
cd examples
cargo run -p example-sse

curl -N http://127.0.0.1:3000/sse
````

或浏览器打开 `http://127.0.0.1:3000/`（自带前端 JS 用 `EventSource` 接收），看到 `hi!` 每秒出现。

## 解读

### SSE vs WebSocket vs 流式 HTTP

| 维度 | SSE | WebSocket（ch41） | 流式 HTTP（ch15） |
| --- | --- | --- | --- |
| 协议 | HTTP/1.1 或 HTTP/2 | HTTP upgrade 后切 WebSocket | 普通 HTTP |
| 方向 | server → client 单向 | 双向 | server → client 单向 |
| 重连 | 浏览器自动重连（Last-Event-ID 续传） | 手动实现 | 不支持 |
| 代理穿透 | 好（就是 HTTP） | 需代理支持 upgrade | 好 |
| 适合 | 通知、推送、流式响应 | 聊天、协作编辑 | 大文件下载 |

**优先选 SSE**：简单（纯 HTTP）、自带重连、代理友好。只有需要双向才用 WebSocket。

### SSE 的浏览器 API

```javascript
const es = new EventSource("/sse");
es.onmessage = (e) => console.log(e.data);          // 默认事件
es.addEventListener("login", (e) => {               // 自定义事件类型
    console.log("logged in:", e.data);
});
es.onerror = () => { /* 浏览器自动重连 */ };
```

浏览器原生支持，不需要任何库。

## 常见问题

**SSE 和 WebSocket 哪个简单？** SSE 简单得多——纯 HTTP、不需要握手升级、浏览器自带重连。单向推送场景必选 SSE。

**`Infallible` 错误类型是什么意思？** "永不失败"——这章 stream 不会产错误。如果业务 stream 可能失败（如数据库连接断），用真实错误类型。

**为什么用 `tokio_stream::StreamExt as _`？** 防 trait 名冲突——`futures_util::StreamExt` 和 `tokio_stream::StreamExt` 都叫 `StreamExt`，`as _` 导入但不绑名字（只为了它的方法在作用域内）。

## 手写任务

1. 改 stream 从 `tokio::sync::broadcast` 接收消息（参考 ch43）。
2. 加事件类型：`.event("notification")`，前端 `addEventListener("notification", ...)`。
3. 加断线重连：handler 读 `Last-Event-ID` header，从该 ID 之后开始推。
4. 改成"流式 ChatGPT"——每秒产一个字符，最后发 `[DONE]`。

## 小结

这章用 2 步讲了 SSE：

1. **`Sse<Stream>` 响应**：handler 返回 `Sse::new(stream)`，stream 不断 yield `Event`，axum 自动编码成 SSE 格式。
2. **`KeepAlive` 心跳**：防止代理超时断连，业务事件稀疏时必备。

核心：SSE 是**单向推送**的简单方案——纯 HTTP、浏览器自带重连、代理友好。优先选 SSE 而非 WebSocket。

## 源码对照

- `examples/sse/Cargo.toml`
- `examples/sse/src/main.rs`
- `examples/sse/assets/index.html`
