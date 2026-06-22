# 40. sse

对应示例：`examples/sse`

本章目标：使用 Axum 的 `Sse` 返回服务端事件流，理解 EventSource、Stream、keep-alive 和 SSE 集成测试。

SSE 全称是 Server-Sent Events。  
它适合服务端持续推送消息给浏览器。

## 这个小项目在做什么

应用做两件事：

```text
GET /    -> 返回静态页面 assets/index.html
GET /sse -> 每秒推送一次 SSE 消息 hi!
```

浏览器页面里的 JavaScript：

````javascript
var eventSource = new EventSource('sse');

eventSource.onmessage = function(event) {
    console.log('Message from server ', event.data);
}
````

请求主线是：

```text
浏览器打开 /
-> 加载 script.js
-> script.js 创建 EventSource('sse')
-> 浏览器请求 /sse
-> 服务端保持 HTTP 连接不断开
-> 每秒发送一个 data: hi!
-> 浏览器 onmessage 收到消息
```

## 先理解 SSE 和普通响应的区别

普通 HTTP 响应通常是：

```text
请求
-> 服务端一次性返回响应
-> 连接结束
```

SSE 是：

```text
请求
-> 服务端返回事件流
-> 连接保持打开
-> 服务端不断发送事件
```

它是单向的：

```text
服务端 -> 浏览器
```

如果你需要浏览器和服务端双向实时通信，通常看 WebSocket。  
如果你只需要服务端持续推送通知、进度、日志、状态更新，SSE 更简单。

## 文件和依赖

这个 example 有四个主要文件：

1. `examples/sse/Cargo.toml`：声明 Axum、axum-extra、futures-util、tokio-stream、tower-http。
2. `examples/sse/src/main.rs`：实现 SSE handler、静态文件服务和集成测试。
3. `examples/sse/assets/index.html`：加载前端 JS。
4. `examples/sse/assets/script.js`：创建 `EventSource` 并打印服务端消息。

关键依赖：

- `axum`：提供 `Sse`、`Event`、Router。
- `futures-util`：创建 stream，并使用 `map`。
- `tokio-stream`：提供 `throttle`，控制事件发送频率。
- `axum-extra` / `headers`：提取 `User-Agent`。
- `tower-http`：提供静态文件服务和 trace 日志。
- `eventsource-stream` / `reqwest`：测试中读取 SSE 流。

## 第一步：静态文件服务

源码：

````rust
fn app() -> Router {
    let assets_dir = PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("assets");
    let static_files_service = ServeDir::new(assets_dir).append_index_html_on_directories(true);

    Router::new()
        .fallback_service(static_files_service)
        .route("/sse", get(sse_handler))
        .layer(TraceLayer::new_for_http())
}
````

这里先找到 assets 目录：

````rust
PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("assets")
````

`CARGO_MANIFEST_DIR` 是当前 crate 的目录。  
这样比直接写 `"assets"` 更稳定，不容易受运行目录影响。

`fallback_service(static_files_service)` 表示没有匹配到其他路由时，就去静态目录找文件。

## 第二步：前端 EventSource

`assets/script.js`：

````javascript
var eventSource = new EventSource('sse');

eventSource.onmessage = function(event) {
    console.log('Message from server ', event.data);
}
````

浏览器原生支持 `EventSource`。  
它会向 `/sse` 建立 SSE 连接。

服务端发送事件后，浏览器会触发：

````javascript
onmessage
````

`event.data` 就是服务端事件里的数据。

## 第三步：SSE handler 返回类型

源码：

````rust
async fn sse_handler(
    TypedHeader(user_agent): TypedHeader<headers::UserAgent>,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    ...
}
````

返回类型比较长，拆开看：

```text
Sse<...>                         -> Axum SSE 响应
Stream<Item = Result<Event, _>>   -> 不断产出 SSE Event 的流
Event                             -> 单个 SSE 事件
Infallible                        -> 这个 stream 不会产生错误
```

参数：

````rust
TypedHeader(user_agent): TypedHeader<headers::UserAgent>
````

从请求头提取浏览器 User-Agent，只是为了打印谁连接了。

## 第四步：创建每秒一次的事件流

源码：

````rust
let stream = stream::repeat_with(|| Event::default().data("hi!"))
    .map(Ok)
    .throttle(Duration::from_secs(1));
````

这段创建了一个无限 stream。

拆开看：

````rust
stream::repeat_with(|| Event::default().data("hi!"))
````

不断生成事件：

```text
data: hi!
```

````rust
.map(Ok)
````

把 `Event` 包成 `Result<Event, Infallible>`。

````rust
.throttle(Duration::from_secs(1))
````

限制每秒最多发一次。

所以客户端会每秒收到一个：

```text
hi!
```

## 第五步：返回 Sse 并设置 keep-alive

源码：

````rust
Sse::new(stream).keep_alive(
    axum::response::sse::KeepAlive::new()
        .interval(Duration::from_secs(1))
        .text("keep-alive-text"),
)
````

`Sse::new(stream)` 把 stream 包成 SSE HTTP 响应。

`keep_alive` 用来定期发送注释或心跳，避免连接长期空闲被代理或浏览器断开。

本例设置：

```text
每 1 秒发送 keep-alive-text
```

真实项目中 keep-alive 间隔可以根据代理、网关和业务需求调整。

## 第六步：测试如何读取 SSE

测试中先启动应用到随机端口：

````rust
let listener = TcpListener::bind(format!("{host}:0")).await.unwrap();
let port = listener.local_addr().unwrap().port();
tokio::spawn(async {
    axum::serve(listener, app()).await;
});
````

端口 `0` 表示让操作系统自动分配可用端口。  
这适合测试，避免固定端口冲突。

然后用 reqwest 请求 `/sse`：

````rust
let mut event_stream = reqwest::Client::new()
    .get(format!("{listening_url}/sse"))
    .header("User-Agent", "integration_test")
    .send()
    .await
    .unwrap()
    .bytes_stream()
    .eventsource()
    .take(1);
````

`eventsource()` 把字节流解析成 SSE event。  
`.take(1)` 只取一个事件，避免测试一直等无限流。

最后断言：

````rust
assert!(event_data[0] == "hi!");
````

## 函数职责速查

- `main`：初始化日志，创建 app，绑定端口并启动服务。
- `app`：配置静态文件 fallback、`/sse` 路由和 TraceLayer。
- `sse_handler`：创建每秒发送 `hi!` 的 SSE stream，并设置 keep-alive。
- `integration_test`：启动测试服务，请求 `/sse`，读取一个事件并断言内容。

## 带中文注释的手写版

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
    // 初始化日志。
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
    // 使用 CARGO_MANIFEST_DIR 定位 assets 目录，避免受当前工作目录影响。
    let assets_dir = PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("assets");
    let static_files_service = ServeDir::new(assets_dir).append_index_html_on_directories(true);

    Router::new()
        // 静态文件兜底，访问 / 会返回 index.html。
        .fallback_service(static_files_service)
        // SSE 事件流接口。
        .route("/sse", get(sse_handler))
        .layer(TraceLayer::new_for_http())
}

async fn sse_handler(
    TypedHeader(user_agent): TypedHeader<headers::UserAgent>,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    println!("`{}` connected", user_agent.as_str());

    // 创建一个无限 stream，每秒产出一个 data 为 hi! 的事件。
    let stream = stream::repeat_with(|| Event::default().data("hi!"))
        .map(Ok)
        .throttle(Duration::from_secs(1));

    // 把 stream 包成 SSE 响应，并设置 keep-alive。
    Sse::new(stream).keep_alive(
        axum::response::sse::KeepAlive::new()
            .interval(Duration::from_secs(1))
            .text("keep-alive-text"),
    )
}
````

前端脚本：

````javascript
var eventSource = new EventSource('sse');

eventSource.onmessage = function(event) {
    console.log('Message from server ', event.data);
}
````

## 运行和验证

运行：

````bash
cargo run -p example-sse
````

浏览器打开：

```text
http://127.0.0.1:3000/
```

打开浏览器 Console，应该每秒看到：

```text
Message from server hi!
```

也可以用 curl 看原始事件流：

````bash
curl -N http://127.0.0.1:3000/sse
````

运行测试：

````bash
cargo test -p example-sse
````

## 常见卡点

### 1. 为什么 curl 要加 -N？

`-N` 让 curl 不缓冲输出。  
SSE 是持续流，如果缓冲，终端可能不会立刻显示事件。

### 2. SSE 能从浏览器发消息到服务端吗？

不能。  
SSE 是服务端到客户端的单向推送。

客户端要发消息，仍然用普通 HTTP 请求，或者改用 WebSocket。

### 3. 为什么需要 keep-alive？

长连接如果长时间没有数据，可能被浏览器、代理、负载均衡断开。  
keep-alive 用来维持连接活跃。

### 4. 这个 stream 会结束吗？

不会。  
`repeat_with` 创建的是无限 stream。

真实项目里可以根据业务事件、channel 关闭或用户断开来结束 stream。

### 5. SSE 适合什么场景？

适合：

```text
通知
任务进度
日志流
状态更新
只需要服务端推送的实时数据
```

不适合强双向通信场景。

## 手写任务

建议按下面顺序自己敲一遍：

1. 先写一个 `/sse` handler，返回 `Sse`。
2. 用 `stream::repeat_with` 创建事件。
3. 用 `.map(Ok)` 变成 `Result<Event, Infallible>`。
4. 用 `.throttle(Duration::from_secs(1))` 控制频率。
5. 加上 `keep_alive`。
6. 写一个 `index.html` 和 `script.js`，用 `EventSource` 连接。
7. 用浏览器 Console 验证消息。

加深练习：

1. 把 `hi!` 改成当前时间。
2. 给 Event 设置 event name 和 id。
3. 用 Tokio channel 代替 `repeat_with`，从后台任务推送消息。

## 本章真正要记住什么

SSE 的核心模型是：

```text
服务端返回一个不会立刻结束的 HTTP 响应
响应体是事件流
浏览器用 EventSource 接收事件
```

Axum 中最重要的结构是：

````rust
Sse<impl Stream<Item = Result<Event, Infallible>>>
````

它把 Rust stream 变成浏览器能理解的 Server-Sent Events。

## 源码对照

本章手写版对应源码：

- `examples/sse/src/main.rs`
- `examples/sse/assets/index.html`
- `examples/sse/assets/script.js`
- `examples/sse/Cargo.toml`
