# 第 34 章 websockets:双向通信

## 本章你将做出什么

一个 WebSocket 服务和**两个客户端**:

```
GET /ws     WebSocket 端点
GET /       浏览器页面(JS 客户端)
```

外加一个 Rust 客户端(`src/bin/client.rs`),同时开两条连接压测服务端。这是本教程第一个**多可执行文件**的项目。

服务端会先 ping、收一条消息、连发 4 条问候,然后**把连接拆成读写两半**并发工作:一个任务持续推 20 条消息,另一个任务持续接收客户端消息。

学完本章你将掌握:

- WebSocket 的升级握手,以及它和普通 HTTP 的关系
- 六种消息帧类型,以及 `Ping`/`Pong` 为什么不用你管
- **`split()` 把一条连接拆成独立的读写两半**,以及为什么必须这么做
- `tokio::select!` 协调两个任务的生命周期
- 一个 Cargo 技巧:一个项目产出多个二进制

## 前置知识

### WebSocket 与 SSE 的根本差别

[第 33 章](33-sse.md#服务端推送的三种方案)那张表对比过三种方案,这里说清 WebSocket 的定位。

**SSE 是"一个不结束的 HTTP 响应"** —— 它始终在 HTTP 的框架内,只是响应体一直不结束。

**WebSocket 是"用 HTTP 握手,然后换协议"** —— 握手成功后,这条 TCP 连接上跑的**不再是 HTTP**,而是 WebSocket 自己的帧协议。

这个区别带来几个实际后果:

**双向。** 客户端和服务端都能随时发消息,不需要"客户端先问"。

**二进制友好。** SSE 只能传 UTF-8 文本([第 33 章](33-sse.md#sse-的报文格式)讲过),WebSocket 原生支持二进制帧。

**不能用普通 HTTP 工具调试。** curl 发不了 WebSocket 消息(要用 `websocat` 之类),浏览器 Network 标签也需要专门的 Messages 视图。

**中间设备可能不认。** 老旧的代理、某些企业防火墙会拦住 `Upgrade` 请求。SSE 因为是普通 HTTP,通过率更高。

**心跳要自己管。** 这是最容易忽略的一点,后面详说。

### 升级握手

WebSocket 连接从一个普通 HTTP 请求开始:

```
GET /ws HTTP/1.1
Host: localhost:3000
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

服务端同意的话返回:

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**`101 Switching Protocols`** —— [第 00 章](00-web-basics.md#状态码一个三位数的分类回答)讲状态码分类时提过 `1xx` 很少见,`101` 是其中最常用的一个。它的含义是"从现在起,这条连接上跑别的协议"。

**`Sec-WebSocket-Accept` 是对 `Key` 的固定变换**(拼一个魔法字符串后做 SHA-1 再 base64)。它不是安全机制,只是确认"对端真的懂 WebSocket 协议",防止普通 HTTP 服务器被误当成 WebSocket 端点。

**握手是最后能访问 HTTP 信息的时刻。** 官方示例的注释专门强调了这一点:

> This is the last point where we can extract TCP/IP metadata such as IP address of the client as well as things from HTTP headers such as user-agent

握手之后就没有"HTTP 请求"了,所以**认证、限流、来源检查必须在这里做完**。这也是为什么 `ws_handler` 的签名里有 `TypedHeader<UserAgent>` 和 `ConnectInfo<SocketAddr>`。

### WebSocketUpgrade 与 on_upgrade

axum 把握手封装成一个提取器:

```rust
async fn ws_handler(ws: WebSocketUpgrade, /* 其它提取器 */) -> impl IntoResponse {
    // 这里还在 HTTP 世界,能访问所有请求信息
    ws.on_upgrade(move |socket| handle_socket(socket, addr))
}
```

**`WebSocketUpgrade` 提取器**会检查握手头部是否合法,不合法直接拒绝(返回 `400`)。

**`ws.on_upgrade(callback)`** 返回一个 `Response`(那个 `101`)。axum 发出这个响应后,**在后台调用你的回调**,把已升级的 `WebSocket` 交给它。

关键在于时序:

1. `ws_handler` 返回 → axum 发送 `101` 响应
2. 连接升级完成
3. **回调被调用**,`handle_socket` 开始跑

所以 `ws_handler` 里的代码在升级**之前**执行,回调里的代码在升级**之后**执行。回调是 `async` 的,可以一直跑到连接关闭。

**注意路由用的是 `any` 而不是 `get`:**

```rust
.route("/ws", any(ws_handler))
```

握手请求确实是 `GET`,用 `get` 也能工作。用 `any` 是更宽容的写法——某些客户端实现可能用别的方法,而且升级后"方法"这个概念就不存在了。

### 六种消息帧

WebSocket 的数据以**帧**(frame)为单位,axum 的 `Message` 枚举对应六种:

| 变体 | 用途 | 谁发 |
|---|---|---|
| `Text(Utf8Bytes)` | UTF-8 文本 | 双方 |
| `Binary(Bytes)` | 任意字节 | 双方 |
| `Ping(Bytes)` | 探测对端是否活着 | 双方 |
| `Pong(Bytes)` | 回应 Ping,内容必须原样返回 | 双方 |
| `Close(Option<CloseFrame>)` | 请求关闭连接 | 双方 |
| `Frame(_)` | 底层原始帧 | 仅在直接用 tungstenite 时出现 |

**`Ping` 你不用手动回复。** 官方示例的注释说得很明确:

> You should never need to manually handle `Message::Ping`, as axum's websocket library will do so for you automagically by replying with Pong

axum 底层的 tungstenite 会自动回 `Pong` 并复制内容。你在 `match` 里看到 `Ping` 只是**能观察到它**,不需要处理。

**`Close` 是"请求"关闭,不是立即断开。** 规范要求收到 `Close` 后回一个 `Close` 再关连接(挥手)。axum 也会自动处理这个,你的代码只需要**收到 `Close` 就退出循环**。

`CloseFrame` 里有 `code` 和 `reason`。常用的码:

- `1000` (`close_code::NORMAL`) —— 正常关闭
- `1001` —— 端点离开(比如页面关闭)
- `1011` —— 服务端内部错误
- `3000`~`3999` —— 应用自定义(本章前端用了 `3000`)

### split():为什么必须拆

**这是本章最重要的技术点。**

`WebSocket` 同时是 `Stream`(收)和 `Sink`(发)。直接用它的话:

```rust
socket.send(msg).await;      // 需要 &mut socket
socket.recv().await;         // 也需要 &mut socket
```

**两个操作都要可变借用,所以不能同时进行。** 想"一边收一边发"就得等一个完成再做另一个——这在真实场景里行不通:服务端要推消息时,可能正阻塞在等客户端消息上。

`futures_util::StreamExt::split()` 解决这个问题:

```rust
let (mut sender, mut receiver) = socket.split();
```

它把一条连接拆成两个独立的句柄:`SplitSink`(只能发)和 `SplitStream`(只能收)。**两者可以移进不同的任务并发工作。**

内部实现是用一个 `BiLock` 协调对底层连接的访问——发和收都需要碰底层 socket,但因为它们操作的是不同方向,不会真正冲突。

**拆开之后就不能再用原来的 `socket` 了**(所有权被移走)。想合回去可以用 `reunite`,但很少需要。

`split` 来自 `StreamExt`,`send` 来自 `SinkExt` —— 所以那行 `use futures_util::{sink::SinkExt, stream::StreamExt};` 两个都要导入。

### tokio::select!:谁先结束就收拾谁

拆成两个任务后,面临一个协调问题:**任一方向出问题,另一个也该停下。**

- 客户端断开 → 接收任务结束 → 发送任务再发也没意义
- 发送任务发完了要关连接 → 接收任务该退出

```rust
tokio::select! {
    rv_a = (&mut send_task) => {
        // send_task 先完成
        recv_task.abort();
    },
    rv_b = (&mut recv_task) => {
        // recv_task 先完成
        send_task.abort();
    }
}
```

**`tokio::select!` 等待多个 future,谁先完成就执行对应分支,其余的被丢弃。**

这里等的是两个 `JoinHandle`(`tokio::spawn` 的返回值),完成意味着那个任务跑完了。然后调 `abort()` 主动取消另一个。

**`&mut send_task`** 是因为 `select!` 会消耗 future,而后面还要用 `send_task.abort()`。取可变引用让所有权留在外面。

**为什么要 `abort()` 而不是让它自己结束?** 因为另一个任务可能正阻塞在 `receiver.next().await` 上,永远等不到消息。不主动取消它会一直挂着,泄露资源。

**这让一个 crate 产出两个可执行文件。** 用途:服务端和客户端共享依赖和类型定义,但分开运行。

运行时用 `--bin` 指定:

```bash
cargo run
cargo run --bin client
```

约定俗成的做法是把额外的二进制放 `src/bin/` 目录下:文件名就是 bin 名,连 `[[bin]]` 声明都不需要。官方示例放在 `src/` 并用 `[[bin]]` 显式声明,是为了自定义 `example-*` 名字;教程项目叫 `websockets`,默认 bin 名就是 `websockets`,`src/bin/client.rs` 自动叫 `client`。

**注意 `src/main.rs` 和 `src/bin/client.rs` 是两个独立的 crate 根**,彼此看不到对方的类型——不是模块关系。

## 项目搭建

```bash
cargo new websockets
cd websockets
```

```bash
cargo add axum --features ws
cargo add axum-extra --features typed-header
cargo add futures-util --no-default-features --features sink,std
cargo add headers
cargo add tokio --features full
cargo add tokio-tungstenite
cargo add tower-http --features fs,trace
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

```bash
mkdir src/bin
```

第二个可执行文件就放这里,`client.rs` 直接写在 `src/` 下不会被编译。

**`axum` 的 `ws` feature 必须显式开启**,默认不带。

`tokio-tungstenite` 是**客户端**用的(`src/bin/client.rs`),服务端不需要——axum 内部已经用了 tungstenite。

### 前端页面

```bash
mkdir assets
```

`assets/index.html`:

```html
<a>Open the console to see stuff, then refresh to initiate exchange.</a>
<script src='script.js'></script>
```

`assets/script.js`:

```js
const socket = new WebSocket('ws://localhost:3000/ws');

socket.addEventListener('open', function (event) {
    socket.send('Hello Server!');
});

socket.addEventListener('message', function (event) {
    console.log('Message from server ', event.data);
});


setTimeout(() => {
    const obj = { hello: "world" };
    const blob = new Blob([JSON.stringify(obj, null, 2)], {
      type: "application/json",
    });
    console.log("Sending blob over websocket");
    socket.send(blob);
}, 1000);

setTimeout(() => {
    socket.send('About done here...');
    console.log("Sending close over websocket");
    socket.close(3000, "Crash and Burn!");
}, 3000);
```

**前端做了四件事**:连上后发一条文本、1 秒后发一个 Blob(二进制)、3 秒后发一条文本并主动关闭(自定义 code `3000`)、以及打印所有收到的消息。

## 分步构建

### 步骤 1:握手与最简单的收发

**要做什么**:实现 `/ws`,完成握手后收发几条消息。

```rust
// src/main.rs
use axum::{
    body::Bytes,
    extract::ws::{Message, WebSocket, WebSocketUpgrade},
    response::IntoResponse,
    routing::any,
    Router,
};
use axum_extra::TypedHeader;
use std::net::SocketAddr;
use std::ops::ControlFlow;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

use axum::extract::connect_info::ConnectInfo;

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

    let app = Router::new().route("/ws", any(ws_handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(
        listener,
        app.into_make_service_with_connect_info::<SocketAddr>(),
    )
    .await
    .unwrap();
}

/// The handler for the HTTP request (this gets called when the HTTP request lands at the start
/// of websocket negotiation). After this completes, the actual switching from HTTP to
/// websocket protocol will occur.
/// This is the last point where we can extract TCP/IP metadata such as IP address of the client
/// as well as things from HTTP headers such as user-agent of the browser etc.
async fn ws_handler(
    ws: WebSocketUpgrade,
    user_agent: Option<TypedHeader<headers::UserAgent>>,
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
) -> impl IntoResponse {
    let user_agent = if let Some(TypedHeader(user_agent)) = user_agent {
        user_agent.to_string()
    } else {
        String::from("Unknown browser")
    };
    println!("`{user_agent}` at {addr} connected.");
    // finalize the upgrade process by returning upgrade callback.
    // we can customize the callback by sending additional info such as address.
    ws.on_upgrade(move |socket| handle_socket(socket, addr))
}

/// Actual websocket statemachine (one will be spawned per connection)
async fn handle_socket(mut socket: WebSocket, who: SocketAddr) {
    // send a ping (unsupported by some browsers) just to kick things off and get a response
    if socket
        .send(Message::Ping(Bytes::from_static(&[1, 2, 3])))
        .await
        .is_ok()
    {
        println!("Pinged {who}...");
    } else {
        println!("Could not send ping {who}!");
        // no Error here since the only thing we can do is to close the connection.
        // If we can not send messages, there is no way to salvage the statemachine anyway.
        return;
    }

    // receive single message from a client (we can either receive or send with socket).
    if let Some(msg) = socket.recv().await {
        if let Ok(msg) = msg {
            if process_message(msg, who).is_break() {
                return;
            }
        } else {
            println!("client {who} abruptly disconnected");
            return;
        }
    }

    for i in 1..5 {
        if socket
            .send(Message::Text(format!("Hi {i} times!").into()))
            .await
            .is_err()
        {
            println!("client {who} abruptly disconnected");
            return;
        }
        tokio::time::sleep(std::time::Duration::from_millis(100)).await;
    }
}

/// helper to print contents of messages to stdout. Has special treatment for Close.
fn process_message(msg: Message, who: SocketAddr) -> ControlFlow<(), ()> {
    match msg {
        Message::Text(t) => {
            println!(">>> {who} sent str: {t:?}");
        }
        Message::Binary(d) => {
            println!(">>> {who} sent {} bytes: {d:?}", d.len());
        }
        Message::Close(c) => {
            if let Some(cf) = c {
                println!(
                    ">>> {who} sent close with code {} and reason `{}`",
                    cf.code, cf.reason
                );
            } else {
                println!(">>> {who} somehow sent close message without CloseFrame");
            }
            return ControlFlow::Break(());
        }

        Message::Pong(v) => {
            println!(">>> {who} sent pong with {v:?}");
        }
        // You should never need to manually handle Message::Ping, as axum's websocket library
        // will do so for you automagically by replying with Pong and copying the v according to
        // spec. But if you need the contents of the pings you can see them here.
        Message::Ping(v) => {
            println!(">>> {who} sent ping with {v:?}");
        }
    }
    ControlFlow::Continue(())
}
```

**`into_make_service_with_connect_info::<SocketAddr>()`** —— `ConnectInfo` 提取器需要真实的连接地址,这个方法把它注入进去。[第 21 章](21-testing.md#mockconnectinfo)讲过测试里怎么模拟它。

**`user_agent: Option<TypedHeader<...>>`** 用 `Option` 包着,因为不是所有客户端都发 `User-Agent`。这是[第 31 章](31-oauth.md#optionalfromrequestparts)那个"可选提取器"的另一种形式——`TypedHeader` 本身支持 `Option` 包装。

**`ws.on_upgrade(move |socket| handle_socket(socket, addr))`** 前置知识讲过时序。`move` 是为了把 `addr` 移进闭包。

**`ControlFlow<(), ()>`** 是标准库类型,表示"继续还是中断"。用它而不是 `bool` 让意图更明确:`is_break()` 比 `if !ok` 可读。

**`socket.send(...)` 的错误处理都是 `return`。** 注释解释了原因:

> If we can not send messages, there is no way to salvage the statemachine anyway.

发送失败意味着连接已经断了,能做的只有退出。

**跑起来看**:

```bash
cargo run
```

用 `websocat` 测试(如果没装:`brew install websocat` 或 `cargo install websocat`):

```bash
echo 'hello from websocat' | websocat ws://127.0.0.1:3000/ws
```

```
Hi 1 times!
Hi 2 times!
Hi 3 times!
Hi 4 times!
```

服务端终端:

```
`Unknown browser` at 127.0.0.1:54321 connected.
Pinged 127.0.0.1:54321...
>>> 127.0.0.1:54321 sent str: "hello from websocat"
```

**没装 `websocat` 也能用 curl 观察握手:**

```bash
curl -i -s -N \
  -H 'Connection: Upgrade' \
  -H 'Upgrade: websocket' \
  -H 'Sec-WebSocket-Version: 13' \
  -H 'Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==' \
  http://127.0.0.1:3000/ws | head -5
```

```
HTTP/1.1 101 Switching Protocols
upgrade: websocket
connection: upgrade
sec-websocket-accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**`101 Switching Protocols`** —— 前置知识讲的握手响应。之后 curl 收到的就是二进制帧了,显示成乱码。

**验证握手校验。** 不带升级头部直接访问:

```bash
curl -i -s http://127.0.0.1:3000/ws | head -3
```

```
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8

Connection header did not include 'upgrade'
```

**`WebSocketUpgrade` 提取器拒绝了它** —— 这就是提取器作为守卫的又一个例子([第 30 章](30-jwt.md#提取器即认证守卫)讲过这个模式)。

### 步骤 2:拆成读写两半并发工作

**要做什么**:用 `split()` 把连接拆开,两个任务并发收发。

```rust
// src/main.rs
use axum::{
    body::Bytes,
    extract::ws::{Message, Utf8Bytes, WebSocket, WebSocketUpgrade},   // 改动:加上 Utf8Bytes
    response::IntoResponse,
    routing::any,
    Router,
};
// ...(其余 use 不变)
use axum::extract::ws::CloseFrame;                                     // 新增

// Allows splitting the websocket stream into separate TX and RX branches
use futures_util::{sink::SinkExt, stream::StreamExt};                  // 新增

// ...(main、ws_handler 不变)

async fn handle_socket(mut socket: WebSocket, who: SocketAddr) {
    // ...(ping、收一条消息、发 4 条问候都不变)

    // 新增:By splitting socket we can send and receive at the same time.
    let (mut sender, mut receiver) = socket.split();

    // Spawn a task that will push several messages to the client (does not matter what client does)
    let mut send_task = tokio::spawn(async move {
        let n_msg = 20;
        for i in 0..n_msg {
            // In case of any websocket error, we exit.
            if sender
                .send(Message::Text(format!("Server message {i} ...").into()))
                .await
                .is_err()
            {
                return i;
            }

            tokio::time::sleep(std::time::Duration::from_millis(300)).await;
        }

        println!("Sending close to {who}...");
        if let Err(e) = sender
            .send(Message::Close(Some(CloseFrame {
                code: axum::extract::ws::close_code::NORMAL,
                reason: Utf8Bytes::from_static("Goodbye"),
            })))
            .await
        {
            println!("Could not send Close due to {e}, probably it is ok?");
        }
        n_msg
    });

    // This second task will receive messages from client and print them on server console
    let mut recv_task = tokio::spawn(async move {
        let mut cnt = 0;
        while let Some(Ok(msg)) = receiver.next().await {
            cnt += 1;
            // print message and break if instructed to do so
            if process_message(msg, who).is_break() {
                break;
            }
        }
        cnt
    });

    // If any one of the tasks exit, abort the other.
    tokio::select! {
        rv_a = (&mut send_task) => {
            match rv_a {
                Ok(a) => println!("{a} messages sent to {who}"),
                Err(a) => println!("Error sending messages {a:?}")
            }
            recv_task.abort();
        },
        rv_b = (&mut recv_task) => {
            match rv_b {
                Ok(b) => println!("Received {b} messages"),
                Err(b) => println!("Error receiving messages {b:?}")
            }
            send_task.abort();
        }
    }

    // returning from the handler closes the websocket connection
    println!("Websocket context {who} destroyed");
}

// ...(process_message 不变)
```

**`socket.split()`** 是本章的核心,前置知识讲过为什么必须拆。

**两个任务的分工:**

- **`send_task`** —— 主动推 20 条消息(每 300ms 一条),完了发 `Close`。**完全不管客户端在做什么**。
- **`recv_task`** —— 循环接收客户端消息并打印,收到 `Close` 就退出。

**`while let Some(Ok(msg)) = receiver.next().await`** 这个模式值得注意:它同时处理了两种结束情况——`None`(流结束,连接关了)和 `Err`(接收出错)。**两种都跳出循环**,因为都意味着连接不可用了。

**两个任务的返回值**分别是"发了多少条"和"收了多少条",`select!` 分支里打印出来。

**`tokio::select!` 加 `abort()`** 前置知识讲过。

**函数返回就关闭连接** —— 注释写着 `returning from the handler closes the websocket connection`。`sender` 和 `receiver` 被 drop 时底层连接也就关了。

**跑起来看**:

```bash
cargo run
```

用 `websocat` 交互测试:

```bash
websocat ws://127.0.0.1:3000/ws
```

输入几条消息(每行一条),同时会看到服务端推来的消息交错出现:

```
Hi 1 times!
Hi 2 times!
Hi 3 times!
Hi 4 times!
Server message 0 ...
hello                      ← 你输入的
Server message 1 ...
Server message 2 ...
world                      ← 你输入的
Server message 3 ...
```

**收发是真正并发的** —— 服务端在推消息的同时也在接收你的输入。这就是 `split()` 的效果。

服务端终端:

```
>>> 127.0.0.1:54322 sent str: "hello"
>>> 127.0.0.1:54322 sent str: "world"
Sending close to 127.0.0.1:54322...
20 messages sent to 127.0.0.1:54322
Websocket context 127.0.0.1:54322 destroyed
```

**验证 `split()` 的必要性。** 临时把发送和接收改成串行(不拆、不 spawn):

```rust
    // 临时:不拆,串行收发
    for i in 0..20 {
        socket.send(Message::Text(format!("Server message {i} ...").into())).await.ok();
        // 想接收就得等
        if let Some(Ok(msg)) = socket.recv().await {
            process_message(msg, who);
        }
        tokio::time::sleep(std::time::Duration::from_millis(300)).await;
    }
```

跑起来会发现:**服务端在 `socket.recv()` 那里卡住了**,因为客户端不一定有消息要发。推送停滞,直到客户端说话才继续。**这就是不拆连接的后果**——收和发互相阻塞。

改回 `split()` 版本。

### 步骤 3:前端页面

**要做什么**:加上静态文件服务,用浏览器测试。

```rust
// src/main.rs
use std::{net::SocketAddr, path::PathBuf};              // 改动:加上 PathBuf
use tower_http::{                                        // 新增
    services::ServeDir,
    trace::{DefaultMakeSpan, TraceLayer},
};
// ...(其余 use 不变)

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    let assets_dir = PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("assets");

    // 改动:加上静态文件服务和日志中间件
    let app = Router::new()
        .fallback_service(ServeDir::new(assets_dir).append_index_html_on_directories(true))
        .route("/ws", any(ws_handler))
        // logging so we can see what's going on
        .layer(
            TraceLayer::new_for_http()
                .make_span_with(DefaultMakeSpan::default().include_headers(true)),
        );

    // ...(listener 和 serve 不变)
}

// ...(其余不变)
```

**`fallback_service(ServeDir::new(...))`** 和[第 33 章](33-sse.md#步骤-3配上前端页面)一样,静态文件兜底。

**`DefaultMakeSpan::default().include_headers(true)`** 让 `TraceLayer` 的 span 带上所有请求头部——**对调试 WebSocket 握手特别有用**,能看到 `Upgrade`、`Sec-WebSocket-*` 这些头部到底发过来了没有。

[第 17 章](17-tracing-aka-logging.md#步骤-2自定义-span)讲 `make_span_with` 时用的是自己写闭包,这里用的是 `tower-http` 提供的默认实现加配置。

> **更好的写法**:`include_headers(true)` 会把**所有**请求头部记进日志,包括 `Cookie` 和 `Authorization`。[第 17 章](17-tracing-aka-logging.md#代价与边界)强调过"日志里不要放敏感信息"——放进去就等于公开。调试时开着没问题,生产环境应该关掉,或者用自定义 span 只记必要的头部。

**跑起来看**:

```bash
cargo run
```

浏览器访问 `http://localhost:3000/`,按 F12 打开控制台,**刷新页面**(脚本在加载时就发起连接):

```
Message from server  Hi 1 times!
Message from server  Hi 2 times!
Message from server  Hi 3 times!
Message from server  Hi 4 times!
Message from server  Server message 0 ...
Sending blob over websocket
Message from server  Server message 1 ...
Message from server  Server message 2 ...
Sending close over websocket
```

服务端终端:

```
`Mozilla/5.0 (...)` at 127.0.0.1:54323 connected.
Pinged 127.0.0.1:54323...
>>> 127.0.0.1:54323 sent str: "Hello Server!"
>>> 127.0.0.1:54323 sent 28 bytes: b"{\n  \"hello\": \"world\"\n}"
>>> 127.0.0.1:54323 sent str: "About done here..."
>>> 127.0.0.1:54323 sent close with code 3000 and reason `Crash and Burn!`
Received 4 messages
Websocket context 127.0.0.1:54323 destroyed
```

**四种消息都出现了**:文本、二进制(那个 Blob)、又一条文本、以及带自定义 code `3000` 的关闭帧。

**注意 `Pong` 没有出现在日志里。** 服务端发了 `Ping`,但浏览器的 `Pong` 回复被 tungstenite 在协议层处理掉了(而且浏览器的 WebSocket API 根本不暴露 ping/pong)。前置知识讲的"不用手动处理"在这里体现了。

**看 Network 标签。** 找到 `ws` 请求,状态是 `101 Switching Protocols`,点进去有个 **Messages** 标签,能看到每一帧的方向、内容、时间。**这是调试 WebSocket 的主要工具。**

### 步骤 4:Rust 客户端

**要做什么**:写 `src/bin/client.rs`,用两条并发连接测试服务端。

```rust
// src/bin/client.rs
use futures_util::{SinkExt, StreamExt};
use std::ops::ControlFlow;
use std::time::Instant;
use tokio::task::JoinSet;
use tokio_tungstenite::tungstenite::Utf8Bytes;

// we will use tungstenite for websocket client impl (same library as what axum is using)
use tokio_tungstenite::{
    connect_async,
    tungstenite::protocol::{frame::coding::CloseCode, CloseFrame, Message},
};

const N_CLIENTS: usize = 2; //set to desired number
const SERVER: &str = "ws://127.0.0.1:3000/ws";

#[tokio::main]
async fn main() {
    let start_time = Instant::now();
    //spawn several clients that will concurrently talk to the server
    let mut clients = (0..N_CLIENTS).map(spawn_client).collect::<JoinSet<_>>();

    //wait for all our clients to exit
    while clients.join_next().await.is_some() {}

    let end_time = Instant::now();

    //total time should be the same no matter how many clients we spawn
    println!(
        "Total time taken {:#?} with {N_CLIENTS} concurrent clients, should be about 6.45 seconds.",
        end_time - start_time
    );
}

//creates a client. quietly exits on failure.
async fn spawn_client(who: usize) {
    let ws_stream = match connect_async(SERVER).await {
        Ok((stream, response)) => {
            println!("Handshake for client {who} has been completed");
            // This will be the HTTP response, same as with server this is the last moment we
            // can still access HTTP stuff.
            println!("Server response was {response:?}");
            stream
        }
        Err(e) => {
            println!("WebSocket handshake for client {who} failed with {e}!");
            return;
        }
    };

    let (mut sender, mut receiver) = ws_stream.split();

    //we can ping the server for start
    sender
        .send(Message::Ping(axum::body::Bytes::from_static(
            b"Hello, Server!",
        )))
        .await
        .expect("Can not send!");

    //spawn an async sender to push some more messages into the server
    let mut send_task = tokio::spawn(async move {
        for i in 1..30 {
            // In any websocket error, break loop.
            if sender
                .send(Message::Text(format!("Message number {i}...").into()))
                .await
                .is_err()
            {
                //just as with server, if send fails there is nothing we can do but exit.
                return;
            }

            tokio::time::sleep(std::time::Duration::from_millis(300)).await;
        }

        // When we are done we may want our client to close connection cleanly.
        println!("Sending close to {who}...");
        if let Err(e) = sender
            .send(Message::Close(Some(CloseFrame {
                code: CloseCode::Normal,
                reason: Utf8Bytes::from_static("Goodbye"),
            })))
            .await
        {
            println!("Could not send Close due to {e:?}, probably it is ok?");
        };
    });

    //receiver just prints whatever it gets
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(msg)) = receiver.next().await {
            // print message and break if instructed to do so
            if process_message(msg, who).is_break() {
                break;
            }
        }
    });

    //wait for either task to finish and kill the other task
    tokio::select! {
        _ = (&mut send_task) => {
            recv_task.abort();
        },
        _ = (&mut recv_task) => {
            send_task.abort();
        }
    }
}

/// Function to handle messages we get (with a slight twist that Frame variant is visible
/// since we are working with the underlying tungstenite library directly without axum here).
fn process_message(msg: Message, who: usize) -> ControlFlow<(), ()> {
    match msg {
        Message::Text(t) => {
            println!(">>> {who} got str: {t:?}");
        }
        Message::Binary(d) => {
            println!(">>> {who} got {} bytes: {d:?}", d.len());
        }
        Message::Close(c) => {
            if let Some(cf) = c {
                println!(
                    ">>> {who} got close with code {} and reason `{}`",
                    cf.code, cf.reason
                );
            } else {
                println!(">>> {who} somehow got close message without CloseFrame");
            }
            return ControlFlow::Break(());
        }

        Message::Pong(v) => {
            println!(">>> {who} got pong with {v:?}");
        }
        // Just as with axum server, the underlying tungstenite websocket library
        // will handle Ping for you automagically by replying with Pong and copying the
        // v according to spec. But if you need the contents of the pings you can see them here.
        Message::Ping(v) => {
            println!(">>> {who} got ping with {v:?}");
        }

        Message::Frame(_) => {
            unreachable!("This is never supposed to happen")
        }
    }
    ControlFlow::Continue(())
}
```

**注意客户端代码和服务端结构惊人地相似**:都是 `split()` + 两个任务 + `select!`。官方注释也提到这一点:

> how the client-side and server-side code can be quite similar

**`connect_async(SERVER)`** 返回 `(stream, response)`。那个 `response` 是握手的 HTTP 响应——**客户端侧最后能访问 HTTP 信息的时刻**,和服务端的 `ws_handler` 对应。

**`JoinSet`** 是 tokio 的任务集合,方便管理一批任务:

```rust
let mut clients = (0..N_CLIENTS).map(spawn_client).collect::<JoinSet<_>>();
while clients.join_next().await.is_some() {}
```

比手动收集 `JoinHandle` 再逐个 `await` 干净。

**`Message::Frame(_)` 分支只在客户端出现。** 因为这里直接用 tungstenite,能看到底层帧类型;axum 的 `Message` 枚举没有这个变体。注释里的 `unreachable!` 说明它在正常收发中不会出现(只在手动构造帧时用)。

**跑起来看**:

先启动服务端:

```bash
cargo run
```

另开一个终端跑客户端:

```bash
cargo run --bin client
```

客户端输出(两个客户端交错):

```
Handshake for client 0 has been completed
Server response was Response { status: 101, ... }
Handshake for client 1 has been completed
>>> 0 got pong with b"Hello, Server!"
>>> 0 got str: "Hi 1 times!"
>>> 1 got str: "Hi 1 times!"
...
>>> 0 got close with code Normal and reason `Goodbye`
Total time taken 6.451s with 2 concurrent clients, should be about 6.45 seconds.
```

**注意最后那句"无论开多少客户端总时间都差不多"** —— 这是并发的证明。服务端为每条连接跑一个独立的状态机,互不阻塞。

**试试把 `N_CLIENTS` 改成 10**,总时间应该基本不变:

```rust
const N_CLIENTS: usize = 10;
```

**服务端的 `Pong` 也出现了** —— 客户端发的 `Ping` 被服务端的 tungstenite 自动回复,客户端在 `process_message` 里看到了那个 `Pong`。

## 完整代码

`assets/index.html`:

```html
<a>Open the console to see stuff, then refresh to initiate exchange.</a>
<script src='script.js'></script>
```

`assets/script.js`:

```js
const socket = new WebSocket('ws://localhost:3000/ws');

socket.addEventListener('open', function (event) {
    socket.send('Hello Server!');
});

socket.addEventListener('message', function (event) {
    console.log('Message from server ', event.data);
});


setTimeout(() => {
    const obj = { hello: "world" };
    const blob = new Blob([JSON.stringify(obj, null, 2)], {
      type: "application/json",
    });
    console.log("Sending blob over websocket");
    socket.send(blob);
}, 1000);

setTimeout(() => {
    socket.send('About done here...');
    console.log("Sending close over websocket");
    socket.close(3000, "Crash and Burn!");
}, 3000);
```

`src/main.rs`:

```rust
use axum::{
    body::Bytes,
    extract::ws::{Message, Utf8Bytes, WebSocket, WebSocketUpgrade},
    response::IntoResponse,
    routing::any,
    Router,
};
use axum_extra::TypedHeader;

use std::ops::ControlFlow;
use std::{net::SocketAddr, path::PathBuf};
use tower_http::{
    services::ServeDir,
    trace::{DefaultMakeSpan, TraceLayer},
};

use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// Allows extracting the IP of the connecting user
use axum::extract::connect_info::ConnectInfo;
use axum::extract::ws::CloseFrame;

// Allows splitting the websocket stream into separate TX and RX branches
use futures_util::{sink::SinkExt, stream::StreamExt};

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

    let assets_dir = PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("assets");

    // build our application with some routes
    let app = Router::new()
        .fallback_service(ServeDir::new(assets_dir).append_index_html_on_directories(true))
        .route("/ws", any(ws_handler))
        // logging so we can see what's going on
        .layer(
            TraceLayer::new_for_http()
                .make_span_with(DefaultMakeSpan::default().include_headers(true)),
        );

    // run it with hyper
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(
        listener,
        app.into_make_service_with_connect_info::<SocketAddr>(),
    )
    .await
    .unwrap();
}

/// The handler for the HTTP request (this gets called when the HTTP request lands at the start
/// of websocket negotiation). After this completes, the actual switching from HTTP to
/// websocket protocol will occur.
/// This is the last point where we can extract TCP/IP metadata such as IP address of the client
/// as well as things from HTTP headers such as user-agent of the browser etc.
async fn ws_handler(
    ws: WebSocketUpgrade,
    user_agent: Option<TypedHeader<headers::UserAgent>>,
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
) -> impl IntoResponse {
    let user_agent = if let Some(TypedHeader(user_agent)) = user_agent {
        user_agent.to_string()
    } else {
        String::from("Unknown browser")
    };
    println!("`{user_agent}` at {addr} connected.");
    // finalize the upgrade process by returning upgrade callback.
    // we can customize the callback by sending additional info such as address.
    ws.on_upgrade(move |socket| handle_socket(socket, addr))
}

/// Actual websocket statemachine (one will be spawned per connection)
async fn handle_socket(mut socket: WebSocket, who: SocketAddr) {
    // send a ping (unsupported by some browsers) just to kick things off and get a response
    if socket
        .send(Message::Ping(Bytes::from_static(&[1, 2, 3])))
        .await
        .is_ok()
    {
        println!("Pinged {who}...");
    } else {
        println!("Could not send ping {who}!");
        // no Error here since the only thing we can do is to close the connection.
        // If we can not send messages, there is no way to salvage the statemachine anyway.
        return;
    }

    // receive single message from a client (we can either receive or send with socket).
    // this will likely be the Pong for our Ping or a hello message from client.
    // waiting for message from a client will block this task, but will not block other client's
    // connections.
    if let Some(msg) = socket.recv().await {
        if let Ok(msg) = msg {
            if process_message(msg, who).is_break() {
                return;
            }
        } else {
            println!("client {who} abruptly disconnected");
            return;
        }
    }

    // Since each client gets individual statemachine, we can pause handling
    // when necessary to wait for some external event (in this case illustrated by sleeping).
    // Waiting for this client to finish getting its greetings does not prevent other clients from
    // connecting to server and receiving their greetings.
    for i in 1..5 {
        if socket
            .send(Message::Text(format!("Hi {i} times!").into()))
            .await
            .is_err()
        {
            println!("client {who} abruptly disconnected");
            return;
        }
        tokio::time::sleep(std::time::Duration::from_millis(100)).await;
    }

    // By splitting socket we can send and receive at the same time. In this example we will send
    // unsolicited messages to client based on some sort of server's internal event (i.e .timer).
    let (mut sender, mut receiver) = socket.split();

    // Spawn a task that will push several messages to the client (does not matter what client does)
    let mut send_task = tokio::spawn(async move {
        let n_msg = 20;
        for i in 0..n_msg {
            // In case of any websocket error, we exit.
            if sender
                .send(Message::Text(format!("Server message {i} ...").into()))
                .await
                .is_err()
            {
                return i;
            }

            tokio::time::sleep(std::time::Duration::from_millis(300)).await;
        }

        println!("Sending close to {who}...");
        if let Err(e) = sender
            .send(Message::Close(Some(CloseFrame {
                code: axum::extract::ws::close_code::NORMAL,
                reason: Utf8Bytes::from_static("Goodbye"),
            })))
            .await
        {
            println!("Could not send Close due to {e}, probably it is ok?");
        }
        n_msg
    });

    // This second task will receive messages from client and print them on server console
    let mut recv_task = tokio::spawn(async move {
        let mut cnt = 0;
        while let Some(Ok(msg)) = receiver.next().await {
            cnt += 1;
            // print message and break if instructed to do so
            if process_message(msg, who).is_break() {
                break;
            }
        }
        cnt
    });

    // If any one of the tasks exit, abort the other.
    tokio::select! {
        rv_a = (&mut send_task) => {
            match rv_a {
                Ok(a) => println!("{a} messages sent to {who}"),
                Err(a) => println!("Error sending messages {a:?}")
            }
            recv_task.abort();
        },
        rv_b = (&mut recv_task) => {
            match rv_b {
                Ok(b) => println!("Received {b} messages"),
                Err(b) => println!("Error receiving messages {b:?}")
            }
            send_task.abort();
        }
    }

    // returning from the handler closes the websocket connection
    println!("Websocket context {who} destroyed");
}

/// helper to print contents of messages to stdout. Has special treatment for Close.
fn process_message(msg: Message, who: SocketAddr) -> ControlFlow<(), ()> {
    match msg {
        Message::Text(t) => {
            println!(">>> {who} sent str: {t:?}");
        }
        Message::Binary(d) => {
            println!(">>> {who} sent {} bytes: {d:?}", d.len());
        }
        Message::Close(c) => {
            if let Some(cf) = c {
                println!(
                    ">>> {who} sent close with code {} and reason `{}`",
                    cf.code, cf.reason
                );
            } else {
                println!(">>> {who} somehow sent close message without CloseFrame");
            }
            return ControlFlow::Break(());
        }

        Message::Pong(v) => {
            println!(">>> {who} sent pong with {v:?}");
        }
        // You should never need to manually handle Message::Ping, as axum's websocket library
        // will do so for you automagically by replying with Pong and copying the v according to
        // spec. But if you need the contents of the pings you can see them here.
        Message::Ping(v) => {
            println!(">>> {who} sent ping with {v:?}");
        }
    }
    ControlFlow::Continue(())
}
```

`src/bin/client.rs`:

```rust
use futures_util::{SinkExt, StreamExt};
use std::ops::ControlFlow;
use std::time::Instant;
use tokio::task::JoinSet;
use tokio_tungstenite::tungstenite::Utf8Bytes;

// we will use tungstenite for websocket client impl (same library as what axum is using)
use tokio_tungstenite::{
    connect_async,
    tungstenite::protocol::{frame::coding::CloseCode, CloseFrame, Message},
};

const N_CLIENTS: usize = 2; //set to desired number
const SERVER: &str = "ws://127.0.0.1:3000/ws";

#[tokio::main]
async fn main() {
    let start_time = Instant::now();
    //spawn several clients that will concurrently talk to the server
    let mut clients = (0..N_CLIENTS).map(spawn_client).collect::<JoinSet<_>>();

    //wait for all our clients to exit
    while clients.join_next().await.is_some() {}

    let end_time = Instant::now();

    //total time should be the same no matter how many clients we spawn
    println!(
        "Total time taken {:#?} with {N_CLIENTS} concurrent clients, should be about 6.45 seconds.",
        end_time - start_time
    );
}

//creates a client. quietly exits on failure.
async fn spawn_client(who: usize) {
    let ws_stream = match connect_async(SERVER).await {
        Ok((stream, response)) => {
            println!("Handshake for client {who} has been completed");
            // This will be the HTTP response, same as with server this is the last moment we
            // can still access HTTP stuff.
            println!("Server response was {response:?}");
            stream
        }
        Err(e) => {
            println!("WebSocket handshake for client {who} failed with {e}!");
            return;
        }
    };

    let (mut sender, mut receiver) = ws_stream.split();

    //we can ping the server for start
    sender
        .send(Message::Ping(axum::body::Bytes::from_static(
            b"Hello, Server!",
        )))
        .await
        .expect("Can not send!");

    //spawn an async sender to push some more messages into the server
    let mut send_task = tokio::spawn(async move {
        for i in 1..30 {
            // In any websocket error, break loop.
            if sender
                .send(Message::Text(format!("Message number {i}...").into()))
                .await
                .is_err()
            {
                //just as with server, if send fails there is nothing we can do but exit.
                return;
            }

            tokio::time::sleep(std::time::Duration::from_millis(300)).await;
        }

        // When we are done we may want our client to close connection cleanly.
        println!("Sending close to {who}...");
        if let Err(e) = sender
            .send(Message::Close(Some(CloseFrame {
                code: CloseCode::Normal,
                reason: Utf8Bytes::from_static("Goodbye"),
            })))
            .await
        {
            println!("Could not send Close due to {e:?}, probably it is ok?");
        };
    });

    //receiver just prints whatever it gets
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(msg)) = receiver.next().await {
            // print message and break if instructed to do so
            if process_message(msg, who).is_break() {
                break;
            }
        }
    });

    //wait for either task to finish and kill the other task
    tokio::select! {
        _ = (&mut send_task) => {
            recv_task.abort();
        },
        _ = (&mut recv_task) => {
            send_task.abort();
        }
    }
}

/// Function to handle messages we get (with a slight twist that Frame variant is visible
/// since we are working with the underlying tungstenite library directly without axum here).
fn process_message(msg: Message, who: usize) -> ControlFlow<(), ()> {
    match msg {
        Message::Text(t) => {
            println!(">>> {who} got str: {t:?}");
        }
        Message::Binary(d) => {
            println!(">>> {who} got {} bytes: {d:?}", d.len());
        }
        Message::Close(c) => {
            if let Some(cf) = c {
                println!(
                    ">>> {who} got close with code {} and reason `{}`",
                    cf.code, cf.reason
                );
            } else {
                println!(">>> {who} somehow got close message without CloseFrame");
            }
            return ControlFlow::Break(());
        }

        Message::Pong(v) => {
            println!(">>> {who} got pong with {v:?}");
        }
        // Just as with axum server, the underlying tungstenite websocket library
        // will handle Ping for you automagically by replying with Pong and copying the
        // v according to spec. But if you need the contents of the pings you can see them here.
        Message::Ping(v) => {
            println!(">>> {who} got ping with {v:?}");
        }

        Message::Frame(_) => {
            unreachable!("This is never supposed to happen")
        }
    }
    ControlFlow::Continue(())
}
```

## 代价与边界

**心跳要自己实现。** 这是 WebSocket 相对 SSE 最大的额外负担。

[第 33 章](33-sse.md#心跳为什么是必需的)讲过 axum 的 `Sse` 有内置的 `keep_alive`。**WebSocket 没有这个** —— 你得自己起一个定时任务周期发 `Ping`,并且**记录上次收到 `Pong` 的时间**,超时就主动关闭。

为什么必须这么做:TCP 连接可能变成**半开**(half-open)——一端已经断了(拔网线、进程被杀),另一端却毫不知情,`recv()` 会永远挂着。只有主动 `Ping` 才能发现。

本章示例只在开头 ping 了一次,**没有持续心跳** —— 生产环境不能这样。

**没有认证。** `/ws` 对所有人开放。前置知识强调过认证必须在握手阶段做完,做法是在 `ws_handler` 里加认证提取器(比如[第 30 章](30-jwt.md#提取器即认证守卫)的 `Claims`),失败就不调用 `on_upgrade`。

**没有校验 Origin。** WebSocket **不受同源策略约束**——[第 20 章](20-cors.md#同源策略浏览器的基本安全边界)讲的那套对它不适用!任何网站的 JS 都能连你的 WebSocket 端点,而且**浏览器会自动带上 cookie**。这叫 **Cross-Site WebSocket Hijacking**。

防御办法是在握手时检查 `Origin` 头部,只允许你自己的域名。**这是 WebSocket 特有的、极容易遗漏的安全问题。**

**没有消息大小限制。** 客户端可以发一个 1GB 的帧。axum 的 `WebSocketUpgrade` 支持 `max_message_size()` 和 `max_frame_size()` 配置,示例没设。

**没有限流。** 客户端可以每秒发一万条消息。需要自己实现速率限制。

**每条连接一个任务对 + 一个状态机。** 10000 个连接就是 20000 个 tokio 任务。这在 tokio 下可行,但要注意内存开销和文件描述符上限——和[第 33 章](33-sse.md#代价与边界)一样的考虑。

**水平扩展同样需要消息中间件。** 和 SSE 一样:用户连在实例 1,消息产生在实例 2。[第 37 章](37-chat.md)的聊天室会遇到这个问题的简化版(单实例内广播),真正的跨实例方案需要 Redis Pub/Sub 之类。

**`include_headers(true)` 会记录敏感头部。** 步骤 3 的「更好的写法」指出了。

**客户端断开的检测靠 `recv()` 返回 `None`/`Err`。** 但半开连接下这不会发生,回到上面第一条——必须有心跳。

## 动手练习

**1. 加上持续心跳和超时检测。**

提示:这是「代价与边界」里最该补的一条。在 `handle_socket` 里再 spawn 一个任务,每 30 秒发一个 `Ping`;同时用一个 `Arc<AtomicU64>` 记录上次收到任何消息的时间戳(在 `recv_task` 里更新),心跳任务检查距离上次超过 90 秒就 `abort` 另外两个任务。验证:连上后拔掉网线(或用 `kill -STOP` 暂停客户端进程),看服务端能否在 90 秒后检测到并清理。

**2. 校验 Origin 头部,防 Cross-Site WebSocket Hijacking。**

提示:在 `ws_handler` 里提取 `TypedHeader<headers::Origin>`,检查是否是 `http://localhost:3000`,不是就返回 `403` 而不调用 `on_upgrade`。验证:从另一个源的页面尝试连接(可以用[第 20 章](20-cors.md#步骤-1在一个进程里跑两个服务)那个双服务的技巧起一个 4000 端口的页面),应该被拒。**这是 WebSocket 特有的安全问题,而且极容易遗漏。**

**3. 加上认证。**

提示:把[第 30 章](30-jwt.md)的 `Claims` 提取器加到 `ws_handler` 的参数里。你会立刻遇到一个实际问题:**浏览器的 `WebSocket` 构造函数不能自定义请求头**(和[第 33 章](33-sse.md#eventsource-与自动重连)的 `EventSource` 一样)。三个绕法:用 cookie、把 token 放查询参数、或者连上后先发一条认证消息再决定是否继续。分别想想它们的取舍。

## 小结与下一章预告

WebSocket 用 HTTP 握手(`101 Switching Protocols`),**握手之后这条连接上跑的就不是 HTTP 了**。这意味着**认证、限流、Origin 检查必须在握手阶段做完** —— `ws_handler` 是最后能访问 HTTP 信息的地方,`on_upgrade` 的回调在升级之后才跑。

六种消息帧里,`Ping` 和 `Close` 的协议层应答由 axum 底层的 tungstenite 自动处理,你只需要**观察**它们、并在收到 `Close` 时退出循环。

**`split()` 是本章的核心**:`WebSocket` 同时是 `Stream` 和 `Sink`,直接用会因为都需要可变借用而无法并发。拆成 `sender`/`receiver` 后可以移进不同任务真正并发收发——本章验证过不拆的后果:服务端会卡在 `recv()` 上,推送停滞。

`tokio::select!` 配合 `abort()` 协调两个任务:任一方向结束就取消另一个,避免那个挂在 `recv()` 上的任务永远泄露。

**心跳要自己实现,这是 WebSocket 相对 SSE 最大的额外负担。** [第 33 章](33-sse.md#心跳为什么是必需的)的 `Sse` 有内置 `keep_alive`,WebSocket 没有——而 TCP 半开连接下 `recv()` 会永远挂着,只有主动 `Ping` 才能发现。

还有一个极易遗漏的安全问题:**WebSocket 不受同源策略约束**,任何网站的 JS 都能连你的端点而且浏览器会带上 cookie(Cross-Site WebSocket Hijacking)。必须在握手时校验 `Origin`。

顺带认识了一个 Cargo 技巧:`src/bin/` 下的每个文件自动成为一个可执行文件,服务端和客户端共享依赖但分开运行。

WebSocket 的收发是异步、双向、并发的,这让它比普通 handler 难测得多——不能像[第 21 章](21-testing.md#oneshot-与-readycall)那样 `oneshot` 发一个请求断言响应。[第 35 章 testing-websockets](35-testing-websockets.md) 讲怎么测:内存里直接驱动 `WebSocket` 的办法、以及什么时候不得不起真实服务器。
