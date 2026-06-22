# 49. serve-with-hyper

对应示例：`examples/serve-with-hyper`

`axum::serve(listener, app)` 一行启动 server——但有时候需要更多控制（自定义握手、连接级 middleware、特殊协议）。这章演示**手动用 hyper 启动**，理解 axum server 内部到底跑了什么。

分 3 步：先用 hyper 服务最简连接（不用 ConnectInfo），再加 ConnectInfo 拿到 remote_addr，最后对比两种用法的差异。

相比前面章节新引入：**`tower::Service` / `ServiceExt::oneshot`、`hyper_util::server::conn::auto::Builder`、`TokioIo` 适配器、`hyper::service::service_fn`、手动 TCP accept loop**。

## Cargo.toml

````toml
[package]
name = "example-serve-with-hyper"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
hyper = "1.0.0"
hyper-util = { version = "0.1", features = ["server", "tokio"] }
tokio = { version = "1.0", features = ["full"] }
tower = { version = "0.5", features = ["util"] }
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：用 hyper 服务最简连接（不用 ConnectInfo）

这步展开 `axum::serve` 的内部：手动 accept TCP 连接，spawn task，每条连接用 hyper 的 `auto::Builder` 服务。Router 通过 `service_fn` 包成 hyper Service。

````rust
use axum::{extract::Request, routing::get, Router};
use hyper::body::Incoming;
use hyper_util::rt::{TokioExecutor, TokioIo};
use hyper_util::server;
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    serve_plain().await;
}

async fn serve_plain() {
    // 普通的 axum app
    let app = Router::new().route("/", get(|| async { "Hello!" }));

    // 用 tokio 建 TCP listener
    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

    // 持续接受新连接
    loop {
        // 这步丢弃 remote_addr，下一步演示怎么暴露它
        let (socket, _remote_addr) = listener.accept().await.unwrap();

        // Router 总是 ready，不用 poll_ready
        let tower_service = app.clone();

        // 每条连接 spawn 一个 task，多连接并发
        tokio::spawn(async move {
            // hyper 有自己的 AsyncRead/AsyncWrite，TokioIo 在 tokio IO 和 hyper IO 间适配
            let socket = TokioIo::new(socket);

            // hyper 也有自己的 Service trait，service_fn 把 axum 的 Router 包成 hyper Service
            let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
                // hyper Service 用 &self，tower Service 用 &mut self
                // 所以每次 clone 一个新 Router 处理这条连接
                tower_service.clone().call(request)
            });

            // auto::Builder 自动选 HTTP/1.1 或 HTTP/2
            // serve_connection_with_upgrades 支持 WebSocket upgrade
            if let Err(err) = server::conn::auto::Builder::new(TokioExecutor::new())
                .serve_connection_with_upgrades(socket, hyper_service)
                .await
            {
                eprintln!("failed to serve connection: {err:#}");
            }
        });
    }
}
````

验证：

````bash
cd examples
cargo run -p example-serve-with-hyper
````

````bash
curl http://127.0.0.1:3000/
# 返回 "Hello!"
````

> **新面孔：手动 TCP accept loop**
>
> 所有 HTTP server 的标准骨架：
>
> ```text
> loop {
>     let (socket, addr) = listener.accept().await;   // 等新连接
>     tokio::spawn(async move { /* 服务这条连接 */ });  // 每条连接独立 task
> }
> ```
>
> 这正是 `axum::serve` 内部循环。这章把它显式展开。

> **新面孔：`TokioIo` 适配器**
>
> tokio IO 实现 `tokio::io::{AsyncRead, AsyncWrite}`，hyper 要求 `hyper::rt::{Read, Write}`——两套独立 trait，不兼容。`TokioIo::new(socket)` 把 tokio IO 适配成 hyper IO。
>
> 所有 hyper low-level 代码都要这个适配。ch50、ch53 都见过它。

> **新面孔：`hyper::service::service_fn`**
>
> hyper 不用 `tower::Service`，有自己的 `hyper::service::Service` trait（接口略不同）。`service_fn(closure)` 把闭包包成 hyper Service。
>
> 闭包内 `tower_service.clone().call(request)` 调用 axum Router——必须 clone 因为 hyper Service 是 `&self`，tower Service 是 `&mut self`，每次 clone 一个新 Router 处理请求。Router 的 clone 很便宜（内部 `Arc`）。

> **新面孔：`auto::Builder::serve_connection_with_upgrades`**
>
> hyper-util 提供的服务端连接 builder：
> - `auto`：根据 ALPN 自动选 HTTP/1.1 或 HTTP/2
> - `serve_connection_with_upgrades`：服务连接 + 支持 HTTP upgrade（如 WebSocket）
> - `TokioExecutor::new()`：告诉 hyper 用 tokio runtime spawn task
>
> 这就是 `axum::serve` 内部最终调用的东西。

---

## 第二步：加 `ConnectInfo`——拿到 remote_addr

第一步丢弃了 remote_addr。这步演示怎么把它暴露给 handler，让 handler 能拿到客户端 IP（用于日志、限流、地理定位）。

关键差异：用 `into_make_service_with_connect_info::<SocketAddr>()` 而不是直接 `app.clone()`——它在每条连接来时生成带 remote_addr 的 Router。

````rust
use axum::extract::ConnectInfo;
use std::{convert::Infallible, net::SocketAddr};
use tower::{Service, ServiceExt};

async fn serve_with_connect_info() {
    let app = Router::new().route(
        "/",
        get(|ConnectInfo(remote_addr): ConnectInfo<SocketAddr>| async move {
            format!("Hello {remote_addr}")
        }),
    );

    // 关键：用 make_service_with_connect_info，每条连接注入 remote_addr
    let mut make_service = app.into_make_service_with_connect_info::<SocketAddr>();

    let listener = TcpListener::bind("0.0.0.0:3001").await.unwrap();

    loop {
        let (socket, remote_addr) = listener.accept().await.unwrap();

        // make_service.call(remote_addr) 返回带 ConnectInfo 注入的 Router
        let tower_service = unwrap_infallible(make_service.call(remote_addr).await);

        tokio::spawn(async move {
            let socket = TokioIo::new(socket);

            let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
                // 和第一步的差异：用 oneshot 而不是 call
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
````

> **新面孔：`into_make_service_with_connect_info`**
>
> 第一步用 `app.clone()` 直接复制 Router。这步用 `into_make_service_with_connect_info::<SocketAddr>()`——它返回一个**工厂**，每条连接来时 `make_service.call(remote_addr)` 生成一个**注入了 remote_addr** 的 Router。
>
> 注入的 remote_addr 存进 request extensions，handler通过 `ConnectInfo<SocketAddr>` extractor 取出。ch50 UDS 用同名方法注入 `UdsConnectInfo`。

> **新面孔：`Service::call` vs `ServiceExt::oneshot`**
>
> 第一步 handler 里 `tower_service.clone().call(request)`，这步用 `oneshot(request)`。区别：
>
> - `Service::call(&mut self, req)` → 返回 future（self 还能复用，需要先 `poll_ready`）
> - `ServiceExt::oneshot(self, req)` → 返回 future（消费 self，不用 `poll_ready`）
>
> `oneshot` 是 `call` 的便捷版——一次调用后 service 就消费掉了。这里每条连接 clone 一个新 service 用一次，正好适合。

> **新面孔：`ConnectInfo<SocketAddr>` extractor**
>
> 第 50 章见过。提取注入到 request extensions 的连接信息。TCP 用 `ConnectInfo<SocketAddr>`（remote IP+port），UDS 用自定义 `ConnectInfo<UdsConnectInfo>`。

### `unwrap_infallible` 解释

`make_service.call(addr).await` 返回 `Result<T, Infallible>`——错误类型是 `Infallible`（不可能存在的类型）。所以 `unwrap_infallible` 用 `match err {}` 空匹配（编译器知道 Err 分支不可能到达）。

---

## 第三步：对比两种用法

```text
serve_plain (无 ConnectInfo):
    app.clone() → tower_service → clone().call(request)

serve_with_connect_info (有 ConnectInfo):
    app.into_make_service_with_connect_info() → make_service
    make_service.call(remote_addr) → tower_service（带 remote_addr 注入）
    tower_service.clone().oneshot(request)
```

| 维度 | serve_plain | serve_with_connect_info |
| --- | --- | --- |
| Router 来源 | `app.clone()` 直接复制 | `make_service.call(addr)` 生成 |
| ConnectInfo | handler 拿不到 | handler 用 `ConnectInfo<SocketAddr>` 取 |
| 多一层抽象 | 没有 | 多一个 make_service 工厂 |
| 适用场景 | 不需要 remote_addr | 需要客户端 IP 做日志/限流 |

完整代码同时跑两个 server（用 `tokio::join!`）：

````rust
#[tokio::main]
async fn main() {
    tokio::join!(serve_plain(), serve_with_connect_info());
}
````

---

## 完整代码

````rust
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
````

## 运行

````bash
cd examples
cargo run -p example-serve-with-hyper
````

两个端口同时跑：

````bash
curl http://127.0.0.1:3000/     # 返回 "Hello!"
curl http://127.0.0.1:3001/     # 返回 "Hello 127.0.0.1:xxxxx"（remote_addr）
````

## 解读

### `axum::serve` 内部就是这章

```text
axum::serve(listener, app) 内部等价于：
    loop {
        let (socket, addr) = listener.accept();
        spawn(service_connection(socket, addr));
    }
```

`axum::serve` 帮你包好了：TCP accept loop、TokioIo 适配、service_fn 包 Router、auto Builder、错误处理。这章全部手动写出来。

### 什么时候手动写

99% 场景用 `axum::serve` 就够。手动 hyper 适合：

- 自定义传输层（TLS、QUIC、UDS、自定义协议）——ch53 TLS 走这条路
- 连接级 middleware（每条连接独立 middleware 链）
- axum 还没封装的高级特性

## 常见问题

**为什么必须 spawn task？** 不 spawn 的话一次只能处理一条连接——前一条连接没结束下一条进不来。spawn 让每条连接独立运行，互不阻塞。

**`serve_connection_with_upgrades` 和 `serve_connection` 区别？** 前者支持 HTTP upgrade（主要是 WebSocket），后者不支持。要用 WebSocket 必须前者。

**`Infallible` 是什么？** "不可能存在的错误"——类型系统保证这个 Result 永远是 Ok。`unwrap_infallible` 用空 `match err {}` 处理（编译器知道 Err 不可能）。

## 手写任务

1. 加 graceful shutdown：用 `tokio::select!` 同时等 Ctrl+C 和 `serve_connection_with_upgrades`。
2. 加每条连接的 tracing span：`tracing::info_span!("conn", %remote_addr)` 包住 task。
3. 改成只支持 HTTP/1.1（用 `server::conn::http1::Builder` 而不是 `auto::Builder`）。
4. 加 connection 超时：用 `tokio::time::timeout` 包住 `serve_connection_with_upgrades`。

## 小结

这章用 3 步展开了 hyper low-level server：

1. **基础 hyper 服务**：手动 TCP accept loop + `TokioIo` + `service_fn` 包 Router + `auto::Builder::serve_connection_with_upgrades`。
2. **加 ConnectInfo**：`into_make_service_with_connect_info` + `make_service.call(addr)` 生成带 remote_addr 注入的 Router，handler 用 `ConnectInfo<SocketAddr>` 取。
3. **对比两种**：直接 `app.clone()` 简单但拿不到 remote_addr；make_service 工厂能注入但多一层。

核心理解：**`axum::serve` 内部就是这章的代码**。手动 hyper 是为了自定义传输层、连接级 middleware、axum 没封装的特性。

## 源码对照

- `examples/serve-with-hyper/Cargo.toml`
- `examples/serve-with-hyper/src/main.rs`
