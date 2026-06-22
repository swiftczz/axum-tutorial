# 41. websockets

对应示例：`examples/websockets`

上一章 SSE 是服务端单向推送。这章 WebSocket 是**双向通信**（浏览器 ↔ 服务端）。我们分 4 步从零搭出来。

相比上一章新引入：`WebSocketUpgrade`（HTTP→WebSocket 升级）、`Message`（消息类型）、`socket.split()`（拆分收发）、`tokio::select!`（并发收发）。

## Cargo.toml

````toml
[package]
name = "example-websockets"
version = "0.1.0"
edition = "2024"
publish = false

[[bin]]
name = "example-websockets"
path = "src/main.rs"

[[bin]]
name = "example-client"
path = "src/client.rs"

[dependencies]
axum = { version = "0.8", features = ["ws"] }
axum-extra = { version = "0.12", features = ["typed-header"] }
futures-util = { version = "0.3", default-features = false, features = ["sink", "std"] }
headers = "0.4"
tokio = { version = "1.0", features = ["full"] }
tokio-tungstenite = "0.29.0"
tower-http = { version = "0.6.1", features = ["fs", "trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

启用 axum 的 `ws` feature 才能用 `WebSocketUpgrade`。两个 `[[bin]]` 分别是服务端和 Rust 客户端示例。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：HTTP upgrade——最小 WebSocket 服务

WebSocket 不是凭空建立的。客户端先发一个 HTTP 请求要求"升级协议"，服务端同意后切换到 WebSocket。所以代码分两阶段：`ws_handler` 处理升级前的 HTTP 请求，`handle_socket` 处理升级后的 WebSocket 连接。

````rust
use axum::{
    extract::ws::{WebSocket, WebSocketUpgrade},
    response::IntoResponse,
    routing::any,
    Router,
};
use axum::extract::connect_info::ConnectInfo;
use axum_extra::TypedHeader;
use std::net::SocketAddr;

#[tokio::main]
async fn main() {
    let app = Router::new().route("/ws", any(ws_handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await.unwrap();
    axum::serve(
        listener,
        app.into_make_service_with_connect_info::<SocketAddr>(),
    )
    .await;
}

async fn ws_handler(
    ws: WebSocketUpgrade,
    user_agent: Option<TypedHeader<headers::UserAgent>>,
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
) -> impl IntoResponse {
    let user_agent = if let Some(TypedHeader(ua)) = user_agent {
        ua.to_string()
    } else {
        String::from("Unknown browser")
    };
    println!("`{user_agent}` at {addr} connected.");

    // upgrade 成功后，把 WebSocket 交给 handle_socket
    ws.on_upgrade(move |socket| handle_socket(socket, addr))
}

async fn handle_socket(socket: WebSocket, who: SocketAddr) {
    println!("WebSocket connection from {who} established!");
    // 连接建立后什么都不做，连接会保持直到客户端断开
}
````

> **新面孔：`WebSocketUpgrade` + `on_upgrade`**
>
> `WebSocketUpgrade` 是 axum 的提取器。当客户端发来 WebSocket 升级请求时，axum 通过它完成 HTTP→WebSocket 协议切换。`ws.on_upgrade(closure)` 表示升级成功后把 `WebSocket` 交给处理函数。
>
> **新面孔：`ConnectInfo` + `into_make_service_with_connect_info`**
>
> `ConnectInfo(addr): ConnectInfo<SocketAddr>` 提取客户端 IP 地址。必须在 `axum::serve` 里用 `into_make_service_with_connect_info::<SocketAddr>()`，否则拿不到地址。

---

## 第二步：消息收发——Ping、文本、Close

连接建立后，可以收发消息。WebSocket 有 5 种消息类型：

````rust
use axum::{body::Bytes, extract::ws::{Message, CloseFrame, Utf8Bytes}};
use std::ops::ControlFlow;

async fn handle_socket(mut socket: WebSocket, who: SocketAddr) {
    // 1. 先发一个 Ping，测试连接是否通
    if socket.send(Message::Ping(Bytes::from_static(&[1, 2, 3]))).await.is_ok() {
        println!("Pinged {who}...");
    } else {
        println!("Could not send ping {who}!");
        return;
    }

    // 2. 接收一条消息
    if let Some(msg) = socket.recv().await {
        if let Ok(msg) = msg {
            if process_message(msg, who).is_break() {
                return;  // 收到 Close，退出
            }
        } else {
            println!("client {who} abruptly disconnected");
            return;
        }
    }

    // 3. 连发几条欢迎消息
    for i in 1..5 {
        if socket.send(Message::Text(format!("Hi {i} times!").into())).await.is_err() {
            println!("client {who} abruptly disconnected");
            return;
        }
        tokio::time::sleep(std::time::Duration::from_millis(100)).await;
    }
}

fn process_message(msg: Message, who: SocketAddr) -> ControlFlow<(), ()> {
    match msg {
        Message::Text(t) => println!(">>> {who} sent str: {t:?}"),
        Message::Binary(d) => println!(">>> {who} sent {} bytes: {d:?}", d.len()),
        Message::Close(c) => {
            if let Some(cf) = c {
                println!(">>> {who} sent close with code {} and reason `{}`", cf.code, cf.reason);
            }
            return ControlFlow::Break(());  // Close → 退出
        }
        Message::Pong(v) => println!(">>> {who} sent pong with {v:?}"),
        Message::Ping(v) => println!(">>> {who} sent ping with {v:?}"),
    }
    ControlFlow::Continue(())
}
````

> **新面孔：`Message` 类型 + `ControlFlow`**
>
> WebSocket 消息有 5 种：`Text`（文本）、`Binary`（二进制）、`Close`（关闭）、`Ping`/`Pong`（心跳检测）。
>
> `socket.recv()` 和 `socket.send()` 分别收发单条消息。`recv()` 返回 `Option<Result<Message, _>>`——`None` 表示连接关闭。
>
> `ControlFlow::Break` 表示收到 Close 应退出读取循环，`Continue` 表示继续处理。

---

## 第三步：split + 并发收发

到目前为止发送和接收都在同一个流程里串行执行。真实场景需要**同时收发**——后台持续发消息，同时接收客户端消息。

`socket.split()` 把 WebSocket 拆成 `sender` 和 `receiver` 两半，用两个 `tokio::spawn` 任务分别处理：

````rust
use futures_util::{sink::SinkExt, stream::StreamExt};

// split 后可以同时收发
let (mut sender, mut receiver) = socket.split();

// 任务一：每 300ms 发一条消息，发 20 条后发 Close
let mut send_task = tokio::spawn(async move {
    let n_msg = 20;
    for i in 0..n_msg {
        if sender.send(Message::Text(format!("Server message {i} ...").into())).await.is_err() {
            return i;  // 发送失败，客户端可能断了
        }
        tokio::time::sleep(std::time::Duration::from_millis(300)).await;
    }

    println!("Sending close to {who}...");
    let _ = sender.send(Message::Close(Some(CloseFrame {
        code: axum::extract::ws::close_code::NORMAL,
        reason: Utf8Bytes::from_static("Goodbye"),
    }))).await;
    n_msg
});

// 任务二：持续接收客户端消息
let mut recv_task = tokio::spawn(async move {
    let mut cnt = 0;
    while let Some(Ok(msg)) = receiver.next().await {
        cnt += 1;
        if process_message(msg, who).is_break() {
            break;
        }
    }
    cnt
});

// 任意一边结束就取消另一边
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
````

> **新面孔：`split` + `SinkExt`/`StreamExt` + `tokio::select!`**
>
> `socket.split()` 把 WebSocket 拆成 sender（只发）和 receiver（只收），这样可以两个任务同时工作。
>
> `SinkExt` 提供 `sender.send()`，`StreamExt` 提供 `receiver.next()`。
>
> `tokio::select!` 同时等待多个 future，谁先完成就执行对应分支。发送或接收任意一侧结束，就 abort 另一个——WebSocket 连接生命周期该收尾了。

---

## 完整代码

````rust
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

use axum::extract::connect_info::ConnectInfo;
use axum::extract::ws::CloseFrame;

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

    let app = Router::new()
        .fallback_service(ServeDir::new(assets_dir).append_index_html_on_directories(true))
        .route("/ws", any(ws_handler))
        .layer(
            TraceLayer::new_for_http()
                .make_span_with(DefaultMakeSpan::default().include_headers(true)),
        );

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(
        listener,
        app.into_make_service_with_connect_info::<SocketAddr>(),
    )
    .await;
}

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

    ws.on_upgrade(move |socket| handle_socket(socket, addr))
}

async fn handle_socket(mut socket: WebSocket, who: SocketAddr) {
    if socket
        .send(Message::Ping(Bytes::from_static(&[1, 2, 3])))
        .await
        .is_ok()
    {
        println!("Pinged {who}...");
    } else {
        println!("Could not send ping {who}!");
        return;
    }

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

    let (mut sender, mut receiver) = socket.split();

    let mut send_task = tokio::spawn(async move {
        let n_msg = 20;
        for i in 0..n_msg {
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

    let mut recv_task = tokio::spawn(async move {
        let mut cnt = 0;
        while let Some(Ok(msg)) = receiver.next().await {
            cnt += 1;
            if process_message(msg, who).is_break() {
                break;
            }
        }
        cnt
    });

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

    println!("Websocket context {who} destroyed");
}

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
        Message::Ping(v) => {
            println!(">>> {who} sent ping with {v:?}");
        }
    }
    ControlFlow::Continue(())
}
````

> 上面是完整 main.rs。配套的 `src/client.rs`（Rust 客户端示例）、`assets/index.html`、`assets/script.js` 见源码对照。

## 运行

````bash
cargo run -p example-websockets --bin example-websockets   # 服务端
````

浏览器打开 `http://localhost:3000`，打开 Console 观察服务端消息。或运行 Rust 客户端：

````bash
cargo run -p example-websockets --bin example-client
````

## ⚠️ 生产级提示：心跳与超时

本示例只在连接建立时发了一个 Ping，没有周期性心跳和超时。生产环境会引发**半开连接泄漏**——客户端断网后 TCP 层没正常关闭，服务端 socket 还"活着"一直等消息。

标准做法：周期性发 Ping（如每 30 秒）+ 给 recv 加超时（60 秒收不到消息就断）。详见手写任务。

## 手写任务

1. 写 `/ws` 只完成 `WebSocketUpgrade`。
2. 写 `handle_socket` 连接后发一条文本。
3. 接收一条客户端消息并打印。
4. 加 `socket.split()`，用两个任务分别发送和接收。
5. 浏览器 `new WebSocket(...)` 验证。
6. **（进阶）**加周期性 Ping + recv 超时防半开连接。

## 小结

这章分 4 止从零搭了 WebSocket 服务：

1. **HTTP upgrade**：`WebSocketUpgrade` + `on_upgrade` 完成 HTTP→WebSocket 切换，`ConnectInfo` 提取客户端 IP。
2. **消息收发**：`Message` 有 5 种类型（Text/Binary/Close/Ping/Pong），`socket.recv()`/`socket.send()` 收发。
3. **split + 并发**：`socket.split()` 拆成 sender/receiver，两个 `tokio::spawn` 同时收发，`tokio::select!` 谁先结束 abort 另一个。

WebSocket vs SSE：双向实时用 WebSocket，单向推送用 SSE。

## 源码对照

- `examples/websockets/Cargo.toml`
- `examples/websockets/src/main.rs`
- `examples/websockets/src/client.rs`
- `examples/websockets/assets/index.html`
- `examples/websockets/assets/script.js`
