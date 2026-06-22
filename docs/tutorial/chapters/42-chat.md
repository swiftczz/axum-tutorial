# 42. chat

对应示例：`examples/chat`

上一章演示了单个 WebSocket 连接的收发。这章把多个连接串起来：实现多人聊天室——一个用户发消息，所有用户都收到。代码约 120 行，分 4 步搭。

相比上一章的新东西：**共享状态**（谁在线）、**broadcast channel**（一条消息发给所有人）、**用户名管理**。

## Cargo.toml

````toml
[package]
name = "example-chat"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["ws"] }
futures-util = { version = "0.3", default-features = false, features = ["sink", "std"] }
tokio = { version = "1", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：理解 broadcast channel

单个 WebSocket 连接只能管"这个客户端发了什么"。聊天室需要管"所有在线用户"和"把消息发给所有人"。

`tokio::sync::broadcast` 解决"一条消息发给多个接收者"的问题：

````rust
// 创建广播通道，容量 100（缓冲未消费消息）
let (tx, _rx) = broadcast::channel(100);

// 发送端：任何人发消息都通过它广播
tx.send("Alice joined.".to_string());

// 接收端：每个连接 subscribe 一个，收到所有广播
let mut rx = tx.subscribe();
let msg = rx.recv().await; // "Alice joined."
````

> **新面孔：`broadcast::channel`**
>
> `broadcast` 是"一条消息给多个订阅者"的 channel。所有连接共享同一个 `tx`（发送端），每个连接有自己的 `rx`（接收端）。
>
> 对比 `mpsc`（多生产者一消费者），broadcast 是"多生产者多消费者"——聊天室里每个用户既是生产者（发消息）也是消费者（收消息）。

---

## 第二步：定义共享状态和用户名管理

聊天室需要知道"谁在线"（用户名唯一），所以要有共享状态：

````rust
use std::{
    collections::HashSet,
    sync::{Arc, Mutex},
};
use tokio::sync::broadcast;

struct AppState {
    user_set: Mutex<HashSet<String>>,   // 已占用的用户名
    tx: broadcast::Sender<String>,       // 广播发送端
}
````

`user_set` 用 `Mutex<HashSet<String>>`——和 ch16 的 `RwLock<HashMap>` 同理，多个连接可能同时加入/离开，需要锁保护。

`tx` 是上一章的 broadcast 发送端，所有连接共享。

用户名检查逻辑：

````rust
fn check_username(state: &AppState, string: &mut String, name: &str) {
    let mut user_set = state.user_set.lock().unwrap();

    if !user_set.contains(name) {
        user_set.insert(name.to_owned());
        string.push_str(name);  // username 可用 → 写入
    }
    // username 已存在 → string 仍为空，调用方就知道不可用
}
````

> **新面孔：`Mutex<HashSet<String>>`**
>
> 和 ch16 的 `RwLock<HashMap>` 一样是并发保护。这里用 `Mutex`（不是 `RwLock`）因为用户名检查是"读+写"操作（先查再插），`Mutex` 更简单。
>
> `Arc` 包外层（`Arc<AppState>`），让多个 WebSocket 连接共享同一份状态。

---

## 第三步：WebSocket handler——第一条消息是用户名

浏览器连接后第一条消息发送用户名。服务端检查是否可用，可用就进聊天室：

````rust
use axum::{
    extract::{
        ws::{Message, Utf8Bytes, WebSocket, WebSocketUpgrade},
        State,
    },
    response::IntoResponse,
};
use futures_util::{sink::SinkExt, stream::StreamExt};

async fn websocket_handler(
    ws: WebSocketUpgrade,
    State(state): State<Arc<AppState>>,
) -> impl IntoResponse {
    ws.on_upgrade(|socket| websocket(socket, state))
}

async fn websocket(stream: WebSocket, state: Arc<AppState>) {
    let (mut sender, mut receiver) = stream.split();

    // 等第一条文本消息作为用户名
    let mut username = String::new();
    while let Some(Ok(message)) = receiver.next().await {
        if let Message::Text(name) = message {
            check_username(&state, &mut username, name.as_str());

            if !username.is_empty() {
                break;  // 用户名可用，进聊天室
            } else {
                // 用户名重复，通知客户端并断开
                let _ = sender.send(Message::Text(Utf8Bytes::from_static(
                    "Username already taken.",
                ))).await;
                return;
            }
        }
    }

    // ... 下一步：订阅广播 + 收发消息
}
````

和 ch41 的 WebSocket upgrade 一样（`ws.on_upgrade`），但多了 `State(state)` 提取共享状态。

---

## 第四步：两个并发任务——一个发一个收

用户进聊天室后，需要同时做两件事：
- **收客户端消息 → 广播给所有人**
- **收广播消息 → 发给当前客户端**

这需要 `split` 后用两个 `tokio::spawn` 任务：

````rust
// 先 subscribe，再发 joined（这样自己也能收到）
let mut rx = state.tx.subscribe();

let msg = format!("{username} joined.");
let _ = state.tx.send(msg);

// 任务一：从广播通道收消息 → 发给当前客户端
let mut send_task = tokio::spawn(async move {
    while let Ok(msg) = rx.recv().await {
        if sender.send(Message::text(msg)).await.is_err() {
            break;  // 客户端断开
        }
    }
});

// 任务二：从当前客户端收消息 → 广播给所有人
let tx = state.tx.clone();
let name = username.clone();
let mut recv_task = tokio::spawn(async move {
    while let Some(Ok(Message::Text(text))) = receiver.next().await {
        let _ = tx.send(format!("{name}: {text}"));  // Alice: hello
    }
});

// 任意一边结束就取消另一边，然后清理
tokio::select! {
    _ = &mut send_task => recv_task.abort(),
    _ = &mut recv_task => send_task.abort(),
};

// 广播离开 + 释放用户名
let _ = state.tx.send(format!("{username} left."));
state.user_set.lock().unwrap().remove(&username);
````

> **新面孔：`subscribe` + 顺序很重要**
>
> 先 `subscribe()` 再 `send(joined)`——这样当前用户自己也能收到 `Alice joined.`。如果反过来，当前用户收不到自己的 joined 消息。
>
> `tokio::select!` 谁先结束就 abort 另一个：客户端断开发送任务结束 → abort 接收任务；反之亦然。
>
> 最后广播 `left` 消息并从 `user_set` 移除用户名，新连接可以重新使用。

---

## 完整代码

````rust
use axum::{
    extract::{
        ws::{Message, Utf8Bytes, WebSocket, WebSocketUpgrade},
        State,
    },
    response::{Html, IntoResponse},
    routing::get,
    Router,
};
use futures_util::{sink::SinkExt, stream::StreamExt};
use std::{
    collections::HashSet,
    sync::{Arc, Mutex},
};
use tokio::sync::broadcast;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

struct AppState {
    user_set: Mutex<HashSet<String>>,
    tx: broadcast::Sender<String>,
}

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=trace", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let user_set = Mutex::new(HashSet::new());
    let (tx, _rx) = broadcast::channel(100);

    let app_state = Arc::new(AppState { user_set, tx });

    let app = Router::new()
        .route("/", get(index))
        .route("/websocket", get(websocket_handler))
        .with_state(app_state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn websocket_handler(
    ws: WebSocketUpgrade,
    State(state): State<Arc<AppState>>,
) -> impl IntoResponse {
    ws.on_upgrade(|socket| websocket(socket, state))
}

async fn websocket(stream: WebSocket, state: Arc<AppState>) {
    let (mut sender, mut receiver) = stream.split();

    let mut username = String::new();
    while let Some(Ok(message)) = receiver.next().await {
        if let Message::Text(name) = message {
            check_username(&state, &mut username, name.as_str());

            if !username.is_empty() {
                break;
            } else {
                let _ = sender
                    .send(Message::Text(Utf8Bytes::from_static(
                        "Username already taken.",
                    )))
                    .await;
                return;
            }
        }
    }

    let mut rx = state.tx.subscribe();

    let msg = format!("{username} joined.");
    tracing::debug!("{msg}");
    let _ = state.tx.send(msg);

    let mut send_task = tokio::spawn(async move {
        while let Ok(msg) = rx.recv().await {
            if sender.send(Message::text(msg)).await.is_err() {
                break;
            }
        }
    });

    let tx = state.tx.clone();
    let name = username.clone();
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(Message::Text(text))) = receiver.next().await {
            let _ = tx.send(format!("{name}: {text}"));
        }
    });

    tokio::select! {
        _ = &mut send_task => recv_task.abort(),
        _ = &mut recv_task => send_task.abort(),
    };

    let msg = format!("{username} left.");
    tracing::debug!("{msg}");
    let _ = state.tx.send(msg);

    state.user_set.lock().unwrap().remove(&username);
}

fn check_username(state: &AppState, string: &mut String, name: &str) {
    let mut user_set = state.user_set.lock().unwrap();

    if !user_set.contains(name) {
        user_set.insert(name.to_owned());
        string.push_str(name);
    }
}

async fn index() -> Html<&'static str> {
    Html(std::include_str!("../chat.html"))
}
````

## 运行

````bash
cd examples
cargo run -p example-chat
````

打开两个浏览器窗口 `http://127.0.0.1:3000/`，分别输入不同用户名加入。在任意窗口输入消息按 Enter，另一个窗口能收到。

## 手写任务

1. 先写 WebSocket echo（收到什么回什么）。
2. 加 `AppState { user_set, tx }`。
3. 第一条消息作为用户名。
4. 用 `broadcast::channel` 广播 joined/message/left。
5. split socket，分别写 send_task 和 recv_task。
6. 断开时移除用户名。

## 小结

这章在 ch41 WebSocket 基础上搭了多人聊天室，4 步增量构建：

1. **broadcast channel**：一条消息发给所有订阅者（聊天室核心）。
2. **共享状态**：`Mutex<HashSet>` 管理用户名唯一，`broadcast::Sender` 群发消息。
3. **第一条消息是用户名**：检查是否可用，可用就进聊天室。
4. **两个并发任务**：send_task（广播→客户端）+ recv_task（客户端→广播），`tokio::select!` 谁先结束 abort 另一个。

和 ch41 的区别：ch41 只管单个连接收发，ch42 用 broadcast 把多个连接串起来。

## 源码对照

- `examples/chat/Cargo.toml`
- `examples/chat/src/main.rs`
- `examples/chat/chat.html`
