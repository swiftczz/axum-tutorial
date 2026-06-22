# 49. serve-with-hyper

对应示例：`examples/serve-with-hyper`

本章目标：理解不用 `axum::serve` 时，如何用 Hyper 的低层 API 手动 accept TCP 连接、包装 IO、桥接 Hyper service 和 Tower service。

前面我们一直用：

````rust
axum::serve(listener, app)
````

这一章往下看一层：自己写 accept loop，把每个 TCP 连接交给 Hyper。

## 这个小项目在做什么

程序同时启动两个服务：

```text
0.0.0.0:3000 -> 普通 Hello
0.0.0.0:3001 -> 使用 ConnectInfo 显示客户端地址
```

主线是：

```text
TcpListener accept 一个 socket
-> TokioIo 包装 socket
-> 把 Axum Router 桥接成 Hyper service
-> hyper_util server builder 处理连接
-> 每个连接 tokio::spawn 并发处理
```

## 先理解为什么要这么底层

大多数项目不需要这一章。  
`axum::serve` 已经够用。

但你可能在这些场景需要低层控制：

```text
自定义 accept loop
接入特殊 IO
手动控制 hyper connection builder
同时支持 http1/http2 或 upgrades
和已有 Hyper 代码集成
```

这一章的价值是理解 Axum 和 Hyper/Tower 的边界。

### 关键心智：`axum::serve` 内部就是 hyper

这一章标题叫 "serve-with-hyper"，核心看点就是：**`axum::serve(listener, app)` 内部其实就是用 hyper 实现的**。它帮你做了这些事：

```text
axum::serve(listener, app) 内部：
  1. 进入 accept loop，从 listener 收 TCP 连接
  2. 用 TokioIo 把 TcpStream 适配成 hyper 能用的 IO
  3. 用 service_fn 把你的 Router（一个 Tower Service）桥接成 Hyper Service
  4. 用 hyper 的 serve_connection 处理 HTTP 协议
  5. 把 hyper 的 Response 发回客户端
```

所以本章不是"学一个新东西"，而是**把 `axum::serve` 脱了外套看一遍**——这 5 步手动做出来。看清了边界，你就知道什么时候 `axum::serve` 够用、什么时候必须自己写。这也是理解第 53-55 章低层 TLS 接入的基础（那几章也是手动 accept loop + TLS + hyper）。

## 文件和依赖

这个 example 有两个文件：

1. `examples/serve-with-hyper/Cargo.toml`：声明 Axum、Hyper、hyper-util、tower。
2. `examples/serve-with-hyper/src/main.rs`：实现低层 Hyper 服务接入。

关键依赖：

- `hyper`：HTTP 协议实现。
- `hyper-util`：Tokio IO 适配、server builder、executor。
- `tower`：Axum Router 的 service trait 来自 Tower。
- `tokio`：TCP listener 和任务调度。

## 第一步：同时启动两个服务

源码：

````rust
#[tokio::main]
async fn main() {
    tokio::join!(serve_plain(), serve_with_connect_info());
}
````

两个服务都是无限 accept loop。  
所以要用 `tokio::join!` 同时运行。

## 第二步：普通服务的 accept loop

源码：

````rust
let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

loop {
    let (socket, _remote_addr) = listener.accept().await.unwrap();
    let tower_service = app.clone();

    tokio::spawn(async move {
        ...
    });
}
````

每来一个 TCP 连接：

1. `accept()` 得到 socket。
2. clone 一份 Router service。
3. spawn 一个任务处理这个连接。

这样多个连接可以并发处理。

## 第三步：TokioIo 适配 IO trait

源码：

````rust
let socket = TokioIo::new(socket);
````

Hyper 使用自己的 async IO trait。  
Tokio 的 `TcpStream` 不能直接给 Hyper 低层 API 用。

`TokioIo` 负责适配：

```text
tokio TcpStream -> hyper 可用 IO
```

## 第四步：桥接 Hyper service 和 Tower service

源码：

````rust
let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
    tower_service.clone().call(request)
});
````

Axum Router 是 Tower service。  
Hyper 需要 Hyper service。

所以用 `service_fn` 包一层：

```text
Hyper 收到 Request
-> 调用 tower_service.clone().call(request)
-> 得到 Response
```

这里要 clone，是因为 Hyper service 的调用方式和 Tower service 的可变借用模型不同。

## 第五步：Hyper 处理连接

源码：

````rust
server::conn::auto::Builder::new(TokioExecutor::new())
    .serve_connection_with_upgrades(socket, hyper_service)
    .await
````

`auto::Builder` 可以自动处理 HTTP/1 和 HTTP/2。  
`TokioExecutor` 告诉 Hyper 用 Tokio spawn 任务。

`serve_connection_with_upgrades` 支持 upgrades。  
如果你的服务需要 WebSocket，这点很重要。

## 第六步：带 ConnectInfo 的服务

源码：

````rust
let mut make_service = app.into_make_service_with_connect_info::<SocketAddr>();
...
let (socket, remote_addr) = listener.accept().await.unwrap();
let tower_service = unwrap_infallible(make_service.call(remote_addr).await);
````

普通 `ConnectInfo` 需要知道客户端地址。  
这里是手写 accept loop，所以你自己拿到了：

```text
remote_addr
```

然后把它传给：

````rust
make_service.call(remote_addr).await
````

生成带连接信息的 service。

## 第七步：unwrap_infallible

源码：

````rust
fn unwrap_infallible<T>(result: Result<T, Infallible>) -> T {
    match result {
        Ok(value) => value,
        Err(err) => match err {},
    }
}
````

`Infallible` 表示不可能出错。  
所以 Err 分支不可能发生。

这个函数只是把：

```text
Result<T, Infallible>
```

变成：

```text
T
```

## 函数职责速查

- `main`：同时启动普通服务和 ConnectInfo 服务。
- `serve_plain`：手写 accept loop，把普通 Router 交给 Hyper。
- `serve_with_connect_info`：手写 accept loop，并把 remote_addr 注入 ConnectInfo。
- `unwrap_infallible`：处理不可能失败的 Result。


## 带中文注释的手写版

完整 `src/main.rs`：

````rust
//! 使用 Hyper 低层 API 运行 Axum。
//!
//! Run with
//!
//! ```not_rust
//! cargo run -p example-serve-with-hyper
//! ```

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
    // 同时启动两个低层 Hyper 服务。
    tokio::join!(serve_plain(), serve_with_connect_info());
}

async fn serve_plain() {
    let app = Router::new().route("/", get(|| async { "Hello!" }));

    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

    loop {
        let (socket, _remote_addr) = listener.accept().await.unwrap();

        // Router 总是 ready，这里每个连接 clone 一份。
        let tower_service = app.clone();

        tokio::spawn(async move {
            // 把 Tokio TCP stream 适配成 Hyper IO。
            let socket = TokioIo::new(socket);

            // 把 Tower service 桥接成 Hyper service。
            let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
                tower_service.clone().call(request)
            });

            // auto Builder 支持 http1/http2；with_upgrades 支持 WebSocket 等 upgrade。
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
        get(
            |ConnectInfo(remote_addr): ConnectInfo<SocketAddr>| async move {
                format!("Hello {remote_addr}")
            },
        ),
    );

    // 这个 make_service 会把 remote_addr 注入 ConnectInfo。
    let mut make_service = app.into_make_service_with_connect_info::<SocketAddr>();

    let listener = TcpListener::bind("0.0.0.0:3001").await.unwrap();

    loop {
        let (socket, remote_addr) = listener.accept().await.unwrap();

        // 手写 accept loop 时，remote_addr 在这里拿到，再传给 make_service。
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

// Infallible 表示不可能失败，所以 Err 分支不可达。
fn unwrap_infallible<T>(result: Result<T, Infallible>) -> T {
    match result {
        Ok(value) => value,
        Err(err) => match err {},
    }
}
````

## 运行和验证

运行：

````bash
cargo run -p example-serve-with-hyper
````

普通服务：

````bash
curl http://127.0.0.1:3000/
````

带 ConnectInfo：

````bash
curl http://127.0.0.1:3001/
````

## 常见卡点

### 1. 平时需要这样写吗？

通常不需要。  
优先用 `axum::serve`。

### 2. TokioIo 是干什么的？

把 Tokio 的 TCP stream 适配成 Hyper 低层 API 需要的 IO 类型。

### 3. 为什么要 service_fn？

Hyper 和 Tower 的 service trait 不同。  
`service_fn` 用来把请求转发给 Axum Router。

### 4. ConnectInfo 为什么要特殊处理？

因为手写 accept loop 时，客户端地址在 `listener.accept()` 这里拿到。  
你要手动把它传给 `into_make_service_with_connect_info` 生成的 make service。

## 手写任务

1. 先写普通 Axum Router。
2. 手写 `TcpListener::bind` 和 accept loop。
3. 用 `TokioIo::new(socket)` 包装 socket。
4. 用 `service_fn` 调用 `tower_service.call(request)`。
5. 用 Hyper builder 处理连接。
6. 再加一个 ConnectInfo 版本。

## 本章真正要记住什么

`axum::serve` 帮你隐藏了很多底层细节。  
低层写法的核心链路是：

```text
TcpListener
accept socket
TokioIo
Hyper service_fn
Tower Router call
Hyper serve_connection
```

理解它能帮你在需要底层控制时知道该接在哪里。

## 源码对照

本章手写版对应源码：

- `examples/serve-with-hyper/src/main.rs`
- `examples/serve-with-hyper/Cargo.toml`
