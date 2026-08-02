# 第 35 章 testing-websockets:两种测法

## 本章你将做出什么

**两个功能完全一样的 echo handler**,以及各自对应的测试:

```
GET /integration-testable   普通写法,用真实客户端连上去测
GET /unit-testable          泛型写法,用 channel 冒充 socket 测
```

[第 21 章](21-testing.md)的 `oneshot` 在这里失效了——WebSocket 的收发是双向、异步、持续的,没法"发一个请求断言一个响应"。这一章讲两条可行的路,以及它们对**代码结构**的不同要求。

学完本章你将掌握:

- 为什么 `oneshot` 测不了 WebSocket
- 集成测试:起真实服务器 + `tokio-tungstenite` 客户端
- **单元测试:把 handler 写成泛型,用 channel 替代 socket**
- 为什么必须用 `futures_channel` 而不是 `tokio::sync::mpsc`
- 这两种测法对代码结构的影响,以及官方的推荐

## 前置知识

### 为什么 oneshot 测不了 WebSocket

[第 21 章](21-testing.md#router-是一个-service)讲过 `Router` 是个 `Service`,可以直接喂请求:

```rust
let response = app.oneshot(request).await.unwrap();
assert_eq!(response.status(), StatusCode::OK);
```

对 WebSocket 这么做只能验证到**握手那一步**:

```rust
// 能测到这个
assert_eq!(response.status(), StatusCode::SWITCHING_PROTOCOLS);
```

但**握手之后的收发完全测不到**。原因是 `on_upgrade` 的回调需要一个**真正升级过的连接**——[第 34 章](34-websockets.md#websocketupgrade-与-on_upgrade)讲过时序:axum 先发 `101` 响应,再在后台把升级后的连接交给回调。

`oneshot` 只走到"拿到响应"就结束了,那个后台回调根本不会被调用,更没有连接可以读写。

**所以 WebSocket 的业务逻辑只有两条测法:**

1. **起真实服务器**,用真实客户端连上去
2. **绕过 socket**,直接测那个处理逻辑的函数

本章两条路由分别演示这两种。

### 集成测试:真实客户端

思路和[第 21 章](21-testing.md#什么时候仍需真实服务器)那个 `the_real_deal` 测试一样:

```rust
let listener = TcpListener::bind(SocketAddr::from((Ipv4Addr::UNSPECIFIED, 0))).await.unwrap();
let addr = listener.local_addr().unwrap();
tokio::spawn(axum::serve(listener, app()).into_future());

let (mut socket, _response) =
    tokio_tungstenite::connect_async(format!("ws://{addr}/integration-testable")).await.unwrap();
```

**端口写 0 让系统分配**,`local_addr()` 问出实际端口——[第 21 章](21-testing.md#什么时候仍需真实服务器)讲过这个技巧,测试并行跑也不会抢端口。

**`.into_future()`** 是这里唯一的新东西。`axum::serve(...)` 返回的是一个实现了 `IntoFuture` 的类型,不是 `Future` 本身。平时写 `axum::serve(...).await` 时,`.await` 会自动调用 `IntoFuture::into_future`。但 `tokio::spawn` 要求的是 `Future`,所以得显式转换。

需要 `use std::future::IntoFuture;`。

**优点是测的东西最真实**:完整的握手、真实的帧编解码、真实的 TCP。任何中间环节的问题都能发现。

**缺点是需要能起服务器。** 如果 `app()` 依赖数据库连接、外部服务、复杂配置,测试就变重了。官方注释也提到这点:

> Generally we recommend the integration test version unless your app has a lot of setup that makes it hard to run in a test.

### 单元测试:把 socket 抽象掉

**这是本章的核心技巧。**

问题的根源是 `WebSocket` 是个具体类型,只能从真实连接得到。解法是**让处理函数不依赖它**:

```rust
async fn unit_testable_handle_socket<W, R>(mut write: W, mut read: R)
where
    W: Sink<Message> + Unpin,
    R: Stream<Item = Result<Message, axum::Error>> + Unpin,
{
    while let Some(Ok(msg)) = read.next().await {
        // ...
        write.send(...).await
    }
}
```

**函数只要求"能发消息的东西"和"能收消息的东西"**,不关心它们具体是什么。

`WebSocket` 拆开后正好满足([第 34 章](34-websockets.md#split为什么必须拆)讲过 `split()`):

```rust
ws.on_upgrade(|socket| {
    let (write, read) = socket.split();
    unit_testable_handle_socket(write, read)
})
```

而测试里可以传别的东西——**channel 也实现了 `Sink` 和 `Stream`**:

```rust
let (socket_write, mut test_rx) = futures_channel::mpsc::channel(1024);
let (mut test_tx, socket_read) = futures_channel::mpsc::channel(1024);

tokio::spawn(unit_testable_handle_socket(socket_write, socket_read));
```

**两个 channel 交叉接线**:handler 的"写"端连到测试的"读"端,测试的"写"端连到 handler 的"读"端。这样测试就能"冒充"客户端。

这本质上是[第 16 章](16-dependency-injection.md#依赖注入解决什么问题)讲的依赖注入,只是注入的对象从"数据库"换成了"通信通道"。

**`Unpin` 约束是必需的。** `read.next()` 和 `write.send()` 这些 `StreamExt`/`SinkExt` 方法要求 `Self: Unpin`(它们内部要移动 `&mut self`)。`SplitSink`/`SplitStream` 和 channel 都满足,加上这个约束就能同时兼容两者。

### 为什么必须用 futures_channel

官方注释专门提醒了这一点:

> Need to use "futures" channels rather than "tokio" channels as they implement `Sink` and `Stream`

**`tokio::sync::mpsc` 不实现 `Sink`/`Stream`**,只有自己的 `send()`/`recv()` 方法。它是 tokio 生态的原生设计,更高效,但不接 futures 的 trait 体系。

**`futures_channel::mpsc` 实现了这两个 trait**,所以能冒充 socket 的两半。

| | `tokio::sync::mpsc` | `futures_channel::mpsc` |
|---|---|---|
| 实现 `Sink`/`Stream` | **否** | **是** |
| API | `send()` / `recv()` | `send()` (来自 `SinkExt`) / `next()` |
| 性能 | 更好 | 略差 |
| 用途 | 一般的任务间通信 | 需要 `Sink`/`Stream` 抽象时 |

**平时用 tokio 的,需要接 futures trait 体系时用 futures 的。** tokio 也提供了 `tokio_stream::wrappers::ReceiverStream` 把 tokio 的 receiver 包成 `Stream`,但 sender 那边没有对应的 `Sink` 包装,所以本章场景只能用 `futures_channel`。

### 两种测法对代码结构的要求

**这是选择时的关键考虑,不只是"测试怎么写"的问题。**

| | 集成测试 | 单元测试 |
|---|---|---|
| handler 写法 | **普通写法** | 必须泛型化 + `split()` |
| 需要起服务器 | 是 | 否 |
| 需要真实客户端库 | 是 | 否 |
| 测试速度 | 毫秒级 | 微秒级 |
| 覆盖握手/编解码 | **是** | 否 |
| 对 `app()` 的依赖 | 整个应用 | **只有那个函数** |

**注意第一行:** 单元测试的代价是**改变生产代码的结构**。对比两个 handler:

```rust
// 集成测试版:直接用 socket
async fn integration_testable_handle_socket(mut socket: WebSocket) {
    while let Some(Ok(msg)) = socket.recv().await { ... }
}

// 单元测试版:必须先 split,再传两个泛型参数
async fn unit_testable_handle_socket<W, R>(mut write: W, mut read: R)
where W: Sink<Message> + Unpin, R: Stream<...> + Unpin
{ ... }
```

后者多了泛型参数和 trait bound,`on_upgrade` 里也多了一步 `split()`。**这是"可测试性"的成本**——它让代码更抽象,但也更啰嗦。

[第 16 章](16-dependency-injection.md#该选哪个)讨论过同样的取舍:类型参数会扩散、增加复杂度。**官方的建议是默认用集成测试**,只在应用启动成本太高时才为了单元测试改结构。

## 项目搭建

```bash
cargo new testing-websockets
cd testing-websockets
```

```bash
cargo add axum --features ws
cargo add futures-channel --features sink
cargo add futures-util --no-default-features --features sink,std
cargo add tokio --features full
cargo add tokio-tungstenite
```

**注意这次 `futures-channel` 和 `tokio-tungstenite` 都在 `[dependencies]` 里,不是 `dev-dependencies`。** 严格说它们只在测试用到,应该放 dev 依赖——官方示例没这么做,「代价与边界」会提。

`futures-channel` 的 `sink` feature 让 `Sender` 实现 `Sink`。

## 分步构建

### 步骤 1:集成测试版的 handler

**要做什么**:用最自然的方式写一个 echo handler。

```rust
// src/main.rs
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
```

**这是最朴素的 WebSocket handler**:收到文本就回一句 `You said: xxx`。

**注意它没有 `split()`。** 因为是严格的"收一条、回一条"模式,不需要并发收发——[第 34 章](34-websockets.md#split为什么必须拆)讲的那个限制在这里不成问题。echo 服务天然是串行的。

**`if let Message::Text(msg) = msg`** 只处理文本消息,其它类型(`Binary`、`Ping`、`Pong`、`Close`)静默忽略。注意 `Close` 也被忽略了,但 `recv()` 会在连接关闭后返回 `None`,循环自然退出。

**`ws.on_upgrade(integration_testable_handle_socket)`** 直接传函数名,不需要闭包——因为函数签名正好匹配 `FnOnce(WebSocket) -> Future`。

**`fn app() -> Router`** 这个模式[第 21 章](21-testing.md#fn-app-router-的价值兑现)兑现过价值,这一章会再用一次。

**跑起来看**:

```bash
cargo run
```

```bash
websocat ws://127.0.0.1:3000/integration-testable
```

输入几行:

```
hello
You said: hello
world
You said: world
```

**echo 正常工作。**

### 步骤 2:集成测试

**要做什么**:写测试,起真实服务器 + 真实客户端。

在 `src/main.rs` 末尾追加:

```rust
// src/main.rs

#[cfg(test)]
mod tests {
    use super::*;
    use futures_util::{SinkExt, StreamExt};
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
}
```

**`SocketAddr::from((Ipv4Addr::UNSPECIFIED, 0))`** 等价于 `0.0.0.0:0` —— 前置知识讲过端口 0 的作用。

**`tokio::spawn(axum::serve(listener, app()).into_future())`** —— `into_future()` 是前置知识讲的那个细节。忘了它会报类型错误:`tokio::spawn` 要 `Future`,而 `Serve` 只是 `IntoFuture`。

**注意 spawn 之后没有任何等待就直接 `connect_async`。** [第 21 章](21-testing.md#什么时候仍需真实服务器)解释过为什么能工作:`bind` 已经完成,操作系统开始接受连接并排队,即使 `serve` 还没 accept,客户端的连接也不会被拒绝。

**`socket.next().await.unwrap().unwrap()`** 两层 `unwrap`:第一层是 `Option`(流可能结束),第二层是 `Result`(接收可能出错)。

**`match ... other => panic!(...)`** 这个写法比 `if let` 好:**收到非文本消息时会明确报错**,而不是静默跳过导致测试卡在等待下一条消息上。

**跑起来看**:

```bash
cargo test
```

```
running 1 test
test tests::integration_test ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; finished in 0.01s
```

**验证测试确实在测真东西。** 临时把 handler 的回复改一下:

```rust
                .send(Message::Text(format!("You wrote: {msg}").into()))
```

```bash
cargo test
```

```
test tests::integration_test ... FAILED

assertion `left == right` failed
  left: "You wrote: foo"
 right: "You said: foo"
```

改回去。

**这个测试覆盖了完整链路**:HTTP 握手、协议升级、帧编解码、handler 逻辑。任何一环出问题都会被发现。

### 步骤 3:单元测试版的 handler

**要做什么**:把同样的逻辑写成泛型版本,让它能接受任何 `Sink`/`Stream`。

```rust
// src/main.rs
use axum::{
    extract::{
        ws::{Message, WebSocket},
        WebSocketUpgrade,
    },
    response::Response,
    routing::get,
    Router,
};
use futures_util::{Sink, SinkExt, Stream, StreamExt};       // 新增

// ...(main 不变)

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
        .route("/unit-testable", get(unit_testable_handler))       // 新增
}

// ...(integration_testable_* 不变)

// 新增:The unit testable version requires some changes.
//
// By splitting the socket into an `impl Sink` and `impl Stream` we can test without providing a
// real socket and instead using channels, which also implement `Sink` and `Stream`.
async fn unit_testable_handler(ws: WebSocketUpgrade) -> Response {
    ws.on_upgrade(|socket| {
        let (write, read) = socket.split();
        unit_testable_handle_socket(write, read)
    })
}

// 新增:The implementation is largely the same as `integration_testable_handle_socket` except we call
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
```

**把两个版本的函数体并排看,几乎一模一样:**

```rust
// 集成测试版
while let Some(Ok(msg)) = socket.recv().await {
    ...
    socket.send(...).await
}

// 单元测试版
while let Some(Ok(msg)) = read.next().await {
    ...
    write.send(...).await
}
```

**差别只在方法来源**:`socket.recv()`/`socket.send()` 是 `WebSocket` 自己的方法,`read.next()`/`write.send()` 来自 `StreamExt`/`SinkExt`。官方注释也说了 `largely the same except we call methods from SinkExt and StreamExt`。

**`R: Stream<Item = Result<Message, axum::Error>>`** 这个 `Item` 类型是关键——它必须和 `SplitStream<WebSocket>` 产出的类型一致,所以是 `Result<Message, axum::Error>`。测试里的 channel 也必须传这个类型。

**`Unpin` 约束**前置知识讲过为什么必需。

**`on_upgrade` 里用闭包而非直接传函数名**,因为要先 `split()` 再调用。

**跑起来看**:

```bash
cargo run
```

两条路由行为应该一致:

```bash
websocat ws://127.0.0.1:3000/unit-testable
```

```
hello
You said: hello
```

**功能完全相同**,只是内部结构不同。

### 步骤 4:单元测试

**要做什么**:用两个交叉接线的 channel 冒充 socket。

```rust
// src/main.rs 的 tests 模块内

    // ...(integration_test 不变)

    // 新增:We can unit test the other handler by creating channels to read and write from.
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
```

**两个 channel 的接线值得画清楚:**

```
channel 1:  socket_write ──────► test_rx      (handler 写,测试读)
channel 2:  test_tx ──────────► socket_read   (测试写,handler 读)
```

- **`socket_write`** 传给 handler 当 `W`(它是 `Sender`,实现 `Sink`)
- **`test_rx`** 留给测试当接收端,读 handler 的输出
- **`test_tx`** 留给测试,模拟客户端发消息
- **`socket_read`** 传给 handler 当 `R`(它是 `Receiver`,实现 `Stream`)

**注意 `test_tx.send(Ok(Message::Text(...)))` 里那个 `Ok`。** 因为 `R` 的 `Item` 是 `Result<Message, axum::Error>`,channel 传的就是这个类型。**这里可以顺便测试错误路径**——发一个 `Err` 看 handler 怎么处理。

**`channel(1024)`** 是缓冲大小。给足够大避免测试因为缓冲满而阻塞。

**`tokio::spawn(...)` 之后不需要等待**,因为 channel 有缓冲,测试可以直接往里发。

**跑起来看**:

```bash
cargo test
```

```
running 2 tests
test tests::unit_test ... ok
test tests::integration_test ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; finished in 0.00s
```

**`finished in 0.00s`** —— 两个测试加起来不到 10 毫秒。

**看单个测试的耗时对比**(需要 nightly 或者手工计时,这里用 `--nocapture` 加日志的方式感受):

单元测试完全在内存里跑,没有 TCP、没有握手、没有帧编解码。集成测试虽然也很快,但多了这些开销。**在有成百上千个 WebSocket 测试时,这个差距会显现。**

**验证单元测试的隔离性。** 临时把泛型 handler 的逻辑改错:

```rust
            if write
                .send(Message::Text(format!("Echo: {msg}").into()))
                .await
```

```bash
cargo test
```

```
test tests::integration_test ... ok        ← 不受影响
test tests::unit_test ... FAILED
```

**只有 `unit_test` 失败** —— 因为它测的是 `unit_testable_handle_socket`,而集成测试测的是另一个函数。**两条路由的逻辑是独立的**,这也说明单元测试的粒度更细。

改回去。

**试试测错误路径**(这是集成测试很难做到的)。在 `unit_test` 里发一个 `Err`:

```rust
        test_tx
            .send(Err(axum::Error::new(std::io::Error::other("boom"))))
            .await
            .unwrap();
```

handler 的 `while let Some(Ok(msg))` 遇到 `Err` 会跳出循环,所以后续的 `test_rx.next()` 会返回 `None`。**用真实客户端很难精确制造"接收出错"这种情况**,而 channel 可以。

## 完整代码

`src/main.rs`:

```rust
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
```

## 代价与边界

**单元测试要求改造生产代码。** 前置知识那张表的第一行——泛型化 + 强制 `split()`。这是最实际的代价:为了可测试性,handler 变得更抽象、签名更长。

**官方推荐默认用集成测试。** 注释写得很明确:

> Generally we recommend the integration test version unless your app has a lot of setup that makes it hard to run in a test.

**判断标准是 `app()` 的启动成本**:如果它需要数据库连接、外部服务、复杂配置,起真实服务器就变重了,那时单元测试的价值才体现出来。

**单元测试覆盖不到握手和协议层。** 它只测那个处理函数。以下问题它发现不了:

- 握手头部处理错误
- `WebSocketUpgrade` 的配置(消息大小限制等)
- 帧编解码
- `Ping`/`Pong` 的自动应答
- 路由配置错误

**所以两种测法不是替代关系,而是互补**:单元测试覆盖业务逻辑分支,集成测试覆盖"整条链路能通"。

**测试依赖放在了 `[dependencies]` 而非 `[dev-dependencies]`。** `futures-channel` 和 `tokio-tungstenite` 只在测试用到,放主依赖会让**生产构建也编译它们**,增加编译时间和二进制体积。正确做法:

```toml
[dev-dependencies]
futures-channel = { version = "0.3", features = ["sink"] }
tokio-tungstenite = "0.29"
```

[第 21 章](21-testing.md#项目搭建)那个示例就把 `tower` 正确放进了 dev 依赖。

**echo handler 太简单,掩盖了真实的测试难点。** 真实的 WebSocket 逻辑通常有状态(连接列表、房间、订阅关系),测试时需要构造这些状态。本章的 handler 是无状态的,所以两种测法都很轻松。[第 37 章](37-chat.md)那个聊天室会引入共享状态,测试难度完全不同。

**没有测超时和并发。** WebSocket 的常见 bug 是竞态:两个消息几乎同时到达、发送方比接收方快导致背压。这些需要专门设计的测试(注入延迟、控制时序),两种基础测法都覆盖不到。

**集成测试没有清理服务器任务。** `tokio::spawn` 起的服务器在测试结束时随运行时销毁,这没问题([第 21 章](21-testing.md#什么时候仍需真实服务器)讲过)。但如果测试里有需要显式清理的资源(临时文件、数据库记录),要自己处理。

## 动手练习

**1. 把测试依赖挪到 `[dev-dependencies]`。**

提示:`futures-channel` 和 `tokio-tungstenite` 只在测试里用。挪过去之后 `cargo build` 仍应成功,`cargo test` 也仍应通过。用 `cargo tree --edges normal` 确认它们不在正常依赖树里。**这是「代价与边界」里最直接的一处改进。**

**2. 测试错误路径。**

提示:在 `unit_test` 里发一个 `Err(axum::Error::new(...))`,断言 handler 退出了循环(后续 `test_rx.next()` 返回 `None`)。**这是单元测试独有的能力** —— 用真实客户端很难精确制造"接收出错"。做完想想:集成测试要怎么测这个?(很难,需要构造非法帧或者中途断开连接。)

**3. 给两个 handler 加上处理 `Close` 消息,并各写一个测试。**

提示:在 `match` 里加 `Message::Close(_) => break`。集成测试版发 `tungstenite::Message::Close(None)`,断言 socket 随后关闭;单元测试版发 `Ok(Message::Close(None))`,断言 `test_rx` 返回 `None`。做完对比两个测试的代码量和可读性。

## 小结与下一章预告

**`oneshot` 测不了 WebSocket** —— 它只走到"拿到 `101` 响应"就结束,而 `on_upgrade` 的回调需要一个真正升级过的连接。业务逻辑只有两条测法。

**集成测试**起真实服务器(端口 0 让系统分配)+ 真实客户端。优点是覆盖完整链路:握手、编解码、路由;缺点是需要能起服务器。注意 `tokio::spawn(axum::serve(...).into_future())` 那个 `into_future()`——`Serve` 只是 `IntoFuture`,不是 `Future`。

**单元测试**把处理函数写成泛型,只要求 `Sink<Message>` 和 `Stream<Item = Result<Message, axum::Error>>`,测试时用**两个交叉接线的 channel** 冒充 socket。必须用 `futures_channel::mpsc` 而非 `tokio::sync::mpsc`——**只有前者实现 `Sink`/`Stream`**。

**代价是改造生产代码**:泛型参数、trait bound、强制 `split()`。这和[第 16 章](16-dependency-injection.md#该选哪个)讨论的取舍一样——可测试性有成本。**官方建议默认用集成测试**,只在应用启动成本高时才为单元测试改结构。

两者是**互补而非替代**:单元测试能精确构造错误路径(发一个 `Err` 给 handler),集成测试能覆盖协议层。

本章的 handler 是无状态的 echo,掩盖了真实的测试难点——有共享状态的 WebSocket 逻辑难测得多。

[第 36 章 websockets-http2](36-websockets-http2.md) 换个角度:WebSocket over HTTP/2。HTTP/2 的握手机制和 HTTP/1.1 完全不同(用 `CONNECT` 而非 `Upgrade`),axum 需要显式开启支持。你会看到为什么这个组合不常见,以及它解决了什么问题。
