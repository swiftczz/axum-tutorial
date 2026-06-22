# 41. websockets

对应示例：`examples/websockets`

本章目标：理解 WebSocket 的 HTTP upgrade、连接状态机、消息类型、并发收发，以及 Axum 中 `WebSocketUpgrade` 和 `WebSocket` 的基本用法。

上一章 SSE 是服务端单向推送。  
这一章 WebSocket 是双向通信：

```text
浏览器 <-> 服务端
```

## 这个小项目在做什么

服务端提供：

```text
GET /    -> 静态页面
ANY /ws  -> WebSocket 连接入口
```

浏览器页面会创建：

````javascript
const socket = new WebSocket('ws://localhost:3000/ws');
````

连接成功后：

```text
浏览器给服务端发消息
服务端给浏览器发 ping 和文本消息
服务端 split socket 后并发发送和接收
双方最后发送 close
```

请求主线是：

```text
浏览器请求 /ws
-> HTTP 请求进入 ws_handler
-> Axum 执行 WebSocket upgrade
-> upgrade 成功后进入 handle_socket
-> handle_socket 处理这个连接的生命周期
```

## 先理解 HTTP upgrade

WebSocket 一开始不是直接凭空建立的。  
它先发一个 HTTP 请求，要求升级协议：

```text
HTTP -> WebSocket
```

所以源码里有两个阶段：

```text
ws_handler     -> 处理 HTTP upgrade 前的请求
handle_socket  -> 处理 upgrade 后的 WebSocket 连接
```

在 `ws_handler` 里还可以读取 HTTP 信息：

```text
User-Agent
客户端 IP
headers
```

一旦 upgrade 完成，后面主要处理 WebSocket message。

## 文件和依赖

这个 example 有五个主要文件：

1. `examples/websockets/Cargo.toml`：声明 Axum ws feature、tokio-tungstenite、tower-http。
2. `examples/websockets/src/main.rs`：WebSocket 服务端。
3. `examples/websockets/src/client.rs`：Rust WebSocket 客户端。
4. `examples/websockets/assets/index.html`：浏览器页面。
5. `examples/websockets/assets/script.js`：浏览器 WebSocket 客户端脚本。

关键依赖：

- `axum`：启用 `ws` feature，提供 WebSocket extractor。
- `futures-util`：提供 `SinkExt`、`StreamExt`，用于 split 后发送和接收。
- `tokio-tungstenite`：Rust 客户端示例使用。
- `tower-http`：静态文件和 trace 日志。
- `axum-extra`：提取 `User-Agent`。

## 第一步：Router 同时提供静态文件和 /ws

源码：

````rust
let app = Router::new()
    .fallback_service(ServeDir::new(assets_dir).append_index_html_on_directories(true))
    .route("/ws", any(ws_handler))
    .layer(
        TraceLayer::new_for_http()
            .make_span_with(DefaultMakeSpan::default().include_headers(true)),
    );
````

这里有两类请求：

```text
/       -> 静态页面
/ws     -> WebSocket upgrade
```

`any(ws_handler)` 表示多种 HTTP 方法都可以进入这个 handler。  
浏览器 WebSocket 通常会发 GET upgrade 请求。

## 第二步：开启 ConnectInfo

源码：

````rust
axum::serve(
    listener,
    app.into_make_service_with_connect_info::<SocketAddr>(),
)
````

这让 handler 可以提取客户端地址：

````rust
ConnectInfo(addr): ConnectInfo<SocketAddr>
````

如果不用 `into_make_service_with_connect_info`，`ConnectInfo` extractor 就拿不到地址。

## 第三步：ws_handler 处理 upgrade 前的信息

源码：

````rust
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
````

`WebSocketUpgrade` 是 Axum 提供的 upgrade extractor。  
最后返回：

````rust
ws.on_upgrade(move |socket| handle_socket(socket, addr))
````

这表示：

```text
HTTP upgrade 成功后，把 WebSocket 交给 handle_socket
```

## 第四步：每个连接一个 handle_socket

源码：

````rust
async fn handle_socket(mut socket: WebSocket, who: SocketAddr) {
    ...
}
````

每个 WebSocket 客户端都会有自己的 `handle_socket` 任务。  
一个客户端 sleep 或等待消息，不会阻塞其他客户端连接。

这就是 async server 的基本并发模型：

```text
每个连接各自推进自己的 future
Tokio 负责调度
```

## 第五步：发送 Ping 并接收第一条消息

源码：

````rust
socket
    .send(Message::Ping(Bytes::from_static(&[1, 2, 3])))
    .await
````

服务端先发一个 Ping。  
WebSocket 协议中，Ping/Pong 常用于心跳和连接检测。

然后接收一条消息：

````rust
if let Some(msg) = socket.recv().await {
    if let Ok(msg) = msg {
        if process_message(msg, who).is_break() {
            return;
        }
    }
}
````

`process_message` 会处理 Text、Binary、Close、Ping、Pong 等消息类型。

## 第六步：连续发送几条文本消息

源码：

````rust
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
````

这段演示服务端主动发消息给客户端。  
每 100ms 发一条。

如果发送失败，说明连接可能已经断开，直接结束当前连接处理。

## 第七步：split 后并发发送和接收

源码：

````rust
let (mut sender, mut receiver) = socket.split();
````

split 后得到两半：

```text
sender   -> 负责发送消息
receiver -> 负责接收消息
```

这样可以同时做两件事：

```text
后台任务不断给客户端发消息
另一个后台任务不断读客户端消息
```

这比单个循环里一会儿发、一会儿收更灵活。

## 第八步：send_task 和 recv_task

发送任务：

````rust
let mut send_task = tokio::spawn(async move {
    for i in 0..20 {
        sender.send(Message::Text(format!("Server message {i} ...").into())).await?;
        tokio::time::sleep(std::time::Duration::from_millis(300)).await;
    }
    sender.send(Message::Close(...)).await
});
````

接收任务：

````rust
let mut recv_task = tokio::spawn(async move {
    while let Some(Ok(msg)) = receiver.next().await {
        if process_message(msg, who).is_break() {
            break;
        }
    }
});
````

源码里错误处理写得更完整。  
核心是：

```text
一个任务负责发
一个任务负责收
```

## 第九步：tokio::select! 谁先结束就停另一个

源码：

````rust
tokio::select! {
    rv_a = (&mut send_task) => {
        ...
        recv_task.abort();
    },
    rv_b = (&mut recv_task) => {
        ...
        send_task.abort();
    }
}
````

WebSocket 连接里，发送或接收任意一侧结束，都通常意味着这个连接生命周期该收尾了。

所以这里：

```text
send_task 结束 -> abort recv_task
recv_task 结束 -> abort send_task
```

最后 `handle_socket` 返回，WebSocket 连接关闭。

## 第十步：process_message 处理消息类型

源码：

````rust
fn process_message(msg: Message, who: SocketAddr) -> ControlFlow<(), ()> {
    match msg {
        Message::Text(t) => { ... }
        Message::Binary(d) => { ... }
        Message::Close(c) => { return ControlFlow::Break(()); }
        Message::Pong(v) => { ... }
        Message::Ping(v) => { ... }
    }
    ControlFlow::Continue(())
}
````

`ControlFlow::Break` 表示应该结束当前连接处理。  
特别是收到 Close 时，应该退出读取循环。

## 浏览器客户端

前端脚本：

````javascript
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
    socket.send(blob);
}, 1000);

setTimeout(() => {
    socket.send('About done here...');
    socket.close(3000, "Crash and Burn!");
}, 3000);
````

它演示了：

- 连接打开后发文本。
- 收到服务端消息后打印。
- 发送二进制 Blob。
- 主动关闭连接。

## Rust 客户端

`src/client.rs` 使用 `tokio-tungstenite`。  
它会同时启动两个客户端连接：

```text
N_CLIENTS = 2
```

用来验证多个 WebSocket 连接可以并发运行。

## 函数职责速查

- `main`：初始化日志，提供静态文件和 `/ws`，启用 ConnectInfo，启动服务。
- `ws_handler`：处理 HTTP upgrade，提取 User-Agent 和客户端地址。
- `handle_socket`：管理单个 WebSocket 连接的完整生命周期。
- `process_message`：打印并判断 WebSocket 消息是否要求结束连接。
- `client.rs::spawn_client`：启动一个 Rust WebSocket 客户端并并发收发消息。

## 带中文注释的手写版

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

// upgrade 前的 HTTP handler。
async fn ws_handler(
    ws: WebSocketUpgrade,
    user_agent: Option<TypedHeader<headers::UserAgent>>,
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
) -> impl IntoResponse {
    let user_agent = user_agent
        .map(|TypedHeader(ua)| ua.to_string())
        .unwrap_or_else(|| "Unknown browser".to_string());

    println!("`{user_agent}` at {addr} connected.");

    // upgrade 成功后进入真正的 WebSocket 处理函数。
    ws.on_upgrade(move |socket| handle_socket(socket, addr))
}

// 每个客户端连接都会运行一个 handle_socket。
async fn handle_socket(mut socket: WebSocket, who: SocketAddr) {
    // 先发一个 ping。
    if socket
        .send(Message::Ping(Bytes::from_static(&[1, 2, 3])))
        .await
        .is_err()
    {
        return;
    }

    // 先接收一条消息。
    if let Some(Ok(msg)) = socket.recv().await {
        if process_message(msg, who).is_break() {
            return;
        }
    }

    // 发送几条问候消息。
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

    // split 后可以同时发送和接收。
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

    // 任意一边结束，就取消另一边。
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

## 运行和验证

运行服务端：

````bash
cargo run -p example-websockets --bin example-websockets
````

浏览器打开：

```text
http://localhost:3000
```

打开 Console，观察服务端消息。

也可以运行 Rust 客户端：

````bash
cargo run -p example-websockets --bin example-client
````

## 常见卡点

### 1. WebSocket 和 SSE 最大区别是什么？

SSE 是服务端到客户端单向推送。  
WebSocket 是双向通信。

### 2. 为什么要 split？

不 split 时，同一个 socket 的发送和接收通常写在同一流程里。  
split 后可以一个任务发，一个任务收，更适合实时双向通信。

### 3. 为什么收到 Close 要 break？

Close 表示对方想关闭连接。  
继续处理通常没有意义，应该结束当前连接状态机。

### 4. 为什么要 ConnectInfo？

它让 handler 能拿到客户端 SocketAddr。  
这对日志、审计、调试都很有用。

## 手写任务

1. 先写 `/ws`，只完成 `WebSocketUpgrade`。
2. 写 `handle_socket`，连接后发送一条文本。
3. 接收一条客户端消息并打印。
4. 加 `socket.split()`。
5. 用两个任务分别发送和接收。
6. 用浏览器 `new WebSocket(...)` 验证。

## 生产级 WebSocket 必做：心跳与超时

本示例只在连接建立时发了一个 Ping，**没有任何周期性心跳和超时机制**。这在教学里没问题，但在生产环境会引发**半开连接泄漏**——这是生产 WebSocket 服务的头号问题。

### 什么是半开连接

```text
客户端断网（比如手机切到电梯、电脑休眠）
→ TCP 层可能没有正常发 FIN 关闭
→ 服务端的 socket 还"活着"，以为连接正常
→ 服务端一直 await receiver.next()，永远等不到消息
→ 连接、内存、任务句柄全部泄漏
```

TCP 默认的 keep-alive 探测间隔通常以小时计，等它发现连接已死要很久。所以应用层必须自己做心跳。

### 标准做法：周期性 Ping + recv 超时

生产 WebSocket 的核心是两件事：

1. **周期性发 Ping**：服务端每隔一段时间（如 30 秒）发一个 Ping，客户端会自动回 Pong。如果对端已死，发 Ping 会失败。
2. **给 recv 加超时**：用 `tokio::time::timeout` 包住 `receiver.next()`，如果一段时间内既没有消息也没有 Pong，就认定连接已死，主动关闭。

伪代码示意（接在 split 之后）：

````rust
let mut send_interval = tokio::time::interval(Duration::from_secs(30));
loop {
    tokio::select! {
        // 周期性发心跳
        _ = send_interval.tick() => {
            if sender.send(Message::Ping(Bytes::new())).await.is_err() {
                break; // 发送失败，对端已断开
            }
        }
        // 读消息，带超时（60 秒收不到任何消息/Pong 就断）
        msg = tokio::time::timeout(Duration::from_secs(60), receiver.next()) => {
            match msg {
                Ok(Some(Ok(message))) => { /* 处理消息 */ }
                _ => break, // 超时或错误，关闭连接
            }
        }
    }
}
````

> 注意：本示例的 handler 里 `socket` 在 `tokio::select!` 中被移动，依赖 WebSocket 操作的 **cancel-safety**（第 44 章测试里会讲到这个概念——`WebSocket::recv` 和 `broadcast::Receiver::recv` 都是 cancel-safe，所以可以安全用在 select! 里）。

记住这条工程经验：**写 WebSocket 不加心跳和超时，上线后一定会被半开连接拖垮。** 这比 handler 业务逻辑重要得多。

## 本章真正要记住什么

WebSocket 在 Axum 中分两步：

```text
WebSocketUpgrade 处理 HTTP upgrade
WebSocket 处理 upgrade 后的双向消息流
```

连接内的核心模式是：

```text
send / recv
split
tokio::spawn
tokio::select!
Close 后结束连接
```

## 源码对照

本章手写版对应源码：

- `examples/websockets/src/main.rs`
- `examples/websockets/src/client.rs`
- `examples/websockets/assets/index.html`
- `examples/websockets/assets/script.js`
- `examples/websockets/Cargo.toml`
