# 43. websockets-http2

对应示例：`examples/websockets-http2`

上一章是普通 WebSocket,这章重点不是聊天业务,而是 **WebSocket over HTTP/2**:学会用 `axum-server` + rustls 启动 HTTPS 服务,启用 HTTP/2 CONNECT protocol。



相比前面章节新引入：**HTTP/2 WebSocket、`enable_connect_protocol`、ALPN 协商**。

## Cargo.toml

````toml
[package]
name = "example-websockets-http2"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["ws", "http2"] }
axum-server = { version = "0.8", features = ["tls-rustls"] }
tokio = { version = "1", features = ["full"] }
tower-http = { version = "0.6", features = ["fs"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

axum 启用 `ws` + `http2` feature;`axum-server` 启用 `tls-rustls`。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 关键概念

> **新面孔：HTTP/2 WebSocket + `enable_connect_protocol`**
>
> HTTP/2 WebSocket 使用扩展 CONNECT 机制。`axum_server::bind_rustls` 需手动启用 `enable_connect_protocol`（`axum::serve` 默认启用）。ALPN 协商 h2/http1.1。


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

    let config = RustlsConfig::from_pem_file(
        PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("self_signed_certs").join("cert.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("self_signed_certs").join("key.pem"),
    )
    .await
    .unwrap();

    let app = Router::new()
        .fallback_service(ServeDir::new(assets_dir).append_index_html_on_directories(true))
        .route("/ws", any(ws_handler))
        .with_state(broadcast::channel::<String>(16).0);

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {}", addr);

    let mut server = axum_server::bind_rustls(addr, config);

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

浏览器打开 `https://localhost:3000/`(注意 `https`)。证书不可信需手动信任本地自签名证书或重新生成开发证书。打开两个窗口互相发消息,终端日志关注 `accepted a WebSocket using HTTP/2`(或 HTTP/1.1)。

## 解读

### 和普通 WebSocket 的差异

```text
普通示例:  ws://localhost:3000/ws          axum::serve(listener, app)
本章示例:  wss://localhost:3000/ws         axum_server::bind_rustls + enable_connect_protocol
```

多出来:TLS 配置 + HTTP/2 CONNECT protocol。前端用 `wss://`(基于 TLS 的 WebSocket)。

### 加载自签名证书

````rust
let config = RustlsConfig::from_pem_file(cert_path, key_path).await.unwrap();
````

读取 `cert.pem` 和 `key.pem` 启动 HTTPS。⚠️ 仓库证书是教学用自签名证书,可能已过期,浏览器会拦截/拒绝。真实本地测试建议重新生成开发证书并让浏览器信任。

### `axum_server::bind_rustls`(不用 `axum::serve`)

````rust
let mut server = axum_server::bind_rustls(addr, config);
````

服务运行在 `https://localhost:3000`,WebSocket 地址 `wss://localhost:3000/ws`。

### 启用 HTTP/2 CONNECT protocol(本章关键)

````rust
server.http_builder().http2().enable_connect_protocol();
````

**这是本章最关键的一行**。HTTP/2 WebSocket 使用扩展 CONNECT 机制,服务端必须声明支持,否则客户端无法按 HTTP/2 WebSocket 方式连接。源码注释:"This is required to advertise our support for HTTP/2 websockets to the client."

`axum::serve` 默认启用 CONNECT protocol;这里用 `axum_server` 需手动启用。

### `Version` extractor 打印 HTTP 版本

````rust
async fn ws_handler(
    ws: WebSocketUpgrade,
    version: Version,                              // 提取 HTTP 版本
    State(sender): State<broadcast::Sender<String>>,
) -> axum::response::Response {
    tracing::debug!("accepted a WebSocket using {version:?}");
````

`version: Version` 能告诉你连接是 HTTP/1.1 还是 HTTP/2,验证本章很有用。

### 连接内 `tokio::select!` 同时收发(不 split)

````rust
ws.on_upgrade(|mut ws| async move {
    loop {
        tokio::select! {
            res = ws.recv() => { /* 客户端消息 → broadcast */ }
            res = receiver.recv() => { /* broadcast → 发回客户端 */ }
        }
    }
})
````

没 split,直接在一个 loop 里用 `tokio::select!` 同时等 WebSocket 收客户端消息和 broadcast 收其他连接消息。和 42 章的 split + 两个 spawn 任务不同,这是另一种常见写法。

## 常见问题

**为什么用 `wss`?** HTTP/2 WebSocket 在浏览器里通常需 TLS,前端用 `wss://localhost:3000/ws`。

**示例证书不能直接用?** 自签名开发证书,可能已过期,浏览器会拦。真实本地测试重新生成。

**`enable_connect_protocol` 干什么?** 让服务端声明支持 HTTP/2 扩展 CONNECT,这是 HTTP/2 WebSocket 连接需要的能力。`axum::serve` 默认启用,`axum_server` 需手动开。

**和 chat 区别?** chat 重点聊天室业务和用户名管理;本章重点 HTTP/2 + TLS 下的 WebSocket 连接方式。

## 手写任务

1. `RustlsConfig::from_pem_file` 加载证书。
2. `axum_server::bind_rustls` 启动 HTTPS。
3. 加 `/ws` WebSocket 路由。
4. 启用 `server.http_builder().http2().enable_connect_protocol()`。
5. 前端用 `wss://localhost:3000/ws` 连接。
6. 用 `Version` extractor 打印 HTTP 版本。

## 小结

- WebSocket over HTTP/2 关键不是 handler 写法而是启动层配置:TLS(rustls) + HTTP/2 + `enable_connect_protocol()`。
- 用 `axum_server::bind_rustls` 而不是 `axum::serve`;`axum::serve` 默认启用 CONNECT protocol,`axum_server` 需手动开。
- 前端用 `wss://`(基于 TLS);自签名证书浏览器会拦,本地测试需重新生成并信任。
- 连接内可复用熟悉模式:recv 客户端消息 → broadcast → send 回客户端;可用 split + 两个任务(42 章)或 `tokio::select!` 同时收发(本章)。
- `Version` extractor 能拿到 HTTP 版本验证连接是 HTTP/1.1 还是 HTTP/2。

## 源码对照

- `examples/websockets-http2/Cargo.toml`
- `examples/websockets-http2/src/main.rs`
- `examples/websockets-http2/assets/index.html`
- `examples/websockets-http2/assets/script.js`
- `examples/websockets-http2/self_signed_certs/cert.pem`
- `examples/websockets-http2/self_signed_certs/key.pem`
