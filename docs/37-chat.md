# 第 37 章 chat:聊天室(综合项目)

## 本章你将做出什么

一个能用的聊天室。开两个浏览器窗口,各自起一个用户名加入,然后互相发消息:

```
alice joined.
bob joined.
alice: 你好
bob: 嗨
bob left.
```

功能:**唯一用户名校验、加入/离开广播、消息带发送者名字**。

这是「实时通信」阶段的综合项目,把前面几章的东西拼在一起:[第 22 章](22-key-value-store.md)的共享可变状态、[第 34 章](34-websockets.md)的 `split()` + `select!`、[第 36 章](36-websockets-http2.md)的 `broadcast` channel。

学完本章你将掌握:

- 一个完整 WebSocket 应用的结构
- **握手后的"应用层握手"**:第一条消息用来认证/注册
- `HashSet` + `broadcast` 组合成的共享状态
- 连接断开时的资源清理
- 这个实现里几个真实存在的 bug(而且都很典型)

## 前置知识

### 应用层的第二次握手

[第 34 章](34-websockets.md#升级握手)讲过 WebSocket 的协议握手(`101` 或 `CONNECT`)。但**协议握手完成不等于业务就绪** —— 服务端还不知道这个连接属于谁。

本章的做法是**把第一条消息当作注册请求**:

```rust
// Username gets set in the receive loop, if it's valid.
let mut username = String::new();
// Loop until a text message is found.
while let Some(Ok(message)) = receiver.next().await {
    if let Message::Text(name) = message {
        check_username(&state, &mut username, name.as_str());
        if !username.is_empty() { break; } else { /* 拒绝并断开 */ }
    }
}
```

前端配合:

```js
websocket.onopen = function() {
    websocket.send(username.value);      // 第一条消息就是用户名
}
```

**为什么要这么绕?** 因为[第 34 章](34-websockets.md#代价与边界)提过的那个限制:**浏览器的 `WebSocket` 构造函数不能自定义请求头**,所以没法在握手阶段用 `Authorization: Bearer xxx`。三个绕法:

| 做法 | 优点 | 缺点 |
|---|---|---|
| cookie | 握手时就能验证 | 受 `SameSite` 限制,且有 CSWSH 风险 |
| token 放查询参数 | 握手时就能验证 | **会进访问日志** |
| **连上后先发认证消息** | 不受头部限制 | 连接已建立,要处理"未认证状态" |

本章用第三种。代价是**存在一个"已连接但未注册"的中间状态**,这个状态需要超时保护——否则恶意客户端可以连上后什么都不发,白占资源。「代价与边界」会说这个问题。

### 状态设计:HashSet + broadcast

```rust
struct AppState {
    // We require unique usernames. This tracks which usernames have been taken.
    user_set: Mutex<HashSet<String>>,
    // Channel used to send messages to all connected clients.
    tx: broadcast::Sender<String>,
}
```

两个字段各管一件事:

- **`user_set`** 记录哪些用户名被占用了,用 `Mutex` 保护([第 22 章](22-key-value-store.md#mutex-与-rwlock)讲过 `Mutex` 与 `RwLock` 的选择——这里读写都很少,`Mutex` 够了)
- **`tx`** 是广播通道([第 36 章](36-websockets-http2.md#broadcast-channel一对多)讲过),`Sender` 存在状态里供所有连接使用

**注意 `Mutex` 只包住了 `user_set`,而不是整个 `AppState`。** 这是有意的:`broadcast::Sender` 本身就是线程安全的(内部有自己的同步),不需要外部加锁。**只锁真正需要保护的部分**,减少竞争。

外层用 `Arc<AppState>` 共享 —— [第 05 章](05-templates-minijinja.md#state-与-with_state)讲过为什么。

### 订阅时机决定了能否看到自己的加入消息

代码里有一处注释值得注意:

```rust
// We subscribe *before* sending the "joined" message, so that we will also
// display it to our client.
let mut rx = state.tx.subscribe();

// Now send the "joined" message to all subscribers.
let msg = format!("{username} joined.");
let _ = state.tx.send(msg);
```

**`broadcast` channel 的语义是"订阅之后的消息才能收到"。** 所以:

- 先 `subscribe()` 再 `send()` → **自己也能看到 "xxx joined."**
- 先 `send()` 再 `subscribe()` → 自己看不到自己的加入消息

这是个很小的顺序差别,但直接影响用户体验。**类似的时序问题在事件驱动的系统里很常见** —— 订阅晚了就漏消息。

### 为什么不能"发给除某人以外的所有人"

[第 36 章](36-websockets-http2.md#动手练习)的练习题 2 提过这个局限,这里说清。

`broadcast::Sender::send` 是**无条件广播**:所有订阅者都收到,没有"排除某人"的选项。所以本章的效果是:

**你发的消息自己也会收到。** 前端把它显示出来,看起来像"发送成功的回显"。

**这实际上简化了前端** —— 不需要本地先显示自己的消息再等服务端确认,统一由服务端回显保证顺序一致。**很多聊天应用故意这么设计**,因为它保证了所有人看到的消息顺序完全相同。

想实现"不回显给自己"有几种办法:

- 广播的消息里带上发送者 ID,接收时比对跳过(要改消息格式)
- 每个连接一个独立的 `mpsc`,服务端维护连接列表并逐个发送(不用 `broadcast`,但要自己管理列表)
- 前端过滤(需要知道自己的标识)

**本章选择最简单的:不做。** 这在聊天场景是合理的默认。

### 三段式的连接生命周期

把 `websocket` 函数的结构抽出来看,是很典型的三段:

```rust
async fn websocket(stream: WebSocket, state: Arc<AppState>) {
    // ① 准备:split、注册用户名、订阅、广播加入
    // ② 运行:两个任务并发收发,select! 等任一结束
    // ③ 清理:广播离开、释放用户名
}
```

**第 ③ 段是最容易被忘的。** 连接断开时必须:

- 广播 "xxx left." 让其他人知道
- **把用户名从 `user_set` 里删掉**,否则这个名字永远被占用

```rust
let msg = format!("{username} left.");
let _ = state.tx.send(msg);

// Remove username from map so new clients can take it again.
state.user_set.lock().unwrap().remove(&username);
```

**这段代码在 `select!` 之后执行** —— 因为 `select!` 返回意味着某个方向已经结束,连接不可用了。

**注意这里有个隐患:如果这个 async 函数被取消(比如整个服务关闭),清理代码不会执行。** 真正健壮的做法是用一个实现了 `Drop` 的守卫类型,保证无论如何都会清理。「代价与边界」会展开。

### include_str! 内嵌 HTML

```rust
// Include utf-8 file at **compile** time.
async fn index() -> Html<&'static str> {
    Html(std::include_str!("../chat.html"))
}
```

这和前面几章用 `ServeDir` 提供静态文件不同 —— [第 05 章](05-templates-minijinja.md#include_str-把文件内容嵌进二进制)讲过 `include_str!` 在**编译期**把文件读成 `&'static str` 嵌进二进制。

**取舍很清楚:**

| | `include_str!` | `ServeDir` |
|---|---|---|
| 部署 | **只需一个二进制** | 要带上资源目录 |
| 改 HTML | 要重新编译 | 直接改文件 |
| 缓存头部 | 自己处理 | 自动处理 |
| 多文件 | 每个都要写一行 | 整个目录 |

**单个 HTML 文件用 `include_str!` 很合适**,资源多了就该用 `ServeDir`。

## 项目搭建

```bash
cargo new chat
cd chat
```

```bash
cargo add axum --features ws
cargo add futures-util --no-default-features --features sink,std
cargo add tokio --features full
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

依赖很少 —— 这一章不需要 TLS、不需要静态文件服务、不需要额外的 channel 库。

### 前端页面

`chat.html`(注意在**项目根目录**,不是 `assets/`):

```html
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

                const websocket = new WebSocket("ws://localhost:3000/websocket");

                websocket.onopen = function() {
                    console.log("connection opened");
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
```

**前端的流程**:填用户名 → 点 Join → 建立 WebSocket → `onopen` 时发用户名 → 之后输入框回车发消息 → `onmessage` 追加到文本域。

**注意 `input.onkeydown` 是在 Join 按钮的点击回调里注册的** —— 所以没加入之前回车不会发送。

## 分步构建

### 步骤 1:状态与路由骨架

**要做什么**:定义共享状态,搭出两条路由。

```rust
// src/main.rs
use axum::{
    extract::{ws::WebSocketUpgrade, State},
    response::{Html, IntoResponse},
    routing::get,
    Router,
};
use std::{
    collections::HashSet,
    sync::{Arc, Mutex},
};
use tokio::sync::broadcast;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// Our shared state
struct AppState {
    // We require unique usernames. This tracks which usernames have been taken.
    user_set: Mutex<HashSet<String>>,
    // Channel used to send messages to all connected clients.
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

    // Set up application state for use with with_state().
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

async fn websocket(_stream: axum::extract::ws::WebSocket, _state: Arc<AppState>) {
    // 下一步实现
}

// Include utf-8 file at **compile** time.
async fn index() -> Html<&'static str> {
    Html(std::include_str!("../chat.html"))
}
```

**`let (tx, _rx) = broadcast::channel(100);`** —— 创建通道时会同时得到一个 `Receiver`,但这里不需要它(每个连接自己 `subscribe()`)。用 `_rx` 而不是 `_` 是因为:**如果直接丢弃它没问题,但如果所有 `Receiver` 都被丢弃且没有新的订阅,`send()` 会返回 `Err`** —— 不过这里 `tx` 存在状态里,随时能 `subscribe()`,所以无所谓。

**容量 100** 比[第 36 章](36-websockets-http2.md#broadcast-channel一对多)那个 16 大不少。聊天消息可能突发,给足缓冲减少 `Lagged`。

**日志级别默认是 `trace`** 而非前面章节的 `debug`,官方示例如此。

**`route("/websocket", get(websocket_handler))`** 用 `get` 而不是[第 34 章](34-websockets.md#websocketupgrade-与-on_upgrade)那个 `any` —— 这里只支持 HTTP/1.1 握手(方法是 `GET`),所以 `get` 够用。

**跑起来看**:

```bash
cargo run
```

浏览器访问 `http://localhost:3000/`,能看到用户名输入框、Join 按钮、聊天区域。

点 Join,控制台显示:

```
connection opened
```

**连接建立了**,但 `websocket` 函数什么都不做,所以立刻返回、连接关闭:

```
connection closed
```

按钮重新变成可用状态。

### 步骤 2:用户名注册

**要做什么**:实现 `websocket` 的第一段——读第一条消息当用户名,校验唯一性。

```rust
// src/main.rs
use axum::{
    extract::{
        ws::{Message, Utf8Bytes, WebSocket, WebSocketUpgrade},      // 改动
        State,
    },
    response::{Html, IntoResponse},
    routing::get,
    Router,
};
use futures_util::{sink::SinkExt, stream::StreamExt};                // 新增
// ...(其余 use 不变)

// ...(AppState、main、websocket_handler、index 不变)

// 改动:实现用户名注册
async fn websocket(stream: WebSocket, state: Arc<AppState>) {
    // By splitting, we can send and receive at the same time.
    let (mut sender, mut receiver) = stream.split();

    // Username gets set in the receive loop, if it's valid.
    let mut username = String::new();
    // Loop until a text message is found.
    while let Some(Ok(message)) = receiver.next().await {
        if let Message::Text(name) = message {
            // If username that is sent by client is not taken, fill username string.
            check_username(&state, &mut username, name.as_str());

            // If not empty we want to quit the loop else we want to quit function.
            if !username.is_empty() {
                break;
            } else {
                // Only send our client that username is taken.
                let _ = sender
                    .send(Message::Text(Utf8Bytes::from_static(
                        "Username already taken.",
                    )))
                    .await;

                return;
            }
        }
    }

    tracing::debug!("{username} registered");
}

// 新增
fn check_username(state: &AppState, string: &mut String, name: &str) {
    let mut user_set = state.user_set.lock().unwrap();

    if !user_set.contains(name) {
        user_set.insert(name.to_owned());

        string.push_str(name);
    }
}
```

**`stream.split()`** 是[第 34 章](34-websockets.md#split为什么必须拆)讲的关键操作。这里在注册阶段就拆开了,因为拒绝时要单独给这个客户端发消息(而不是广播)。

**`check_username` 的设计有点绕:** 它不返回 `bool`,而是**通过 `&mut String` 参数传出结果** —— 成功就填上名字,失败就保持空。调用方靠 `username.is_empty()` 判断。

这个写法可以商榷。更直白的是返回 `bool` 或 `Option<String>`:

> **更好的写法**:`check_username` 用"填充 `&mut String`"表达成败,调用方靠 `is_empty()` 判断,这让控制流不够直白——而且**用户名为空字符串时会被误判为"已占用"**(前端的 `required` 属性没加在用户名框上,直接点 Join 会发一个空串,`check_username` 会把空串插入 `user_set` 并"成功",但 `username.is_empty()` 为真,于是走到拒绝分支——用户看到 "Username already taken.",而空串已经被占用了)。返回 `Result<String, ()>` 或 `Option<String>` 更清晰,同时应该显式拒绝空用户名。
>
> 顺带说:锁在 `check_username` 里获取并在函数结束时释放,这个粒度是对的——[第 22 章](22-key-value-store.md#持锁跨-await能让服务卡死的陷阱)强调过临界区要短,不能跨 `await`。

**`Utf8Bytes::from_static("Username already taken.")`** 用静态字符串构造消息,避免一次分配。

**`return` 而非 `break`** —— 用户名被占用时直接退出整个函数,连接随之关闭。

**跑起来看**:

```bash
cargo run
```

浏览器访问,输入 `alice`,点 Join。服务端日志:

```
DEBUG chat: alice registered
```

连接随后关闭(函数返回了),但注册成功了。

**开第二个窗口,也用 `alice`:**

控制台:

```
received message: Username already taken.
connection closed
```

**唯一性校验生效。**

**换个名字 `bob`:**

```
DEBUG chat: bob registered
```

**验证那个空用户名的 bug。** 不填用户名直接点 Join:

浏览器控制台:

```
received message: Username already taken.
```

**明明是第一次连接,却说用户名被占用** —— 这正是上面「更好的写法」指出的问题:空串被插入了 `user_set`,但 `username.is_empty()` 判断为失败。

而且**空串现在真的被占用了** —— 后续再有人发空用户名,会真的因为"已存在"而被拒。练习题会让你修。

### 步骤 3:加入广播与消息转发

**要做什么**:实现第二段——订阅通道、广播加入、两个任务并发收发。

```rust
// src/main.rs
// ...(use 不变)

async fn websocket(stream: WebSocket, state: Arc<AppState>) {
    // ...(split 和用户名注册不变)

    // 新增:We subscribe *before* sending the "joined" message, so that we will also
    // display it to our client.
    let mut rx = state.tx.subscribe();

    // Now send the "joined" message to all subscribers.
    let msg = format!("{username} joined.");
    tracing::debug!("{msg}");
    let _ = state.tx.send(msg);

    // Spawn the first task that will receive broadcast messages and send text
    // messages over the websocket to our client.
    let mut send_task = tokio::spawn(async move {
        while let Ok(msg) = rx.recv().await {
            // In any websocket error, break loop.
            if sender.send(Message::text(msg)).await.is_err() {
                break;
            }
        }
    });

    // Clone things we want to pass (move) to the receiving task.
    let tx = state.tx.clone();
    let name = username.clone();

    // Spawn a task that takes messages from the websocket, prepends the user
    // name, and sends them to all broadcast subscribers.
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(Message::Text(text))) = receiver.next().await {
            // Add username before message.
            let _ = tx.send(format!("{name}: {text}"));
        }
    });

    // If any one of the tasks run to completion, we abort the other.
    tokio::select! {
        _ = &mut send_task => recv_task.abort(),
        _ = &mut recv_task => send_task.abort(),
    };
}

// ...(check_username、index 不变)
```

**`subscribe()` 在 `send()` 之前** —— 前置知识讲过这个顺序决定了自己能否看到自己的加入消息。

**两个任务的分工:**

- **`send_task`** —— 从广播通道收消息,写进 WebSocket。**它是"广播 → 这个客户端"的桥梁。**
- **`recv_task`** —— 从 WebSocket 收消息,加上用户名前缀后广播。**它是"这个客户端 → 广播"的桥梁。**

这个结构和[第 34 章](34-websockets.md#步骤-2拆成读写两半并发工作)一样,但两个任务的内容完全不同 —— 那边是定时推送和打印,这边是真正的消息中转。

**`let tx = state.tx.clone(); let name = username.clone();`** 两个克隆是为了 `move` 进 `recv_task`。`broadcast::Sender` 的克隆很廉价(内部是 `Arc`)。

**`while let Some(Ok(Message::Text(text))) = receiver.next().await`** 这个模式**只处理文本消息**,而且**遇到非文本消息就退出循环**(模式不匹配导致 `while let` 结束)。

这有个副作用:客户端发一个 `Ping` 或 `Binary` 消息就会导致 `recv_task` 结束,进而 `select!` 触发、连接关闭。**这算不算 bug 取决于你怎么看** —— 对聊天室来说只接受文本是合理的,但用"退出循环"来处理非文本消息不够明确。

**`select!` 后面没有 `match` 取返回值**,直接 `_ =` 忽略。比[第 34 章](34-websockets.md#步骤-2拆成读写两半并发工作)那个打印计数的版本简洁。

**跑起来看**:

```bash
cargo run
```

**开两个窗口**,分别用 `alice` 和 `bob` 加入。

窗口 A(alice)会看到:

```
alice joined.
bob joined.
```

窗口 B(bob)只看到:

```
bob joined.
```

**为什么 bob 看不到 "alice joined."?** 因为那条消息在 bob 订阅之前就发出去了 —— `broadcast` 只投递订阅之后的消息。**这是合理的**(新人不需要看到历史事件),但如果你想让新人看到"当前在线列表",需要单独实现。

**在 alice 窗口输入 `你好` 回车:**

两个窗口都显示:

```
alice: 你好
```

**alice 自己也看到了** —— 前置知识讲过 `broadcast` 无法"排除某人"。

**在 bob 窗口输入 `嗨`:**

两个窗口都显示 `bob: 嗨`。

**消息顺序在两个窗口完全一致** —— 这是服务端统一回显带来的好处。

服务端日志:

```
DEBUG chat: alice joined.
DEBUG chat: bob joined.
```

**现在关掉 bob 的窗口。** alice 窗口没有任何变化 —— 因为还没实现离开广播。而且服务端也没释放 `bob` 这个名字:

**重新开一个窗口用 `bob` 加入:**

```
received message: Username already taken.
```

**名字被永久占用了。** 下一步修。

### 步骤 4:清理与离开广播

**要做什么**:实现第三段——连接断开时广播离开、释放用户名。

```rust
// src/main.rs
// ...(前面不变)

async fn websocket(stream: WebSocket, state: Arc<AppState>) {
    // ...(前两段不变)

    // If any one of the tasks run to completion, we abort the other.
    tokio::select! {
        _ = &mut send_task => recv_task.abort(),
        _ = &mut recv_task => send_task.abort(),
    };

    // 新增:Send "user left" message (similar to "joined" above).
    let msg = format!("{username} left.");
    tracing::debug!("{msg}");
    let _ = state.tx.send(msg);

    // Remove username from map so new clients can take it again.
    state.user_set.lock().unwrap().remove(&username);
}

// ...(其余不变)
```

**两行清理代码,但都很重要:**

**广播离开** 让其他人知道有人走了。注意这条消息发出去时,这个连接的 `send_task` 已经结束了,所以**离开的人自己收不到**(也不需要)。

**从 `user_set` 删除** 让名字能被重新使用。忘了这一步就是步骤 3 最后看到的那个问题。

**跑起来看**:

```bash
cargo run
```

两个窗口分别用 `alice`、`bob` 加入,然后**关掉 bob 的窗口**。

alice 窗口立刻显示:

```
bob left.
```

服务端日志:

```
DEBUG chat: bob left.
```

**重新用 `bob` 加入**:

```
bob joined.
```

**名字释放了。** 完整的生命周期跑通了。

**完整走一遍**:

```
alice joined.        ← alice 加入(自己可见)
bob joined.          ← bob 加入(两人可见)
alice: 你好
bob: 嗨
bob left.            ← bob 关窗口(alice 可见)
bob joined.          ← bob 重新加入,名字可用
```

**验证清理代码不执行的情况。** 这个比较难在浏览器里演示,但可以理解:如果整个服务被 `Ctrl-C` 关闭,`websocket` 函数会被取消,**那两行清理代码不会执行**。对本例无所谓(进程都退出了),但如果状态存在外部(Redis、数据库),就会留下脏数据。

**正确做法是用 `Drop` 守卫**:

```rust
struct UserGuard { state: Arc<AppState>, username: String }

impl Drop for UserGuard {
    fn drop(&mut self) {
        let _ = self.state.tx.send(format!("{} left.", self.username));
        self.state.user_set.lock().unwrap().remove(&self.username);
    }
}
```

创建一个 `UserGuard` 放在函数里,**无论函数怎么结束(正常返回、panic、被取消)都会执行清理**。练习题会让你实现。

## 完整代码

`chat.html`:

```html
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

                const websocket = new WebSocket("ws://localhost:3000/websocket");

                websocket.onopen = function() {
                    console.log("connection opened");
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
```

`src/main.rs`:

```rust
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

// Our shared state
struct AppState {
    // We require unique usernames. This tracks which usernames have been taken.
    user_set: Mutex<HashSet<String>>,
    // Channel used to send messages to all connected clients.
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

    // Set up application state for use with with_state().
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

// This function deals with a single websocket connection, i.e., a single
// connected client / user, for which we will spawn two independent tasks (for
// receiving / sending chat messages).
async fn websocket(stream: WebSocket, state: Arc<AppState>) {
    // By splitting, we can send and receive at the same time.
    let (mut sender, mut receiver) = stream.split();

    // Username gets set in the receive loop, if it's valid.
    let mut username = String::new();
    // Loop until a text message is found.
    while let Some(Ok(message)) = receiver.next().await {
        if let Message::Text(name) = message {
            // If username that is sent by client is not taken, fill username string.
            check_username(&state, &mut username, name.as_str());

            // If not empty we want to quit the loop else we want to quit function.
            if !username.is_empty() {
                break;
            } else {
                // Only send our client that username is taken.
                let _ = sender
                    .send(Message::Text(Utf8Bytes::from_static(
                        "Username already taken.",
                    )))
                    .await;

                return;
            }
        }
    }

    // We subscribe *before* sending the "joined" message, so that we will also
    // display it to our client.
    let mut rx = state.tx.subscribe();

    // Now send the "joined" message to all subscribers.
    let msg = format!("{username} joined.");
    tracing::debug!("{msg}");
    let _ = state.tx.send(msg);

    // Spawn the first task that will receive broadcast messages and send text
    // messages over the websocket to our client.
    let mut send_task = tokio::spawn(async move {
        while let Ok(msg) = rx.recv().await {
            // In any websocket error, break loop.
            if sender.send(Message::text(msg)).await.is_err() {
                break;
            }
        }
    });

    // Clone things we want to pass (move) to the receiving task.
    let tx = state.tx.clone();
    let name = username.clone();

    // Spawn a task that takes messages from the websocket, prepends the user
    // name, and sends them to all broadcast subscribers.
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(Message::Text(text))) = receiver.next().await {
            // Add username before message.
            let _ = tx.send(format!("{name}: {text}"));
        }
    });

    // If any one of the tasks run to completion, we abort the other.
    tokio::select! {
        _ = &mut send_task => recv_task.abort(),
        _ = &mut recv_task => send_task.abort(),
    };

    // Send "user left" message (similar to "joined" above).
    let msg = format!("{username} left.");
    tracing::debug!("{msg}");
    let _ = state.tx.send(msg);

    // Remove username from map so new clients can take it again.
    state.user_set.lock().unwrap().remove(&username);
}

fn check_username(state: &AppState, string: &mut String, name: &str) {
    let mut user_set = state.user_set.lock().unwrap();

    if !user_set.contains(name) {
        user_set.insert(name.to_owned());

        string.push_str(name);
    }
}

// Include utf-8 file at **compile** time.
async fn index() -> Html<&'static str> {
    Html(std::include_str!("../chat.html"))
}
```

## 代价与边界

**这是本教程"能跑但离生产很远"最典型的一个示例。** 逐条说。

**空用户名的 bug。** 步骤 2 验证过:不填用户名会被误判为"已占用",而且空串会永久占据一个位置。根源是 `check_username` 用 `is_empty()` 表达失败。

**没有注册超时。** 前置知识提过:连上后不发用户名就会一直停在 `while let` 循环里。**恶意客户端可以开成千上万个连接,每个都不发消息**,白占内存和文件描述符。修法是用 `tokio::time::timeout` 包住注册阶段。

**清理代码可能不执行。** 步骤 4 讲过,应该用 `Drop` 守卫。

**没有认证。** 任何人都能用任何用户名加入。真实聊天室需要先登录([第 30 章](30-jwt.md)/[第 31 章](31-oauth.md)),用户名从认证信息里来而不是客户端自称。**"客户端说自己是谁就是谁"是最基本的安全问题。**

**没有校验 Origin。** [第 34 章](34-websockets.md#代价与边界)强调过 Cross-Site WebSocket Hijacking。

**消息没有长度限制、没有限流。** 客户端可以发 100MB 的消息,或者每秒发一万条。前者需要 `WebSocketUpgrade::max_message_size()`,后者需要自己实现速率限制。

**消息没有转义,前端有 XSS 隐患。** 本例前端用 `textarea.value +=`,而 `textarea` 的 `value` 是纯文本,**恰好没有 XSS 风险**。但如果改成 `innerHTML +=`,发一条 `<img src=x onerror=alert(1)>` 就能在所有人浏览器里执行脚本。[第 04 章](04-templates.md#模板引擎解决什么问题)讲过这个问题。

**消息不持久化。** 刷新页面就什么都没了,新加入的人看不到历史消息。真实应用要存数据库并支持拉取历史。

**没有"当前在线列表"。** 新人只能看到自己加入之后的事件。前置知识讲过 `broadcast` 的语义决定了这一点,想要在线列表得单独实现(遍历 `user_set` 发给新人)。

**`broadcast` 的 `Lagged` 没有处理。** `while let Ok(msg) = rx.recv().await` 遇到 `Err`(包括 `Lagged`)就退出循环 —— 也就是说**某个客户端消费慢到落后 100 条,它的连接就会被断开**。[第 36 章](36-websockets-http2.md#代价与边界)那边是 `continue` 跳过,这边是直接断连。两种都不理想,合理的做法是记日志 + 通知客户端"你落后了,请刷新"。

**非文本消息导致连接关闭。** 步骤 3 提过:`while let Some(Ok(Message::Text(text)))` 遇到 `Binary`/`Ping` 会因模式不匹配而结束循环。**客户端发一个 ping 就被踢下线**,这显然不对。

**单实例限制。** `user_set` 和 `broadcast` 都是进程内的。多实例部署时用户名唯一性无法保证,消息也传不到别的实例。需要 Redis 之类的外部协调。

**没有心跳。** 和[第 34 章](34-websockets.md#代价与边界)一样:半开连接检测不到,那个连接会一直占着用户名。

## 动手练习

**1. 修掉空用户名的 bug,并改造 `check_username` 的签名。**

提示:改成 `fn check_username(state: &AppState, name: &str) -> Option<String>`,内部先判断 `name.trim().is_empty()` 就返回 `None`。调用方用 `match` 或 `let Some(u) = ... else { 拒绝并 return }`。同时给前端的用户名输入框加 `required`。验证:不填名字点 Join 应该看到明确的"用户名不能为空"而不是"已被占用"。**这是「代价与边界」里最该修的一处。**

**2. 用 `Drop` 守卫保证清理一定执行。**

提示:定义前置知识里那个 `UserGuard`,在注册成功后立刻创建一个,删掉函数末尾的两行清理代码。验证:在 `select!` 之后加一行 `panic!("test")`,确认 `bob left.` 仍然广播了、名字仍然被释放了(用另一个窗口试着用同名加入)。**这个练习能让你理解 RAII 在异步代码里的价值。**

**3. 加上注册超时。**

提示:用 `tokio::time::timeout(Duration::from_secs(10), async { /* 注册循环 */ })` 包住注册阶段,超时就关闭连接。验证:用 `websocat ws://localhost:3000/websocket` 连上但什么都不发,10 秒后应该被断开。

**4. 给新加入的人发送当前在线列表。**

提示:在广播 "joined" 之前,遍历 `user_set` 拼一条 `"Online: alice, bob"` **只发给这个客户端**(用 `sender.send(...)` 而不是 `state.tx.send(...)`)。注意此时 `sender` 还没被 move 进 `send_task`,所以可以直接用。做完你会理解为什么"只发给一个人"和"广播"需要不同的通道。

## 小结与下一章预告

一个完整的 WebSocket 应用是**三段式**的:准备(split、注册、订阅、广播加入)→ 运行(两任务并发 + `select!`)→ 清理(广播离开、释放资源)。**第三段最容易被忘,而且如果函数被取消它根本不会执行** —— 稳妥的做法是用 `Drop` 守卫。

因为浏览器的 `WebSocket` 不能自定义请求头,本章用了**应用层的第二次握手**:第一条消息当作注册请求。代价是存在"已连接但未注册"的中间状态,**必须有超时保护**,否则会被用来耗资源。

`broadcast` channel 的两个语义细节直接影响体验:**只投递订阅之后的消息**(所以 `subscribe()` 要在广播 "joined" 之前,新人也看不到历史事件),以及**无法"排除某人"**(所以自己发的消息自己也收到——这在聊天场景反而是好事,保证了所有人看到的顺序一致)。

`Mutex` 只包住 `user_set` 而不是整个 `AppState`,因为 `broadcast::Sender` 自带同步。**只锁真正需要保护的部分。**

这个示例是本教程"能跑但离生产很远"最典型的一个,「代价与边界」列了十几条,其中几个是真 bug:空用户名会被误判为已占用且永久占位、客户端发一个 `Ping` 就会被踢下线、某个客户端消费慢到落后 100 条就被断连。**它演示的是结构和 API 用法,不是可以照搬的聊天室。**

---

「实时通信」阶段(33~37)到此结束。你现在能用 SSE 做单向推送、用 WebSocket 做双向通信、知道两者怎么测、了解 HTTP/2 上的差别,也见过一个完整应用的结构。

接下来是「内容分发与代理」阶段。[第 38 章 static-file-server](38-static-file-server.md) 终于讲那个从[第 33 章](33-sse.md#步骤-3配上前端页面)起就在用、但一直挂着"本章不深入"的 `ServeDir`:静态文件服务怎么配、路径穿越怎么防、和 `fallback` 怎么组合、以及为什么它被排在这么后面(因为它依赖 `Layer` 和测试的知识)。
