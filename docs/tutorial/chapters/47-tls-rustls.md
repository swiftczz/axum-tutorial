# 47. tls-rustls

对应示例：`examples/tls-rustls`

本章目标：使用 `axum-server` 和 rustls 启动 HTTPS 服务，并理解如何额外启动一个 HTTP 服务把请求重定向到 HTTPS。

前面的服务大多是：

```text
http://127.0.0.1:3000
```

这一章变成：

```text
https://127.0.0.1:3000
```

## 这个小项目在做什么

应用启动两个服务：

```text
HTTPS 服务：127.0.0.1:3000
HTTP  跳转：127.0.0.1:7878
```

访问 HTTPS：

```text
https://localhost:3000/
-> Hello, World!
```

访问 HTTP：

```text
http://localhost:7878/
-> 301/308 类永久重定向到 https://localhost:3000/
```

## 先理解 TLS、HTTPS、rustls

HTTPS 可以理解成：

```text
HTTP + TLS
```

TLS 提供：

```text
加密传输
服务器身份验证
防止中间人篡改
```

`rustls` 是 Rust 生态里的 TLS 实现。  
本章通过 `axum-server` 的 `tls-rustls` feature 使用它。

### 为什么 HTTPS 需要证书？cert.pem 和 key.pem 是什么？

初学者看到这两个文件会困惑。简单解释：

- **`key.pem` 是私钥**：一段随机数，**必须保密**。用它来"签名"和"解密"通信。
- **`cert.pem` 是证书**：包含你的**公钥** + 你的身份信息（域名等）+ **CA 的签名**。可以公开。

证书的作用是**身份验证**：浏览器怎么知道 `https://example.com` 真的是 example.com，而不是中间人伪造的？答案靠的是**信任链**：

```text
你的证书  ←  由某个中间 CA 签名
中间 CA  ←  由某个根 CA 签名
根 CA    ←  浏览器/操作系统内置信任（预装了一批根证书）
```

浏览器收到你的证书后，沿着这条链验证签名，如果最终能追溯到它内置信任的根 CA，就认为你的证书可信。

本例用的是**自签名证书**——你自己给自己签的，不在浏览器内置信任链里，所以浏览器会警告"不安全"。生产环境必须用受信任 CA 签发的证书（如 Let's Encrypt 免费签发）。本地开发可以用 `mkcert` 生成被本机信任的本地证书。

### ALPN：TLS 里协商 HTTP/2

本章和第 48 章用 `axum_server::bind_rustls`，它在 TLS 握手时会通过 **ALPN**（Application-Layer Protocol Negotiation）自动协商协议。

为什么要协商？因为同一个 HTTPS 端口要同时支持 HTTP/1.1 和 HTTP/2。TLS 握手时，客户端在 ClientHello 里带上自己支持的协议列表（如 `["h2", "http/1.1"]`），服务端根据自己的配置选一个。`axum_server` 默认会广播 `h2` 和 `http/1.1`，所以同一个 HTTPS 端口能同时服务两种协议。

如果你用第 53 章的低层 rustls 接入，需要手动设置 `config.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1"]`。删掉这一行，浏览器走 HTTP/2 会失败。

> 关键：**HTTP/2 在实践中必须配合 TLS + ALPN**。明文 HTTP/2（h2c）虽然标准支持，但浏览器不支持，所以你在生产环境看到的 HTTP/2 基本都是 over TLS 的。

## 文件和依赖

这个 example 有四类文件：

1. `examples/tls-rustls/Cargo.toml`：声明 axum-server tls-rustls。
2. `examples/tls-rustls/src/main.rs`：加载证书、启动 HTTPS、启动 HTTP 重定向。
3. `examples/tls-rustls/self_signed_certs/cert.pem`：示例证书。
4. `examples/tls-rustls/self_signed_certs/key.pem`：示例私钥。

关键依赖：

- `axum-server`：提供 `bind_rustls`。
- `axum`：提供 Router、Redirect、handler 转 service。
- `tokio`：异步运行时。

## 第一步：定义端口

源码：

````rust
#[derive(Clone, Copy)]
struct Ports {
    http: u16,
    https: u16,
}

let ports = Ports {
    http: 7878,
    https: 3000,
};
````

把端口放进结构体，方便 HTTP 重定向服务知道 HTTPS 端口是多少。

## 第二步：启动 HTTP 到 HTTPS 重定向服务

源码：

````rust
tokio::spawn(redirect_http_to_https(ports));
````

这是一个后台任务。  
主任务继续启动 HTTPS 服务。

这样程序同时监听：

```text
7878 HTTP
3000 HTTPS
```

## 第三步：加载证书和私钥

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

HTTPS 服务需要：

```text
证书 cert.pem
私钥 key.pem
```

示例使用自签名证书，只适合教学和本地调试。  
浏览器会提示不受信任，真实项目必须使用正式证书或受信任的本地开发证书。

## 第四步：启动 HTTPS 服务

源码：

````rust
let app = Router::new().route("/", get(handler));

let addr = SocketAddr::from(([127, 0, 0, 1], ports.https));
axum_server::bind_rustls(addr, config)
    .serve(app.into_make_service())
    .await
    .unwrap();
````

和普通 Axum 的区别是：

```text
普通 HTTP：axum::serve(listener, app)
HTTPS：axum_server::bind_rustls(addr, config).serve(...)
```

handler 本身不变：

````rust
async fn handler() -> &'static str {
    "Hello, World!"
}
````

TLS 是传输层能力，不应该污染业务 handler。

## 第五步：构造 HTTPS URI

重定向服务里的核心函数：

````rust
fn make_https(uri: Uri, https_port: u16) -> Result<Uri, BoxError> {
    let mut parts = uri.into_parts();

    parts.scheme = Some(axum::http::uri::Scheme::HTTPS);
    parts.authority = Some(format!("localhost:{https_port}").parse()?);

    if parts.path_and_query.is_none() {
        parts.path_and_query = Some("/".parse().unwrap());
    }

    Ok(Uri::from_parts(parts)?)
}
````

它把原始 HTTP URI 改成 HTTPS URI：

```text
scheme -> https
authority -> localhost:3000
path/query -> 保留原路径和 query
```

例如：

```text
http://localhost:7878/foo?a=1
```

会变成：

```text
https://localhost:3000/foo?a=1
```

## 第六步：返回永久重定向

源码：

````rust
let redirect = move |uri: Uri| async move {
    match make_https(uri, ports.https) {
        Ok(uri) => Ok(Redirect::permanent(uri.to_string())),
        Err(error) => {
            tracing::warn!(%error, "failed to convert URI to HTTPS");
            Err(StatusCode::BAD_REQUEST)
        }
    }
};
````

`Redirect::permanent` 返回永久重定向。  
如果 URI 构造失败，就返回 400。

## 第七步：handler 转 service

源码：

````rust
axum::serve(listener, redirect.into_make_service()).await;
````

`redirect` 是一个 handler closure。  
`into_make_service()` 把它变成可以交给 `axum::serve` 的 service。

这个 HTTP 服务不需要 Router，因为它只做一件事：

```text
所有请求都重定向到 HTTPS
```

## 函数职责速查

- `main`：初始化日志，定义端口，启动 HTTP 重定向服务，加载 TLS 配置，启动 HTTPS 服务。
- `handler`：返回 `Hello, World!`。
- `redirect_http_to_https`：启动 HTTP 服务，把请求重定向到 HTTPS。
- `make_https`：把原始 URI 改写成 HTTPS URI。


## 带中文注释的手写版

完整 `src/main.rs`：

````rust
//! Rustls HTTPS 示例。
//!
//! Run with
//!
//! ```not_rust
//! cargo run -p example-tls-rustls
//! ```

#![allow(unused_imports)]

use axum::{
    handler::HandlerWithoutStateExt,
    http::{uri::Authority, StatusCode, Uri},
    response::Redirect,
    routing::get,
    BoxError, Router,
};
use axum_server::tls_rustls::RustlsConfig;
use std::{net::SocketAddr, path::PathBuf};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[allow(dead_code)]
#[derive(Clone, Copy)]
struct Ports {
    http: u16,
    https: u16,
}

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

    let ports = Ports {
        http: 7878,
        https: 3000,
    };

    // 可选：启动一个 HTTP 服务，把 HTTP 请求重定向到 HTTPS。
    tokio::spawn(redirect_http_to_https(ports));

    // 加载 HTTPS 证书和私钥。
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

    let app = Router::new().route("/", get(handler));

    // 启动 HTTPS 服务。
    let addr = SocketAddr::from(([127, 0, 0, 1], ports.https));
    tracing::debug!("listening on {}", addr);
    axum_server::bind_rustls(addr, config)
        .serve(app.into_make_service())
        .await
        .unwrap();
}

#[allow(dead_code)]
async fn handler() -> &'static str {
    "Hello, World!"
}

#[allow(dead_code)]
async fn redirect_http_to_https(ports: Ports) {
    // 把原始 URI 改写成 HTTPS URI。
    fn make_https(uri: Uri, https_port: u16) -> Result<Uri, BoxError> {
        let mut parts = uri.into_parts();

        parts.scheme = Some(axum::http::uri::Scheme::HTTPS);
        parts.authority = Some(format!("localhost:{https_port}").parse()?);

        if parts.path_and_query.is_none() {
            parts.path_and_query = Some("/".parse().unwrap());
        }

        Ok(Uri::from_parts(parts)?)
    }

    // 所有 HTTP 请求都进入这个 handler，然后永久重定向到 HTTPS。
    let redirect = move |uri: Uri| async move {
        match make_https(uri, ports.https) {
            Ok(uri) => Ok(Redirect::permanent(uri.to_string())),
            Err(error) => {
                tracing::warn!(%error, "failed to convert URI to HTTPS");
                Err(StatusCode::BAD_REQUEST)
            }
        }
    };

    let addr = SocketAddr::from(([127, 0, 0, 1], ports.http));
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, redirect.into_make_service())
        .await
        .unwrap();
}
````

## 运行和验证

运行：

````bash
cargo run -p example-tls-rustls
````

访问 HTTPS：

````bash
curl -k https://localhost:3000/
````

`-k` 表示允许不受信任的自签名证书。

验证 HTTP 重定向：

````bash
curl -i http://localhost:7878/
````

应该看到 `Location` 指向：

```text
https://localhost:3000/
```

## 常见卡点

### 1. 浏览器为什么提示不安全？

因为示例用的是自签名证书。  
浏览器不信任它是正常的。

### 2. 生产环境能用仓库里的 key.pem 吗？

不能。  
私钥必须由你自己生成并安全保存，生产证书应来自受信任 CA 或你的基础设施。

### 3. handler 需要知道 HTTPS 吗？

不需要。  
TLS 在服务启动层处理，业务 handler 仍然返回普通响应。

## 手写任务

1. 准备 cert.pem 和 key.pem。
2. 用 `RustlsConfig::from_pem_file` 加载。
3. 用 `axum_server::bind_rustls` 启动 HTTPS。
4. 再启动一个 HTTP 服务。
5. 把 HTTP URI 改写成 HTTPS URI。
6. 返回 `Redirect::permanent`。

## 本章真正要记住什么

TLS 接入 Axum 的核心是：

```text
证书 + 私钥 -> RustlsConfig
RustlsConfig -> axum_server::bind_rustls
业务 Router 不需要改
```

HTTP 到 HTTPS 重定向可以作为一个独立的小服务运行。

## 源码对照

本章手写版对应源码：

- `examples/tls-rustls/src/main.rs`
- `examples/tls-rustls/self_signed_certs/cert.pem`
- `examples/tls-rustls/self_signed_certs/key.pem`
- `examples/tls-rustls/Cargo.toml`
