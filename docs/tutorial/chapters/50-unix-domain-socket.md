# 50. unix-domain-socket

对应示例：`examples/unix-domain-socket`

理解 axum 如何监听 Unix domain socket(UDS)而不是 TCP 端口,并为 UDS 提供自定义 `ConnectInfo`。UDS 用于同一台机器上的进程通信。

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

## src/main.rs

````rust
// Unix domain socket 是 Unix 平台能力,只在 unix 平台编译 main
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
        tokio::fs::create_dir_all(path.parent().unwrap()).await.unwrap();

        let uds = UnixListener::bind(path.clone()).unwrap();

        tokio::spawn(async move {
            let app = Router::new()
                .route("/", get(handler))
                .into_make_service_with_connect_info::<UdsConnectInfo>();

            axum::serve(uds, app).await;
        });

        // 本地客户端验证:通过 UnixStream 连 socket 文件
        let stream = TokioIo::new(UnixStream::connect(path).await.unwrap());

        let (mut sender, conn) = hyper::client::conn::http1::handshake(stream).await.unwrap();
        tokio::task::spawn(async move {
            if let Err(err) = conn.await {
                println!("Connection failed: {err:?}");
            }
        });

        // URI host 不重要,真实连接已由 UnixStream 建好
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

示例自己启动服务并用 UnixStream 发请求,断言通过说明返回了 `Hello, World!`。

## 解读

### 什么时候用 UDS

UDS 适合本机进程之间通信:Nginx 反向代理到本机应用、systemd socket activation、本机 sidecar、本地工具调后台服务。**不走网络端口**,访问权限通过文件路径和文件权限控制。

### 只在 Unix 平台

````rust
#[cfg(unix)]
#[tokio::main]
async fn main() { unix::server().await; }

#[cfg(not(unix))]
fn main() { println!("This example requires unix") }
````

UDS 是 Unix 平台能力,非 Unix 系统直接打印提示。

### 绑定文件路径而不是端口

````rust
let path = PathBuf::from("/tmp/axum/helloworld");
let _ = tokio::fs::remove_file(&path).await;                    // 先删旧 socket 文件
tokio::fs::create_dir_all(path.parent().unwrap()).await.unwrap();  // 确保父目录存在
let uds = UnixListener::bind(path.clone()).unwrap();
````

UDS 监听文件路径(`/tmp/axum/helloworld`)不是端口。**先删旧 socket 文件**——如果旧文件还在 `UnixListener::bind` 可能失败。后面 `axum::serve(uds, app)` 仍能处理这个 listener。

### 自定义 `UdsConnectInfo`

````rust
#[derive(Clone, Debug)]
struct UdsConnectInfo {
    peer_addr: Arc<tokio::net::unix::SocketAddr>,
    peer_cred: UCred,    // 对端进程的用户等凭据信息
}

impl connect_info::Connected<IncomingStream<'_, UnixListener>> for UdsConnectInfo {
    fn connect_info(stream: IncomingStream<'_, UnixListener>) -> Self {
        let peer_addr = stream.io().peer_addr().unwrap();
        let peer_cred = stream.io().peer_cred().unwrap();
        Self { peer_addr: Arc::new(peer_addr), peer_cred }
    }
}
````

UDS 连接信息不是 `SocketAddr`。`Connected` trait 告诉 axum:当 UnixListener 连接进来时如何从 stream 提取连接信息(`peer_addr` + `peer_cred`)。然后 handler 用 `ConnectInfo(info): ConnectInfo<UdsConnectInfo>`。

### 启动服务

````rust
let app = Router::new()
    .route("/", get(handler))
    .into_make_service_with_connect_info::<UdsConnectInfo>();   // 注册自定义 ConnectInfo 类型

axum::serve(uds, app).await;
````

因为 handler 需 `ConnectInfo<UdsConnectInfo>`,要用 `into_make_service_with_connect_info::<UdsConnectInfo>()`(对比第 41 章 TCP 的 `into_make_service_with_connect_info::<SocketAddr>()`)。

### 客户端通过 UnixStream

````rust
let stream = TokioIo::new(UnixStream::connect(path).await.unwrap());
let (mut sender, conn) = hyper::client::conn::http1::handshake(stream).await.unwrap();
// ...
let request = Request::get("http://uri-doesnt-matter.com").body(Body::empty()).unwrap();
````

创建 UnixStream 连接 socket 文件,用 hyper HTTP/1 client handshake 把它当 HTTP 连接用。**URI host 不重要**——真实连接已由 UnixStream 建好。

## 常见问题

**为什么删旧 socket 文件?** UDS 绑定路径时如果旧文件还在,bind 可能失败。

**URI 为什么写 `uri-doesnt-matter`?** 真实连接目标是 UnixStream 不是 URI 里的 host。

**Windows 能跑吗?** 用 `#[cfg(unix)]`,非 Unix 系统只打印提示。

## 手写任务

1. 创建 `/tmp/axum/helloworld` 路径。
2. 删除旧 socket 文件。
3. 绑定 `UnixListener`。
4. 写 `UdsConnectInfo` 并实现 `Connected`。
5. `axum::serve(uds, app)` 启动。
6. `UnixStream` + Hyper client 发测试请求。

## 小结

- UDS 版 axum 核心:`UnixListener` 替代 `TcpListener`,`UnixStream` 替代 TCP stream,自定义 `Connected` 提取 UDS 连接信息(`peer_addr` + `peer_cred`),HTTP 协议仍跑在这个 stream 上。
- UDS 监听文件路径不是端口;绑定前先删旧 socket 文件,否则 bind 可能失败。
- UDS 适合本机进程通信(Nginx 反代、systemd socket activation、sidecar),不走网络,访问权限通过文件权限控制。
- 只 Unix 平台有;客户端通过 `UnixStream` + hyper handshake 连接,URI host 不重要。
- `into_make_service_with_connect_info::<UdsConnectInfo>()` 注册自定义 ConnectInfo 类型(对比 TCP 的 `::<SocketAddr>()`)。

## 源码对照

- `examples/unix-domain-socket/Cargo.toml`
- `examples/unix-domain-socket/src/main.rs`
