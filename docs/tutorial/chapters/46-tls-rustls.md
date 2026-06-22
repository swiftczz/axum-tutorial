# 46. tls-rustls

对应示例：`examples/tls-rustls`

HTTPS = HTTP over TLS。前面章节都用 HTTP，这章用 `axum-server` + rustls 启动 HTTPS server，并提供一个 HTTP→HTTPS 重定向服务（典型的生产配置）。

rustls 是 Rust 实现的 TLS 库（不带 C 依赖、内存安全），`axum-server` 把它和 hyper 包装成一行启动 HTTPS server。

分 3 步：先用 `axum_server::bind_rustls` 启动 HTTPS server，再加 HTTP 重定向到 HTTPS，最后说明证书怎么准备。

相比前面章节新引入：**`axum-server` crate、`RustlsConfig::from_pem_file`、`bind_rustls`、`Redirect::permanent`、`HandlerWithoutStateExt::into_make_service`**。

## Cargo.toml

````toml
[package]
name = "example-tls-rustls"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
axum-extra = "0.12"
axum-server = { version = "0.8", features = ["tls-rustls"] }
tokio = { version = "1", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

本章相比前面章节新增 `axum-server`（启用 `tls-rustls` feature）。`axum::serve` 只支持明文 HTTP，要 HTTPS 必须用 `axum-server::bind_rustls` 一行启动（high-level API）。对比第 53 章手动 hyper + rustls（low-level）。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：HTTPS server 骨架

`axum-server::bind_rustls` 是启动 HTTPS server 的高层 API——比第 53 章手动 hyper+rustls 简单得多。需要一份证书（PEM 格式 cert + key）。

````rust
use axum::{routing::get, Router};
use axum_server::tls_rustls::RustlsConfig;
use std::{net::SocketAddr, path::PathBuf};
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

    // 配置证书（cert + key）
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

    // 启动 HTTPS server
    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {}", addr);
    axum_server::bind_rustls(addr, config)
        .serve(app.into_make_service())
        .await
        .unwrap();
}

async fn handler() -> &'static str {
    "Hello, World!"
}
````

验证（用 examples 自带的自签名证书）：

````bash
cd examples
cargo run -p example-tls-rustls
````

````bash
curl -k https://localhost:3000/
# 返回 "Hello, World!"
````

`-k` 忽略自签名证书警告（生产用受信任 CA 签发的证书就不需要 `-k`）。

> **新面孔：`axum-server` crate**
>
> axum 生态的 server 启动库，支持 TCP 和 TLS（rustls/native-tls/openssl），比 axum 内置的 `axum::serve` 多了 TLS 支持。
>
> `axum::serve` 只支持 TCP（HTTP），不支持 HTTPS。要 HTTPS 必须用 `axum-server` 或手动 hyper（ch53）。

> **新面孔：`RustlsConfig::from_pem_file`**
>
> 从 PEM 文件加载证书和私钥。两个参数：
> - `cert.pem`：证书链（服务器证书 + 中间证书），公开的，给客户端验证身份
> - `key.pem`：私钥，保密的，用于 TLS 握手时解密
>
> 返回 `RustlsConfig`，传给 `bind_rustls`。

> **新面孔：`bind_rustls(addr, config)`**
>
> 启动 HTTPS server 的高层 API：
>
> ```text
> axum_server::bind_rustls(addr, config)
>     .serve(app.into_make_service())
>     .await
> ```
>
> 内部就是 ch53 的 hyper+rustls 流程，但全包装好了。

### 自签名证书（开发用）

生产证书要从 CA（Let's Encrypt 等）申请。开发用 `openssl` 生成自签名证书：

````bash
mkdir -p self_signed_certs
openssl req -x509 -newkey rsa:4096 -keyout self_signed_certs/key.pem \
    -out self_signed_certs/cert.pem -days 365 -nodes \
    -subj "/CN=localhost"
````

examples 仓库已自带 `self_signed_certs/{cert.pem, key.pem}`，**不能用于生产**。

---

## 第二步：HTTP→HTTPS 重定向

只开 HTTPS 的话，用户输入 `http://example.com` 会失败。生产常见做法：开一个额外的 HTTP 端口（80），把所有请求**永久重定向**到 HTTPS。

````rust
use axum::{
    handler::HandlerWithoutStateExt,
    http::{uri::Authority, StatusCode, Uri},
    response::Redirect,
    BoxError,
};

#[derive(Clone, Copy)]
struct Ports {
    http: u16,
    https: u16,
}

async fn redirect_http_to_https(ports: Ports) {
    // 把 URI 从 http://localhost:7878/path 改成 https://localhost:3000/path
    fn make_https(uri: Uri, https_port: u16) -> Result<Uri, BoxError> {
        let mut parts = uri.into_parts();
        parts.scheme = Some(axum::http::uri::Scheme::HTTPS);
        parts.authority = Some(format!("localhost:{https_port}").parse()?);
        if parts.path_and_query.is_none() {
            parts.path_and_query = Some("/".parse().unwrap());
        }
        Ok(Uri::from_parts(parts)?)
    }

    let redirect = move |uri: Uri| async move {
        match make_https(uri, ports.https) {
            Ok(uri) => Ok(Redirect::permanent(uri.to_string())),
            Err(error) => {
                tracing::warn!(%error, "failed to convert URI to HTTPS");
                Err(StatusCode::BAD_REQUEST)
            }
        }
    };

    // 启动 HTTP server（端口 7878），所有请求重定向到 HTTPS（端口 3000）
    let addr = SocketAddr::from(([127, 0, 0, 1], ports.http));
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, redirect.into_make_service()).await;
}
````

> **新面孔：`Redirect::permanent`**
>
> 返回 301 永久重定向响应。`Redirect::permanent(uri)` 等价于 `HTTP/1.1 301 Moved Permanently\nLocation: <uri>`。
>
> 浏览器/搜索引擎收到 301 会缓存这个重定向——以后直接访问 HTTPS。临时重定向用 `Redirect::temporary`（307）。

> **新面孔：`Uri::into_parts` / `from_parts`**
>
> URI 是结构化对象（scheme、authority、path、query 等部分），`into_parts()` 拆开，改完 `from_parts()` 拼回。这章把 scheme 从 `http` 改成 `https`，authority 改成新端口，path 保留。

> **新面孔：`HandlerWithoutStateExt::into_make_service`**
>
> `redirect` 是个闭包 handler，`axum::serve` 要 MakeService（service 工厂）。`into_make_service()` 把单个 handler 转成工厂——等价于 `Router::new().route("/", get(redirect)).into_make_service()`，但更简洁。
>
> 第 47 章已见过它（同样的 redirect 服务）。这里没有 `with_state` 所以用 `HandlerWithoutStateExt`（不带 state 版）。

---

## 第三步：两个服务并行跑

最终用 `tokio::spawn` + `tokio::main` 同时跑两个 server：HTTPS 主服务在主 task，HTTP 重定向在 spawn 出来的后台 task。

````rust
#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(/* tracing 初始化 */)
        .with(tracing_subscriber::fmt::layer())
        .init();

    let ports = Ports { http: 7878, https: 3000 };

    // 后台跑 HTTP→HTTPS 重定向
    tokio::spawn(redirect_http_to_https(ports));

    // 主 task 跑 HTTPS server
    let config = RustlsConfig::from_pem_file(/* ... */).await.unwrap();
    let app = Router::new().route("/", get(handler));
    let addr = SocketAddr::from(([127, 0, 0, 1], ports.https));
    axum_server::bind_rustls(addr, config)
        .serve(app.into_make_service())
        .await
        .unwrap();
}
````

为什么 spawn 重定向服务、HTTPS 留主 task？axum 的 `serve` 是无限循环不会返回。两个无限循环要并行必须 spawn 一个。让 HTTPS 在主 task 因为它更重要——主 task 退出整个程序退出。

> **新面孔：两个 server 并行（`tokio::spawn`）**
>
> 两个 server 都是无限循环。用 `tokio::spawn(redirect_server)` 把它扔后台，主 task 跑 HTTPS。这样两个同时跑。
>
> 也可以用 `tokio::join!`（ch47 那样）——区别是 `spawn` 后台跑不阻塞主 task，`join!` 等两个都完成。这章主 server 在主 task 上 `await`，spawn 让重定向服务并发。

---

## 完整代码

````rust
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
    // optional: spawn a second server to redirect http requests to this server
    tokio::spawn(redirect_http_to_https(ports));

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

    let app = Router::new().route("/", get(handler));

    // run https server
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
    fn make_https(uri: Uri, https_port: u16) -> Result<Uri, BoxError> {
        let mut parts = uri.into_parts();

        parts.scheme = Some(axum::http::uri::Scheme::HTTPS);
        parts.authority = Some(format!("localhost:{https_port}").parse()?);

        if parts.path_and_query.is_none() {
            parts.path_and_query = Some("/".parse().unwrap());
        }

        Ok(Uri::from_parts(parts)?)
    }

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
    axum::serve(listener, redirect.into_make_service()).await;
}
````

## 运行

````bash
cd examples
cargo run -p example-tls-rustls
````

测试两个端口：

````bash
curl -k https://localhost:3000/         # HTTPS 直连
curl -i http://localhost:7878/some/path # HTTP 被 301 重定向到 HTTPS
````

第二个返回：

```text
HTTP/1.1 301 Moved Permanently
location: https://localhost:3000/some/path
```

## 解读

### HTTPS vs HTTP

| 维度 | HTTP | HTTPS |
| --- | --- | --- |
| 传输 | 明文 | TLS 加密 |
| 端口 | 80（默认） | 443（默认） |
| 证书 | 不需要 | 需要 CA 签发 |
| 性能 | 快 | 多 TLS 握手开销（HTTP/2 多路复用可补偿） |
| 现代特性 | 不支持 HSTS/HTTP2 push 等 | 支持 |

**生产环境必须 HTTPS**——浏览器对 HTTP 站点显示不安全警告，新 API（如 service worker、geolocation）只在 HTTPS 启用。

### rustls vs native-tls vs openssl

axum-server 支持三种 TLS backend：

| Backend | 特点 |
| --- | --- |
| rustls（这章） | Rust 实现，无 C 依赖，内存安全，编译简单 |
| native-tls（ch54） | 用系统 OpenSSL/SChannel，兼容性好 |
| openssl（ch55） | 直接链接 OpenSSL，功能最全 |

新项目优先 rustls。

## 常见问题

**自签名证书能用生产吗？** 不能。浏览器不信任会警告。生产用 Let's Encrypt（免费）或商业 CA 签发的证书。

**为什么需要 HTTP→HTTPS 重定向？** 用户常输 `http://` 或没输协议（浏览器默认 http）。不重定向用户会看到连接失败。301 永久重定向让浏览器记住以后直接走 HTTPS。

**`axum::serve` 能跑 HTTPS 吗？** 不能。`axum::serve` 只支持 TCP（明文 HTTP）。HTTPS 必须用 `axum-server::bind_rustls`（这章）或手动 hyper+rustls（ch53）。

## 手写任务

1. 把自签名证书换成 Let's Encrypt 签发的真证书，用浏览器访问验证（不需 `-k`）。
2. 加 HSTS 头：`Strict-Transport-Security: max-age=31536000`，强制浏览器以后只用 HTTPS。
3. 重定向服务改成只支持特定路径（如 `/health` 不重定向）。
4. 加 graceful shutdown（ch47）让两个服务都能优雅退出。

## 小结

这章用 3 步讲了 axum 的 HTTPS 启动：

1. **HTTPS server**：`RustlsConfig::from_pem_file` 加载证书 + `axum_server::bind_rustls(addr, config).serve(app.into_make_service())` 一行启动。
2. **HTTP→HTTPS 重定向**：`Redirect::permanent(https_uri)` + `into_make_service()`，所有 HTTP 请求 301 到 HTTPS。
3. **并行双服务**：`tokio::spawn` 后台跑重定向，主 task 跑 HTTPS。

核心：HTTPS 是生产标配，axum-server 是启动 HTTPS 最简单的方式（比 ch53 手动 hyper+rustls 简单）。HTTP→HTTPS 重定向解决用户漏输 `https://` 的体验问题。

## 源码对照

- `examples/tls-rustls/Cargo.toml`
- `examples/tls-rustls/src/main.rs`
- `examples/tls-rustls/self_signed_certs/cert.pem`
- `examples/tls-rustls/self_signed_certs/key.pem`
