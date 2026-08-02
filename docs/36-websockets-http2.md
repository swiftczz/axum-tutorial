# 第 36 章 websockets-http2:HTTP/2 上的 WebSocket

## 本章你将做出什么

一个跑在 **HTTPS + HTTP/2** 上的 WebSocket 广播服务:打开两个浏览器窗口,在一边输入消息,另一边立刻看到。

```
wss://localhost:3000/ws   WebSocket 端点(HTTP/2 握手)
https://localhost:3000/   带输入框的页面
```

这一章要解决的是[第 34 章](34-websockets.md)那个握手机制的局限:**HTTP/2 里没有 `Upgrade` 头部**,所以 WebSocket 的握手方式必须换。

学完本章你将掌握:

- HTTP/2 的多路复用,以及它为什么废掉了 `Upgrade`
- **扩展 `CONNECT` 方法:HTTP/2 上 WebSocket 的握手机制**
- 为什么 `axum_server` 需要显式 `enable_connect_protocol()`
- `broadcast` channel:一对多消息分发
- `tokio::select!` 的**取消安全**(cancel safety)—— 这是本章最深的一个概念

## 前置知识

### HTTP/2 与多路复用

[第 34 章](34-websockets.md#升级握手)那套 `Upgrade: websocket` 握手是 HTTP/1.1 的机制。它的前提是:**一条 TCP 连接同一时刻只处理一个请求**,所以可以把整条连接"交给"另一个协议。

HTTP/2 改变了这个前提。它引入了**流**(stream)的概念:一条 TCP 连接上可以并发跑多个逻辑流,每个流处理一个请求-响应。

| | HTTP/1.1 | HTTP/2 |
|---|---|---|
| 一条连接上的并发请求 | 1 个(队头阻塞) | **多个**(多路复用) |
| 头部格式 | 文本 | 二进制 + HPACK 压缩 |
| 服务端推送 | 无 | 有(已被主流浏览器废弃) |
| `Upgrade` 头部 | 支持 | **禁止** |

**最后一行是本章的起点。** HTTP/2 明确禁止 `Upgrade` 和 `Connection` 头部([RFC 9113](https://www.rfc-editor.org/rfc/rfc9113#section-8.2.2) 把它们列为"连接特定头部"),因为"把整条连接换协议"在多路复用下毫无意义——那会影响连接上的其它流。

**所以 HTTP/1.1 的 WebSocket 握手在 HTTP/2 上根本发不出去。**

### 扩展 CONNECT 方法

[RFC 8441](https://www.rfc-editor.org/rfc/rfc8441) 给出了解决方案:**用 `CONNECT` 方法在单个流上建立 WebSocket 隧道**,而不是接管整条连接。

对比两种握手:

**HTTP/1.1**([第 34 章](34-websockets.md#升级握手)):

```
GET /ws HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: ...
Sec-WebSocket-Version: 13

→ 101 Switching Protocols
```

**HTTP/2**(伪头部形式):

```
:method = CONNECT
:protocol = websocket      ← 这是 RFC 8441 新增的伪头部
:scheme = https
:path = /ws
sec-websocket-version = 13

→ :status = 200            ← 注意是 200,不是 101
```

三处关键差别:

- **方法是 `CONNECT` 而非 `GET`**
- **多了 `:protocol` 伪头部**,值为 `websocket`
- **成功响应是 `200` 而非 `101`** —— 因为没有"切换协议"这回事,只是这个流变成了隧道

**`:protocol` 这个扩展需要服务端明确声明支持。** 服务端通过在 HTTP/2 的 `SETTINGS` 帧里发送 `SETTINGS_ENABLE_CONNECT_PROTOCOL = 1` 来告知客户端"我支持这个"。客户端看到才会尝试用 `CONNECT` 握手。

**这就是本章那行关键代码的作用:**

```rust
// IMPORTANT: This is required to advertise our support for HTTP/2 websockets to the client.
// If you use axum::serve, it is enabled by default.
server.http_builder().http2().enable_connect_protocol();
```

注释里那句"If you use `axum::serve`, it is enabled by default"很重要:**`axum::serve` 默认就开了这个设置**,但[第 32 章](32-tls-rustls.md#为什么不能用-axumserve)讲过它不支持 TLS,所以这里必须用 `axum_server`——而 `axum_server` 默认**没有**开启。

**忘了这一行会怎样:** 客户端不知道服务端支持 `CONNECT` 协议,于是不会尝试 HTTP/2 的 WebSocket 握手。浏览器通常会回退到 HTTP/1.1(如果 ALPN 协商允许),或者直接失败。**这是个静默的降级,不容易发现。**

### 为什么这个组合不常见

**先说结论:HTTP/2 上的 WebSocket 用得很少,而且大多数场景不需要它。**

**支持面窄。** 浏览器侧 Chrome 和 Firefox 支持,Safari 一直有问题。服务端和中间层的支持更参差——很多反向代理不支持 `CONNECT` + `:protocol`,会直接拒绝或降级。

**收益有限。** HTTP/2 WebSocket 的主要好处是**共用一条 TCP 连接**:页面的 HTTP 请求和 WebSocket 走同一条连接,省掉一次 TCP + TLS 握手。但 WebSocket 是长连接,那一次握手的成本本来就被摊薄了。

**HTTP/2 的流控可能拖慢 WebSocket。** HTTP/2 有连接级和流级的流量控制窗口,大量数据传输时可能引入额外的等待。

**HTTP/3 是更好的方向。** [RFC 9220](https://www.rfc-editor.org/rfc/rfc9220) 定义了 HTTP/3 上的 WebSocket,基于 QUIC,没有队头阻塞问题。而 WebTransport 则是为这类场景设计的新协议。

**那什么时候值得用:**

- 客户端环境受控(比如自家 App,不用管 Safari)
- 已经全站 HTTP/2,想减少连接数
- 有严格的连接数限制(某些移动网络环境)

**必须走 TLS。** 实际上浏览器只在 HTTPS 下使用 HTTP/2(虽然规范允许明文 h2c),所以本章必然要配证书——这也是为什么它依赖 [第 32 章](32-tls-rustls.md)的内容。

### Version 提取器

本章 handler 有个参数能直接看出协议版本:

```rust
async fn ws_handler(
    ws: WebSocketUpgrade,
    version: Version,          // ← 这个
    State(sender): State<broadcast::Sender<String>>,
) -> Response {
    tracing::debug!("accepted a WebSocket using {version:?}");
```

`http::Version` 实现了 `FromRequestParts`,所以能直接当提取器用。可能的值:`HTTP_09`、`HTTP_10`、`HTTP_11`、`HTTP_2`、`HTTP_3`。

**这行日志是本章最直接的验证手段** —— 它会告诉你握手实际走的是 HTTP/1.1 还是 HTTP/2。

### broadcast channel:一对多

前面章节的共享状态都是"存数据"([第 22 章](22-key-value-store.md)的 KV、[第 23 章](23-todos.md)的 todos)。这一章的状态是**一个消息通道**:

```rust
.with_state(broadcast::channel::<String>(16).0)
```

`tokio::sync::broadcast` 是**一对多**广播:

- 一个 `Sender` 可以克隆到任意多处
- 每个 `Sender::subscribe()` 产生一个新的 `Receiver`
- **发一条消息,所有 `Receiver` 都能收到**

对比其它几种 channel:

| | 语义 | 每条消息被谁收到 |
|---|---|---|
| `mpsc` | 多生产者单消费者 | **一个**消费者 |
| `broadcast` | 多生产者多消费者 | **所有**订阅者 |
| `watch` | 单值,只关心最新 | 所有订阅者(可能跳过中间值) |
| `oneshot` | 只发一次 | 一个 |

**广播正是聊天室需要的语义**:一个人发消息,所有在线的人都收到。

**`channel::<String>(16)` 里的 16 是缓冲容量。** 这个参数有个重要含义:如果某个 `Receiver` 消费太慢,落后超过 16 条,它会**丢失中间的消息**,`recv()` 返回 `Err(RecvError::Lagged(n))`。

本章代码对这种情况的处理是:

```rust
Err(_) => continue,
```

**直接忽略,继续循环。** 意思是"落后了就跳过丢掉的消息"。这对聊天室可以接受(丢几条历史消息不致命),但要清楚它的含义——「代价与边界」会展开。

`.0` 取的是元组的第一个元素,即 `Sender`。`Receiver` 被丢弃了——每个连接会用 `sender.subscribe()` 自己创建一个。

### 取消安全:本章最深的概念

**这是 `tokio::select!` 使用中最容易出错、也最少被理解的一点。**

`select!` 的行为是:等多个 future,谁先完成就走对应分支,**其余的 future 被直接丢弃**(drop)。

问题在于:**被丢弃的那个 future 可能已经做了一部分工作。** 如果它在丢弃时会丢失状态或数据,那这个操作就是**取消不安全**的(not cancel-safe)。

举个反例:

```rust
// 假设有个方法从流里读数据到内部缓冲区,凑满一行才返回
tokio::select! {
    line = reader.read_line() => { ... }     // 如果这个被取消
    _ = other_future => { ... }              // 已读进缓冲的半行数据可能丢失
}
```

**本章代码里两处注释专门在说这件事:**

```rust
// Since `ws` is a `Stream`, it is by nature cancel-safe.
res = ws.recv() => { ... }

// Tokio guarantees that `broadcast::Receiver::recv` is cancel-safe.
res = receiver.recv() => { ... }
```

**为什么 `Stream::next()`(以及 `ws.recv()`)天然取消安全:** `Stream` 的契约是"要么返回一项,要么什么都不做"。`poll_next` 返回 `Pending` 时不消耗任何数据,下次轮询从同一位置继续。丢弃它不会丢数据。

**为什么 `broadcast::Receiver::recv` 取消安全:** tokio 的文档明确保证了这一点——取消时消息不会被消耗,下次 `recv()` 还能拿到。

**这个概念的实际重要性:** 如果本章的循环里用了取消不安全的操作,**每次 `select!` 选中另一个分支,就会静默丢失数据**。而且这类 bug 极难发现——它只在特定的时序下出现,日志里什么都看不到。

**判断办法:查文档。** tokio 为所有可能用在 `select!` 里的方法都标注了取消安全性。**不确定就别放进 `select!`**,改用 `spawn` 加 channel(就像[第 34 章](34-websockets.md#split为什么必须拆)那个 `split()` + 两个任务的做法)。

**注意本章没有用 `split()`。** 因为 `select!` 让单个 `ws` 也能"同时"处理收和发——每轮循环只做一件事,不需要并发。这是另一种可行的结构,代价是收发不能真正同时进行(发消息时不能同时收)。

## 项目搭建

```bash
cargo new websockets-http2
cd websockets-http2
```

```bash
cargo add axum --features ws,http2
cargo add axum-server --features tls-rustls
cargo add tokio --features full
cargo add tower-http --features fs
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

**`axum` 需要 `ws` 和 `http2` 两个 feature。** `http2` 默认是开着的(在 `default` feature 里),但这里显式写出来更清楚。

### 生成证书

和[第 32 章](32-tls-rustls.md#生成证书)完全一样:

```bash
mkdir self_signed_certs
cd self_signed_certs

openssl req -x509 -newkey rsa:4096 \
  -keyout key.pem -out cert.pem \
  -days 365 -nodes \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

cd ..
echo 'self_signed_certs/*.pem' >> .gitignore
```

`subjectAltName` 是必需的,理由见[第 32 章](32-tls-rustls.md#生成自签名证书)。

### 前端页面

```bash
mkdir assets
```

`assets/index.html`:

```html
<p>Open this page in two windows and try sending some messages!</p>
<form action="javascript:void(0)">
    <input type="text" name="content" required>
    <button>Send</button>
</form>
<div id="messages"></div>
<script src='script.js'></script>
```

`assets/script.js`:

```js
const socket = new WebSocket('wss://localhost:3000/ws');

socket.addEventListener('message', e => {
    document.getElementById("messages").append(e.data, document.createElement("br"));
});

const form = document.querySelector("form");
form.addEventListener("submit", () => {
    socket.send(form.elements.namedItem("content").value);
    form.elements.namedItem("content").value = "";
});
```

**注意是 `wss://` 而不是 `ws://`** —— WebSocket over TLS,和 `https://` 对应。

**`action="javascript:void(0)"`** 阻止表单默认提交(那会刷新页面)。

## 分步构建

### 步骤 1:HTTPS + 广播通道骨架

**要做什么**:搭出 TLS 服务和 broadcast channel,先不管 HTTP/2。

```rust
// src/main.rs
use axum::{
    extract::{ws::WebSocketUpgrade, State},
    http::Version,
    routing::any,
    Router,
};
use axum_server::tls_rustls::RustlsConfig;
use std::{net::SocketAddr, path::PathBuf};
use tokio::sync::broadcast;
use tower_http::services::ServeDir;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let assets_dir = PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("assets");

    // configure certificate and private key used by https
    let config = RustlsConfig::from_pem_file(
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("cert.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("key.pem"),
    )
    .await
    .unwrap();

    // build our application with some routes and a broadcast channel
    let app = Router::new()
        .fallback_service(ServeDir::new(assets_dir).append_index_html_on_directories(true))
        .route("/ws", any(ws_handler))
        .with_state(broadcast::channel::<String>(16).0);

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {}", addr);

    let server = axum_server::bind_rustls(addr, config);

    server.serve(app.into_make_service()).await.unwrap();
}

async fn ws_handler(
    ws: WebSocketUpgrade,
    version: Version,
    State(_sender): State<broadcast::Sender<String>>,
) -> axum::response::Response {
    tracing::debug!("accepted a WebSocket using {version:?}");
    ws.on_upgrade(|_ws| async move {
        // 下一步实现
    })
}
```

**TLS 部分和[第 32 章](32-tls-rustls.md#步骤-1跑起-https-服务)一模一样**:`RustlsConfig::from_pem_file` + `axum_server::bind_rustls` + `into_make_service()`。

**`fallback_service(ServeDir::new(...))`** 和[第 33](33-sse.md#步骤-3配上前端页面)、[34 章](34-websockets.md)一样,把 `assets/` 目录当静态文件兜底。

> **本章不深入**:`ServeDir` 来自 `tower-http`,负责把目录映射成 HTTP 路由。这里照抄即可,它的完整用法(路径穿越防护、缓存头部、`fallback` 组合)见[第 38 章](38-static-file-server.md)。

**`broadcast::channel::<String>(16).0`** 前置知识讲过:取 `Sender` 当状态,每个连接自己 `subscribe()`。

**`version: Version`** 是本步骤的观察点。

**`any(ws_handler)`** 而非 `get` —— [第 34 章](34-websockets.md#websocketupgrade-与-on_upgrade)讲过。**这在本章尤其重要**:HTTP/2 的握手用的是 `CONNECT` 方法,不是 `GET`!用 `get(...)` 会导致 HTTP/2 握手被路由拒绝(`405`)。

**跑起来看**:

```bash
cargo run
```

```
DEBUG websockets_http2: listening on 127.0.0.1:3000
```

浏览器访问 `https://localhost:3000/`(注意 `https`),点过证书警告([第 32 章](32-tls-rustls.md#步骤-1跑起-https-服务)讲过为什么会有),能看到输入框。

打开 F12 控制台,应该没有 WebSocket 错误(连接建立了,只是不做任何事)。

**看服务端日志里的版本:**

```
DEBUG websockets_http2: accepted a WebSocket using HTTP/1.1
```

**是 HTTP/1.1,不是 HTTP/2！** 尽管 TLS 已经启用、浏览器也支持 HTTP/2,WebSocket 握手仍然走的老路。

**原因就是前置知识讲的那个设置没开** —— 服务端没有告知客户端"我支持 `CONNECT` 协议",所以浏览器只能用 HTTP/1.1 的 `Upgrade` 握手。

下一步修。

### 步骤 2:开启 CONNECT 协议

**要做什么**:加上那行关键配置,让握手真正走 HTTP/2。

```rust
// src/main.rs
// ...(前面不变)

#[tokio::main]
async fn main() {
    // ...(证书和 app 不变)

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {}", addr);

    // 改动:需要 mut 才能调 http_builder
    let mut server = axum_server::bind_rustls(addr, config);

    // 新增:IMPORTANT: This is required to advertise our support for HTTP/2 websockets to the client.
    // If you use axum::serve, it is enabled by default.
    server.http_builder().http2().enable_connect_protocol();

    server.serve(app.into_make_service()).await.unwrap();
}

// ...(ws_handler 不变)
```

**就这一行。** `server.http_builder().http2().enable_connect_protocol()` 让服务端在 HTTP/2 的 `SETTINGS` 帧里声明支持 `CONNECT` 协议扩展。

**`let mut server`** —— `http_builder()` 需要可变引用。

**注释里那句话值得再强调一次:** `axum::serve` 默认开启这个设置,但它不支持 TLS;`axum_server` 支持 TLS 但默认不开这个设置。**两个库的默认值不一致,这是个容易踩的坑。**

**跑起来看**:

```bash
cargo run
```

浏览器刷新 `https://localhost:3000/`,看服务端日志:

```
DEBUG websockets_http2: accepted a WebSocket using HTTP/2.0
```

**现在是 HTTP/2 了。** 一行配置的差别。

**在浏览器 Network 标签里看更清楚。** 找到 `ws` 请求:

- **Protocol 列显示 `h2`**(而不是 `http/1.1`)
- **Status 是 `200`**(而不是 `101 Switching Protocols`)
- **Method 是 `CONNECT`**

这三点正是前置知识讲的 RFC 8441 握手特征。

**对比[第 34 章](34-websockets.md#步骤-3前端页面)那个 HTTP/1.1 的例子**:那边是 `101` + `GET`,这边是 `200` + `CONNECT`。同样的 WebSocket API,底层握手完全不同。

**把那行配置注释掉再试一次**,日志会退回 `HTTP/1.1`。**这个降级是静默的** —— 功能照常工作,只是没用上 HTTP/2。如果你以为配好了却没有,很难发现。

### 步骤 3:广播逻辑

**要做什么**:实现 `on_upgrade` 里的循环,收到消息就广播给所有连接。

```rust
// src/main.rs
use axum::{
    extract::{
        ws::{self, WebSocketUpgrade},        // 改动:加上 ws 模块
        State,
    },
    http::Version,
    routing::any,
    Router,
};
// ...(其余 use 不变)

// ...(main 不变)

// 改动:实现广播逻辑
async fn ws_handler(
    ws: WebSocketUpgrade,
    version: Version,
    State(sender): State<broadcast::Sender<String>>,
) -> axum::response::Response {
    tracing::debug!("accepted a WebSocket using {version:?}");
    let mut receiver = sender.subscribe();
    ws.on_upgrade(|mut ws| async move {
        loop {
            tokio::select! {
                // Since `ws` is a `Stream`, it is by nature cancel-safe.
                res = ws.recv() => {
                    match res {
                        Some(Ok(ws::Message::Text(s))) => {
                            let _ = sender.send(s.to_string());
                        }
                        Some(Ok(_)) => {}
                        Some(Err(e)) => tracing::debug!("client disconnected abruptly: {e}"),
                        None => break,
                    }
                }
                // Tokio guarantees that `broadcast::Receiver::recv` is cancel-safe.
                res = receiver.recv() => {
                    match res {
                        Ok(msg) => if let Err(e) = ws.send(ws::Message::Text(msg.into())).await {
                            tracing::debug!("client disconnected abruptly: {e}");
                        }
                        Err(_) => continue,
                    }
                }
            }
        }
    })
}
```

**`let mut receiver = sender.subscribe();`** 在 `on_upgrade` **之前**执行——这很重要。如果放在闭包内部,会因为闭包需要 `move` 而产生所有权问题;更关键的是**订阅时机**:提前订阅确保连接建立瞬间开始的消息不会漏掉。

**`select!` 的两个分支:**

**分支一 `ws.recv()`** —— 收到客户端消息就广播出去:

```rust
Some(Ok(ws::Message::Text(s))) => { let _ = sender.send(s.to_string()); }
```

`let _ =` 忽略发送结果。`broadcast::Sender::send` 在**没有任何订阅者**时返回 `Err`,但这里不算错误——没人在线就没人收,继续跑就是。

**分支二 `receiver.recv()`** —— 从广播通道收到消息就发给这个客户端:

```rust
Ok(msg) => if let Err(e) = ws.send(...).await { tracing::debug!(...) }
Err(_) => continue,
```

`Err(_) => continue` 就是前置知识讲的**落后处理**:缓冲区满了导致消息丢失时,跳过继续。

**注意发送者自己也会收到自己的消息** —— 因为它也订阅了同一个通道。所以在浏览器里发消息,自己的页面也会显示。这是这个实现的行为,不是 bug(聊天室通常也这样)。

**四个 `match` 分支覆盖了所有情况:**

| 分支 | 含义 | 处理 |
|---|---|---|
| `Some(Ok(Text(s)))` | 收到文本 | 广播 |
| `Some(Ok(_))` | 收到其它类型(Binary/Ping/Pong/Close) | 忽略 |
| `Some(Err(e))` | 接收出错 | 记日志,**但继续循环** |
| `None` | 流结束(连接关了) | `break` |

**第三个分支值得注意:出错了只记日志不退出。** 这有点可疑——如果连接已经坏了,继续循环只会一直出错。「代价与边界」会说这个问题。

**跑起来看**:

```bash
cargo run
```

**打开两个浏览器窗口**都访问 `https://localhost:3000/`(页面上的提示就是这么说的)。

在窗口 A 输入 `hello from A`,点 Send:

**两个窗口都立刻显示 `hello from A`。**

在窗口 B 输入 `hi from B`:

**两个窗口都显示 `hi from B`。**

**广播生效了。** 服务端日志:

```
DEBUG websockets_http2: accepted a WebSocket using HTTP/2.0
DEBUG websockets_http2: accepted a WebSocket using HTTP/2.0
```

两条连接都是 HTTP/2。

**验证广播的"一对多"语义。** 开三四个窗口,在任一个发消息,**所有窗口都会收到** —— 这就是 `broadcast` 相对 `mpsc` 的区别([前置知识](#broadcast-channel一对多)那张表)。

**验证落后丢消息。** 通道容量是 16。临时把它改成 2:

```rust
        .with_state(broadcast::channel::<String>(2).0);
```

然后在一个窗口里快速连发很多条消息(可以在控制台跑):

```js
for (let i = 0; i < 20; i++) socket.send(`msg ${i}`);
```

**另一个窗口可能会漏掉中间的消息** —— 因为它的 `Receiver` 落后超过 2 条,`recv()` 返回了 `Lagged`,而代码 `continue` 跳过了。

改回 16。

**验证关掉连接的行为。** 关掉一个窗口,服务端不会报错(`recv()` 返回 `None`,循环 `break`),另一个窗口照常工作。

## 完整代码

`assets/index.html`:

```html
<p>Open this page in two windows and try sending some messages!</p>
<form action="javascript:void(0)">
    <input type="text" name="content" required>
    <button>Send</button>
</form>
<div id="messages"></div>
<script src='script.js'></script>
```

`assets/script.js`:

```js
const socket = new WebSocket('wss://localhost:3000/ws');

socket.addEventListener('message', e => {
    document.getElementById("messages").append(e.data, document.createElement("br"));
});

const form = document.querySelector("form");
form.addEventListener("submit", () => {
    socket.send(form.elements.namedItem("content").value);
    form.elements.namedItem("content").value = "";
});
```

`src/main.rs`:

```rust
use axum::{
    extract::{
        ws::{self, WebSocketUpgrade},
        State,
    },
    http::Version,
    routing::any,
    Router,
};
use axum_server::tls_rustls::RustlsConfig;
use std::{net::SocketAddr, path::PathBuf};
use tokio::sync::broadcast;
use tower_http::services::ServeDir;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let assets_dir = PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("assets");

    // configure certificate and private key used by https
    let config = RustlsConfig::from_pem_file(
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("cert.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("key.pem"),
    )
    .await
    .unwrap();

    // build our application with some routes and a broadcast channel
    let app = Router::new()
        .fallback_service(ServeDir::new(assets_dir).append_index_html_on_directories(true))
        .route("/ws", any(ws_handler))
        .with_state(broadcast::channel::<String>(16).0);

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {}", addr);

    let mut server = axum_server::bind_rustls(addr, config);

    // IMPORTANT: This is required to advertise our support for HTTP/2 websockets to the client.
    // If you use axum::serve, it is enabled by default.
    server.http_builder().http2().enable_connect_protocol();

    server.serve(app.into_make_service()).await.unwrap();
}

async fn ws_handler(
    ws: WebSocketUpgrade,
    version: Version,
    State(sender): State<broadcast::Sender<String>>,
) -> axum::response::Response {
    tracing::debug!("accepted a WebSocket using {version:?}");
    let mut receiver = sender.subscribe();
    ws.on_upgrade(|mut ws| async move {
        loop {
            tokio::select! {
                // Since `ws` is a `Stream`, it is by nature cancel-safe.
                res = ws.recv() => {
                    match res {
                        Some(Ok(ws::Message::Text(s))) => {
                            let _ = sender.send(s.to_string());
                        }
                        Some(Ok(_)) => {}
                        Some(Err(e)) => tracing::debug!("client disconnected abruptly: {e}"),
                        None => break,
                    }
                }
                // Tokio guarantees that `broadcast::Receiver::recv` is cancel-safe.
                res = receiver.recv() => {
                    match res {
                        Ok(msg) => if let Err(e) = ws.send(ws::Message::Text(msg.into())).await {
                            tracing::debug!("client disconnected abruptly: {e}");
                        }
                        Err(_) => continue,
                    }
                }
            }
        }
    })
}
```

## 代价与边界

**这个组合本身就是小众选择。** 前置知识讲过:支持面窄(Safari 有问题、很多代理不支持)、收益有限(省一次握手,而 WebSocket 是长连接)、HTTP/3 是更好的方向。**默认用 HTTP/1.1 的 WebSocket 就好**,除非有明确理由。

**`enable_connect_protocol()` 漏了会静默降级。** 步骤 1 和 2 验证过这个差别。**没有任何报错**,功能照常,只是没用上 HTTP/2 —— 这类"配置没生效但看起来正常"的问题最难发现。建议像本章一样**用日志打出实际协议版本**。

**`Some(Err(e))` 分支不退出循环。** 收到错误只记日志然后继续。如果错误是持久性的(连接损坏),循环会疯狂空转刷日志。更稳妥的做法是 `break`,或者累计错误次数超过阈值才退出。

**`Err(_) => continue` 掩盖了两种不同的情况。** `broadcast::error::RecvError` 有两个变体:

- **`Lagged(n)`** —— 落后了 n 条,应该继续(可能值得记一条 warn)
- **`Closed`** —— 通道关闭了(所有 `Sender` 都被丢弃),这时**应该退出循环**而不是继续

现在两种都 `continue`,`Closed` 的情况会导致死循环空转。实际上因为 `Sender` 存在 `State` 里永不释放,`Closed` 不会发生——但这是依赖了外部条件的巧合,不是稳健的写法。

**没有 `split()`,收发不能真正并发。** 前置知识提过:`select!` 每轮只处理一件事。给某个客户端发消息时,不能同时接收它的消息。消息量大时会成为瓶颈。[第 34 章](34-websockets.md#split为什么必须拆)那个 `split()` + 两任务的结构在这方面更好。

**没有心跳。** 和[第 34 章](34-websockets.md#代价与边界)一样的问题:TCP 半开连接下 `recv()` 会永远挂着。而且这里连开头那次 `Ping` 都没有。

**没有校验 Origin。** [第 34 章](34-websockets.md#代价与边界)强调过 Cross-Site WebSocket Hijacking —— WebSocket 不受同源策略约束,必须自己检查 `Origin`。本章同样没做。

**没有认证、没有限流、没有消息大小限制。** 和[第 34 章](34-websockets.md)相同的一组问题。

**没有用户标识。** 所有消息都是裸文本,看不出谁发的。[第 37 章](37-chat.md)会解决这个。

**证书路径绑定 `CARGO_MANIFEST_DIR`。** [第 32 章](32-tls-rustls.md#步骤-1跑起-https-服务)指出过:编译期路径,二进制换机器就找不到。

**广播的水平扩展问题。** 和[第 33 章](33-sse.md#代价与边界)一样:`broadcast` channel 是进程内的,多实例部署时实例 A 的消息传不到实例 B 的客户端。需要 Redis Pub/Sub 之类的跨进程方案。

## 动手练习

**1. 正确区分 `Lagged` 和 `Closed`。**

提示:把 `Err(_) => continue` 展开成:

```rust
Err(broadcast::error::RecvError::Lagged(n)) => {
    tracing::warn!("client lagged, skipped {n} messages");
    continue;
}
Err(broadcast::error::RecvError::Closed) => break,
```

**这是「代价与边界」里最该修的一处** —— 现在 `Closed` 会导致死循环空转。做完用步骤 3 那个"通道容量改成 2"的办法验证 `Lagged` 的日志出现了。

**2. 加上发送者标识,并且不回显给自己。**

提示:给消息加个前缀(用 `ConnectInfo<SocketAddr>` 或一个递增的连接 ID),广播时带上。想"不回显给自己"需要在广播的消息里带上发送者 ID,收到时比对——**这正是 `broadcast` 的一个局限**:它没法"发给除某人以外的所有人"。[第 37 章](37-chat.md)会看到官方怎么处理这个。

**3. 对比 HTTP/1.1 和 HTTP/2 的握手报文。**

提示:注释掉 `enable_connect_protocol()`,在浏览器 Network 标签里对比两次的 `ws` 请求:Protocol 列、Status、Method 三处都不同(`http/1.1`/`101`/`GET` vs `h2`/`200`/`CONNECT`)。**这个对比能让 RFC 8441 的机制变得具体。**

## 小结与下一章预告

HTTP/2 的多路复用让"把整条连接换协议"失去意义,所以它**禁止 `Upgrade` 头部** —— [第 34 章](34-websockets.md#升级握手)那套握手在 HTTP/2 上根本发不出去。

[RFC 8441](https://www.rfc-editor.org/rfc/rfc8441) 的方案是**用扩展 `CONNECT` 方法在单个流上建隧道**:方法从 `GET` 变成 `CONNECT`、多一个 `:protocol = websocket` 伪头部、成功响应从 `101` 变成 `200`。

**服务端必须显式声明支持这个扩展**,否则客户端不会尝试。`axum::serve` 默认开启但不支持 TLS,`axum_server` 支持 TLS 但默认不开——**两个库的默认值不一致**,所以要那行 `server.http_builder().http2().enable_connect_protocol()`。漏了它会**静默降级到 HTTP/1.1**,功能正常但没报错,本章用 `Version` 提取器的日志验证了这个差别。

路由必须用 `any` 而非 `get`,因为 HTTP/2 握手的方法是 `CONNECT`。

**这个组合本身是小众选择**:支持面窄、收益有限、HTTP/3 + WebTransport 是更好的方向。默认用 HTTP/1.1 就好。

`broadcast` channel 提供一对多语义,正是广播需要的。要注意它的**落后丢消息**行为:`Receiver` 落后超过缓冲容量时 `recv()` 返回 `Lagged`,示例的 `Err(_) => continue` 把它和 `Closed` 混为一谈,后者会导致死循环。

**本章最深的概念是取消安全**:`select!` 会丢弃未完成的 future,如果那个操作在丢弃时会丢数据,就会**静默出错且极难排查**。`Stream::next` 和 `broadcast::Receiver::recv` 都有取消安全保证,所以能放心用在 `select!` 里。**不确定就查文档,或者改用 `spawn` + channel。**

本章的广播已经是聊天室的雏形了,但缺了用户标识、加入/离开通知、以及"不回显给自己"的能力。[第 37 章 chat](37-chat.md) 是「实时通信」阶段的综合项目:完整的聊天室,有用户名、有在线列表、有加入退出广播。你会看到共享状态和 WebSocket 结合时的完整形态,以及 `broadcast` 那个"发给除某人以外的所有人"的局限该怎么绕。
