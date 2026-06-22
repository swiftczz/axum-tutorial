# 42. chat

对应示例：`examples/chat`

本章目标：在 WebSocket 基础上实现一个多人聊天室，理解共享状态、唯一用户名、广播 channel、以及每个连接的收发任务。

上一章只是演示单个 WebSocket 连接如何收发消息。  
这一章把多个连接串起来：一个用户发消息，所有用户都能收到。

## 这个小项目在做什么

应用有两个路由：

```text
GET /          -> 返回 chat.html
GET /websocket -> WebSocket 聊天连接
```

浏览器打开页面后：

```text
输入 username
点击 Join Chat
建立 WebSocket
第一条消息发送 username
之后输入聊天内容并按 Enter
```

服务端做三件事：

1. 检查用户名是否已被占用。
2. 把用户消息广播给所有连接。
3. 用户离开时释放用户名并广播离开消息。

## 先理解聊天室和单连接 WebSocket 的区别

单连接 WebSocket 只需要关心：

```text
这个客户端发了什么
服务端给这个客户端回什么
```

聊天室需要关心：

```text
所有在线用户是谁
如何把一个用户的消息发给所有用户
用户断开后如何清理状态
```

所以本章新增了共享状态 `AppState`。

## 文件和依赖

这个 example 有三个主要文件：

1. `examples/chat/Cargo.toml`：声明 Axum ws feature、futures-util、Tokio。
2. `examples/chat/src/main.rs`：实现聊天室后端。
3. `examples/chat/chat.html`：实现浏览器聊天页面。

关键依赖：

- `axum`：WebSocket、Router、State。
- `tokio::sync::broadcast`：一条消息广播给多个接收者。
- `futures-util`：split WebSocket 并收发消息。
- `Arc<Mutex<_>>`：共享用户名集合。

## 第一步：AppState 保存共享状态

源码：

````rust
struct AppState {
    user_set: Mutex<HashSet<String>>,
    tx: broadcast::Sender<String>,
}
````

`user_set` 保存已经占用的用户名。  
它用 `Mutex<HashSet<String>>`，因为多个连接可能同时加入或离开。

`tx` 是广播发送端。  
每个 WebSocket 连接都会订阅它，收到其他用户发出的消息。

## 第二步：创建状态并放进 Router

源码：

````rust
let user_set = Mutex::new(HashSet::new());
let (tx, _rx) = broadcast::channel(100);

let app_state = Arc::new(AppState { user_set, tx });

let app = Router::new()
    .route("/", get(index))
    .route("/websocket", get(websocket_handler))
    .with_state(app_state);
````

`Arc<AppState>` 让多个连接共享同一份状态。

`broadcast::channel(100)` 创建一个广播通道。  
容量 100 表示通道内部最多缓冲一定数量的消息。

## 第三步：首页直接 include HTML

源码：

````rust
async fn index() -> Html<&'static str> {
    Html(std::include_str!("../chat.html"))
}
````

`include_str!` 会在编译时把 `chat.html` 内容嵌入二进制。

这样访问 `/` 时直接返回 HTML 页面。

## 第四步：WebSocket upgrade

源码：

````rust
async fn websocket_handler(
    ws: WebSocketUpgrade,
    State(state): State<Arc<AppState>>,
) -> impl IntoResponse {
    ws.on_upgrade(|socket| websocket(socket, state))
}
````

这和上一章一样：

```text
HTTP upgrade 成功后
进入 websocket(socket, state)
```

不同点是这里把共享状态 `state` 也传给连接处理函数。

## 第五步：split 连接

源码：

````rust
let (mut sender, mut receiver) = stream.split();
````

聊天室需要同时做两件事：

```text
接收当前用户发来的消息
把其他用户的广播消息发给当前用户
```

所以一开始就 split。

## 第六步：第一条 Text 当用户名

源码：

````rust
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
````

浏览器连接后第一条消息发送 username。  
服务端把第一条文本消息当作用户名。

如果用户名可用，跳出循环，进入聊天室。  
如果已被占用，只给当前客户端发错误消息，然后结束连接。

## 第七步：检查用户名唯一

源码：

````rust
fn check_username(state: &AppState, string: &mut String, name: &str) {
    let mut user_set = state.user_set.lock().unwrap();

    if !user_set.contains(name) {
        user_set.insert(name.to_owned());

        string.push_str(name);
    }
}
````

这里用 Mutex 锁住 `HashSet`。

检查逻辑：

```text
如果 name 不存在
-> 插入 HashSet
-> 写入 username
```

如果 name 已存在，`username` 仍然为空，调用方就知道用户名不可用。

## 第八步：订阅广播并发送 joined 消息

源码：

````rust
let mut rx = state.tx.subscribe();

let msg = format!("{username} joined.");
let _ = state.tx.send(msg);
````

顺序很重要：

```text
先 subscribe
再 send joined
```

这样当前用户自己也能收到：

```text
Alice joined.
```

每个连接都有自己的 `rx` 接收端。  
所有连接共享同一个 `tx` 发送端。

## 第九步：send_task 把广播消息发给当前客户端

源码：

````rust
let mut send_task = tokio::spawn(async move {
    while let Ok(msg) = rx.recv().await {
        if sender.send(Message::text(msg)).await.is_err() {
            break;
        }
    }
});
````

这个任务负责：

```text
从 broadcast rx 收消息
通过 WebSocket sender 发给当前浏览器
```

不管消息是谁发的，只要进入广播通道，当前连接都会收到。

## 第十步：recv_task 把当前用户消息广播出去

源码：

````rust
let tx = state.tx.clone();
let name = username.clone();

let mut recv_task = tokio::spawn(async move {
    while let Some(Ok(Message::Text(text))) = receiver.next().await {
        let _ = tx.send(format!("{name}: {text}"));
    }
});
````

这个任务负责：

```text
从当前 WebSocket receiver 读消息
加上 username 前缀
发送到 broadcast channel
```

例如 Alice 输入：

```text
hello
```

广播内容是：

```text
Alice: hello
```

## 第十一步：任意任务结束就收尾

源码：

````rust
tokio::select! {
    _ = &mut send_task => recv_task.abort(),
    _ = &mut recv_task => send_task.abort(),
};

let msg = format!("{username} left.");
let _ = state.tx.send(msg);

state.user_set.lock().unwrap().remove(&username);
````

如果发送任务或接收任务任意一个结束，就取消另一个。  
然后：

1. 广播用户离开。
2. 从 `user_set` 移除用户名。

这样其他人能看到离开消息，新连接也可以重新使用这个用户名。

## 前端页面做了什么

`chat.html` 里主要逻辑是：

````javascript
const websocket = new WebSocket("ws://localhost:3000/websocket");

websocket.onopen = function() {
    websocket.send(username.value);
}

websocket.onmessage = function(e) {
    textarea.value += e.data+"\r\n";
}

input.onkeydown = function(e) {
    if (e.key == "Enter") {
        websocket.send(input.value);
        input.value = "";
    }
}
````

浏览器第一条消息发送用户名。  
后续按 Enter 发送聊天内容。

## 函数职责速查

- `main`：初始化共享状态、注册路由、启动服务。
- `index`：返回聊天 HTML 页面。
- `websocket_handler`：完成 WebSocket upgrade。
- `websocket`：处理单个用户连接、用户名校验、广播订阅、并发收发和清理。
- `check_username`：在共享 `HashSet` 中检查并占用用户名。


## 带中文注释的手写版

完整 `src/main.rs`：

````rust
//! 聊天室示例。
//!
//! Run with
//!
//! ```not_rust
//! cargo run -p example-chat
//! ```

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

// 应用共享状态。
struct AppState {
    // 保存已经被占用的用户名，保证用户名唯一。
    user_set: Mutex<HashSet<String>>,
    // 广播通道发送端。任何用户发消息，都通过它广播给所有订阅者。
    tx: broadcast::Sender<String>,
}

#[tokio::main]
async fn main() {
    // 初始化日志。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=trace", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 创建共享用户名集合。
    let user_set = Mutex::new(HashSet::new());
    // 创建广播通道。容量 100 表示可以缓冲一部分未消费消息。
    let (tx, _rx) = broadcast::channel(100);

    // WebSocket 连接会在多个异步任务中共享状态，所以用 Arc 包起来。
    let app_state = Arc::new(AppState { user_set, tx });

    // 首页返回 chat.html，/websocket 负责 WebSocket 连接。
    let app = Router::new()
        .route("/", get(index))
        .route("/websocket", get(websocket_handler))
        .with_state(app_state);

    // 启动服务。
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
    // HTTP upgrade 成功后，把 socket 和共享状态交给 websocket 函数处理。
    ws.on_upgrade(|socket| websocket(socket, state))
}

// 处理单个 WebSocket 连接。
async fn websocket(stream: WebSocket, state: Arc<AppState>) {
    // 拆成发送端和接收端，方便同时收发。
    let (mut sender, mut receiver) = stream.split();

    // 第一条文本消息会被当作用户名。
    let mut username = String::new();
    while let Some(Ok(message)) = receiver.next().await {
        if let Message::Text(name) = message {
            // 检查用户名是否已被占用。
            check_username(&state, &mut username, name.as_str());

            if !username.is_empty() {
                // 用户名可用，进入聊天流程。
                break;
            } else {
                // 用户名重复，只通知当前客户端，然后断开。
                let _ = sender
                    .send(Message::Text(Utf8Bytes::from_static(
                        "Username already taken.",
                    )))
                    .await;

                return;
            }
        }
    }

    // 当前连接订阅广播通道。
    // 先 subscribe，再发送 joined，这样自己也能看到 joined 消息。
    let mut rx = state.tx.subscribe();

    let msg = format!("{username} joined.");
    tracing::debug!("{msg}");
    let _ = state.tx.send(msg);

    // 任务一：从广播通道接收消息，再发给当前 WebSocket 客户端。
    let mut send_task = tokio::spawn(async move {
        while let Ok(msg) = rx.recv().await {
            if sender.send(Message::text(msg)).await.is_err() {
                break;
            }
        }
    });

    // 任务二：从当前 WebSocket 客户端接收消息，再广播给所有人。
    let tx = state.tx.clone();
    let name = username.clone();
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(Message::Text(text))) = receiver.next().await {
            let _ = tx.send(format!("{name}: {text}"));
        }
    });

    // 任意一个任务结束，就取消另一个任务。
    tokio::select! {
        _ = &mut send_task => recv_task.abort(),
        _ = &mut recv_task => send_task.abort(),
    };

    // 连接结束后，广播离开消息。
    let msg = format!("{username} left.");
    tracing::debug!("{msg}");
    let _ = state.tx.send(msg);

    // 释放用户名，后续新连接可以再次使用。
    state.user_set.lock().unwrap().remove(&username);
}

fn check_username(state: &AppState, string: &mut String, name: &str) {
    let mut user_set = state.user_set.lock().unwrap();

    if !user_set.contains(name) {
        user_set.insert(name.to_owned());
        string.push_str(name);
    }
}

// 编译时把 chat.html 嵌入二进制。
async fn index() -> Html<&'static str> {
    Html(std::include_str!("../chat.html"))
}
````

完整 `chat.html`：

````html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>WebSocket Chat</title>
    </head>
    <body>
        <h1>WebSocket Chat Example</h1>

        <input id="username" style="display:block; width:100px; box-sizing: border-box" type="text" placeholder="username">
        <button id="join-chat" type="button">Join Chat</button>
        <textarea id="chat" style="display:block; width:600px; height:400px; box-sizing: border-box" cols="30" rows="10"></textarea>
        <input id="input" style="display:block; width:600px; box-sizing: border-box" type="text" placeholder="chat">

        <script>
            const username = document.querySelector("#username");
            const join_btn = document.querySelector("#join-chat");
            const textarea = document.querySelector("#chat");
            const input = document.querySelector("#input");

            join_btn.addEventListener("click", function(e) {
                this.disabled = true;

                // 连接后端 WebSocket。
                const websocket = new WebSocket("ws://localhost:3000/websocket");

                websocket.onopen = function() {
                    console.log("connection opened");
                    // 第一条消息发送用户名。
                    websocket.send(username.value);
                }

                const btn = this;

                websocket.onclose = function() {
                    console.log("connection closed");
                    btn.disabled = false;
                }

                websocket.onmessage = function(e) {
                    console.log("received message: "+e.data);
                    textarea.value += e.data+"\r\n";
                }

                input.onkeydown = function(e) {
                    if (e.key == "Enter") {
                        websocket.send(input.value);
                        input.value = "";
                    }
                }
            });
        </script>
    </body>
</html>
````

## 运行和验证

运行：

````bash
cargo run -p example-chat
````

打开两个浏览器窗口：

```text
http://127.0.0.1:3000/
```

分别输入不同用户名加入。  
在任意窗口输入消息并按 Enter，另一个窗口应该能收到。

## 常见卡点

### 1. 为什么要 Arc<AppState>？

多个 WebSocket 连接要共享同一份用户名集合和广播发送端。  
`Arc` 让这些异步任务能共享 state。

### 2. 为什么 user_set 要 Mutex？

因为多个连接可能同时检查和插入用户名。  
`Mutex` 保证同一时间只有一个任务修改集合。

### 3. broadcast channel 和 mpsc 有什么区别？

`broadcast` 是一条消息给多个订阅者。  
聊天室需要所有用户收到同一条消息，所以适合 broadcast。

### 4. 为什么第一条消息是用户名？

这是本 example 简化协议的做法。  
真实项目通常会设计明确的消息格式，例如 JSON：`{ "type": "join", "name": "Alice" }`。

## 手写任务

1. 先写 WebSocket echo。
2. 加 `AppState { user_set, tx }`。
3. 第一条消息作为用户名。
4. 用 `broadcast::channel` 广播 joined/message/left。
5. split socket，分别写 send_task 和 recv_task。
6. 断开时移除用户名。

## 本章真正要记住什么

聊天室的核心不是 WebSocket API 本身，而是共享状态和广播：

```text
HashSet 保证用户名唯一
broadcast::Sender 负责群发消息
每个连接 subscribe 得到自己的 receiver
split 后一个任务发，一个任务收
断开时清理状态
```

## 源码对照

本章手写版对应源码：

- `examples/chat/src/main.rs`
- `examples/chat/chat.html`
- `examples/chat/Cargo.toml`
