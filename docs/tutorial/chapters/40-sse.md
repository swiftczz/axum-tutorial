# 40. sse

对应示例：`examples/sse`

SSE(Server-Sent Events)适合服务端持续推送消息给浏览器。用 axum 的 `Sse` 返回服务端事件流,理解 EventSource、Stream、keep-alive、SSE 集成测试。



相比前面章节新引入：**SSE（Server-Sent Events）、`Sse<Stream>`、`Event::default().data(...)`、`keep_alive`**。

## Cargo.toml

````toml
[package]
name = "example-sse"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
axum-extra = { version = "0.12", features = ["typed-header"] }
futures-util = { version = "0.3", default-features = false, features = ["sink", "std"] }
headers = "0.4"
tokio = { version = "1.0", features = ["full"] }
tokio-stream = "0.1"
tower-http = { version = "0.6.1", features = ["fs", "trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

[dev-dependencies]
eventsource-stream = "0.2"
reqwest = { version = "0.12", default-features = false, features = ["stream"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 关键概念

> **新面孔：SSE（Server-Sent Events）**
>
> 服务端→客户端单向推送（WebSocket 是双向）。`Sse<Stream>` 把 Rust stream 变成 SSE 响应。`keep_alive` 防长连接被代理断开。


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

    let app = app();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

fn app() -> Router {
    let assets_dir = PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("assets");
    let static_files_service = ServeDir::new(assets_dir).append_index_html_on_directories(true);

    Router::new()
        .fallback_service(static_files_service)
        .route("/sse", get(sse_handler))
        .layer(TraceLayer::new_for_http())
}

async fn sse_handler(
    TypedHeader(user_agent): TypedHeader<headers::UserAgent>,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    println!("`{}` connected", user_agent.as_str());

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

## assets/script.js

````javascript
var eventSource = new EventSource('sse');

eventSource.onmessage = function(event) {
    console.log('Message from server ', event.data);
}
````

## 运行

````bash
cd examples
cargo run -p example-sse
````

浏览器打开 `http://127.0.0.1:3000/`,Console 每秒看到:

````text
Message from server hi!
````

用 curl 看原始事件流(注意 `-N` 不缓冲):

````bash
curl -N http://127.0.0.1:3000/sse
````

运行测试:

````bash
cargo test -p example-sse
````

## 解读

### SSE vs 普通响应

```text
普通响应:请求 → 服务端一次性返回 → 连接结束
SSE:     请求 → 服务端返回事件流 → 连接保持打开 → 不断发送事件
```

SSE 是**单向**的(服务端 → 浏览器)。需要双向实时通信看 WebSocket(41 章起);只需服务端持续推送(通知/进度/日志/状态更新),SSE 更简单。

### 静态文件 + SSE 路由

````rust
fn app() -> Router {
    let assets_dir = PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("assets");
    let static_files_service = ServeDir::new(assets_dir).append_index_html_on_directories(true);

    Router::new()
        .fallback_service(static_files_service)   // 静态文件兜底
        .route("/sse", get(sse_handler))           // SSE 事件流
        .layer(TraceLayer::new_for_http())
}
````

`CARGO_MANIFEST_DIR` 是当前 crate 目录,比直接写 `"assets"` 稳定不受运行目录影响。`fallback_service` 表示没匹配到路由就去静态目录找文件(见 29 章 ServeDir)。

### 前端 EventSource

浏览器原生支持 `EventSource`,自动向 `/sse` 建立 SSE 连接,服务端发事件后触发 `onmessage`,`event.data` 是事件数据。

### SSE handler 返回类型

````rust
async fn sse_handler(
    TypedHeader(user_agent): TypedHeader<headers::UserAgent>,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
````

返回类型拆开:

```text
Sse<...>                          axum SSE 响应
Stream<Item = Result<Event, _>>   不断产出 SSE Event 的流
Event                             单个 SSE 事件
Infallible                        这个 stream 不会产生错误
```

`TypedHeader(user_agent)` 从请求头提取 User-Agent,只为打印谁连接了。

### 每秒一次的事件流

````rust
let stream = stream::repeat_with(|| Event::default().data("hi!"))
    .map(Ok)
    .throttle(Duration::from_secs(1));
````

- `stream::repeat_with(|| Event::default().data("hi!"))`:无限 stream,不断生成 `data: hi!` 事件。
- `.map(Ok)`:把 `Event` 包成 `Result<Event, Infallible>`。
- `.throttle(Duration::from_secs(1))`:每秒最多发一次。

客户端每秒收到一个 `hi!`。

### `Sse::new` + `keep_alive`

````rust
Sse::new(stream).keep_alive(
    KeepAlive::new()
        .interval(Duration::from_secs(1))
        .text("keep-alive-text"),
)
````

`Sse::new(stream)` 把 stream 包成 SSE HTTP 响应。`keep_alive` 定期发心跳避免连接长期空闲被代理/浏览器断开——长连接长时间没数据可能被浏览器/代理/负载均衡断开。真实项目间隔按代理/网关/业务调整。

### 测试如何读取 SSE

测试启动应用到随机端口(`bind("{host}:0}")` 让系统分配端口,避免固定端口冲突),用 reqwest 请求 `/sse`:

````rust
let mut event_stream = reqwest::Client::new()
    .get(format!("{listening_url}/sse"))
    .send().await.unwrap()
    .bytes_stream()
    .eventsource()    // 把字节流解析成 SSE event
    .take(1);         // 只取一个事件,避免测试一直等无限流
````

`eventsource()` 把字节流解析成 SSE event,`.take(1)` 只取一个。最后断言 `event_data[0] == "hi!"`。

## 常见问题

**curl 为什么加 `-N`?** `-N` 不缓冲输出。SSE 是持续流,缓冲的话终端可能不立刻显示事件。

**SSE 能从浏览器发消息到服务端吗?** 不能,SSE 是服务端到客户端单向推送。客户端发消息用普通 HTTP 请求或改 WebSocket。

**为什么需要 keep-alive?** 长连接长时间没数据可能被浏览器/代理/负载均衡断开,keep-alive 维持连接活跃。

**stream 会结束吗?** `repeat_with` 创建无限 stream。真实项目按业务事件/channel 关闭/用户断开来结束。

**SSE 适合什么场景?** 通知、任务进度、日志流、状态更新、只需服务端推送的实时数据。不适合强双向通信。

## 手写任务

按下面顺序敲:

1. 写 `/sse` handler 返回 `Sse`。
2. `stream::repeat_with` 创建事件。
3. `.map(Ok)` 变成 `Result<Event, Infallible>`。
4. `.throttle(Duration::from_secs(1))` 控制频率。
5. 加 `keep_alive`。
6. 写 `index.html` 和 `script.js` 用 `EventSource` 连接。
7. 浏览器 Console 验证消息。

加深练习:

1. `hi!` 改成当前时间。
2. 给 Event 设置 event name 和 id。
3. 用 Tokio channel 代替 `repeat_with`,从后台任务推送消息。

## 小结

- SSE 核心模型:服务端返回不立刻结束的 HTTP 响应,响应体是事件流,浏览器用 EventSource 接收。
- axum 关键结构:`Sse<impl Stream<Item = Result<Event, Infallible>>>`,把 Rust stream 变成浏览器能理解的 Server-Sent Events。
- SSE 是单向(服务端 → 浏览器),适合推送通知/进度/日志/状态;双向实时通信用 WebSocket。
- `stream::repeat_with` + `.map(Ok)` + `.throttle` 产生节流事件流;`keep_alive` 发心跳防止长连接被代理断开。
- 测试用 `eventsource-stream` 把 reqwest 字节流解析成 SSE event,`.take(1)` 避免等无限流。

## 源码对照

- `examples/sse/Cargo.toml`
- `examples/sse/src/main.rs`
- `examples/sse/assets/index.html`
- `examples/sse/assets/script.js`
