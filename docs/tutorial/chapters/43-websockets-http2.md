# 43. websockets-http2

对应示例：`examples/websockets-http2`

**前置**：第 41 章（WebSocket 基础）、第 46 章（HTTPS / TLS rustls）

第 41 章 WebSocket 走 HTTP/1.1 upgrade 机制（`Connection: Upgrade, Upgrade: websocket`）。HTTP/2 不支持 upgrade，改用 **RFC 8441 Extended CONNECT**——HTTP/2 上的 WebSocket。这章演示怎么启用并写一个简单的广播聊天 server。

分 2 步：先启动支持 HTTP/2 WebSocket 的 HTTPS server + 静态文件 + broadcast channel，再写 ws_handler 处理双向消息。

相比前面章节新引入：**`http_builder().http2().enable_connect_protocol()`（HTTP/2 Extended CONNECT）、`broadcast::channel` 广播、`tokio::select!` 并发读 socket 和 broadcast**。

## Cargo.toml

````toml
[package]
name = "example-websockets-http2"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["ws"] }
axum-server = { version = "0.8", features = ["tls-rustls"] }
tokio = { version = "1.0", features = ["full"] }
tower-http = { version = "0.6", features = ["fs"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：HTTPS server + broadcast channel + 启用 HTTP/2 Extended CONNECT

这步搭骨架。关键差异：必须显式 `enable_connect_protocol()`，否则浏览器在 HTTP/2 上发起 WebSocket 会失败。broadcast channel 用于所有连接共享消息（聊天室效果）。

````rust
use axum::{
    extract::{ws::WebSocketUpgrade, State},
    routing::any,
    Router,
};
use axum_server::tls_rustls::RustlsConfig;
use std::{net::SocketAddr, path::PathBuf};
use tokio::sync::broadcast;
use tower_http::services::ServeDir;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let assets_dir = PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("assets");

    // 证书（参考第 46 章）
    let config = RustlsConfig::from_pem_file(
        PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("self_signed_certs").join("cert.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("self_signed_certs").join("key.pem"),
    )
    .await
    .unwrap();

    // broadcast channel：所有连接共享，发一条大家都能收到
    let app = Router::new()
        .fallback_service(ServeDir::new(assets_dir).append_index_html_on_directories(true))
        .route("/ws", any(ws_handler))
        .with_state(broadcast::channel::<String>(16).0);

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {}", addr);

    let mut server = axum_server::bind_rustls(addr, config);

    // 关键：启用 HTTP/2 Extended CONNECT，让 HTTP/2 上能跑 WebSocket
    // axum::serve 默认启用，但 axum_server 要手动
    server.http_builder().http2().enable_connect_protocol();

    server.serve(app.into_make_service()).await.unwrap();
}

async fn ws_handler(
    ws: WebSocketUpgrade,
    version: axum::http::Version,
    State(sender): State<broadcast::Sender<String>>,
) -> axum::response::Response {
    tracing::debug!("accepted a WebSocket using {version:?}");
    let mut receiver = sender.subscribe();
    ws.on_upgrade(|mut ws| async move {
        // 下一步填这里
    })
}
````

验证（用 examples 自带的 `assets/index.html` 前端，里面会自动连 `/ws`）：

````bash
cd examples
cargo run -p example-websockets-http2
````

浏览器访问 `https://localhost:3000/`（忽略证书警告），开两个标签页，在 console 看到 `accepted a WebSocket using Http2` 就说明走的是 HTTP/2 WebSocket。

> **新面孔：`enable_connect_protocol`（HTTP/2 Extended CONNECT）**
>
> HTTP/1.1 用 `Connection: Upgrade` 切换协议。HTTP/2 不支持 upgrade，需要 RFC 8441 的 Extended CONNECT method——通过 `:protocol` pseudo-header 协商。
>
> `axum::serve` 默认启用它；但 `axum_server` 不启用，必须手动 `server.http_builder().http2().enable_connect_protocol()`。
>
> 不启用的话浏览器在 HTTP/2 连接上发起 WebSocket 会失败。

> **新面孔：`broadcast::channel`**
>
> tokio 的广播 channel：一个发送者 `.send()` → 所有订阅者 `.subscribe().recv()` 都收到。这章用来做聊天室——所有 WebSocket 连接共享一个 channel，发一条广播给所有人。
>
> `broadcast::channel::<String>(16)` 的 16 是 channel 容量。容量满了新消息会挤掉旧的（latency 比 completeness 重要的场景适用）。

> **新面孔：`any` 路由方法**
>
> WebSocket 路由用 `any` 不是 `get`——因为 WebSocket 握手虽然通常是 GET，但严格来说 HTTP/2 Extended CONNECT 用的是 CONNECT 方法。`any` 接受所有方法。

---

## 第二步：ws_handler 用 `tokio::select!` 并发处理双向消息

WebSocket handler 要同时做两件事：**从 socket 读消息广播出去** + **从 broadcast 收消息发回 socket**。用 `tokio::select!` 并发两个 future。

````rust
use axum::extract::ws;

# async fn ws_handler(
#     ws: WebSocketUpgrade,
#     version: axum::http::Version,
#     State(sender): State<broadcast::Sender<String>>,
# ) -> axum::response::Response {
#     let mut receiver = sender.subscribe();
    ws.on_upgrade(|mut ws| async move {
        loop {
            tokio::select! {
                // 从 socket 读消息（客户端 → 服务端），广播给所有订阅者
                res = ws.recv() => {
                    match res {
                        Some(Ok(ws::Message::Text(s))) => {
                            let _ = sender.send(s.to_string());
                        }
                        Some(Ok(_)) => {}
                        Some(Err(e)) => tracing::debug!("client disconnected abruptly: {e}"),
                        None => break,
                    }
                }
                // 从 broadcast 读消息（其他客户端发的），发回这个 socket
                res = receiver.recv() => {
                    match res {
                        Ok(msg) => if let Err(e) = ws.send(ws::Message::Text(msg.into())).await {
                            tracing::debug!("client disconnected abruptly: {e}");
                        }
                        Err(_) => continue,
                    }
                }
            }
        }
    })
# }
````

> **新面孔：`tokio::select!` 多路复用**
>
> `select!` 同时等多个 future，先 ready 的那个分支执行，其他取消。这章两个分支：
>
> - `ws.recv()`：等客户端发消息（输入方向）
> - `receiver.recv()`：等 broadcast 推消息（输出方向）
>
> select! 在每轮 `loop` 都跑一次——双向消息独立到达，互不阻塞。

> **新面孔：cancel safety**
>
> `tokio::select!` 每轮会取消未完成的分支。如果分支的 future 不是 cancel-safe，可能丢数据。源码注释强调：
>
> - `ws.recv()` 是 cancel-safe（axum 文档保证）
> - `broadcast::Receiver::recv()` 是 cancel-safe（tokio 保证）
>
> 如果换成 `tokio::mpsc::Receiver::recv()` 也要确认 cancel safety。

---

## 完整代码

````rust
use axum::{
    extract::{
        ws::{self, WebSocketUpgrade},
        State,
    },
    http::Version,
    routing::any,
    Router,
};
use axum_server::tls_rustls::RustlsConfig;
use std::{net::SocketAddr, path::PathBuf};
use tokio::sync::broadcast;
use tower_http::services::ServeDir;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let assets_dir = PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("assets");

    // configure certificate and private key used by https
    let config = RustlsConfig::from_pem_file(
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("cert.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("key.pem"),
    )
    .await
    .unwrap();

    // build our application with some routes and a broadcast channel
    let app = Router::new()
        .fallback_service(ServeDir::new(assets_dir).append_index_html_on_directories(true))
        .route("/ws", any(ws_handler))
        .with_state(broadcast::channel::<String>(16).0);

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {}", addr);

    let mut server = axum_server::bind_rustls(addr, config);

    // IMPORTANT: This is required to advertise our support for HTTP/2 websockets to the client.
    // If you use axum::serve, it is enabled by default.
    server.http_builder().http2().enable_connect_protocol();

    server.serve(app.into_make_service()).await.unwrap();
}

async fn ws_handler(
    ws: WebSocketUpgrade,
    version: Version,
    State(sender): State<broadcast::Sender<String>>,
) -> axum::response::Response {
    tracing::debug!("accepted a WebSocket using {version:?}");
    let mut receiver = sender.subscribe();
    ws.on_upgrade(|mut ws| async move {
        loop {
            tokio::select! {
                // Since `ws` is a `Stream`, it is by nature cancel-safe.
                res = ws.recv() => {
                    match res {
                        Some(Ok(ws::Message::Text(s))) => {
                            let _ = sender.send(s.to_string());
                        }
                        Some(Ok(_)) => {}
                        Some(Err(e)) => tracing::debug!("client disconnected abruptly: {e}"),
                        None => break,
                    }
                }
                // Tokio guarantees that `broadcast::Receiver::recv` is cancel-safe.
                res = receiver.recv() => {
                    match res {
                        Ok(msg) => if let Err(e) = ws.send(ws::Message::Text(msg.into())).await {
                            tracing::debug!("client disconnected abruptly: {e}");
                        }
                        Err(_) => continue,
                    }
                }
            }
        }
    })
}
````

## 运行

````bash
cd examples
cargo run -p example-websockets-http2
````

浏览器访问 `https://localhost:3000/`（用 `-k` 忽略证书警告的浏览器需要先 `thisisunsafe` 或类似机制），打开两个标签页，在输入框发消息——两个标签都能看到。打开浏览器 dev tools 看 Network → ws 请求，`Request Headers` 里有 `:protocol: websocket`，`Version` 是 `h2`。

## 解读

### HTTP/1.1 vs HTTP/2 WebSocket

| 维度 | HTTP/1.1 upgrade（ch41） | HTTP/2 Extended CONNECT（这章） |
| --- | --- | --- |
| 协议机制 | `Connection: Upgrade` + `Upgrade: websocket` | `:method: CONNECT` + `:protocol: websocket` |
| 标准化 | RFC 6455 | RFC 8441 |
| 多路复用 | 一条 TCP 一个 WebSocket | 多个 WebSocket 共享一条 HTTP/2 连接 |
| 启用 | 默认 | 显式 `enable_connect_protocol()` |

HTTP/2 WebSocket 好处：一条 TCP/TLS 连接可跑多个 WebSocket（HTTP/2 多路复用），少握手开销。

### 聊天室架构

```text
client A  ──┐
client B  ──┼──>  broadcast::Sender  ──>  所有 Receiver
client C  ──┘                                ↑
                                         每个 ws_handler subscribe 一个
```

所有连接的 `ws_handler` 持有同一个 `Sender`（来自 State）+ 自己的 `Receiver`（`.subscribe()`）。某个 client 发消息 → `sender.send()` → 所有 Receiver 收到 → 各自的 ws 发回自己的 socket。

## 常见问题

**为什么 `axum::serve` 不用 `enable_connect_protocol`？** axum::serve 默认启用。`axum_server::bind_rustls` 不启用，必须手动调用——这是 axum 和 axum_server 的差异。

**HTTP/1.1 客户端能用吗？** 能。这个 server 同时支持 HTTP/1.1 和 HTTP/2，浏览器会自动协商。HTTP/1.1 走普通 upgrade（ch41），HTTP/2 走 Extended CONNECT。

**broadcast 容量满了怎么办？** 新消息会挤掉最旧的（`Lagged` 错误）。源码里 `Err(_) => continue` 跳过这种情况。生产可能要 log 警告或调大容量。

## 手写任务

1. 改成 whisper 模式：消息带 `@user` 前缀的只发给指定用户（提示：每个连接 subscribe 后维护自己的 ID）。
2. 加用户列表：每个连接 subscribe 时记录，断开时移除，提供 `/users` 接口查询在线用户。
3. 把消息改成 JSON（带 username + content + timestamp），前端用模板渲染。

## 小结

这章用 2 步讲了 HTTP/2 WebSocket：

1. **server 骨架**：HTTPS（第 46 章）+ `enable_connect_protocol()`（HTTP/2 Extended CONNECT）+ `broadcast::channel` 共享消息。
2. **ws_handler**：`tokio::select!` 并发读 socket 和 broadcast，cancel-safe 的两个 recv。

核心：HTTP/2 WebSocket 用 RFC 8441 Extended CONNECT，需要在 server 显式启用；`broadcast::channel` 让聊天室模式简洁；`tokio::select!` 处理双向消息。

## 源码对照

- `examples/websockets-http2/Cargo.toml`
- `examples/websockets-http2/src/main.rs`
- `examples/websockets-http2/assets/index.html`
- `examples/websockets-http2/self_signed_certs/cert.pem`
- `examples/websockets-http2/self_signed_certs/key.pem`
