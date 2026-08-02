# 第 44 章 serve-with-hyper:把 `axum::serve` 拆开

## 本章你将做出什么

两个功能完全普通的服务,但**都不用 `axum::serve`**:

```
127.0.0.1:3000    Hello!
127.0.0.1:3001    Hello 127.0.0.1:54321   ← 带上了客户端的地址
```

它们跑在同一个进程里,用 `tokio::join!` 并行。代码里没有一行 `axum::serve`——accept 循环、协议协商、连接升级、客户端地址注入,全是手写的。

[第 42 章](42-http-proxy.md#步骤-1不用-axumserve自己写-accept-循环)已经手写过一次 accept 循环,那次是为了拦截 `CONNECT`。**这一章是为了看清 `axum::serve` 本身。**

学完本章你将掌握:

- **`axum::serve` 到底封装了哪几件事**,以及什么时候你真的需要绕开它
- `hyper_util::server::conn::auto::Builder`:**同一个端口自动区分 HTTP/1.1 和 HTTP/2**
- `serve_connection_with_upgrades` 为什么是默认该用的那个
- **`MakeService`:比 `Service` 高一层的抽象**,以及为什么 `ConnectInfo` 非它不可
- `ConnectInfo<SocketAddr>`:客户端地址是**怎么一路传进 handler** 的
- `unwrap_infallible`:又一种消灭 `Infallible` 的写法

## 前置知识

### `axum::serve` 的五件事

前面 43 章都是这一行:

```rust
axum::serve(listener, app).await.unwrap();
```

它实际上在做五件事:

| # | 做什么 | 本章对应的代码 |
|---|---|---|
| ① | 循环 accept TCP 连接 | `loop { listener.accept().await }` |
| ② | 每条连接 spawn 一个任务 | `tokio::spawn(...)` |
| ③ | 把 tokio socket 转成 hyper 的 IO | `TokioIo::new(socket)` |
| ④ | 把 tower `Service` 转成 hyper `Service` | `hyper::service::service_fn(...)` |
| ⑤ | 在这条连接上跑 HTTP(自动协商版本、支持升级) | `auto::Builder::new(...).serve_connection_with_upgrades(...)` |

**这五步没有任何魔法**,加起来不到 30 行。理解它们之后,`axum::serve` 从一个黑盒变成一个便利函数。

**那什么时候需要自己写?** 老实说,**绝大多数情况下不需要**。真正的理由只有这几类:

- **在路由之前拦截请求** —— [第 42 章](42-http-proxy.md)的 `CONNECT` 分流
- **需要 hyper 的连接级配置** —— `preserve_header_case`、`max_header_list_size`、HTTP/2 的窗口大小和并发流上限等,`axum::serve` 没有全部暴露
- **不是标准的 `TcpListener`** —— Unix 域套接字([第 53 章](53-unix-domain-socket.md))、自定义 TLS 握手([第 50 章](50-low-level-rustls.md)起的几章)
- **精细控制连接生命周期** —— 每连接的超时、限流、优雅停机的自定义逻辑

除此之外,**用 `axum::serve` 就好**。本章的价值是「看懂」,不是「以后都这么写」。

### `auto::Builder`:一个端口,两种协议

[第 42 章](42-http-proxy.md)用的是 `hyper::server::conn::http1::Builder`,写死了 HTTP/1.1。本章换成:

```rust
hyper_util::server::conn::auto::Builder::new(TokioExecutor::new())
```

**`auto` 的意思是自动识别版本。** 它的判断方法很朴素:**偷看连接的头几个字节**。

HTTP/2 的连接必须以一个固定的 24 字节前言(preface)开头:

```
PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n
```

`auto::Builder` 读到这串就走 HTTP/2,否则当 HTTP/1.1 处理。

**这就是 `axum::serve` 的默认行为**,所以你之前写的服务其实一直都同时支持两种协议。

> **浏览器是个例外。** 浏览器只在 HTTPS + ALPN 协商成功时才用 HTTP/2,明文端口一律走 HTTP/1.1。所以 `auto` 的 HTTP/2 分支在浏览器场景下基本用不上,主要服务于**明文 HTTP/2(h2c)** —— gRPC、服务间调用、以及带 `--http2-prior-knowledge` 的 curl。
>
> [第 36 章](36-websockets-http2.md)讲过 HTTP/2 上的 WebSocket,那一章就必须让客户端明确用 HTTP/2。

**`TokioExecutor` 是 HTTP/2 必需的。** HTTP/2 在一条连接上跑多个并发流,hyper 需要 spawn 任务来管理它们。[第 43 章](43-reverse-proxy.md#httpconnector-与-tokioexecutor)在客户端一侧见过同一个东西。

### `serve_connection_with_upgrades`

[第 42 章](42-http-proxy.md#步骤-5故意改坏理解-with_upgrades)吃过亏:`serve_connection` 不处理连接升级,`CONNECT` 隧道会**静默挂起**。

`auto::Builder` 上对应的方法是:

```rust
.serve_connection_with_upgrades(socket, hyper_service)
```

**只要你的服务有 WebSocket([第 34 章](34-websockets.md))、有 `CONNECT` 隧道、或者将来可能有,就用这个。** 没有升级需求时用 `serve_connection` 能省一点点开销,但省的量微不足道,而漏掉它的代价是一个查半天的静默 bug。

`axum::serve` 内部用的就是 `serve_connection_with_upgrades`。

### `Service` 与 `MakeService`:多出来的那一层

这是本章最重要的概念,也是 `ConnectInfo` 能工作的全部秘密。

[第 17 章](17-tracing-aka-logging.md)讲过 tower 的 `Service`:

```
Service<Request>:  Request  →  Response
```

`Router` 就是一个 `Service<Request>`。

但一个服务器要处理的不只是请求,还有**连接**。于是有了高一层的抽象:

```
MakeService<Target>:  Target  →  Service<Request>
```

**「给我一个连接的信息,还我一个专门处理这条连接的 Service」。**

它其实不是一个新 trait,就是 `Service` 本身——只不过输入是连接信息、输出是另一个 `Service`:

```rust
// MakeService 就是这个
impl Service<SocketAddr> for MyMakeService {
    type Response = Router;   // ← 响应是一个 Service!
    // ...
}
```

**为什么需要这一层?** 因为**有些信息属于连接,不属于请求**:

| 信息 | 属于 |
|---|---|
| 请求路径、方法、头、body | 请求 |
| **客户端 IP 和端口** | **连接** |
| TLS 证书、协商的协议 | 连接 |
| 本地监听地址 | 连接 |

HTTP 请求里**根本没有客户端 IP 这个字段**。它来自 TCP 层,只有 `accept()` 的返回值里有。`MakeService` 就是把这个信息**注入**到该连接的 Service 里的地方。

axum 提供两个方法:

```rust
app.into_make_service()                                  // 忽略连接信息
app.into_make_service_with_connect_info::<SocketAddr>()  // 把连接信息塞进请求扩展
```

后者产出的 MakeService 在被 `call(remote_addr)` 时,会 clone 一个 `Router` 出来,并让它**给每个进来的请求的 extensions 里插一个 `ConnectInfo(remote_addr)`**。

然后 handler 里的 `ConnectInfo<SocketAddr>` 提取器就是从 extensions 里把它读出来。[第 09 章](09-consume-body-in-extractor-or-middleware.md)讲过 extensions 是请求上的"顺风车",这是它最经典的用法之一。

**串起来是这样:**

```
accept() 拿到 remote_addr
    ↓
make_service.call(remote_addr)  →  一个绑好了地址的 Router
    ↓
每个请求进来,extensions 里被插入 ConnectInfo(remote_addr)
    ↓
handler 的 ConnectInfo<SocketAddr> 提取器读出来
```

**这也解释了一个常见困惑**:为什么用 `ConnectInfo` 时 `axum::serve` 的第二个参数要写成 `app.into_make_service_with_connect_info::<SocketAddr>()` 而不是 `app`?因为普通的 `app` 那条路径压根没有地方拿到 `remote_addr`。

### `unwrap_infallible`

[第 42 章](42-http-proxy.md#match-err消灭-infallible)用 `match err {}` 消灭过 `Infallible`,本章是另一种形式:

```rust
fn unwrap_infallible<T>(result: Result<T, Infallible>) -> T {
    match result {
        Ok(value) => value,
        Err(err) => match err {},
    }
}
```

和 `.unwrap()` 的区别是:**它不可能 panic,编译器能证明这一点**。`Err` 分支里的 `match err {}` 告诉编译器"这个分支不可达",于是整个函数是全函数(total function)。

**为什么不直接 `.unwrap()`?** 因为 `unwrap` 需要 `E: Debug` 而且会留下一个理论上的 panic 点。用 `Infallible` 的模式匹配则是**类型层面的证明**——代码里一个 panic 分支都没有。这是 Rust 里表达"这里绝对不会出错"的惯用法。

### `tokio::join!` vs `tokio::spawn`

[第 43 章](43-reverse-proxy.md#tokiospawn一个进程跑两个服务)用 `tokio::spawn` 在后台跑第二个服务,本章用:

```rust
tokio::join!(serve_plain(), serve_with_connect_info());
```

| | `tokio::spawn` | `tokio::join!` |
|---|---|---|
| 并发方式 | 独立任务,可能跑在别的线程 | **同一个任务里并发轮询** |
| 返回 | 立刻返回 `JoinHandle` | **等全部完成** |
| 需要 `'static` | 是 | 否 |
| 适合 | 后台配角 | **同等地位的多个主流程** |

本章两个服务地位对等,`join!` 更贴切。**注意 `join!` 不是并行执行,是并发**——两个 future 在同一个任务里被交替轮询。对 IO 密集的服务器来说没区别,因为真正的并行发生在它们各自 spawn 出去的连接任务里。

## 项目搭建

```bash
cargo new serve-with-hyper
cd serve-with-hyper
```

```bash
cargo add axum
cargo add hyper
cargo add hyper-util --features tokio,server-auto,http1
cargo add tokio --features full
cargo add tower --features util
```

`hyper` 不用开任何 feature —— 需要的能力(`http1`、`http2`、`server`)会由 `hyper-util` 的 `server-auto` 连带打开。

`tower` 的 `util` 提供 `ServiceExt::oneshot`。

## 分步构建

### 步骤 1:手写第一个服务

**要做什么**:把 `axum::serve` 的五件事一件件写出来。

```rust
// src/main.rs
use axum::{extract::Request, routing::get, Router};
use hyper::body::Incoming;
use hyper_util::rt::{TokioExecutor, TokioIo};
use hyper_util::server;
use tokio::net::TcpListener;
use tower::Service;

#[tokio::main]
async fn main() {
    serve_plain().await;
}

async fn serve_plain() {
    // Create a regular axum app.
    let app = Router::new().route("/", get(|| async { "Hello!" }));

    // Create a `TcpListener` using tokio.
    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

    // Continuously accept new connections.
    loop {
        // In this example we discard the remote address. See `fn serve_with_connect_info` for how
        // to expose that.
        let (socket, _remote_addr) = listener.accept().await.unwrap();

        // We don't need to call `poll_ready` because `Router` is always ready.
        let tower_service = app.clone();

        // Spawn a task to handle the connection. That way we can handle multiple connections
        // concurrently.
        tokio::spawn(async move {
            // Hyper has its own `AsyncRead` and `AsyncWrite` traits and doesn't use tokio.
            // `TokioIo` converts between them.
            let socket = TokioIo::new(socket);

            // Hyper also has its own `Service` trait and doesn't use tower. We can use
            // `hyper::service::service_fn` to create a hyper `Service` that calls our app through
            // `tower::Service::call`.
            let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
                // We have to clone `tower_service` because hyper's `Service` uses `&self` whereas
                // tower's `Service` requires `&mut self`.
                //
                // We don't need to call `poll_ready` since `Router` is always ready.
                tower_service.clone().call(request)
            });

            // `server::conn::auto::Builder` supports both http1 and http2.
            //
            // `TokioExecutor` tells hyper to use `tokio::spawn` to spawn tasks.
            if let Err(err) = server::conn::auto::Builder::new(TokioExecutor::new())
                // `serve_connection_with_upgrades` is required for websockets. If you don't need
                // that you can use `serve_connection` instead.
                .serve_connection_with_upgrades(socket, hyper_service)
                .await
            {
                eprintln!("failed to serve connection: {err:#}");
            }
        });
    }
}
```

**官方注释写得很全,这里补充几个注释没说透的点:**

**`app.clone()` 在循环里,`tower_service.clone()` 在 service_fn 里 —— 克隆了两次。**

- 外层:**每条连接**一个 `Router` 克隆
- 内层:**每个请求**一个 `Router` 克隆

为什么内层还要克隆?因为 hyper 的 `Service::call` 只拿 `&self`,而 tower 的要 `&mut self`([第 42 章](42-http-proxy.md#hyper-的-service--tower-的-service)那张表)。**两次克隆都是 `Arc` 引用计数,几乎零成本。**

**"We don't need to call `poll_ready`" 是什么意思?**

tower 的 `Service` 契约要求调用方先 `poll_ready()` 确认服务就绪,再 `call()`。这是背压(backpressure)机制——比如一个有并发上限的服务会在满载时返回 `Pending`。

`Router` **永远就绪**(它的 `poll_ready` 直接返回 `Ready(Ok(()))`),所以可以跳过。**但如果你在 `Router` 外面套了带背压的 tower 层,跳过 `poll_ready` 就是违约。** `ServiceExt::oneshot` 会帮你做这件事——步骤 4 会用到。

**`{err:#}` 里的 `#`** 是 `Display` 的"替代形式"。对 hyper 的错误类型,它会打印出完整的错误链而不只是最外层。

**`0.0.0.0` 而不是 `127.0.0.1`** —— 前面 43 章都用的 `127.0.0.1`(只监听本机),这里是 `0.0.0.0`(**监听所有网卡**)。「代价与边界」会说这个区别为什么重要。

**验证**:

```bash
cargo run
```

```bash
curl -i http://127.0.0.1:3000/
```

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 6

Hello!
```

**和用 `axum::serve` 跑出来的完全一样。** 这正是本步骤想说明的:我们只是把封装拆开了,行为没变。

现在验证 `auto` 的价值 —— **强制用 HTTP/2**:

```bash
curl -i --http2-prior-knowledge http://127.0.0.1:3000/
```

```
HTTP/2 200
content-type: text/plain; charset=utf-8
content-length: 6

Hello!
```

**`HTTP/2 200`** —— 同一个端口,同一份代码,协议自动切换了。`--http2-prior-knowledge` 的意思是"我知道对面支持 HTTP/2,直接发前言,不要试探"。

### 步骤 2:故意改坏,看看 `auto` 买到了什么

**要做什么**:把 `auto::Builder` 换成 HTTP/1.1 专用的 builder。

```rust
use hyper::server::conn::http1;
```

```rust
            if let Err(err) = http1::Builder::new()
                .serve_connection(socket, hyper_service)
                .with_upgrades()
                .await
```

重新运行:

```bash
curl -i http://127.0.0.1:3000/                          # 还是正常的
curl -i --http2-prior-knowledge http://127.0.0.1:3000/  # 失败
```

第二条会报类似:

```
curl: (16) Error in the HTTP2 framing layer
```

服务端日志:

```
failed to serve connection: invalid HTTP method parsed
```

**这个错误信息很有意思**:HTTP/2 的前言以 `PRI * HTTP/2.0` 开头,HTTP/1.1 的解析器把 `PRI` 当成了一个 HTTP 方法名,发现不认识就报错了。**协议协商失败的现场就是这样。**

**改回 `auto::Builder`。**

顺带注意:`http1::Builder` 的升级方法是 `.serve_connection(...).with_upgrades()`(两步),而 `auto::Builder` 是 `.serve_connection_with_upgrades(...)`(一步)。**两个 API 长得不一样,别记混。**

### 步骤 3:先看看 `ConnectInfo` 直接用会怎样

**要做什么**:在动手实现之前,先亲眼确认"只加提取器是不够的"。这一步是为了让下一步的 `MakeService` 有存在的理由。

临时把路由改成:

```rust
    let app = Router::new().route(
        "/",
        get(|ConnectInfo(addr): ConnectInfo<std::net::SocketAddr>| async move {
            format!("Hello {addr}")
        }),
    );
```

(需要 `use axum::extract::ConnectInfo;`)

**现象**:能编译、能运行,但请求会得到:

```
HTTP/1.1 500 Internal Server Error

Missing request extension: Extension of type `axum::extract::connect_info::ConnectInfo<core::net::socket_addr::SocketAddr>` was not found. Perhaps you forgot to add it? See `axum::Extension`.
```

**错误信息说得很直白:请求扩展里没有这个东西。**

前置知识讲过原因:`ConnectInfo` 提取器只负责**从 extensions 里读**,**放进去是另一个人的活**——那个人就是 `into_make_service_with_connect_info` 产出的 MakeService,而步骤 1 的代码里根本没有它。

**这一步的意义在于:很多人第一次用 `ConnectInfo` 就撞上这个 500,然后不明白为什么。现在你明白了——缺的不是提取器,是上游那一层。**

把路由改回 `get(|| async { "Hello!" })`,下一步在**另一个函数**里正确地实现它。

### 步骤 4:用 MakeService 注入客户端地址

**要做什么**:写第二个服务 `serve_with_connect_info`,监听 3001。

```rust
use axum::extract::ConnectInfo;
use std::convert::Infallible;
use std::net::SocketAddr;
use tower::{Service, ServiceExt};
```

```rust
// Similar setup to `serve_plain` but captures the remote address and exposes it through the
// `ConnectInfo` extractor
async fn serve_with_connect_info() {
    let app = Router::new().route(
        "/",
        get(
            |ConnectInfo(remote_addr): ConnectInfo<SocketAddr>| async move {
                format!("Hello {remote_addr}")
            },
        ),
    );

    let mut make_service = app.into_make_service_with_connect_info::<SocketAddr>();

    let listener = TcpListener::bind("0.0.0.0:3001").await.unwrap();

    loop {
        let (socket, remote_addr) = listener.accept().await.unwrap();

        // We don't need to call `poll_ready` because `IntoMakeServiceWithConnectInfo` is always
        // ready.
        let tower_service = unwrap_infallible(make_service.call(remote_addr).await);

        tokio::spawn(async move {
            let socket = TokioIo::new(socket);

            let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
                tower_service.clone().oneshot(request)
            });

            if let Err(err) = server::conn::auto::Builder::new(TokioExecutor::new())
                .serve_connection_with_upgrades(socket, hyper_service)
                .await
            {
                eprintln!("failed to serve connection: {err:#}");
            }
        });
    }
}

fn unwrap_infallible<T>(result: Result<T, Infallible>) -> T {
    match result {
        Ok(value) => value,
        Err(err) => match err {},
    }
}
```

**和 `serve_plain` 就三处不同**,但每一处都关键:

**① `let mut make_service = app.into_make_service_with_connect_info::<SocketAddr>();`**

在循环**外面**建一次。注意 `mut` —— 因为下面要 `call(&mut self)`。

**② `let (socket, remote_addr) = listener.accept().await.unwrap();`**

这次**不丢掉** `remote_addr` 了。步骤 1 里那个 `_remote_addr` 就是本章的伏笔。

**③ `let tower_service = unwrap_infallible(make_service.call(remote_addr).await);`**

**这是全章的关键一行。** `make_service.call(remote_addr)` 的语义是:

> "这条连接来自 `remote_addr`,给我一个处理它的 Service。"

返回的 `tower_service` 是一个**绑好了地址的 `Router`**,它给每个经过它的请求插一个 `ConnectInfo(remote_addr)` 扩展。

`.call()` 返回的是 `Result<_, Infallible>`(创建过程不可能失败),所以用 `unwrap_infallible` 剥掉。

**另外还有一处小差别:`.oneshot(request)` 而不是 `.call(request)`。**

`oneshot` = `poll_ready` + `call` 打包。前置知识里说过 `Router` 永远就绪所以 `call` 也行;这里用 `oneshot` 是**更规范的写法**——如果将来在 Router 外面套了带背压的层,`oneshot` 会自动处理,`call` 则会违约。

> **更好的写法**:两个函数里 `serve_plain` 用 `.call()`、`serve_with_connect_info` 用 `.oneshot()`,**同一份代码里两种风格并存**,官方示例应该是想展示两种都可以。实际项目里**统一用 `.oneshot()`** 更稳妥:它在 `Router` 上和 `.call()` 等价(零额外成本),但在套了 `tower` 中间件之后是唯一正确的做法。

### 步骤 5:两个服务一起跑

**要做什么**:用 `tokio::join!` 同时跑起来。

```rust
#[tokio::main]
async fn main() {
    tokio::join!(serve_plain(), serve_with_connect_info());
}
```

**验证**:

```bash
cargo run
```

```bash
curl http://127.0.0.1:3000/
```

```
Hello!
```

```bash
curl http://127.0.0.1:3001/
```

```
Hello 127.0.0.1:54832
```

**那个端口号每次都不一样** —— 它是客户端 TCP 连接的**源端口**,由操作系统随机分配。多跑几次:

```bash
for i in 1 2 3; do curl -s http://127.0.0.1:3001/; echo; done
```

```
Hello 127.0.0.1:54841
Hello 127.0.0.1:54842
Hello 127.0.0.1:54843
```

**每次一个新端口,说明每次都是一条新连接。**

反过来,用 keep-alive 在**同一条连接**上发多个请求:

```bash
curl -s http://127.0.0.1:3001/ http://127.0.0.1:3001/
```

```
Hello 127.0.0.1:54850Hello 127.0.0.1:54850
```

**两个请求的端口相同** —— 因为 curl 复用了连接,`accept` 只发生了一次,`make_service.call()` 也只调用了一次。**这个实验直接印证了「连接信息属于连接,不属于请求」。**

### 步骤 6:和 `axum::serve` 对照

**要做什么**:把手写的循环换回 `axum::serve`,确认行为一致。这一步不写进最终代码,但值得跑一遍。

`serve_plain` 的等价写法:

```rust
async fn serve_plain() {
    let app = Router::new().route("/", get(|| async { "Hello!" }));
    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

`serve_with_connect_info` 的等价写法:

```rust
async fn serve_with_connect_info() {
    let app = Router::new().route(
        "/",
        get(|ConnectInfo(remote_addr): ConnectInfo<SocketAddr>| async move {
            format!("Hello {remote_addr}")
        }),
    );
    let listener = TcpListener::bind("0.0.0.0:3001").await.unwrap();
    axum::serve(
        listener,
        app.into_make_service_with_connect_info::<SocketAddr>(),
    )
    .await
    .unwrap();
}
```

**跑一遍,两个端口的行为和手写版一模一样**,包括 `--http2-prior-knowledge`。

**注意第二个:`axum::serve` 的第二个参数就是那个 MakeService。** 这也解释了为什么 `axum::serve` 能同时接受 `app` 和 `app.into_make_service_with_connect_info::<SocketAddr>()` —— 它接受的是"能为一条连接产出 Service 的东西",`Router` 自己也满足(通过一个忽略连接信息的 blanket 实现)。

**看完这个对照再回头看步骤 1 和 4 的三十行,你就知道那三十行值不值得写了。** 大多数时候不值得。

## 完整代码

`src/main.rs`:

```rust
use std::convert::Infallible;
use std::net::SocketAddr;

use axum::extract::ConnectInfo;
use axum::{extract::Request, routing::get, Router};
use hyper::body::Incoming;
use hyper_util::rt::{TokioExecutor, TokioIo};
use hyper_util::server;
use tokio::net::TcpListener;
use tower::{Service, ServiceExt};

#[tokio::main]
async fn main() {
    tokio::join!(serve_plain(), serve_with_connect_info());
}

async fn serve_plain() {
    // Create a regular axum app.
    let app = Router::new().route("/", get(|| async { "Hello!" }));

    // Create a `TcpListener` using tokio.
    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

    // Continuously accept new connections.
    loop {
        // In this example we discard the remote address. See `fn serve_with_connect_info` for how
        // to expose that.
        let (socket, _remote_addr) = listener.accept().await.unwrap();

        // We don't need to call `poll_ready` because `Router` is always ready.
        let tower_service = app.clone();

        // Spawn a task to handle the connection. That way we can handle multiple connections
        // concurrently.
        tokio::spawn(async move {
            // Hyper has its own `AsyncRead` and `AsyncWrite` traits and doesn't use tokio.
            // `TokioIo` converts between them.
            let socket = TokioIo::new(socket);

            // Hyper also has its own `Service` trait and doesn't use tower. We can use
            // `hyper::service::service_fn` to create a hyper `Service` that calls our app through
            // `tower::Service::call`.
            let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
                // We have to clone `tower_service` because hyper's `Service` uses `&self` whereas
                // tower's `Service` requires `&mut self`.
                //
                // We don't need to call `poll_ready` since `Router` is always ready.
                tower_service.clone().call(request)
            });

            // `server::conn::auto::Builder` supports both http1 and http2.
            //
            // `TokioExecutor` tells hyper to use `tokio::spawn` to spawn tasks.
            if let Err(err) = server::conn::auto::Builder::new(TokioExecutor::new())
                // `serve_connection_with_upgrades` is required for websockets. If you don't need
                // that you can use `serve_connection` instead.
                .serve_connection_with_upgrades(socket, hyper_service)
                .await
            {
                eprintln!("failed to serve connection: {err:#}");
            }
        });
    }
}

// Similar setup to `serve_plain` but captures the remote address and exposes it through the
// `ConnectInfo` extractor
async fn serve_with_connect_info() {
    let app = Router::new().route(
        "/",
        get(
            |ConnectInfo(remote_addr): ConnectInfo<SocketAddr>| async move {
                format!("Hello {remote_addr}")
            },
        ),
    );

    let mut make_service = app.into_make_service_with_connect_info::<SocketAddr>();

    let listener = TcpListener::bind("0.0.0.0:3001").await.unwrap();

    loop {
        let (socket, remote_addr) = listener.accept().await.unwrap();

        // We don't need to call `poll_ready` because `IntoMakeServiceWithConnectInfo` is always
        // ready.
        let tower_service = unwrap_infallible(make_service.call(remote_addr).await);

        tokio::spawn(async move {
            let socket = TokioIo::new(socket);

            let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
                tower_service.clone().oneshot(request)
            });

            if let Err(err) = server::conn::auto::Builder::new(TokioExecutor::new())
                .serve_connection_with_upgrades(socket, hyper_service)
                .await
            {
                eprintln!("failed to serve connection: {err:#}");
            }
        });
    }
}

fn unwrap_infallible<T>(result: Result<T, Infallible>) -> T {
    match result {
        Ok(value) => value,
        Err(err) => match err {},
    }
}
```

## 代价与边界

**监听 `0.0.0.0` 意味着对外网开放。** 前面 43 章都用 `127.0.0.1`(只接受本机连接),这一章是 `0.0.0.0`(**接受来自任何网卡的连接**)。开发环境无所谓,但如果这台机器有公网 IP 且防火墙没配好,**服务就直接暴露在互联网上了**。

生产环境的常见做法是:应用只监听 `127.0.0.1` 或内网地址,由反向代理([第 43 章](43-reverse-proxy.md))监听 `0.0.0.0`。容器环境里则必须监听 `0.0.0.0`(否则容器外访问不到),靠网络策略和 Service 定义来限制访问面。**这是个需要显式决策的点,不该抄默认值。**

**`accept().await.unwrap()` 会让整个服务 panic。** 和[第 42 章](42-http-proxy.md#步骤-1不用-axumserve自己写-accept-循环)一样的问题:accept 的错误有一大类是可恢复的(`EMFILE` 文件描述符耗尽、客户端在 accept 前断开),不该直接崩。`axum::serve` 内部会记日志并退避重试,**手写循环就得自己实现这套逻辑**。

**没有连接数上限。** 无脑 spawn,来多少收多少。文件描述符耗尽后配合上一条就是 panic 退出。生产环境需要一个 `Semaphore` 或 `tower::limit::ConcurrencyLimit` 来封顶。

**没有连接超时和空闲超时。** 一个客户端可以建立连接后什么都不发,永久占用一个任务和一个 fd(slowloris 攻击)。hyper 的 builder 上有 `header_read_timeout` 之类的选项,但这个示例一个都没设。

**没有优雅停机。** 进程退出时,所有 spawn 出去的连接任务被直接砍掉,在途请求全部中断。[第 45 章](45-graceful-shutdown.md)专门讲这个——**手写 accept 循环时优雅停机要自己实现,比用 `axum::serve` 的 `with_graceful_shutdown` 麻烦得多**。这也是"能用 `axum::serve` 就别自己写"的一个具体理由。

**`ConnectInfo` 在反向代理后面会失真。** 这是**部署层面最容易出错的一条**:一旦服务跑在 Nginx / ELB / Cloudflare 后面,`ConnectInfo` 拿到的是**代理的地址**,不是真实客户端。所有基于它的 IP 限流、访问日志、地理判断都会失效——所有请求看起来都来自同一个 IP。

正确做法是读 `X-Forwarded-For` 或 `Forwarded` 头([第 43 章](43-reverse-proxy.md#步骤-7看看上游到底收到了什么)详细讲过),并且**只信任来自可信代理的那一层**。axum 生态里 `axum-client-ip` 这类 crate 封装了这套逻辑。

**用 `eprintln!` 而不是 `tracing`。** 这个示例连 `tracing` 依赖都没加。连接级错误(TLS 握手失败、协议错误、客户端异常断开)是排障的重要信号,应该走结构化日志([第 17 章](17-tracing-aka-logging.md))。

**HTTP/2 的参数全是默认值。** 并发流上限、初始窗口大小、最大帧大小、keep-alive 间隔——这些在高吞吐场景下都需要调,`auto::Builder` 上有对应的方法(`http2().max_concurrent_streams(...)` 之类),示例没碰。

**两个服务在一个进程里。** 和[第 43 章](43-reverse-proxy.md)一样,是演示便利。

**`serve_plain` 里的 `.call()` 跳过了 `poll_ready`。** 对裸 `Router` 没问题,但套上带背压的 tower 层之后就是违约。步骤 4 的「更好的写法」说了,统一用 `.oneshot()`。

## 动手练习

**1. 把 accept 错误处理修好。**

提示:按「代价与边界」的说法,把 `unwrap()` 换成 `match` + `continue` + 日志。**想一想哪些错误应该 continue、哪些应该退出循环**(提示:`ConnectionAborted`、`Interrupted`、`EMFILE` 可恢复;监听套接字本身失效则不可恢复)。

**2. 给连接数封顶。**

提示:`Arc<tokio::sync::Semaphore>`,accept 之后先 `acquire_owned().await`,把 permit move 进 spawn 的任务里,任务结束自动释放。设成 2,然后开三个 `curl` 长连接(比如 `curl -N` 一个慢响应),确认第三个被卡住。

**3. 加连接空闲超时。**

提示:用 `tokio::time::timeout` 包住 `serve_connection_with_upgrades`,或者用 hyper builder 上的 `header_read_timeout`。验证:`telnet 127.0.0.1 3000` 连上后什么都不发,确认服务端在超时后断开。

**4. 在同一条连接上验证 `ConnectInfo` 只求值一次。**

提示:在 `make_service.call(remote_addr)` 前面加一行 `eprintln!("new connection from {remote_addr}")`。然后对比 `curl url url`(一条连接两个请求)和两次独立的 `curl`,数一数打印了几行。**这个练习把「连接信息属于连接」变成可观测的事实。**

**5. 换成 `axum::serve` 重写,并加上优雅停机。**

提示:按步骤 6 的写法改完,再链一个 `.with_graceful_shutdown(shutdown_signal())`([第 45 章](45-graceful-shutdown.md)会详细讲)。**然后试着给手写循环也加上优雅停机** —— 你会发现要多写不少东西(`Notify` 或 `watch` channel、`select!`、`TaskTracker` 等着在途连接)。**这个对比就是"能用 `axum::serve` 就别自己写"最有说服力的论证。**

**6.(进阶)让 `serve_with_connect_info` 也能在反向代理后面工作。**

提示:加一个中间件,优先读 `X-Forwarded-For` 的第一段,读不到再回退到 `ConnectInfo`。配合[第 43 章](43-reverse-proxy.md)的反向代理一起跑:代理加上 `X-Forwarded-For`,这个服务把真实 IP 打出来。**这是生产环境的标准配置,值得亲手串一遍。**

## 小结与下一章预告

`axum::serve` 做的是五件事:**accept 循环、每连接 spawn、`TokioIo` 转 IO、`hyper::service::service_fn` 转 Service、`auto::Builder` 跑 HTTP**。加起来不到三十行,没有任何魔法。

`auto::Builder` 靠**偷看头 24 字节**判断是 HTTP/1.1 还是 HTTP/2 —— 步骤 2 把它换成 `http1::Builder` 之后,`--http2-prior-knowledge` 的请求会报 `invalid HTTP method parsed`,因为解析器把 HTTP/2 前言里的 `PRI` 当成了方法名。

`serve_connection_with_upgrades` 是该用的那个,理由[第 42 章](42-http-proxy.md)已经用一次静默挂起证明过了。

全章最重要的概念是 **`MakeService`**:`Service<Request>` 是"请求 → 响应",`MakeService<Target>` 是"**连接信息 → 一个 Service**"。它存在的理由是**有些信息属于连接而不属于请求**——客户端 IP 在 HTTP 请求里根本没有这个字段,只有 `accept()` 的返回值里有。

于是 `ConnectInfo` 的完整链路是:`accept` 拿到地址 → `make_service.call(addr)` 产出一个绑好地址的 `Router` → 每个请求的 extensions 里被插入 `ConnectInfo(addr)` → 提取器读出来。步骤 3 故意只加提取器不加 MakeService,得到的 500 错误直接点明了缺的是哪一环;步骤 5 用 keep-alive 复用连接时两个请求返回同一个端口,则从另一面印证了"属于连接"。

要注意的是,手写循环把 `axum::serve` 帮你做的事也一并丢了:accept 错误恢复、连接数上限、超时、**优雅停机**。练习 5 会让你切身体会最后一条有多麻烦。**结论是:除非有明确理由(路由前拦截、hyper 连接级配置、非 TCP 监听器),否则用 `axum::serve`。**

还有一条部署层面的提醒:`ConnectInfo` 在反向代理后面拿到的是**代理的地址**,得配合 `X-Forwarded-For` 使用——[第 43 章](43-reverse-proxy.md)和本章正好从两端讲了同一件事。

「内容分发与代理」到这里结束了。从[第 38 章](38-static-file-server.md)的静态文件,到压缩、流式写盘、转发响应、正向代理、反向代理,再到本章拆开服务器本身——**你已经把「数据怎么进出这个进程」走了一遍**。

[第 45 章 graceful-shutdown](45-graceful-shutdown.md) 开启「生产部署」阶段,第一个问题就是本章反复提到的那个:**进程要退出时,在途的请求怎么办?** 你会看到信号处理、`select!`、以及为什么"直接 `Ctrl-C`"在生产环境是不可接受的。
