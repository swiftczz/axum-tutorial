# 49. serve-with-hyper

对应示例：`examples/serve-with-hyper`

前面一直用 `axum::serve(listener, app)`,这章往下看一层:不用它,自己写 accept loop,把每个 TCP 连接交给 Hyper。理解 axum 和 hyper/Tower 的边界。

## Cargo.toml

````toml
[package]
name = "example-serve-with-hyper"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
hyper = { version = "1.0", features = [] }
hyper-util = { version = "0.1", features = ["tokio", "server-auto", "http1"] }
tokio = { version = "1.0", features = ["full"] }
tower = { version = "0.5.2", features = ["util"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

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
    let app = Router::new().route("/", get(|| async { "Hello!" }));

    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

    loop {
        let (socket, _remote_addr) = listener.accept().await.unwrap();

        let tower_service = app.clone();

        tokio::spawn(async move {
            let socket = TokioIo::new(socket);

            let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
                tower_service.clone().call(request)
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

async fn serve_with_connect_info() {
    let app = Router::new().route(
        "/",
        get(|ConnectInfo(remote_addr): ConnectInfo<SocketAddr>| async move {
            format!("Hello {remote_addr}")
        }),
    );

    let mut make_service = app.into_make_service_with_connect_info::<SocketAddr>();

    let listener = TcpListener::bind("0.0.0.0:3001").await.unwrap();

    loop {
        let (socket, remote_addr) = listener.accept().await.unwrap();

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

普通服务:

````bash
curl http://127.0.0.1:3000/
````

带 ConnectInfo:

````bash
curl http://127.0.0.1:3001/
````

## 解读

### 关键心智:`axum::serve` 内部就是 hyper

标题叫 "serve-with-hyper",核心是:**`axum::serve(listener, app)` 内部其实就是用 hyper 实现的**,它帮你做了 5 步:

```text
1. 进入 accept loop,从 listener 收 TCP 连接
2. 用 TokioIo 把 TcpStream 适配成 hyper 能用的 IO
3. 用 service_fn 把你的 Router(Tower Service)桥接成 Hyper Service
4. 用 hyper 的 serve_connection 处理 HTTP 协议
5. 把 hyper 的 Response 发回客户端
```

本章不是"学新东西",而是**把 `axum::serve` 脱了外套看一遍**——这 5 步手动做出来。看清边界你就知道什么时候 `axum::serve` 够用、什么时候必须自己写。也是理解第 53-55 章低层 TLS 接入的基础(那几章也是手动 accept loop + TLS + hyper)。

### 为什么需要这么底层

大多数项目不需要这章,`axum::serve` 够用。需要低层控制的场景:自定义 accept loop、接入特殊 IO、手动控制 hyper connection builder、同时支持 http1/http2 或 upgrades、和已有 hyper 代码集成。

### accept loop

````rust
loop {
    let (socket, _remote_addr) = listener.accept().await.unwrap();
    let tower_service = app.clone();
    tokio::spawn(async move { ... });   // 每个连接 spawn 一个任务,多连接并发
}
````

每来一个 TCP 连接:accept 得到 socket → clone Router service → spawn 任务处理。

### `TokioIo` 适配 IO trait

````rust
let socket = TokioIo::new(socket);
````

Hyper 用自己的 async IO trait,Tokio 的 `TcpStream` 不能直接给 hyper 低层 API 用。`TokioIo` 适配:`tokio TcpStream → hyper 可用 IO`。

### `service_fn` 桥接 Hyper service 和 Tower service

````rust
let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
    tower_service.clone().call(request)
});
````

axum Router 是 Tower service,Hyper 需要 Hyper service,两种 trait 不同。`service_fn` 包一层:Hyper 收到 Request → 调用 `tower_service.clone().call(request)` → 得到 Response。clone 是因为 Hyper service 和 Tower service 的调用/可变借用模型不同。

### Hyper 处理连接

````rust
server::conn::auto::Builder::new(TokioExecutor::new())
    .serve_connection_with_upgrades(socket, hyper_service)
    .await
````

- `auto::Builder` 自动处理 HTTP/1 和 HTTP/2。
- `TokioExecutor` 告诉 hyper 用 Tokio spawn 任务。
- `serve_connection_with_upgrades` 支持 upgrades(WebSocket 等需要)。

### 带 ConnectInfo 的版本

````rust
let mut make_service = app.into_make_service_with_connect_info::<SocketAddr>();

loop {
    let (socket, remote_addr) = listener.accept().await.unwrap();
    let tower_service = unwrap_infallible(make_service.call(remote_addr).await);   // remote_addr 注入
    ...
}
````

普通 `ConnectInfo` 需知客户端地址。手写 accept loop 时 `remote_addr` 在 `listener.accept()` 拿到,手动传给 `into_make_service_with_connect_info` 生成的 make service。

### `unwrap_infallible`

````rust
fn unwrap_infallible<T>(result: Result<T, Infallible>) -> T {
    match result {
        Ok(value) => value,
        Err(err) => match err {},   // Infallible 不可能出错,Err 分支不可达
    }
}
````

`Infallible` 表示不可能出错,这个函数把 `Result<T, Infallible>` 变成 `T`。

## 常见问题

**平时需要这样写吗?** 通常不需要,优先用 `axum::serve`。

**`TokioIo` 干什么?** 把 Tokio TCP stream 适配成 hyper 低层 API 需要的 IO 类型。

**为什么 `service_fn`?** Hyper 和 Tower 的 service trait 不同,`service_fn` 把请求转发给 axum Router。

**ConnectInfo 为什么要特殊处理?** 手写 accept loop 时客户端地址在 `listener.accept()` 拿到,要手动传给 `into_make_service_with_connect_info` 生成的 make service。

## 手写任务

1. 写普通 axum Router。
2. 手写 `TcpListener::bind` 和 accept loop。
3. `TokioIo::new(socket)` 包装 socket。
4. `service_fn` 调用 `tower_service.call(request)`。
5. Hyper builder 处理连接。
6. 加 ConnectInfo 版本。

## 小结

- `axum::serve(listener, app)` 内部就是用 hyper 实现的,帮你做了 5 步:accept loop → TokioIo 适配 → service_fn 桥接 Tower/Hyper service → hyper serve_connection → Response 回客户端。
- 本章是**把 `axum::serve` 脱外套看一遍**,理解 axum 和 hyper/Tower 的边界,知道什么时候 `axum::serve` 够用、什么时候必须自己写。
- 低层链路:`TcpListener → accept socket → TokioIo → service_fn → Tower Router call → hyper serve_connection`。
- 大多数项目用 `axum::serve` 够;需自定义 accept loop/特殊 IO/手动控制 hyper builder/和已有 hyper 代码集成时才用低层写法。
- 这是理解第 53-55 章低层 TLS 接入的基础。

## 源码对照

- `examples/serve-with-hyper/Cargo.toml`
- `examples/serve-with-hyper/src/main.rs`
