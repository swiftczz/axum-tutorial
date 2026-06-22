# 50. unix-domain-socket

对应示例：`examples/unix-domain-socket`

本章目标：理解 Axum 如何监听 Unix domain socket，而不是 TCP 端口，并学会为 UDS 提供自定义 `ConnectInfo`。

Unix domain socket 简称 UDS。  
它用于同一台机器上的进程通信。

## 这个小项目在做什么

服务端不监听：

```text
127.0.0.1:3000
```

而是监听文件路径：

```text
/tmp/axum/helloworld
```

客户端也不通过 TCP 连接，而是：

```text
UnixStream::connect("/tmp/axum/helloworld")
```

请求主线是：

```text
创建 UnixListener
-> axum::serve(uds, app)
-> 客户端 UnixStream 连接 socket 文件
-> hyper client 在 UnixStream 上发 HTTP 请求
-> handler 返回 Hello, World!
```

## 什么时候用 Unix domain socket

UDS 适合本机进程之间通信：

```text
Nginx 反向代理到本机应用
systemd socket activation
本机 sidecar
本地工具调用后台服务
```

它不走网络端口。  
访问权限可以通过文件路径和文件权限控制。

## 文件和依赖

这个 example 有两个文件：

1. `examples/unix-domain-socket/Cargo.toml`
2. `examples/unix-domain-socket/src/main.rs`

关键依赖：

- `tokio::net::UnixListener` / `UnixStream`：Unix socket 服务端和客户端。
- `axum::serve`：可以服务 UnixListener。
- `hyper`：示例里用低层 client 在 UnixStream 上发送 HTTP 请求。
- `ConnectInfo`：提取 UDS peer 信息。

## 第一步：只在 Unix 平台运行

源码：

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
````

UDS 是 Unix 平台能力。  
非 Unix 系统直接打印提示。

## 第二步：准备 socket 文件路径

源码：

````rust
let path = PathBuf::from("/tmp/axum/helloworld");

let _ = tokio::fs::remove_file(&path).await;
tokio::fs::create_dir_all(path.parent().unwrap())
    .await
    .unwrap();
````

先删除旧 socket 文件。  
再确保父目录存在。

如果旧文件不删，`UnixListener::bind` 可能失败。

## 第三步：绑定 UnixListener

源码：

````rust
let uds = UnixListener::bind(path.clone()).unwrap();
````

这里绑定的是文件路径，不是端口。

后面仍然可以：

````rust
axum::serve(uds, app).await;
````

Axum 能处理这种 listener。

## 第四步：自定义 UdsConnectInfo

源码：

````rust
#[derive(Clone, Debug)]
struct UdsConnectInfo {
    peer_addr: Arc<tokio::net::unix::SocketAddr>,
    peer_cred: UCred,
}
````

UDS 的连接信息不是 `SocketAddr`。  
这里保存：

```text
peer_addr
peer_cred
```

`peer_cred` 可以包含对端进程的用户等凭据信息。

## 第五步：实现 Connected

源码：

````rust
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
````

这告诉 Axum：

```text
当一个 UnixListener 连接进来时
如何从 stream 中提取连接信息
```

然后 handler 就能用：

````rust
ConnectInfo(info): ConnectInfo<UdsConnectInfo>
````

## 第六步：启动 Axum 服务

源码：

````rust
let app = Router::new()
    .route("/", get(handler))
    .into_make_service_with_connect_info::<UdsConnectInfo>();

axum::serve(uds, app).await;
````

因为 handler 需要 `ConnectInfo<UdsConnectInfo>`，所以要用：

````rust
into_make_service_with_connect_info::<UdsConnectInfo>()
````

## 第七步：客户端通过 UnixStream 请求

源码：

````rust
let stream = TokioIo::new(UnixStream::connect(path).await.unwrap());
let (mut sender, conn) = hyper::client::conn::http1::handshake(stream).await.unwrap();
````

这里创建一个 UnixStream 连接。  
再用 Hyper HTTP/1 client handshake，把它当成 HTTP 连接使用。

请求 URI：

````rust
Request::get("http://uri-doesnt-matter.com")
````

这里域名不重要，因为真正连接目标已经是 UnixStream。

## 函数职责速查

- `main`：按平台选择是否运行 UDS 示例。
- `server`：创建 UDS 服务端，启动 Axum，并用 Hyper client 验证。
- `handler`：打印 UDS connect info，返回 Hello。
- `UdsConnectInfo::connect_info`：从 Unix incoming stream 提取 peer 信息。

## 带中文注释的手写版

完整 `src/main.rs`：

````rust
//! Run with
//!
//! ```not_rust
//! cargo run -p example-unix-domain-socket
//! ```

// Unix domain socket 是 Unix 平台能力，所以只在 unix 平台编译这个 main。
#[cfg(unix)]
#[tokio::main]
async fn main() {
    unix::server().await;
}

// 非 unix 平台没有 tokio::net::UnixListener，直接给出提示。
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
        // 初始化日志，方便观察连接和请求。
        tracing_subscriber::registry()
            .with(
                tracing_subscriber::EnvFilter::try_from_default_env()
                    .unwrap_or_else(|_| "debug".into()),
            )
            .with(tracing_subscriber::fmt::layer())
            .init();

        // UDS 监听的是一个文件路径，不是 127.0.0.1:3000 这样的端口。
        let path = PathBuf::from("/tmp/axum/helloworld");

        // 上次运行可能留下旧 socket 文件。绑定前先删掉，并确保父目录存在。
        let _ = tokio::fs::remove_file(&path).await;
        tokio::fs::create_dir_all(path.parent().unwrap())
            .await
            .unwrap();

        // 创建 UnixListener。后面 axum::serve 可以直接接收这个 listener。
        let uds = UnixListener::bind(path.clone()).unwrap();

        tokio::spawn(async move {
            let app = Router::new()
                .route("/", get(handler))
                // handler 要读取 UDS 连接信息，所以这里注册自定义 ConnectInfo 类型。
                .into_make_service_with_connect_info::<UdsConnectInfo>();

            axum::serve(uds, app).await;
        });

        // 下面这段不是服务端必需逻辑，而是示例里的本地客户端验证。
        // 它通过 UnixStream 连接刚才创建的 socket 文件。
        let stream = TokioIo::new(UnixStream::connect(path).await.unwrap());

        // Hyper HTTP/1 client 可以运行在任何实现了 IO trait 的 stream 上。
        let (mut sender, conn) = hyper::client::conn::http1::handshake(stream).await.unwrap();
        tokio::task::spawn(async move {
            if let Err(err) = conn.await {
                println!("Connection failed: {err:?}");
            }
        });

        // URI 的 host 在这里不重要，因为真实连接已经由 UnixStream 建好了。
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
        // 这里能拿到我们自定义的 UDS 连接信息。
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
            // 从底层 UnixStream 里提取 peer 地址和 peer credential。
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

## 运行和验证

运行：

````bash
cargo run -p example-unix-domain-socket
````

示例会自己启动服务并用 UnixStream 发请求。  
如果断言通过，说明返回了：

```text
Hello, World!
```

## 常见卡点

### 1. 为什么要删除旧 socket 文件？

UDS 绑定路径时，如果旧文件还在，bind 可能失败。

### 2. URI 为什么写 uri-doesnt-matter？

因为真正的连接目标是 UnixStream，不是 URI 里的 host。

### 3. Windows 能跑吗？

这个示例用 `#[cfg(unix)]`，非 Unix 系统只打印提示。

## 手写任务

1. 创建 `/tmp/axum/helloworld` 路径。
2. 删除旧 socket 文件。
3. 绑定 `UnixListener`。
4. 写 `UdsConnectInfo` 并实现 `Connected`。
5. 用 `axum::serve(uds, app)` 启动。
6. 用 `UnixStream` + Hyper client 发送测试请求。

## 本章真正要记住什么

UDS 版 Axum 的核心是：

```text
UnixListener 替代 TcpListener
UnixStream 替代 TCP stream
自定义 Connected 提取 UDS 连接信息
HTTP 协议仍然可以跑在这个 stream 上
```

## 源码对照

本章手写版对应源码：

- `examples/unix-domain-socket/src/main.rs`
- `examples/unix-domain-socket/Cargo.toml`
