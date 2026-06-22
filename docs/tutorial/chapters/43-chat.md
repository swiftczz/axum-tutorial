# 43. chat

对应示例：`examples/chat`

上一章演示单个 WebSocket 连接收发,这章把多个连接串起来:实现多人聊天室——一个用户发消息,所有用户都收到。在 WebSocket 基础上理解共享状态、唯一用户名、广播 channel、每个连接的收发任务。

## Cargo.toml

````toml
[package]
name = "example-chat"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["ws"] }
futures = "0.3"
tokio = { version = "1", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

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
use futures::{sink::SinkExt, stream::StreamExt};
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

> `chat.html` 完整内容见源码对照。

## 运行

````bash
cd examples
cargo run -p example-chat
````

打开两个浏览器窗口 `http://127.0.0.1:3000/`,分别输入不同用户名加入。在任意窗口输入消息按 Enter,另一个窗口能收到。

## 解读

### 聊天室 vs 单连接 WebSocket

```text
单连接 WebSocket:只关心这个客户端发了什么,服务端给这个客户端回什么
聊天室:          所有在线用户是谁、如何把一个用户消息发给所有用户、用户断开如何清理状态
```

所以本章新增共享状态 `AppState`。

### `AppState` 两件共享物

````rust
struct AppState {
    user_set: Mutex<HashSet<String>>,   // 已占用的用户名
    tx: broadcast::Sender<String>,      // 广播发送端
}
````

- `user_set: Mutex<HashSet<String>>`:保存已占用用户名,多个连接可能同时加入/离开,用 Mutex 保护(第 17 章 `Arc<RwLock<HashMap>>` 同理)。
- `tx: broadcast::Sender<String>`:广播发送端。每个连接订阅它收到其他用户发的消息。

````rust
let (tx, _rx) = broadcast::channel(100);  // 容量 100,缓冲未消费消息
let app_state = Arc::new(AppState { user_set, tx });
````

`Arc<AppState>` 让多个连接共享同一份状态。

### broadcast vs mpsc

```text
broadcast:一条消息给多个订阅者(聊天室需要所有用户收到同一条消息,适合)
mpsc:     多个生产者一个消费者
```

### 第一条消息当用户名

````rust
let mut username = String::new();
while let Some(Ok(message)) = receiver.next().await {
    if let Message::Text(name) = message {
        check_username(&state, &mut username, name.as_str());
        if !username.is_empty() { break; }      // 可用 → 进聊天室
        else { sender.send(... "Username already taken.").await; return; }  // 重复 → 断开
    }
}
````

这是 example 简化协议(真实项目用明确消息格式如 `{"type":"join","name":"Alice"}`)。`check_username` 用 Mutex 锁住 HashSet,name 不存在就插入并写入 username;已存在 username 仍空,调用方就知道不可用。

### 订阅广播 + joined 消息(顺序重要)

````rust
let mut rx = state.tx.subscribe();   // 1. 先 subscribe
let msg = format!("{username} joined.");
let _ = state.tx.send(msg);          // 2. 再 send joined
````

**顺序很重要**:先 subscribe 再 send joined,当前用户自己也能收到 `Alice joined.`。每个连接都有自己的 `rx` 接收端,所有连接共享同一个 `tx` 发送端。

### 两个并发任务

**send_task:广播 → 当前客户端**

````rust
let mut send_task = tokio::spawn(async move {
    while let Ok(msg) = rx.recv().await {
        if sender.send(Message::text(msg)).await.is_err() { break; }
    }
});
````

不管消息谁发的,只要进广播通道当前连接都收到。

**recv_task:当前用户消息 → 广播**

````rust
let tx = state.tx.clone();
let name = username.clone();
let mut recv_task = tokio::spawn(async move {
    while let Some(Ok(Message::Text(text))) = receiver.next().await {
        let _ = tx.send(format!("{name}: {text}"));   // 加 username 前缀
    }
});
````

Alice 输入 `hello` → 广播 `Alice: hello` → 所有连接收到。

### 收尾 + 清理

````rust
tokio::select! {
    _ = &mut send_task => recv_task.abort(),
    _ = &mut recv_task => send_task.abort(),
};

let _ = state.tx.send(format!("{username} left."));   // 广播离开
state.user_set.lock().unwrap().remove(&username);     // 释放用户名
````

任意任务结束取消另一个,广播离开消息,释放用户名(新连接可重新使用)。

### `include_str!` 嵌入 HTML

````rust
async fn index() -> Html<&'static str> {
    Html(std::include_str!("../chat.html"))
}
````

编译时把 `chat.html` 嵌入二进制(同 28 章 MiniJinja 的 `include_str!`)。

## 常见问题

**为什么用 `Arc<AppState>`?** 多个 WebSocket 连接共享同一份用户名集合和广播发送端,Arc 让异步任务共享 state。

**`user_set` 为什么 Mutex?** 多个连接可能同时检查和插入用户名,Mutex 保证同一时间只有一个任务修改集合。

**broadcast vs mpsc?** broadcast 一条消息给多个订阅者(聊天室适合),mpsc 多生产者一消费者。

**第一条消息为什么是用户名?** example 简化协议。真实项目用明确消息格式如 JSON `{type,name}`。

## 手写任务

1. 先写 WebSocket echo。
2. 加 `AppState { user_set, tx }`。
3. 第一条消息作为用户名。
4. `broadcast::channel` 广播 joined/message/left。
5. split socket,分别写 send_task 和 recv_task。
6. 断开时移除用户名。

## 小结

- 聊天室核心不是 WebSocket API 本身,而是共享状态 + 广播:`HashSet` 保证用户名唯一,`broadcast::Sender` 群发消息,每个连接 `subscribe()` 得到自己的 receiver,split 后一个任务发一个任务收,断开时清理状态。
- `AppState` 用 `Arc` 让多连接共享;`user_set` 用 `Mutex` 保护并发修改。
- broadcast 顺序:先 `subscribe()` 再 `send(joined)`,当前用户才能收到自己的 joined。
- 两个并发任务:send_task(广播 → 客户端)、recv_task(客户端 → 广播);`tokio::select!` 谁先结束 abort 另一个,再广播 left + 释放用户名。
- example 协议简化(第一条消息当用户名),真实项目用明确 JSON 格式。

## 源码对照

- `examples/chat/Cargo.toml`
- `examples/chat/src/main.rs`
- `examples/chat/chat.html`
