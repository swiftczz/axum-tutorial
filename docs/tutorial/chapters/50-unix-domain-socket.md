# 50. unix-domain-socket

对应示例：`examples/unix-domain-socket`

Unix Domain Socket（UDS）是一种**不走 TCP/IP 网络栈**的本地 IPC 机制——两台进程通过文件系统路径（如 `/tmp/axum/helloworld`）通信，不经过网卡、不序列化成网络包，比 TCP localhost 快很多。常见场景：反向代理（Nginx）→ 后端服务、docker 容器内通信、systemd 服务。

axum 通过 `axum::serve` 直接支持 UDS listener，唯一特别的地方是**连接信息**：UDS 没有远程 IP/端口，对应的是 `peer_addr`（socket 文件地址）和 `peer_cred`（PID/UID/GID）。

分 3 步：先建 UDS server + handler，再加自定义 `UdsConnectInfo` 拿到 peer 凭证，最后用 hyper 客户端测试。

相比前面章节新引入：**`UnixListener`、`into_make_service_with_connect_info`、`Connected` trait 自定义连接信息、hyper client over UDS**。

## Cargo.toml

````toml
[package]
name = "example-unix-domain-socket"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
http-body-util = "0.1"
hyper = { version = "1.0.0", features = ["full"] }
hyper-util = { version = "0.1", features = ["tokio", "server-auto", "http1"] }
tokio = { version = "1.0", features = ["full"] }
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：UDS server 骨架

先用 `UnixListener` 绑一个 socket 文件路径，然后用 `axum::serve` 启动——和 TCP 版几乎一样，只是 listener 换成 Unix 版。

````rust
#[cfg(unix)]
#[tokio::main]
async fn main() {
    unix::server().await;
}

#[cfg(not(unix))]
fn main() {
    println!("This example requires unix")
}

#[cfg(unix)]
mod unix {
    use axum::{routing::get, Router};
    use std::path::PathBuf;
    use tokio::net::UnixListener;
    use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

    pub async fn server() {
        tracing_subscriber::registry()
            .with(
                tracing_subscriber::EnvFilter::try_from_default_env()
                    .unwrap_or_else(|_| "debug".into()),
            )
            .with(tracing_subscriber::fmt::layer())
            .init();

        let path = PathBuf::from("/tmp/axum/helloworld");

        // 清理可能残留的旧 socket 文件（bind 会因文件已存在而失败）
        let _ = tokio::fs::remove_file(&path).await;
        tokio::fs::create_dir_all(path.parent().unwrap())
            .await
            .unwrap();

        let uds = UnixListener::bind(path.clone()).unwrap();
        tokio::spawn(async move {
            let app = Router::new().route("/", get(handler));
            axum::serve(uds, app).await;
        });

        // 这里先留空，下一步加客户端测试
    }

    async fn handler() -> &'static str {
        "Hello, World!"
    }
}
````

> **新面孔：`UnixListener`**
>
> tokio 的 Unix domain socket listener，API 和 `TcpListener` 几乎一样：`bind(path)` 绑定 socket 文件，`axum::serve(uds, app)` 直接复用 axum 的 server。
>
> 关键差异：**bind 前要先删旧 socket 文件**——`bind` 遇到已存在的文件会失败。`let _ = tokio::fs::remove_file(&path).await;` 用 `let _ =` 因为第一次运行时文件不存在，忽略错误。

> **新面孔：`#[cfg(unix)]`**
>
> UDS 是 Unix 独有的（Windows 没有等价物），所以代码用 `#[cfg(unix)]` 包起来。非 Unix 平台走 `#[cfg(not(unix))]` 分支只打印一行提示。这样跨平台编译不会出错。

---

## 第二步：`UdsConnectInfo` 拿到 peer 凭证

UDS 没有远程 IP，但能拿到 socket 文件地址和**进程凭证**（PID/UID/GID）。这步定义 `UdsConnectInfo` 并通过 `into_make_service_with_connect_info` 注入。

````rust
use axum::{
    extract::connect_info::{self, ConnectInfo},
    serve::IncomingStream,
};
use std::sync::Arc;
use tokio::net::{unix::UCred, UnixListener};

# #[cfg(unix)]
# mod unix {
#     use axum::{routing::get, Router};
#     use tokio::net::UnixListener;
#     pub async fn server() {
#         // ...

        let uds = UnixListener::bind(path.clone()).unwrap();
        tokio::spawn(async move {
            let app = Router::new()
                .route("/", get(handler))
                .into_make_service_with_connect_info::<UdsConnectInfo>();
            axum::serve(uds, app).await;
        });
#     }

    // handler 通过 ConnectInfo extractor 拿到 UdsConnectInfo
    async fn handler(ConnectInfo(info): ConnectInfo<UdsConnectInfo>) -> &'static str {
        println!("new connection from `{info:?}`");
        "Hello, World!"
    }

    // 自定义连接信息：包含 peer_addr 和 peer_cred
    #[derive(Clone, Debug)]
    #[allow(dead_code)]
    struct UdsConnectInfo {
        peer_addr: Arc<tokio::net::unix::SocketAddr>,
        peer_cred: UCred,
    }

    // 实现 Connected trait，告诉 axum 怎么从 IncomingStream 提取连接信息
    impl connect_info::Connected<IncomingStream<'_, UnixListener>> for UdsConnectInfo {
        fn connect_info(stream: IncomingStream<'_, UnixListener>) -> Self {
            let peer_addr = stream.io().peer_addr().unwrap();
            let peer_cred = stream.io().peer_cred().unwrap();
            Self {
                peer_addr: Arc::new(peer_addr),
                peer_cred,
            }
        }
    }
# }
````

> **新面孔：`into_make_service_with_connect_info`**
>
> `Router::into_make_service()` 不带连接信息（handler 拿不到 remote IP/peer）。`into_make_service_with_connect_info::<T>()` 让 axum 在建立连接时提取一个 `T` 注入到 request extensions，handler 能通过 `ConnectInfo<T>` extractor 取到。
>
> `T` 必须实现 `Connected<I>`，告诉 axum 怎么从 `I`（这里是 `IncomingStream<UnixListener>`）构造 `T`。

> **新面孔：`Connected` trait**
>
> `connect_info::Connected<I>` 是 axum 的 trait，定义"如何从连接类型 `I` 提取连接信息"。我们实现的版本：
>
> ```rust
> impl Connected<IncomingStream<'_, UnixListener>> for UdsConnectInfo {
>     fn connect_info(stream: IncomingStream<'_, UnixListener>) -> Self {
>         let peer_addr = stream.io().peer_addr().unwrap();  // socket 文件地址
>         let peer_cred = stream.io().peer_cred().unwrap();  // PID/UID/GID
>         Self { peer_addr, peer_cred }
>     }
> }
> ```
>
> `stream.io()` 拿到底层 `UnixStream`，`.peer_addr()` / `.peer_cred()` 是 tokio 提供的方法。`UCred` 在 Linux 上是 `(uid, gid, pid)`，在 macOS 上是 `(uid, gid)`（无 PID）。

> **新面孔：`ConnectInfo<T>` extractor**
>
> 第 5 章 handle-head-request 没用过，但 `ConnectInfo<T>` 是 axum 标准 extractor，能从 request extensions 拿到连接信息。这里 `T = UdsConnectInfo`。
>
> TCP server 用 `ConnectInfo<SocketAddr>`，UDS 用我们自定义的 `ConnectInfo<UdsConnectInfo>`。

---

## 第三步：用 hyper client over UDS 测试

测试 UDS server 不能用 curl（curl 的 UDS 支持有限）。这步用 hyper 客户端通过 UDS 连进来发请求。

````rust
use axum::body::Body;
use axum::http::{Request, StatusCode};
use http_body_util::BodyExt;
use hyper_util::rt::TokioIo;
use tokio::net::UnixStream;

# pub async fn server() {
#     // ... 前两步的 server 代码 ...

        let uds = UnixListener::bind(path.clone()).unwrap();
        tokio::spawn(async move {
            let app = Router::new()
                .route("/", get(handler))
                .into_make_service_with_connect_info::<UdsConnectInfo>();
            axum::serve(uds, app).await;
        });

        // 测试客户端：通过 UDS 连接 server
        let stream = TokioIo::new(UnixStream::connect(path).await.unwrap());
        let (mut sender, conn) = hyper::client::conn::http1::handshake(stream).await.unwrap();
        tokio::task::spawn(async move {
            if let Err(err) = conn.await {
                println!("Connection failed: {err:?}");
            }
        });

        // URL 的 host 不重要（UDS 不走 DNS），路径是 /
        let request = Request::get("http://uri-doesnt-matter.com")
            .body(Body::empty())
            .unwrap();

        let response = sender.send_request(request).await.unwrap();

        assert_eq!(response.status(), StatusCode::OK);

        let body = response.collect().await.unwrap().to_bytes();
        let body = String::from_utf8(body.to_vec()).unwrap();
        assert_eq!(body, "Hello, World!");
# }
````

> **新面孔：`TokioIo` 包装**
>
> hyper 1.x 的 `handshake` 要求 IO 类型实现 `hyper::rt` 的 Read/Write traits，tokio 的 `UnixStream` 实现的是 `tokio::io` 的 AsyncRead/AsyncWrite。`TokioIo::new(stream)` 把 tokio IO 适配成 hyper IO。

> **新面孔：`hyper::client::conn::http1::handshake`**
>
> hyper 1.x 的低层 client API：建立一条 HTTP/1.1 连接，返回 `(sender, conn_future)`。`sender.send_request(req)` 发请求，`conn` 是连接生命周期 future（要 spawn 起来维持连接）。
>
> URL `http://uri-doesnt-matter.com` 的 host 无意义——UDS 走文件路径不走 DNS，只看 path（这里是 `/`）。

---

## 完整代码

````rust
#[cfg(unix)]
#[tokio::main]
async fn main() {
    unix::server().await;
}

#[cfg(not(unix))]
fn main() {
    println!("This example requires unix")
}

#[cfg(unix)]
mod unix {
    use axum::{
        body::Body,
        extract::connect_info::{self, ConnectInfo},
        http::{Request, StatusCode},
        routing::get,
        serve::IncomingStream,
        Router,
    };
    use http_body_util::BodyExt;
    use hyper_util::rt::TokioIo;
    use std::{path::PathBuf, sync::Arc};
    use tokio::net::{unix::UCred, UnixListener, UnixStream};
    use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

    pub async fn server() {
        tracing_subscriber::registry()
            .with(
                tracing_subscriber::EnvFilter::try_from_default_env()
                    .unwrap_or_else(|_| "debug".into()),
            )
            .with(tracing_subscriber::fmt::layer())
            .init();

        let path = PathBuf::from("/tmp/axum/helloworld");

        let _ = tokio::fs::remove_file(&path).await;
        tokio::fs::create_dir_all(path.parent().unwrap())
            .await
            .unwrap();

        let uds = UnixListener::bind(path.clone()).unwrap();
        tokio::spawn(async move {
            let app = Router::new()
                .route("/", get(handler))
                .into_make_service_with_connect_info::<UdsConnectInfo>();

            axum::serve(uds, app).await;
        });

        let stream = TokioIo::new(UnixStream::connect(path).await.unwrap());
        let (mut sender, conn) = hyper::client::conn::http1::handshake(stream).await.unwrap();
        tokio::task::spawn(async move {
            if let Err(err) = conn.await {
                println!("Connection failed: {err:?}");
            }
        });

        let request = Request::get("http://uri-doesnt-matter.com")
            .body(Body::empty())
            .unwrap();

        let response = sender.send_request(request).await.unwrap();

        assert_eq!(response.status(), StatusCode::OK);

        let body = response.collect().await.unwrap().to_bytes();
        let body = String::from_utf8(body.to_vec()).unwrap();
        assert_eq!(body, "Hello, World!");
    }

    async fn handler(ConnectInfo(info): ConnectInfo<UdsConnectInfo>) -> &'static str {
        println!("new connection from `{info:?}`");

        "Hello, World!"
    }

    #[derive(Clone, Debug)]
    #[allow(dead_code)]
    struct UdsConnectInfo {
        peer_addr: Arc<tokio::net::unix::SocketAddr>,
        peer_cred: UCred,
    }

    impl connect_info::Connected<IncomingStream<'_, UnixListener>> for UdsConnectInfo {
        fn connect_info(stream: IncomingStream<'_, UnixListener>) -> Self {
            let peer_addr = stream.io().peer_addr().unwrap();
            let peer_cred = stream.io().peer_cred().unwrap();
            Self {
                peer_addr: Arc::new(peer_addr),
                peer_cred,
            }
        }
    }
}
````

## 运行

````bash
cd examples
cargo run -p example-unix-domain-socket
````

例子自带断言（看到正常退出无 panic 即成功）。如果想手动测，需要支持 UDS 的客户端（如 `socat`、`curl --unix-socket`）：

````bash
# 用 curl 7.40+ 测（--unix-socket 选项）
curl --unix-socket /tmp/axum/helloworld http://localhost/
````

server 端日志会打印 `new connection from UdsConnectInfo { peer_addr: ..., peer_cred: ... }`。

## 解读

### UDS vs TCP

| 维度 | TCP localhost | UDS |
| --- | --- | --- |
| 协议 | 走完整 IP/TCP 栈 | 不走网络栈，纯内核 IPC |
| 地址 | `127.0.0.1:port` | 文件路径（如 `/tmp/axum/helloworld`） |
| 速度 | 慢（序列化、校验和、ack） | 快 2-3 倍 |
| peer 标识 | IP + 端口 | socket 文件地址 + 进程凭证（PID/UID/GID） |
| 用法 | 跨机器 | 同机器 |

### peer_cred 的用途

UDS 能拿到对端进程的 UID/GID/PID——TCP 做不到。用途：

- 用 UID 做访问控制（只允许 root 或特定用户连）
- 用 PID 追踪哪个进程在调用
- systemd 服务间认证

## 常见问题

**为什么必须先 `remove_file`？** `UnixListener::bind` 遇到已存在的 socket 文件会失败。每次启动要清理残留。

**Windows 能用吗？** 不能。UDS 是 Unix 独有，所以代码用 `#[cfg(unix)]` 包起来。

**为什么用 hyper client 不用 reqwest？** reqwest 不直接支持 UDS。hyper 1.x 的低层 API 能把任意 IO（包括 UDS）包装成 HTTP 连接。

**URL host 为什么不重要？** UDS 走文件路径不走 DNS，host 字段被忽略，只看 path。

## 手写任务

1. 把 `handler` 改成返回 `peer_cred` 的 UID（`info.peer_cred.uid()`）。
2. 把 socket 路径改成相对路径 `./helloworld.sock`，观察权限问题。
3. 加一个 middleware，只允许特定 UID 的连接（否则 403）。

## 小结

这章用 3 步讲了 axum 的 Unix Domain Socket 支持：

1. **UDS server**：`UnixListener::bind(path)` + `axum::serve(uds, app)`，bind 前要 `remove_file` 清残留。
2. **`UdsConnectInfo`**：实现 `Connected<IncomingStream<UnixListener>>` 提取 `peer_addr` + `peer_cred`，通过 `into_make_service_with_connect_info::<UdsConnectInfo>()` 注入。
3. **hyper client 测试**：`TokioIo::new(UnixStream)` + `hyper::client::conn::http1::handshake`，URL host 无意义。

核心：UDS 不走网络栈，比 TCP localhost 快；axum 的 `axum::serve` 同时支持 TCP 和 UDS listener；`ConnectInfo<T>` extractor 让 handler 拿到连接元信息（TCP 是 SocketAddr，UDS 是自定义 `UdsConnectInfo` 含进程凭证）。

## 源码对照

- `examples/unix-domain-socket/Cargo.toml`
- `examples/unix-domain-socket/src/main.rs`
