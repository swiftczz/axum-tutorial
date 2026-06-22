# 44. testing-websockets

对应示例：`examples/testing-websockets`

本章目标：学习两种 WebSocket 测试方式：启动真实服务器做集成测试，以及把 socket 拆成 `Sink`/`Stream` 后用 channel 做单元测试。

WebSocket 测试比普通 HTTP 接口复杂。  
因为它不是一次请求一次响应，而是一个持续的双向连接。

## 这个小项目在做什么

应用有两个 WebSocket 路由：

```text
/integration-testable -> 用真实 WebSocket 客户端测试
/unit-testable        -> 用 channel 模拟 socket 测试
```

两个 handler 业务都很简单：

```text
客户端发送 foo
服务端返回 You said: foo
```

区别只在测试方式。

## 两种测试策略

### 集成测试

真实启动 Axum 服务，然后用 `tokio_tungstenite` 连接：

```text
测试代码 -> 启动服务器 -> WebSocket 客户端连接 -> 发消息 -> 收响应
```

优点：

```text
最接近真实行为
能覆盖 HTTP upgrade
能发现路由和服务启动问题
```

### 单元测试

不启动真实服务器。  
把 socket 逻辑写成泛型：

```text
impl Sink<Message>
impl Stream<Item = Result<Message, axum::Error>>
```

测试时用 channel 模拟读写。

优点：

```text
更轻
更容易测试纯消息处理逻辑
不需要真实端口
```

## 文件和依赖

这个 example 有两个文件：

1. `examples/testing-websockets/Cargo.toml`：声明 Axum ws、futures-channel、tokio-tungstenite。
2. `examples/testing-websockets/src/main.rs`：实现两个 WebSocket handler 和两个测试。

关键依赖：

- `axum`：WebSocket 支持。
- `tokio-tungstenite`：集成测试里的真实 WebSocket 客户端。
- `futures-channel`：单元测试里提供实现 `Sink`/`Stream` 的 channel。
- `futures-util`：`SinkExt`、`StreamExt`。

## 第一步：普通可集成测试 handler

源码：

````rust
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
````

这就是最直接的 WebSocket 写法。  
接收文本，回发文本。

适合集成测试，因为测试会用真实 WebSocket 连接它。

## 第二步：可单元测试 handler

源码：

````rust
async fn unit_testable_handler(ws: WebSocketUpgrade) -> Response {
    ws.on_upgrade(|socket| {
        let (write, read) = socket.split();
        unit_testable_handle_socket(write, read)
    })
}
````

它先 split：

```text
write
read
```

然后把读写两半交给业务函数。

## 第三步：泛型消息处理函数

源码：

````rust
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

这段不依赖真实 `WebSocket`。  
只要求：

```text
write 能 Sink<Message>
read 能 Stream<Result<Message, axum::Error>>
```

所以测试时可以用 channel 代替真实 socket。

## 第四步：集成测试启动真实服务

源码：

````rust
let listener = tokio::net::TcpListener::bind(SocketAddr::from((Ipv4Addr::UNSPECIFIED, 0)))
    .await
    .unwrap();
let addr = listener.local_addr().unwrap();
tokio::spawn(axum::serve(listener, app()).into_future());
````

端口 `0` 表示让操作系统分配可用端口。  
测试更稳定，不容易和本地其他服务冲突。

然后连接：

````rust
let (mut socket, _response) =
    tokio_tungstenite::connect_async(format!("ws://{addr}/integration-testable"))
        .await
        .unwrap();
````

再发送并断言：

````rust
socket.send(tungstenite::Message::text("foo")).await.unwrap();
...
assert_eq!(msg.as_str(), "You said: foo");
````

## 第五步：单元测试用 channel

源码：

````rust
let (socket_write, mut test_rx) = futures_channel::mpsc::channel(1024);
let (mut test_tx, socket_read) = futures_channel::mpsc::channel(1024);

tokio::spawn(unit_testable_handle_socket(socket_write, socket_read));

test_tx.send(Ok(Message::Text("foo".into()))).await.unwrap();
````

这里创建两组 channel：

```text
test_tx -> socket_read     模拟客户端发给服务端
socket_write -> test_rx    模拟服务端发给客户端
```

最后从 `test_rx` 读取服务端响应：

````rust
let msg = match test_rx.next().await.unwrap() {
    Message::Text(msg) => msg,
    other => panic!("expected a text message but got {other:?}"),
};

assert_eq!(msg.as_str(), "You said: foo");
````

## 函数职责速查

- `app`：注册两个测试用 WebSocket 路由。
- `integration_testable_handler`：普通 WebSocket upgrade。
- `integration_testable_handle_socket`：真实 socket echo 逻辑。
- `unit_testable_handler`：split socket 后调用泛型处理函数。
- `unit_testable_handle_socket`：可用真实 socket 或 channel 测试的消息处理逻辑。
- `integration_test`：启动真实服务并用真实客户端连接。
- `unit_test`：用 channel 模拟 socket 读写。


## 带中文注释的手写版

完整 `src/main.rs`：

````rust
//! WebSocket 测试示例。
//!
//! Run with
//!
//! ```not_rust
//! cargo test -p example-testing-websockets
//! ```

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

// 普通写法：适合启动真实服务器做集成测试。
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

// 可单元测试写法：先 split，再把读写两半交给泛型函数。
async fn unit_testable_handler(ws: WebSocketUpgrade) -> Response {
    ws.on_upgrade(|socket| {
        let (write, read) = socket.split();
        unit_testable_handle_socket(write, read)
    })
}

// 只依赖 Sink/Stream，不依赖真实 WebSocket，所以测试时可以用 channel 模拟。
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

    // 集成测试：启动真实 Axum 服务，再用真实 WebSocket 客户端连接。
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

    // 单元测试：不用真实 socket，用 futures channel 模拟读写两端。
    #[tokio::test]
    async fn unit_test() {
        // socket_write -> test_rx：服务端写，测试读取。
        let (socket_write, mut test_rx) = futures_channel::mpsc::channel(1024);
        // test_tx -> socket_read：测试写，服务端读取。
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

## 运行和验证

运行测试：

````bash
cargo test -p example-testing-websockets
````

两个测试都应该通过：

```text
integration_test
unit_test
```

## 常见卡点

### 1. WebSocket 测试一定要启动真实服务器吗？

不一定。  
如果你要覆盖 upgrade、路由和真实协议，用集成测试。  
如果只测试消息处理逻辑，可以用泛型 Sink/Stream 单元测试。

### 2. 为什么单元测试不用 tokio channel？

源码注释说明了：这里需要实现 `Sink` 和 `Stream` 的 channel。  
`futures_channel::mpsc` 正好适合这个测试。

### 3. 为什么集成测试绑定端口 0？

让操作系统分配可用端口，避免端口冲突。

## 手写任务

1. 写一个 WebSocket echo handler。
2. 写集成测试，启动真实服务并用 `tokio_tungstenite` 连接。
3. 把 echo 逻辑改成泛型 `Sink`/`Stream`。
4. 用 `futures_channel::mpsc` 写单元测试。

## 本章真正要记住什么

WebSocket 测试有两条路：

```text
集成测试：真实服务 + 真实客户端
单元测试：泛型 Sink/Stream + channel
```

一般优先用集成测试。  
当业务逻辑复杂、启动真实服务成本高时，再把消息处理抽出来做单元测试。

## 源码对照

本章手写版对应源码：

- `examples/testing-websockets/src/main.rs`
- `examples/testing-websockets/Cargo.toml`
