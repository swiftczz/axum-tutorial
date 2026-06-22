# 43. websockets-http2

对应示例：`examples/websockets-http2`

本章目标：理解 WebSocket over HTTP/2 的额外配置，学会用 `axum-server` + rustls 启动 HTTPS 服务，并启用 HTTP/2 CONNECT protocol。

上一章是普通 WebSocket。  
这一章重点不是聊天业务，而是：

```text
HTTPS
HTTP/2
WebSocket over HTTP/2
enable_connect_protocol
```

## 这个小项目在做什么

应用提供：

```text
GET /   -> 静态页面
ANY /ws -> WebSocket 入口
```

前端脚本连接：

````javascript
const socket = new WebSocket('wss://localhost:3000/ws');
````

注意是：

```text
wss://
```

也就是基于 TLS 的 WebSocket。

服务端用自签名证书启动 HTTPS，并启用 HTTP/2 WebSocket 所需的 CONNECT protocol。

## 先理解它和普通 WebSocket 的差异

普通示例里：

```text
ws://localhost:3000/ws
axum::serve(listener, app)
```

本章示例里：

```text
wss://localhost:3000/ws
axum_server::bind_rustls(addr, config)
server.http_builder().http2().enable_connect_protocol()
```

多出来的核心点是：

```text
TLS 配置
HTTP/2 CONNECT protocol
```

源码注释也强调：

```text
如果使用 axum::serve，它默认启用；这里用 axum_server，需要手动启用。
```

## 文件和依赖

这个 example 有五类文件：

1. `examples/websockets-http2/Cargo.toml`：声明 Axum ws/http2、axum-server rustls。
2. `examples/websockets-http2/src/main.rs`：HTTPS + HTTP/2 WebSocket 服务端。
3. `examples/websockets-http2/assets/index.html`：简单消息页面。
4. `examples/websockets-http2/assets/script.js`：浏览器 WebSocket 客户端。
5. `examples/websockets-http2/self_signed_certs/*`：示例自签名证书和私钥。

关键依赖：

- `axum`：启用 `ws` 和 `http2` feature。
- `axum-server`：用 rustls 启动 TLS 服务。
- `tokio::sync::broadcast`：把消息广播给多个连接。
- `tower-http`：托管静态文件。

## 第一步：加载自签名证书

源码：

````rust
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
````

`RustlsConfig::from_pem_file` 读取：

```text
cert.pem
key.pem
```

用于启动 HTTPS。

注意：仓库里的证书是教学用自签名证书，而且可能已经过期。  
浏览器验证时可能会拦截或拒绝。真实本地测试通常需要重新生成开发证书，并让浏览器信任它。

## 第二步：Router 和广播状态

源码：

````rust
let app = Router::new()
    .fallback_service(ServeDir::new(assets_dir).append_index_html_on_directories(true))
    .route("/ws", any(ws_handler))
    .with_state(broadcast::channel::<String>(16).0);
````

这个示例比上一章聊天室更短。  
它只保存一个广播发送端：

````rust
broadcast::Sender<String>
````

每个 WebSocket 连接进入后都会：

```text
subscribe 一个 receiver
收到客户端消息后 broadcast
收到 broadcast 消息后发给当前客户端
```

## 第三步：用 axum_server 启动 TLS

源码：

````rust
let mut server = axum_server::bind_rustls(addr, config);
````

这里没有用：

````rust
axum::serve(listener, app)
````

而是用 `axum_server` 绑定地址并加载 rustls 配置。

这样服务运行在：

```text
https://localhost:3000
```

WebSocket 地址就是：

```text
wss://localhost:3000/ws
```

## 第四步：启用 HTTP/2 CONNECT protocol

源码：

````rust
server.http_builder().http2().enable_connect_protocol();
````

这是本章最关键的一行。

HTTP/2 WebSocket 使用的是扩展 CONNECT 机制。  
如果服务端没有广告自己支持它，客户端就无法按 HTTP/2 WebSocket 方式连接。

源码注释说得很直接：

```text
This is required to advertise our support for HTTP/2 websockets to the client.
```

## 第五步：handler 里记录 HTTP 版本

源码：

````rust
async fn ws_handler(
    ws: WebSocketUpgrade,
    version: Version,
    State(sender): State<broadcast::Sender<String>>,
) -> axum::response::Response {
    tracing::debug!("accepted a WebSocket using {version:?}");
    ...
}
````

`version: Version` 可以提取 HTTP 版本。  
日志会告诉你连接是 HTTP/1.1 还是 HTTP/2。

这对验证本章很有用。

## 第六步：连接内用 select 同时收发

源码：

````rust
let mut receiver = sender.subscribe();
ws.on_upgrade(|mut ws| async move {
    loop {
        tokio::select! {
            res = ws.recv() => { ... }
            res = receiver.recv() => { ... }
        }
    }
})
````

这里没有 split，而是直接在一个 loop 里用 `tokio::select!` 同时等待：

```text
WebSocket 收到客户端消息
broadcast 收到其他连接消息
```

如果收到客户端文本：

````rust
let _ = sender.send(s.to_string());
````

如果收到广播：

````rust
ws.send(ws::Message::Text(msg.into())).await
````

这也是一种常见写法。

## 函数职责速查

- `main`：初始化日志，加载 TLS 证书，创建 Router，启用 HTTP/2 CONNECT，启动 HTTPS 服务。
- `ws_handler`：完成 WebSocket upgrade，订阅广播通道，并在连接内双向转发消息。


## 带中文注释的手写版

完整 `src/main.rs`：

````rust
//! WebSocket over HTTP/2 示例。
//!
//! Run with
//!
//! ```not_rust
//! cargo run -p example-websockets-http2
//! ```

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
    // 初始化日志。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 静态文件目录。
    let assets_dir = PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("assets");

    // 加载 HTTPS 证书和私钥。示例证书是自签名证书，只适合本地教学。
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

    // 用 broadcast channel 做最小聊天室广播状态。
    let app = Router::new()
        .fallback_service(ServeDir::new(assets_dir).append_index_html_on_directories(true))
        .route("/ws", any(ws_handler))
        .with_state(broadcast::channel::<String>(16).0);

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {}", addr);

    // 用 axum-server 启动 rustls HTTPS 服务。
    let mut server = axum_server::bind_rustls(addr, config);

    // HTTP/2 WebSocket 必须启用 CONNECT protocol。
    // 如果使用 axum::serve，它默认启用；这里手写 axum_server，所以显式开启。
    server.http_builder().http2().enable_connect_protocol();

    server.serve(app.into_make_service()).await.unwrap();
}

async fn ws_handler(
    ws: WebSocketUpgrade,
    version: Version,
    State(sender): State<broadcast::Sender<String>>,
) -> axum::response::Response {
    tracing::debug!("accepted a WebSocket using {version:?}");

    // 当前连接订阅广播通道。
    let mut receiver = sender.subscribe();

    ws.on_upgrade(|mut ws| async move {
        loop {
            tokio::select! {
                // 从当前 WebSocket 连接收客户端消息。
                res = ws.recv() => {
                    match res {
                        Some(Ok(ws::Message::Text(s))) => {
                            // 文本消息进入广播通道。
                            let _ = sender.send(s.to_string());
                        }
                        Some(Ok(_)) => {}
                        Some(Err(e)) => tracing::debug!("client disconnected abruptly: {e}"),
                        None => break,
                    }
                }
                // 从广播通道收消息，再发送给当前 WebSocket 客户端。
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

完整 `assets/index.html`：

````html
<p>Open this page in two windows and try sending some messages!</p>
<form action="javascript:void(0)">
    <input type="text" name="content" required>
    <button>Send</button>
</form>
<div id="messages"></div>
<script src='script.js'></script>
````

完整 `assets/script.js`：

````javascript
const socket = new WebSocket('wss://localhost:3000/ws');

socket.addEventListener('message', e => {
    document.getElementById("messages").append(e.data, document.createElement("br"));
});

const form = document.querySelector("form");
form.addEventListener("submit", () => {
    socket.send(form.elements.namedItem("content").value);
    form.elements.namedItem("content").value = "";
});
````

## 运行和验证

运行：

````bash
cargo run -p example-websockets-http2
````

浏览器打开：

```text
https://localhost:3000/
```

如果浏览器提示证书不可信，需要手动信任本地自签名证书，或重新生成有效的本地开发证书。

打开两个窗口，互相发送消息。  
终端日志中关注：

```text
accepted a WebSocket using ...
```

## 常见卡点

### 1. 为什么要 wss？

HTTP/2 WebSocket 在浏览器里通常需要 TLS。  
所以前端用 `wss://localhost:3000/ws`。

### 2. 为什么示例证书可能不能直接用？

它是自签名开发证书，并且可能已经过期。  
真实本地测试建议重新生成证书。

### 3. enable_connect_protocol 是干什么的？

它让服务端声明支持 HTTP/2 扩展 CONNECT。  
这是 HTTP/2 WebSocket 连接需要的能力。

### 4. 这一章和 chat 有什么区别？

chat 重点是聊天室业务和用户名管理。  
本章重点是 HTTP/2 + TLS 下的 WebSocket 连接方式。

## 手写任务

1. 用 `RustlsConfig::from_pem_file` 加载证书。
2. 用 `axum_server::bind_rustls` 启动 HTTPS。
3. 加 `/ws` WebSocket 路由。
4. 启用 `server.http_builder().http2().enable_connect_protocol()`。
5. 前端用 `wss://localhost:3000/ws` 连接。
6. 用 `Version` extractor 打印 HTTP 版本。

## 本章真正要记住什么

WebSocket over HTTP/2 的关键不是 handler 写法，而是启动层配置：

```text
TLS
HTTP/2
enable_connect_protocol
```

连接内仍然可以复用熟悉的 WebSocket 模式：

```text
recv 客户端消息
broadcast 给所有订阅者
send 消息回当前客户端
```

## 源码对照

本章手写版对应源码：

- `examples/websockets-http2/src/main.rs`
- `examples/websockets-http2/assets/index.html`
- `examples/websockets-http2/assets/script.js`
- `examples/websockets-http2/self_signed_certs/cert.pem`
- `examples/websockets-http2/self_signed_certs/key.pem`
- `examples/websockets-http2/Cargo.toml`
