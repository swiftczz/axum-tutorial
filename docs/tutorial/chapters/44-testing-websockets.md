# 44. testing-websockets

对应示例：`examples/testing-websockets`

WebSocket 测试比普通 HTTP 接口复杂——不是一次请求一次响应而是持续双向连接。本章学两种测试方式:启动真实服务器做**集成测试**,把 socket 拆成 `Sink`/`Stream` 后用 channel 做**单元测试**。



相比前面章节新引入：**集成测试（真实服务 + `tokio_tungstenite`）vs 单元测试（泛型 `Sink`/`Stream` + channel）**。

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

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

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
    Router::new()
        .route("/integration-testable", get(integration_testable_handler))
        .route("/unit-testable", get(unit_testable_handler))
}

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

async fn unit_testable_handler(ws: WebSocketUpgrade) -> Response {
    ws.on_upgrade(|socket| {
        let (write, read) = socket.split();
        unit_testable_handle_socket(write, read)
    })
}

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
    use std::{future::IntoFuture, net::{Ipv4Addr, SocketAddr}};
    use tokio_tungstenite::tungstenite;

    #[tokio::test]
    async fn integration_test() {
        let listener = tokio::net::TcpListener::bind(SocketAddr::from((Ipv4Addr::UNSPECIFIED, 0)))
            .await.unwrap();
        let addr = listener.local_addr().unwrap();
        tokio::spawn(axum::serve(listener, app()).into_future());

        let (mut socket, _response) =
            tokio_tungstenite::connect_async(format!("ws://{addr}/integration-testable"))
                .await.unwrap();

        socket.send(tungstenite::Message::text("foo")).await.unwrap();

        let msg = match socket.next().await.unwrap().unwrap() {
            tungstenite::Message::Text(msg) => msg,
            other => panic!("expected a text message but got {other:?}"),
        };

        assert_eq!(msg.as_str(), "You said: foo");
    }

    #[tokio::test]
    async fn unit_test() {
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
| **集成测试** | 启动真实 Axum 服务 + `tokio_tungstenite` 真实客户端连接 | 最接近真实行为,覆盖 HTTP upgrade/路由/服务启动问题 |
| **单元测试** | 不启动服务器,socket 逻辑写成泛型 `Sink`/`Stream`,用 channel 模拟读写 | 更轻,易测纯消息处理逻辑,不需真实端口 |

两个 handler 业务相同(客户端发 `foo` → 服务端返 `You said: foo`),区别只在测试方式。

### 集成测试 handler(普通写法)

````rust
async fn integration_testable_handle_socket(mut socket: WebSocket) {
    while let Some(Ok(msg)) = socket.recv().await {
        if let Message::Text(msg) = msg {
            if socket.send(Message::Text(format!("You said: {msg}").into())).await.is_err() {
                break;
            }
        }
    }
}
````

最直接的 WebSocket 写法,依赖具体 `WebSocket` 类型,适合集成测试用真实连接。

### 单元测试 handler(泛型 Sink/Stream)

````rust
async fn unit_testable_handler(ws: WebSocketUpgrade) -> Response {
    ws.on_upgrade(|socket| {
        let (write, read) = socket.split();          // 先 split
        unit_testable_handle_socket(write, read)     // 把读写两半交给泛型函数
    })
}

async fn unit_testable_handle_socket<W, R>(mut write: W, mut read: R)
where
    W: Sink<Message> + Unpin,
    R: Stream<Item = Result<Message, axum::Error>> + Unpin,
{
    while let Some(Ok(msg)) = read.next().await {
        ...write.send(...).await...
    }
}
````

业务函数不依赖真实 `WebSocket`,只要求 `write: Sink<Message>`、`read: Stream<Item = Result<Message, _>>`——所以测试时可用 channel 代替真实 socket。这是**把业务逻辑从具体 socket 类型解耦**的设计技巧。

### 集成测试:启动真实服务

````rust
let listener = tokio::net::TcpListener::bind(SocketAddr::from((Ipv4Addr::UNSPECIFIED, 0))).await.unwrap();
let addr = listener.local_addr().unwrap();
tokio::spawn(axum::serve(listener, app()).into_future());

let (mut socket, _response) = tokio_tungstenite::connect_async(format!("ws://{addr}/integration-testable")).await.unwrap();
socket.send(tungstenite::Message::text("foo")).await.unwrap();
// 断言 "You said: foo"
````

**端口 `0`** 让操作系统分配可用端口,避免和本地其他服务冲突(同 40 章 SSE 测试)。

### 单元测试:用 channel 模拟 socket

````rust
let (socket_write, mut test_rx) = futures_channel::mpsc::channel(1024);   // 服务端写,测试读
let (mut test_tx, socket_read) = futures_channel::mpsc::channel(1024);    // 测试写,服务端读

tokio::spawn(unit_testable_handle_socket(socket_write, socket_read));

test_tx.send(Ok(Message::Text("foo".into()))).await.unwrap();   // 模拟客户端发消息
// 从 test_rx 读取服务端响应,断言 "You said: foo"
````

两组 channel:

```text
test_tx    → socket_read      模拟客户端 → 服务端
socket_write → test_rx        模拟服务端 → 客户端
```

**为什么用 `futures::channel::mpsc` 不用 tokio channel?** 测试需要实现 `Sink` 和 `Stream` 的 channel,`futures::channel::mpsc` 正好满足。

## 常见问题

**WebSocket 测试一定要启动真实服务器吗?** 不一定。要覆盖 upgrade/路由/真实协议用集成测试;只测消息处理逻辑用泛型 Sink/Stream 单元测试。

**为什么单元测试不用 tokio channel?** 需要实现 `Sink`+`Stream` 的 channel,`futures::channel::mpsc` 适合。

**为什么集成测试绑定端口 0?** 让系统分配可用端口,避免端口冲突。

## 手写任务

1. 写 WebSocket echo handler。
2. 写集成测试启动真实服务,用 `tokio_tungstenite` 连接。
3. echo 逻辑改成泛型 `Sink`/`Stream`。
4. 用 `futures::channel::mpsc` 写单元测试。

## 小结

- WebSocket 测试两条路:集成测试(真实服务 + 真实客户端)和单元测试(泛型 Sink/Stream + channel)。
- 一般优先集成测试;业务逻辑复杂、启动真实服务成本高时,把消息处理抽成泛型函数做单元测试。
- 单元测试的关键技巧:**把业务逻辑从具体 `WebSocket` 类型解耦**,只依赖 `Sink<Message>` + `Stream<Item = Result<Message, _>>`,测试时用 `futures::channel::mpsc` 模拟读写。
- 集成测试绑定端口 0 让系统分配,避免冲突。

## 源码对照

- `examples/testing-websockets/Cargo.toml`
- `examples/testing-websockets/src/main.rs`
