# 41. websockets

对应示例：`examples/websockets`

上一章 SSE 是服务端单向推送,这章 WebSocket 是**双向通信**(浏览器 ↔ 服务端)。理解 WebSocket 的 HTTP upgrade、连接状态机、消息类型、并发收发,以及 axum 的 `WebSocketUpgrade` 和 `WebSocket` 用法。

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

## src/main.rs

````rust
use axum::{
    body::Bytes,
    extract::ws::{Message, Utf8Bytes, WebSocket, WebSocketUpgrade},
    response::IntoResponse,
    routing::any,
    Router,
};
use axum::extract::connect_info::ConnectInfo;
use axum::extract::ws::CloseFrame;
use axum_extra::TypedHeader;
use futures_util::{sink::SinkExt, stream::StreamExt};
use std::{net::SocketAddr, ops::ControlFlow, path::PathBuf};
use tower_http::{services::ServeDir, trace::{DefaultMakeSpan, TraceLayer}};

async fn ws_handler(
    ws: WebSocketUpgrade,
    user_agent: Option<TypedHeader<headers::UserAgent>>,
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
) -> impl IntoResponse {
    let user_agent = user_agent
        .map(|TypedHeader(ua)| ua.to_string())
        .unwrap_or_else(|| "Unknown browser".to_string());

    println!("`{user_agent}` at {addr} connected.");

    ws.on_upgrade(move |socket| handle_socket(socket, addr))
}

async fn handle_socket(mut socket: WebSocket, who: SocketAddr) {
    if socket
        .send(Message::Ping(Bytes::from_static(&[1, 2, 3])))
        .await
        .is_err()
    {
        return;
    }

    if let Some(Ok(msg)) = socket.recv().await {
        if process_message(msg, who).is_break() {
            return;
        }
    }

    for i in 1..5 {
        if socket
            .send(Message::Text(format!("Hi {i} times!").into()))
            .await
            .is_err()
        {
            return;
        }
        tokio::time::sleep(std::time::Duration::from_millis(100)).await;
    }

    let (mut sender, mut receiver) = socket.split();

    let mut send_task = tokio::spawn(async move {
        for i in 0..20 {
            if sender
                .send(Message::Text(format!("Server message {i} ...").into()))
                .await
                .is_err()
            {
                return;
            }
            tokio::time::sleep(std::time::Duration::from_millis(300)).await;
        }

        let _ = sender
            .send(Message::Close(Some(CloseFrame {
                code: axum::extract::ws::close_code::NORMAL,
                reason: Utf8Bytes::from_static("Goodbye"),
            })))
            .await;
    });

    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(msg)) = receiver.next().await {
            if process_message(msg, who).is_break() {
                break;
            }
        }
    });

    tokio::select! {
        _ = (&mut send_task) => recv_task.abort(),
        _ = (&mut recv_task) => send_task.abort(),
    }
}

fn process_message(msg: Message, who: SocketAddr) -> ControlFlow<(), ()> {
    match msg {
        Message::Text(t) => println!(">>> {who} sent str: {t:?}"),
        Message::Binary(d) => println!(">>> {who} sent {} bytes", d.len()),
        Message::Close(_) => return ControlFlow::Break(()),
        Message::Pong(v) => println!(">>> {who} sent pong with {v:?}"),
        Message::Ping(v) => println!(">>> {who} sent ping with {v:?}"),
    }
    ControlFlow::Continue(())
}
````

> main 入口 + 静态文件 ServeDir 见源码对照,逻辑同第 29/40 章。

## 运行

````bash
cargo run -p example-websockets --bin example-websockets   # 服务端
````

浏览器打开 `http://localhost:3000`,打开 Console 观察服务端消息。或运行 Rust 客户端:

````bash
cargo run -p example-websockets --bin example-client
````

## 解读

### WebSocket vs SSE

```text
SSE:       服务端 → 客户端单向推送
WebSocket: 浏览器 ↔ 服务端双向通信
```

需双向实时通信用 WebSocket;只需服务端推送(通知/进度/日志)用 SSE 更简单。

### HTTP upgrade(两个阶段)

WebSocket 不是凭空建立的,先发一个 HTTP 请求要求升级协议(`HTTP → WebSocket`)。所以源码分两阶段:

```text
ws_handler    处理 upgrade 前的 HTTP 请求(可读 User-Agent/客户端 IP/headers)
handle_socket 处理 upgrade 后的 WebSocket 连接
```

### `WebSocketUpgrade` extractor

````rust
async fn ws_handler(
    ws: WebSocketUpgrade,
    user_agent: Option<TypedHeader<headers::UserAgent>>,
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
) -> impl IntoResponse {
    ...
    ws.on_upgrade(move |socket| handle_socket(socket, addr))   // upgrade 成功后交给 handle_socket
}
````

`ws.on_upgrade(closure)` 表示 upgrade 成功后把 `WebSocket` 交给 `handle_socket`。

### `ConnectInfo` 要开 `into_make_service_with_connect_info`

````rust
axum::serve(
    listener,
    app.into_make_service_with_connect_info::<SocketAddr>(),
)
````

不用 `into_make_service_with_connect_info`,`ConnectInfo(addr): ConnectInfo<SocketAddr>` extractor 拿不到客户端地址。

### 每个连接一个 `handle_socket`

每个 WebSocket 客户端都有自己的 `handle_socket` 任务。一个客户端 sleep 或等消息不阻塞其他客户端——async server 的基本并发模型:每个连接各自推进自己的 future,Tokio 负责调度。

### 消息类型

````rust
fn process_message(msg: Message, who: SocketAddr) -> ControlFlow<(), ()> {
    match msg {
        Message::Text(t) => ...,
        Message::Binary(d) => ...,
        Message::Close(_) => return ControlFlow::Break(()),   // Close 应退出读取循环
        Message::Pong(v) => ...,
        Message::Ping(v) => ...,
    }
    ControlFlow::Continue(())
}
````

`ControlFlow::Break` 表示应结束当前连接处理。收到 Close 继续处理通常没意义。

### `split` + 并发收发

````rust
let (mut sender, mut receiver) = socket.split();

let mut send_task = tokio::spawn(async move { /* 一个任务发 */ });
let mut recv_task = tokio::spawn(async move { /* 一个任务收 */ });

tokio::select! {
    _ = (&mut send_task) => recv_task.abort(),   // 任意一边结束就停另一边
    _ = (&mut recv_task) => send_task.abort(),
}
````

split 后 sender 负责发、receiver 负责收,可同时做两件事(后台任务不断发,另一个不断读),比单循环里一会儿发一会儿收更灵活。WebSocket 发送或接收任意一侧结束通常意味着连接生命周期该收尾,`tokio::select!` 谁先结束就 abort 另一个。

### Ping/Pong 心跳

WebSocket 协议中 Ping/Pong 常用于心跳和连接检测。本例只在连接建立时发了一个 Ping。

## ⚠️ 生产级 WebSocket 必做:心跳与超时

本示例没任何周期性心跳和超时,教学没问题,但生产会引发**半开连接泄漏**——生产 WebSocket 服务的头号问题。

### 什么是半开连接

```text
客户端断网(手机切电梯、电脑休眠)
  → TCP 层可能没正常发 FIN 关闭
  → 服务端 socket 还"活着"以为连接正常
  → 服务端一直 await receiver.next() 永远等不到消息
  → 连接、内存、任务句柄全部泄漏
```

TCP 默认 keep-alive 探测间隔通常以小时计,等它发现连接已死要很久。应用层必须自己做心跳。

### 标准做法:周期性 Ping + recv 超时

1. **周期性发 Ping**(如每 30 秒):客户端会自动回 Pong。对端已死时发 Ping 会失败。
2. **给 recv 加超时**:用 `tokio::time::timeout` 包住 `receiver.next()`,一段时间内既没消息也没 Pong 就认定连接已死主动关闭。

````rust
let mut send_interval = tokio::time::interval(Duration::from_secs(30));
loop {
    tokio::select! {
        _ = send_interval.tick() => {
            if sender.send(Message::Ping(Bytes::new())).await.is_err() {
                break;   // 发送失败,对端已断开
            }
        }
        msg = tokio::time::timeout(Duration::from_secs(60), receiver.next()) => {
            match msg {
                Ok(Some(Ok(message))) => { /* 处理消息 */ }
                _ => break,   // 超时或错误,关闭连接
            }
        }
    }
}
````

> 本示例 handler 里 `socket` 在 `tokio::select!` 中被移动,依赖 WebSocket 操作的 **cancel-safety**(第 44 章测试会讲——`WebSocket::recv` 和 `broadcast::Receiver::recv` 都 cancel-safe,可安全用在 select!)。

记住这条工程经验:**写 WebSocket 不加心跳和超时,上线后一定会被半开连接拖垮。** 这比 handler 业务逻辑重要得多。

## 常见问题

**WebSocket 和 SSE 最大区别?** SSE 单向推送,WebSocket 双向通信。

**为什么要 split?** split 后可一个任务发一个任务收,适合实时双向通信;不 split 发送和接收通常在同一流程里。

**收到 Close 为什么 break?** Close 表示对方想关连接,继续处理没意义,结束连接状态机。

**为什么用 ConnectInfo?** 让 handler 拿到客户端 SocketAddr,对日志/审计/调试有用。必须配 `into_make_service_with_connect_info`。

## 手写任务

1. 写 `/ws` 只完成 `WebSocketUpgrade`。
2. 写 `handle_socket` 连接后发一条文本。
3. 接收一条客户端消息并打印。
4. 加 `socket.split()`。
5. 用两个任务分别发送和接收。
6. 浏览器 `new WebSocket(...)` 验证。

## 小结

- WebSocket 在 axum 分两步:`WebSocketUpgrade` 处理 HTTP upgrade,`WebSocket` 处理 upgrade 后的双向消息流。
- `ConnectInfo` 要配 `into_make_service_with_connect_info` 才能拿到客户端地址。
- 连接内核心模式:`send`/`recv` → `split` → `tokio::spawn`(一个任务发一个任务收)→ `tokio::select!`(谁先结束 abort 另一个)→ Close 后结束。
- WebSocket vs SSE:双向实时用 WebSocket,单向推送用 SSE。
- **生产 WebSocket 必加周期性 Ping + recv 超时**,否则半开连接泄漏会拖垮服务。

## 源码对照

- `examples/websockets/Cargo.toml`
- `examples/websockets/src/main.rs`
- `examples/websockets/src/client.rs`
- `examples/websockets/assets/index.html`
- `examples/websockets/assets/script.js`
