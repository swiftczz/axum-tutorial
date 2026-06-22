# 44. testing-websockets

对应示例：`examples/testing-websockets`

WebSocket 测试比普通 HTTP 接口复杂——不是一次请求一次响应，而是持续双向连接。本章学两种测试方式：启动真实服务器做**集成测试**，把 socket 拆成 `Sink`/`Stream` 后用 channel 做**单元测试**。

分 3 步：先写一个集成测试版 handler（普通 WebSocket 写法），再把它重构成泛型 `Sink`/`Stream` 版（可单元测试），最后看两种测试怎么写。

相比前面章节新引入：**`socket.split()` 拆 Sink/Stream、泛型 handler 解耦 socket 类型、集成测试（真实服务 + `tokio_tungstenite`）、单元测试（`futures_channel::mpsc` + channel）**。

## Cargo.toml

````toml
[package]
name = "example-testing-websockets"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["ws"] }
futures-channel = { version = "0.3", features = ["sink"] }
futures-util = { version = "0.3", default-features = false, features = ["sink", "std"] }
tokio = { version = "1.0", features = ["full"] }
tokio-tungstenite = "0.29"
````

`tokio-tungstenite` 是测试用的真实 WebSocket 客户端；`futures-channel` 是单元测试用的 channel。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：集成测试版 echo handler（普通 WebSocket 写法）

先写一个最简单的 WebSocket echo：客户端发什么，服务端原样回 `You said: ...`。这版用普通的 `WebSocket` 类型直接 `socket.recv()` / `socket.send()`，最直观。

````rust
use axum::{
    extract::{
        ws::{Message, WebSocket},
        WebSocketUpgrade,
    },
    response::Response,
    routing::get,
    Router,
};

#[tokio::main]
async fn main() {
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app()).await.unwrap();
}

fn app() -> Router {
    Router::new().route("/integration-testable", get(integration_testable_handler))
}

async fn integration_testable_handler(ws: WebSocketUpgrade) -> Response {
    ws.on_upgrade(integration_testable_handle_socket)
}

// 收到 text 消息就回 "You said: ..."
async fn integration_testable_handle_socket(mut socket: WebSocket) {
    while let Some(Ok(msg)) = socket.recv().await {
        if let Message::Text(msg) = msg {
            if socket
                .send(Message::Text(format!("You said: {msg}").into()))
                .await
                .is_err()
            {
                break;
            }
        }
    }
}
````

验证（参考第 41 章 WebSocket 用法）：

````bash
cd examples
cargo run -p example-testing-websockets
````

用任意 WebSocket 客户端连 `ws://127.0.0.1:3000/integration-testable`，发 `foo`，会收到 `You said: foo`。

这版能跑，但 handler 依赖具体的 `WebSocket` 类型——测试时要么启动真实服务（第三步的集成测试），要么…… 就没别的办法了。下一步把它重构成可单元测试的版本。

---

## 第二步：单元测试版——把 socket 拆成泛型 `Sink`/`Stream`

为了让消息处理逻辑不依赖具体 socket 类型，把 `WebSocket` **拆成两半**：`write`（写端，`Sink`）和 `read`（读端，`Stream`）。业务函数只要求泛型参数实现这两个 trait，测试时就能用 channel 代替真实 socket。

````rust
use axum::{
    extract::{
        ws::{Message, WebSocket},
        WebSocketUpgrade,
    },
    response::Response,
    routing::get,
    Router,
};
use futures_util::{Sink, SinkExt, Stream, StreamExt};

# #[tokio::main]
# async fn main() { /* ... */ }

fn app() -> Router {
    Router::new()
        .route("/integration-testable", get(integration_testable_handler))
        .route("/unit-testable", get(unit_testable_handler))
}

# async fn integration_testable_handler(ws: WebSocketUpgrade) -> Response { /* 第一步的代码 */ }
# async fn integration_testable_handle_socket(mut socket: WebSocket) { /* ... */ }

async fn unit_testable_handler(ws: WebSocketUpgrade) -> Response {
    ws.on_upgrade(|socket| {
        // 关键改动：把 socket 拆成 write + read
        let (write, read) = socket.split();
        unit_testable_handle_socket(write, read)
    })
}

// 业务函数不依赖具体 socket 类型，只要求 write: Sink<Message>, read: Stream<...>
async fn unit_testable_handle_socket<W, R>(mut write: W, mut read: R)
where
    W: Sink<Message> + Unpin,
    R: Stream<Item = Result<Message, axum::Error>> + Unpin,
{
    while let Some(Ok(msg)) = read.next().await {
        if let Message::Text(msg) = msg {
            if write
                .send(Message::Text(format!("You said: {msg}").into()))
                .await
                .is_err()
            {
                break;
            }
        }
    }
}
````

> **新面孔：`socket.split()` 拆 Sink/Stream**
>
> `WebSocket` 同时实现了 `Sink<Message>`（写）和 `Stream<Item = Result<Message, _>>`（读）。调用 `.split()` 把它拆成独立的 `write` 和 `read` 两半，分别用 `SinkExt::send` 和 `StreamExt::next` 操作。
>
> 拆开的好处：业务函数只要求**泛型** `Sink` 和 `Stream`，测试时可以用任意实现这两个 trait 的类型（如 channel）代替。

> **新面孔：`Sink<Message>` / `Stream<Item = Result<Message, _>>`**
>
> 来自 `futures-util` 的两个 trait：
> - `Sink<T>`：能往里**写** `T`（`.send(...).await`），类似发送端
> - `Stream<Item = T>`：能从里**读** `T`（`.next().await`），类似迭代器/接收端
>
> `WebSocket` 同时实现两者，`futures_channel::mpsc::Sender` 实现 `Sink`，`Receiver` 实现 `Stream`——所以 channel 能完美模拟 socket。

这步改完，handler 在生产环境行为和第一步完全一样（`socket.split()` 拿到的就是真实 socket 的两半），但业务函数已可单元测试。

---

## 第三步：写两种测试

### 集成测试——启动真实服务 + `tokio_tungstenite`

````rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::{
        future::IntoFuture,
        net::{Ipv4Addr, SocketAddr},
    };
    use tokio_tungstenite::tungstenite;

    #[tokio::test]
    async fn integration_test() {
        // 绑端口 0 让系统分配可用端口
        let listener = tokio::net::TcpListener::bind(SocketAddr::from((Ipv4Addr::UNSPECIFIED, 0)))
            .await
            .unwrap();
        let addr = listener.local_addr().unwrap();
        tokio::spawn(axum::serve(listener, app()).into_future());

        // 用真实 WebSocket 客户端连
        let (mut socket, _response) =
            tokio_tungstenite::connect_async(format!("ws://{addr}/integration-testable"))
                .await
                .unwrap();

        socket
            .send(tungstenite::Message::text("foo"))
            .await
            .unwrap();

        let msg = match socket.next().await.unwrap().unwrap() {
            tungstenite::Message::Text(msg) => msg,
            other => panic!("expected a text message but got {other:?}"),
        };

        assert_eq!(msg.as_str(), "You said: foo");
    }
}
````

> **新面孔：`tokio_tungstenite::connect_async`**
>
> 真实的 WebSocket 客户端库，用 `connect_async("ws://...")` 连接服务端。返回的 `socket` 也是 `Sink` + `Stream`，能 `.send(...)` 发消息、`.next().await` 收消息。
>
> 注意它用的是 `tungstenite::Message`（不是 `axum::extract::ws::Message`），两者是不同类型，但因为都是 `Sink`/`Stream`，互操作无障碍。

> **新面孔：绑定端口 `0`**
>
> `TcpListener::bind((Ipv4Addr::UNSPECIFIED, 0))` 让操作系统分配可用端口。`.local_addr()` 取到实际分配的端口再连。这样测试不依赖固定端口，避免冲突（同 40 章 SSE 测试）。

### 单元测试——`futures_channel::mpsc` 模拟 socket

````rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn unit_test() {
        // 用 futures channel（不是 tokio channel），因为它实现 Sink + Stream
        let (socket_write, mut test_rx) = futures_channel::mpsc::channel(1024);
        let (mut test_tx, socket_read) = futures_channel::mpsc::channel(1024);

        tokio::spawn(unit_testable_handle_socket(socket_write, socket_read));

        test_tx
            .send(Ok(Message::Text("foo".into())))
            .await
            .unwrap();

        let msg = match test_rx.next().await.unwrap() {
            Message::Text(msg) => msg,
            other => panic!("expected a text message but got {other:?}"),
        };

        assert_eq!(msg.as_str(), "You said: foo");
    }
}
````

> **新面孔：`futures_channel::mpsc`**
>
> 来自 `futures-channel` crate 的多生产者单消费者 channel。和 tokio 的 mpsc 不同，它同时实现了 `Sink` 和 `Stream`，所以能完美模拟 `WebSocket` 的读写两半。
>
> 两组 channel 的对应关系：
>
> ```text
> test_tx    → socket_read      模拟客户端 → 服务端
> socket_write → test_rx        模拟服务端 → 客户端
> ```
>
> 测试代码 `test_tx.send(...)` 模拟客户端发消息，`test_rx.next().await` 读服务端响应——**完全不需要启动真实服务**。

---

## 完整代码

````rust
use axum::{
    extract::{
        ws::{Message, WebSocket},
        WebSocketUpgrade,
    },
    response::Response,
    routing::get,
    Router,
};
use futures_util::{Sink, SinkExt, Stream, StreamExt};

#[tokio::main]
async fn main() {
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app()).await.unwrap();
}

fn app() -> Router {
    // WebSocket routes can generally be tested in two ways:
    //
    // - Integration tests where you run the server and connect with a real WebSocket client.
    // - Unit tests where you mock the socket as some generic send/receive type
    //
    // Which version you pick is up to you. Generally we recommend the integration test version
    // unless your app has a lot of setup that makes it hard to run in a test.
    Router::new()
        .route("/integration-testable", get(integration_testable_handler))
        .route("/unit-testable", get(unit_testable_handler))
}

// A WebSocket handler that echos any message it receives.
//
// This one we'll be integration testing so it can be written in the regular way.
async fn integration_testable_handler(ws: WebSocketUpgrade) -> Response {
    ws.on_upgrade(integration_testable_handle_socket)
}

async fn integration_testable_handle_socket(mut socket: WebSocket) {
    while let Some(Ok(msg)) = socket.recv().await {
        if let Message::Text(msg) = msg {
            if socket
                .send(Message::Text(format!("You said: {msg}").into()))
                .await
                .is_err()
            {
                break;
            }
        }
    }
}

// The unit testable version requires some changes.
//
// By splitting the socket into an `impl Sink` and `impl Stream` we can test without providing a
// real socket and instead using channels, which also implement `Sink` and `Stream`.
async fn unit_testable_handler(ws: WebSocketUpgrade) -> Response {
    ws.on_upgrade(|socket| {
        let (write, read) = socket.split();
        unit_testable_handle_socket(write, read)
    })
}

// The implementation is largely the same as `integration_testable_handle_socket` except we call
// methods from `SinkExt` and `StreamExt`.
async fn unit_testable_handle_socket<W, R>(mut write: W, mut read: R)
where
    W: Sink<Message> + Unpin,
    R: Stream<Item = Result<Message, axum::Error>> + Unpin,
{
    while let Some(Ok(msg)) = read.next().await {
        if let Message::Text(msg) = msg {
            if write
                .send(Message::Text(format!("You said: {msg}").into()))
                .await
                .is_err()
            {
                break;
            }
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::{
        future::IntoFuture,
        net::{Ipv4Addr, SocketAddr},
    };
    use tokio_tungstenite::tungstenite;

    // We can integration test one handler by running the server in a background task and
    // connecting to it like any other client would.
    #[tokio::test]
    async fn integration_test() {
        let listener = tokio::net::TcpListener::bind(SocketAddr::from((Ipv4Addr::UNSPECIFIED, 0)))
            .await
            .unwrap();
        let addr = listener.local_addr().unwrap();
        tokio::spawn(axum::serve(listener, app()).into_future());

        let (mut socket, _response) =
            tokio_tungstenite::connect_async(format!("ws://{addr}/integration-testable"))
                .await
                .unwrap();

        socket
            .send(tungstenite::Message::text("foo"))
            .await
            .unwrap();

        let msg = match socket.next().await.unwrap().unwrap() {
            tungstenite::Message::Text(msg) => msg,
            other => panic!("expected a text message but got {other:?}"),
        };

        assert_eq!(msg.as_str(), "You said: foo");
    }

    // We can unit test the other handler by creating channels to read and write from.
    #[tokio::test]
    async fn unit_test() {
        // Need to use "futures" channels rather than "tokio" channels as they implement `Sink` and
        // `Stream`
        let (socket_write, mut test_rx) = futures_channel::mpsc::channel(1024);
        let (mut test_tx, socket_read) = futures_channel::mpsc::channel(1024);

        tokio::spawn(unit_testable_handle_socket(socket_write, socket_read));

        test_tx.send(Ok(Message::Text("foo".into()))).await.unwrap();

        let msg = match test_rx.next().await.unwrap() {
            Message::Text(msg) => msg,
            other => panic!("expected a text message but got {other:?}"),
        };

        assert_eq!(msg.as_str(), "You said: foo");
    }
}
````

## 运行测试

````bash
cd examples
cargo test -p example-testing-websockets
````

`integration_test` 和 `unit_test` 都应通过。

## 解读

### 两种测试策略

| 策略 | 做法 | 优点 |
| --- | --- | --- |
| **集成测试** | 启动真实 Axum 服务 + `tokio_tungstenite` 真实客户端连接 | 最接近真实行为，覆盖 HTTP upgrade/路由/服务启动问题 |
| **单元测试** | 不启动服务器，socket 逻辑写成泛型 `Sink`/`Stream`，用 channel 模拟读写 | 更轻，易测纯消息处理逻辑，不需真实端口 |

两个 handler 业务相同（客户端发 `foo` → 服务端返 `You said: foo`），区别只在测试方式。

## 常见问题

**WebSocket 测试一定要启动真实服务器吗？** 不一定。要覆盖 upgrade/路由/真实协议用集成测试；只测消息处理逻辑用泛型 Sink/Stream 单元测试。

**为什么单元测试不用 tokio channel？** 需要实现 `Sink`+`Stream` 的 channel，`futures::channel::mpsc` 适合（tokio 的 mpsc 不实现 `Sink`）。

**为什么集成测试绑定端口 0？** 让系统分配可用端口，避免端口冲突。

## 手写任务

1. 写 WebSocket echo handler。
2. 写集成测试启动真实服务，用 `tokio_tungstenite` 连接。
3. echo 逻辑改成泛型 `Sink`/`Stream`。
4. 用 `futures::channel::mpsc` 写单元测试。

## 小结

这章用 3 步讲了 WebSocket 测试：

1. **普通 handler**：依赖具体 `WebSocket` 类型，适合集成测试。
2. **泛型 handler**：`socket.split()` 拆成 `Sink` + `Stream`，业务函数从具体 socket 类型解耦。
3. **两种测试**：集成测试（真实服务 + `tokio_tungstenite`）和单元测试（`futures_channel::mpsc` + channel）。

核心技巧：**把业务逻辑从具体 `WebSocket` 类型解耦**，只依赖 `Sink<Message>` + `Stream<Item = Result<Message, _>>`，测试时用 channel 模拟读写。

## 源码对照

- `examples/testing-websockets/Cargo.toml`
- `examples/testing-websockets/src/main.rs`
