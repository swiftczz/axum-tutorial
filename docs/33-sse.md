# 第 33 章 sse:服务端主动推送

## 本章你将做出什么

一个每秒向浏览器推送一条消息的服务:

```
GET /sse    一个永不结束的响应,持续输出事件
GET /       静态页面,用 EventSource 接收并打印到控制台
```

前面 33 章都是"客户端问一句、服务端答一句"。这一章反过来:**响应打开后不关闭,服务端想什么时候推就什么时候推**。

学完本章你将掌握:

- SSE 是什么,它就是一个不结束的 HTTP 响应
- SSE 与 WebSocket、轮询的取舍
- `Stream` trait 与流式响应在 axum 里的表达
- **心跳(keep-alive)为什么是必需的**
- 浏览器端 `EventSource` 的自动重连
- 为什么 SSE 和[第 18 章](18-print-request-response.md)那个中间件天然冲突

## 前置知识

### 服务端推送的三种方案

需求是"服务端有新数据时通知客户端":通知、进度条、股票价格、聊天消息。

**HTTP 本身是请求-响应的**,服务端无法主动发起。三种绕过方式:

**轮询(polling)** —— 客户端每隔几秒问一次"有新数据吗"。实现最简单,但延迟高(平均是轮询间隔的一半)、浪费严重(大多数请求返回"没有")。

**SSE(Server-Sent Events)** —— 客户端发起一个请求,**服务端保持响应不关闭**,持续往里写数据。单向:只能服务端推给客户端。

**WebSocket** —— 通过 HTTP 升级握手切换到一个全双工协议,双方都能随时发消息。[第 34 章](34-websockets.md)的主题。

| | 轮询 | SSE | WebSocket |
|---|---|---|---|
| 方向 | 客户端拉 | **服务端 → 客户端** | 双向 |
| 协议 | 普通 HTTP | **普通 HTTP** | 升级后的独立协议 |
| 数据格式 | 任意 | 文本(UTF-8) | 文本或二进制 |
| 自动重连 | 不适用 | **浏览器内置** | 要自己写 |
| 经过代理/防火墙 | 无障碍 | 通常无障碍 | 可能被拦 |
| 浏览器 API | `fetch` | `EventSource` | `WebSocket` |
| 实现复杂度 | 最低 | **低** | 中 |

**SSE 最被低估的优势是"它就是普通 HTTP"**:走同样的端口、同样的认证(cookie/token 照常发)、同样的中间件、同样的负载均衡器,不需要为它单独配置任何东西。

**选型建议:只需要服务端推给客户端,就用 SSE。** 通知、日志流、进度、AI 逐字输出,这些都是单向的。只有真正需要客户端也频繁发消息(聊天、协作编辑、游戏)才值得上 WebSocket。

### SSE 的报文格式

SSE 的规范简单得出奇。服务端返回:

```
HTTP/1.1 200 OK
content-type: text/event-stream

data: hi!

data: hello again

event: userLogin
data: {"name":"张三"}

: 这是注释,常用作心跳

```

规则:

- **`Content-Type: text/event-stream`** —— 告诉浏览器这是 SSE 流
- **每个事件由若干 `字段: 值` 行组成,以一个空行结束**
- **`data:`** 是事件内容。多行 `data:` 会被拼成一个多行字符串
- **`event:`** 是事件名(可选),对应 JS 里 `addEventListener('userLogin', ...)`;不写则默认是 `message`
- **`id:`** 事件 ID(可选),用于断线重连时告诉服务端"我收到哪了"
- **`retry:`** 建议客户端的重连间隔(毫秒)
- **以 `:` 开头的行是注释**,内容会被忽略——这正是心跳的实现方式

**注意 `data` 只能是文本。** 传结构化数据就自己序列化成 JSON,客户端 `JSON.parse`。二进制得 base64,那不如用 WebSocket。

### Stream:异步的迭代器

SSE 响应体是一串陆续产生的事件,这在 Rust 里的抽象是 **`Stream`**:

```rust
pub trait Stream {
    type Item;
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;
}
```

**`Stream` 之于 `Iterator`,就像 `Future` 之于普通值。** `Iterator::next` 立刻返回下一项;`Stream::poll_next` 可能返回"还没准备好,等会儿再问我"。

| | `Iterator` | `Stream` |
|---|---|---|
| 取下一项 | `next()` 立即返回 | `poll_next()` 可能挂起 |
| 遍历 | `for x in it` | `while let Some(x) = s.next().await` |
| 组合子 | `map`/`filter`/`take` | 同名,但来自 `StreamExt` |
| 结束 | 返回 `None` | 返回 `Poll::Ready(None)` |

**`Stream` 目前还不在标准库里**(`std::async_iter::AsyncIterator` 仍不稳定),所以用 `futures_util::Stream`,组合子来自 `StreamExt` 或 `tokio_stream::StreamExt` —— 又一个必须显式导入的扩展 trait([第 09 章](09-consume-body-in-extractor-or-middleware.md#collect-与-bodyext)的 `BodyExt`、[第 21 章](21-testing.md#oneshot-与-readycall)的 `ServiceExt` 都是这个套路)。

**流式响应是 SSE 的本质**:handler 不是"算出全部内容再返回",而是**返回一个还没产出完的流**,axum 边取边发。

### 心跳为什么是必需的

**这是 SSE 实践中最容易忽略、也最容易出问题的一点。**

SSE 连接可能长时间没有数据(比如"有新通知才推"),而中间的每一环都可能在空闲时切断连接:

- **反向代理**(Nginx 默认 `proxy_read_timeout 60s`)
- **云负载均衡器**(AWS ALB 默认空闲超时 60 秒)
- **NAT 网关 / 防火墙**,清理长时间无流量的连接映射
- **移动网络**,尤其激进

**心跳就是定期发一点无意义的数据,让链路"看起来"是活的:**

```rust
Sse::new(stream).keep_alive(
    KeepAlive::new()
        .interval(Duration::from_secs(1))
        .text("keep-alive-text"),
)
```

axum 会在流空闲超过 `interval` 时,自动发一行注释:

```
:keep-alive-text
```

以 `:` 开头,客户端**收到但忽略**,不会触发 `onmessage`。

**心跳间隔要小于链路上最短的超时时间。** 常见配置是 15~30 秒。示例设成 1 秒是为了演示,生产环境太频繁了——「代价与边界」会说。

### EventSource 与自动重连

浏览器端的 API 只有三行:

```js
var eventSource = new EventSource('sse');

eventSource.onmessage = function(event) {
    console.log('Message from server ', event.data);
}
```

**`EventSource` 内置了自动重连**,这是它相对手写 `fetch` 流处理的最大优势:连接断了会自动重试,间隔递增。

配合 `id:` 字段还能做**断点续传**:客户端重连时会把最后收到的 ID 放在 `Last-Event-ID` 请求头里发过来,服务端据此从断点继续推。**本章示例没有实现这个**,但它是生产级 SSE 的标配。

**`EventSource` 的两个限制**:

- **只支持 `GET`**,不能发请求体
- **不能自定义请求头**,所以无法用 `Authorization: Bearer xxx`([第 30 章](30-jwt.md#typedheader类型化的头部)那套)。认证只能靠 cookie 或查询参数

第二条很实际:**用 JWT 的服务想上 SSE,得改用 cookie 认证或把 token 放 URL 里**(后者会进日志,不理想)。想保留自定义头部就得放弃 `EventSource`,改用 `fetch` + 手动解析流。

## 项目搭建

```bash
cargo new sse
cd sse
```

```bash
cargo add axum
cargo add axum-extra --features typed-header
cargo add futures-util --no-default-features --features sink,std
cargo add headers
cargo add tokio --features full
cargo add tokio-stream
cargo add tower-http --features fs,trace
cargo add tracing
cargo add tracing-subscriber --features env-filter
cargo add --dev eventsource-stream
cargo add --dev reqwest --no-default-features --features stream
```

`tower-http` 的 `fs` feature 提供 `ServeDir`(静态文件服务,[第 38 章](38-static-file-server.md)的主题)。

建立静态资源目录:

```bash
mkdir assets
```

`assets/index.html`:

```html
<script src='script.js'></script>
```

`assets/script.js`:

```js
var eventSource = new EventSource('sse');

eventSource.onmessage = function(event) {
    console.log('Message from server ', event.data);
}
```

**页面上没有任何可见内容** —— 观察点在浏览器控制台。

## 分步构建

### 步骤 1:一个最简单的 SSE 流

**要做什么**:实现 `/sse`,每秒推一条消息。

```rust
// src/main.rs
use axum::{
    response::sse::{Event, Sse},
    routing::get,
    Router,
};
use futures_util::stream::{self, Stream};
use std::{convert::Infallible, time::Duration};
use tokio_stream::StreamExt as _;
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

    let app = Router::new().route("/sse", get(sse_handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn sse_handler() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    // A `Stream` that repeats an event every second
    let stream = stream::repeat_with(|| Event::default().data("hi!"))
        .map(Ok)
        .throttle(Duration::from_secs(1));

    Sse::new(stream)
}
```

**返回类型 `Sse<impl Stream<Item = Result<Event, Infallible>>>`** 拆开看:

- **`Sse<S>`** 是 axum 提供的响应类型,它实现了 `IntoResponse`,负责设置 `Content-Type: text/event-stream` 并把流里的 `Event` 序列化成 SSE 格式
- **`impl Stream<...>`** 是[第 04 章](04-templates.md#步骤-3为自己的类型实现-intoresponse)讲过的 `impl Trait`——具体类型由编译器推断
- **`Item = Result<Event, Infallible>`** —— 每一项是一个事件或一个错误。`Infallible` 表示"永不出错"([第 31 章](31-oauth.md#optionalfromrequestparts)见过这个类型)

**为什么是 `Result` 而不是直接 `Event`?** 因为真实场景里产生事件可能失败(查数据库出错、序列化失败)。流可以产出 `Err` 让 axum 中断连接。本例永不出错,所以用 `Infallible`。

**三个组合子的作用:**

- **`stream::repeat_with(|| ...)`** 创建一个**无限流**,每次取值时调用闭包。这就是"永不结束的响应"的来源。
- **`.map(Ok)`** 把 `Event` 包成 `Result<Event, Infallible>`,匹配 `Sse` 要求的类型
- **`.throttle(Duration::from_secs(1))`** 限速:每秒最多产出一项。**没有它,这个无限流会以 CPU 全速产生事件**,瞬间打爆网络和客户端

`throttle` 来自 `tokio_stream::StreamExt`,所以有那行 `use tokio_stream::StreamExt as _;`(`as _` 表示"只要 trait 方法,不要名字")。

**`Event::default().data("hi!")`** 构造一个事件。还能链式调用 `.event("name")` 设事件名、`.id("123")` 设 ID、`.retry(...)` 设重连间隔。

**跑起来看**:

```bash
cargo run
```

用 curl 直接看原始流(注意 `-N` 禁用缓冲,否则看不到实时输出):

```bash
curl -N http://127.0.0.1:3000/sse
```

```
data: hi!

data: hi!

data: hi!

```

**每秒出现一行,而且请求不会结束** —— 这就是"不结束的响应"。按 `Ctrl-C` 停止。

看响应头:

```bash
curl -N -i http://127.0.0.1:3000/sse | head -6
```

```
HTTP/1.1 200 OK
content-type: text/event-stream
cache-control: no-cache
transfer-encoding: chunked

data: hi!
```

三个关键头部:

- **`content-type: text/event-stream`** —— `Sse` 自动设的
- **`cache-control: no-cache`** —— 防止代理缓存这个流
- **`transfer-encoding: chunked`** —— **没有 `content-length`**,因为长度未知。这是流式响应的标志

**验证 `throttle` 的必要性。** 临时去掉它:

```rust
    let stream = stream::repeat_with(|| Event::default().data("hi!"))
        .map(Ok);
    // 临时:没有 .throttle(...)
```

```bash
cargo run
curl -N http://127.0.0.1:3000/sse | head -20
```

**屏幕瞬间被刷屏**,同时 CPU 占用飙升——无限流以最快速度产生事件。这在真实场景里会立刻打垮客户端和网络。改回去。

### 步骤 2:加上心跳和 User-Agent

**要做什么**:配置 keep-alive,并用 `TypedHeader` 记录客户端信息。

```rust
// src/main.rs
use axum::{
    response::sse::{Event, Sse},
    routing::get,
    Router,
};
use axum_extra::TypedHeader;                          // 新增
use futures_util::stream::{self, Stream};
use std::{convert::Infallible, time::Duration};
use tokio_stream::StreamExt as _;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// ...(main 不变)

// 改动:加上 User-Agent 提取和 keep_alive
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
```

**`TypedHeader<headers::UserAgent>`** 是[第 30 章](30-jwt.md#typedheader类型化的头部)讲的类型化头部,这次取的是 `User-Agent`。

**注意:handler 里的 `println!` 只在连接建立时执行一次。** 之后的事件产生发生在流里,和 handler 函数体无关。**理解这个时序很重要**——handler 返回的是"如何产生事件"的描述,不是事件本身。

**`keep_alive(...)`** 配置心跳,前置知识讲过。

> **更好的写法**:心跳间隔设成 1 秒太频繁了。前置知识讲过它的作用是"防止链路上的代理/防火墙因空闲而断开",而这些超时通常是 60 秒起步。1 秒的心跳意味着**每分钟 60 次无意义的网络往返**,连接数一多就是可观的开销。生产环境常见配置是 15~30 秒。示例设 1 秒是为了让你能快速观察到心跳,不是推荐值。

另外这里用 `println!` 而非 `tracing::info!`,和[第 29 章](29-mongodb.md#步骤-1连接与数据模型)指出的问题一样——日志系统已经初始化了,应该统一走 tracing。

**跑起来看**:

```bash
cargo run
```

```bash
curl -N http://127.0.0.1:3000/sse
```

服务端终端:

```
`curl/8.7.1` connected
```

**只打印一次**,证实了上面说的时序。

**观察心跳。** 因为示例每秒都有数据,心跳不会触发(流不空闲)。临时把数据间隔改长:

```rust
        .throttle(Duration::from_secs(10));    // 临时:10 秒一条数据
```

心跳仍是 1 秒:

```bash
cargo run
curl -N http://127.0.0.1:3000/sse
```

```
:keep-alive-text

:keep-alive-text

...(9 次之后)
data: hi!

:keep-alive-text
```

**以 `:` 开头的是注释行,客户端收到但忽略。** 这就是心跳的样子——它让链路上的中间设备看到流量,但不会触发客户端的 `onmessage`。

改回 1 秒。

### 步骤 3:配上前端页面

**要做什么**:加上静态文件服务,用浏览器的 `EventSource` 接收。

```rust
// src/main.rs
use axum::{
    response::sse::{Event, Sse},
    routing::get,
    Router,
};
use axum_extra::TypedHeader;
use futures_util::stream::{self, Stream};
use std::{convert::Infallible, path::PathBuf, time::Duration};        // 改动:加上 PathBuf
use tokio_stream::StreamExt as _;
use tower_http::{services::ServeDir, trace::TraceLayer};              // 新增
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    // 改动:抽出 app()
    let app = app();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

// 新增
fn app() -> Router {
    let assets_dir = PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("assets");
    let static_files_service = ServeDir::new(assets_dir).append_index_html_on_directories(true);
    // build our application with a route
    Router::new()
        .fallback_service(static_files_service)
        .route("/sse", get(sse_handler))
        .layer(TraceLayer::new_for_http())
}

// ...(sse_handler 不变)
```

**`fallback_service(static_files_service)`** —— [第 06 章](06-global-404-handler.md#fallback匹配不上时交给谁)讲过 `fallback` 接受 handler,`fallback_service` 则接受一个 `Service`。`ServeDir` 是个 `Service`,所以用后者。

效果是:**先匹配 `/sse`,匹配不上的一律交给静态文件服务**。

**`append_index_html_on_directories(true)`** 让访问 `/` 时返回 `index.html`。

> **本章不深入**:`ServeDir` 来自 `tower-http`,负责把目录映射成 HTTP 路由。这里照抄即可,它的完整用法(路径穿越防护、缓存头部、`fallback` 组合)见[第 38 章](38-static-file-server.md)。

**`env!("CARGO_MANIFEST_DIR")`** 的问题[第 32 章](32-tls-rustls.md#步骤-1跑起-https-服务)指出过:编译期路径,二进制换机器就找不到资源。

**跑起来看**:

```bash
cargo run
```

浏览器访问 `http://127.0.0.1:3000/`,页面空白,按 F12 打开控制台:

```
Message from server  hi!
Message from server  hi!
Message from server  hi!
```

**每秒新增一条。**

服务端终端:

```
`Mozilla/5.0 (...) Chrome/...` connected
```

**验证自动重连。** 保持浏览器开着,在终端按 `Ctrl-C` 停掉服务,观察控制台——会出现连接错误。然后重新 `cargo run`:

**浏览器会自己重连上,消息继续打印。** 你不用刷新页面,也不用写任何重连代码——这是 `EventSource` 内置的。

服务端终端会再次打印 `connected`,证明是一条新连接。

**看 Network 标签。** 找到那个 `sse` 请求,你会看到它的状态一直是 "pending"(进行中),而 EventStream 标签里能看到收到的事件列表。**这个请求永远不会完成** —— 这就是 SSE 的本质。

## 完整代码

`assets/index.html`:

```html
<script src='script.js'></script>
```

`assets/script.js`:

```js
var eventSource = new EventSource('sse');

eventSource.onmessage = function(event) {
    console.log('Message from server ', event.data);
}
```

`src/main.rs`:

```rust
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
    axum::serve(listener, app).await.unwrap();
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

#[cfg(test)]
mod tests {
    use eventsource_stream::Eventsource;
    use tokio::net::TcpListener;

    use super::*;

    #[tokio::test]
    async fn integration_test() {
        // A helper function that spawns our application in the background
        async fn spawn_app(host: impl Into<String>) -> String {
            let host = host.into();
            // Bind to localhost at the port 0, which will let the OS assign an available port to us
            let listener = TcpListener::bind(format!("{host}:0")).await.unwrap();
            // Retrieve the port assigned to us by the OS
            let port = listener.local_addr().unwrap().port();
            tokio::spawn(async {
                axum::serve(listener, app()).await;
            });
            // Returns address (e.g. http://127.0.0.1{random_port})
            format!("http://{host}:{port}")
        }
        let listening_url = spawn_app("127.0.0.1").await;

        let mut event_stream = reqwest::Client::new()
            .get(format!("{listening_url}/sse"))
            .header("User-Agent", "integration_test")
            .send()
            .await
            .unwrap()
            .bytes_stream()
            .eventsource()
            .take(1);

        let mut event_data: Vec<String> = vec![];
        while let Some(event) = event_stream.next().await {
            match event {
                Ok(event) => {
                    // break the loop at the end of SSE stream
                    if event.data == "[DONE]" {
                        break;
                    }

                    event_data.push(event.data);
                }
                Err(_) => {
                    panic!("Error in event stream");
                }
            }
        }

        assert!(event_data[0] == "hi!");
    }
}
```

## 代价与边界

**每个连接占用一个服务端资源。** SSE 连接是长连接,10000 个在线用户就是 10000 条 TCP 连接。tokio 的异步模型能扛住(不是每连接一线程),但要注意:

- 文件描述符上限(`ulimit -n`)
- 每条连接的内存开销
- 负载均衡器的连接数限制

**水平扩展需要额外设计。** 用户 A 连在实例 1 上,而"给 A 的通知"这个事件产生在实例 2 —— 实例 2 不知道 A 在哪。解决办法是引入消息中间件(Redis Pub/Sub、NATS、Kafka),所有实例订阅同一个频道。**这是从"能跑"到"能上线"之间最大的一道坎。**

**和缓冲 body 的中间件天然冲突。** [第 18 章](18-print-request-response.md#代价与边界)预告过这一点,现在可以说清了:那个中间件对响应体调用 `collect()`,而 `collect()` 会**等到流结束**才返回。SSE 的流永不结束,所以:

- 客户端**一个字节都收不到**,一直等待
- 服务端的内存里不断累积事件,直到连接断开
- **SSE 功能完全失效,而且没有任何报错**

同理,响应压缩([第 39 章](39-compression.md))对 SSE 也要小心——压缩中间件可能会缓冲数据等待更好的压缩率,导致事件不能及时送达。**给 SSE 路由单独配置中间件,别全局套用。**

**心跳间隔设成 1 秒太频繁。** 步骤 2 的「更好的写法」指出了。

**没有实现断点续传。** 前置知识讲过 `id:` 字段和 `Last-Event-ID` 头部的机制,示例没用。这意味着**断线重连后中间的事件会永久丢失**。需要可靠投递的场景必须实现它。

**`EventSource` 不能带自定义头部。** 前置知识讲过——用 JWT 的服务想上 SSE,认证方式要么改 cookie,要么把 token 放查询参数(会进访问日志)。这是个实际的架构约束。

**无限流没有终止条件。** 示例的流永远产生事件,只能靠客户端断开来结束。真实场景通常需要:任务完成后发一个结束事件、超时后主动关闭、或者检测到客户端断开就清理资源。

**客户端断开的检测是隐式的。** axum 会在客户端断开时丢弃这个流(future 被取消),所以流里的资源会释放。但**如果你在流外面持有了什么资源**(比如往一个 map 里注册了这个连接),需要用 `Drop` 或 `tokio::select!` 显式清理。

**没有认证。** `/sse` 对所有人开放。真实场景要加认证,而且受限于上面说的 `EventSource` 限制。

**`env!("CARGO_MANIFEST_DIR")` 硬编码资源路径。** 和[第 32 章](32-tls-rustls.md#步骤-1跑起-https-服务)同样的问题。

## 动手练习

**1. 用 channel 驱动事件,而不是固定循环。**

提示:这是真实场景的形态——事件由业务逻辑产生,不是定时器。用 `tokio::sync::broadcast` 建一个通道,把 `Sender` 放进 `State`([第 22 章](22-key-value-store.md#从只读到可写)),`/sse` 里用 `tokio_stream::wrappers::BroadcastStream` 把 `Receiver` 变成流。再加一条 `POST /notify` 往通道里发消息。验证:打开浏览器控制台,然后 `curl -X POST -d 'hello' http://127.0.0.1:3000/notify`,消息应该立刻出现在控制台。**做完你就有了一个可用的通知系统骨架。**

**2. 实现断点续传。**

提示:给事件加 ID(`Event::default().id(n.to_string())`),handler 里提取 `Last-Event-ID` 头部(`TypedHeader` 没有现成类型,用 `headers.get("last-event-id")`),从那个 ID 之后开始推。验证:连上后杀掉服务、重启,观察浏览器重连时是否带上了 `Last-Event-ID`(在 Network 标签里看请求头)。

**3. 验证 SSE 和缓冲中间件的冲突。**

提示:把[第 18 章](18-print-request-response.md#完整代码)那个 `print_request_response` 中间件加到本章的 app 上,然后 `curl -N http://127.0.0.1:3000/sse`。**你会一个字节都收不到**,而且服务端内存缓慢增长。这印证了「代价与边界」里说的问题。做完把中间件改成只对非 SSE 路由生效。

## 小结与下一章预告

SSE 就是**一个不结束的 HTTP 响应**:`Content-Type: text/event-stream`,每个事件由 `data:` 等字段行组成、以空行分隔,以 `:` 开头的行是注释。它最被低估的优势是**"就是普通 HTTP"** —— 同样的端口、认证、中间件、负载均衡,不用为它单独配置任何东西。

**只需要服务端推给客户端,就用 SSE**;只有真正需要双向频繁通信才值得上 WebSocket。

axum 里用 `Sse<S>` 包一个 `Stream`。`Stream` 是异步版的 `Iterator`,组合子来自 `StreamExt`(又一个要显式导入的扩展 trait)。**无限流必须限速**——本章验证过去掉 `throttle` 会瞬间刷屏并打满 CPU。

**心跳是必需的**,不是可选优化:代理、负载均衡器、NAT 都会切断空闲连接。间隔要小于链路上最短的超时,生产环境通常 15~30 秒(示例的 1 秒太频繁了)。

浏览器的 `EventSource` **内置自动重连**——本章验证过杀掉服务再重启,页面会自己连上。但它**只支持 GET、不能自定义请求头**,所以用 JWT 的服务上 SSE 时认证方式会受限。

最后兑现了[第 18 章](18-print-request-response.md)的预告:**SSE 和缓冲 body 的中间件天然冲突**。`collect()` 要等流结束,而 SSE 的流永不结束——结果是客户端一个字节都收不到、服务端内存持续增长、**而且没有任何报错**。给流式路由单独配中间件,别全局套用。

SSE 是单向的。[第 34 章 websockets](34-websockets.md) 讲双向通信:HTTP 升级握手是怎么回事、消息帧有哪些类型、`split()` 如何把一条连接拆成独立的读写两半,以及一个 SSE 不需要操心但 WebSocket 必须面对的问题——**心跳要自己实现,因为协议层的 ping/pong 不会自动帮你检测半开连接**。
