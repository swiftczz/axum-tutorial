# 46. tls-rustls

对应示例：`examples/tls-rustls`

用 `axum-server` 和 rustls 启动 HTTPS 服务,并启动一个 HTTP 服务把请求重定向到 HTTPS。理解 TLS、HTTPS、证书、ALPN。



相比前面章节新引入：**`RustlsConfig::from_pem_file`、`axum_server::bind_rustls`、HTTP→HTTPS 重定向**。

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

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 关键概念

> **新面孔：`RustlsConfig::from_pem_file` + `axum_server::bind_rustls`**
>
> 读证书/私钥启动 HTTPS。HTTP→HTTPS 重定向用 `Redirect::permanent`。


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

    let ports = Ports { http: 7878, https: 3000 };

    tokio::spawn(redirect_http_to_https(ports));

    let config = RustlsConfig::from_pem_file(
        PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("self_signed_certs").join("cert.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("self_signed_certs").join("key.pem"),
    )
    .await
    .unwrap();

    let app = Router::new().route("/", get(handler));

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
    axum::serve(listener, redirect.into_make_service())
        .await
        .unwrap();
}
````

## 运行

````bash
cd examples
cargo run -p example-tls-rustls
````

访问 HTTPS:

````bash
curl -k https://localhost:3000/      # -k 允许自签名证书
````

验证 HTTP 重定向:

````bash
curl -i http://localhost:7878/
# Location: https://localhost:3000/
````

## 解读

### TLS / HTTPS / rustls

```text
HTTPS = HTTP + TLS
TLS 提供:加密传输 + 服务器身份验证 + 防中间人篡改
rustls:Rust 生态的 TLS 实现,通过 axum-server 的 tls-rustls feature 使用
```

### 证书和私钥是什么

- **`key.pem` 是私钥**:一段随机数,**必须保密**,用来签名和解密通信。
- **`cert.pem` 是证书**:含**公钥** + 身份信息(域名等)+ **CA 签名**,可以公开。

证书的作用是**身份验证**——浏览器怎么知道 `https://example.com` 真的是 example.com?靠**信任链**:

```text
你的证书  ←  中间 CA 签名
中间 CA  ←  根 CA 签名
根 CA    ←  浏览器/操作系统内置信任(预装一批根证书)
```

浏览器沿链验证签名,追溯到内置信任的根 CA 就认为可信。本例用**自签名证书**——自己给自己签,不在浏览器信任链里所以警告"不安全"。生产必须用受信任 CA 签发(Let's Encrypt 免费签);本地开发用 `mkcert` 生成被本机信任的本地证书。

### ALPN:TLS 里协商 HTTP/2

本章和第 47 章用 `axum_server::bind_rustls`,TLS 握手时通过 **ALPN**(Application-Layer Protocol Negotiation)自动协商协议。为什么协商?同一 HTTPS 端口要同时支持 HTTP/1.1 和 HTTP/2——TLS 握手时客户端在 ClientHello 带支持的协议列表(如 `["h2","http/1.1"]`),服务端根据配置选一个。`axum_server` 默认广播 `h2` 和 `http/1.1`,同端口能同时服务两种协议。

用第 53 章低层 rustls 接入时需手动 `config.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1"]`,删掉这行浏览器走 HTTP/2 会失败。**关键:HTTP/2 实践中必须配合 TLS + ALPN**,明文 HTTP/2(h2c)浏览器不支持。

### 启动两个服务

```text
HTTPS 服务 127.0.0.1:3000(主)
HTTP 跳转  127.0.0.1:7878(把请求重定向到 HTTPS)
```

````rust
tokio::spawn(redirect_http_to_https(ports));   // 后台跑 HTTP 重定向
// 主任务启动 HTTPS
axum_server::bind_rustls(addr, config).serve(app.into_make_service()).await.unwrap();
````

### HTTPS 启动差异

```text
普通 HTTP:axum::serve(listener, app)
HTTPS:    axum_server::bind_rustls(addr, config).serve(app.into_make_service())
```

**handler 本身不变**——TLS 是传输层能力,不应该污染业务 handler。

### HTTP → HTTPS 重定向

````rust
fn make_https(uri: Uri, https_port: u16) -> Result<Uri, BoxError> {
    let mut parts = uri.into_parts();
    parts.scheme = Some(Scheme::HTTPS);
    parts.authority = Some(format!("localhost:{https_port}").parse()?);
    if parts.path_and_query.is_none() { parts.path_and_query = Some("/".parse().unwrap()); }
    Ok(Uri::from_parts(parts)?)
}

let redirect = move |uri: Uri| async move {
    match make_https(uri, ports.https) {
        Ok(uri) => Ok(Redirect::permanent(uri.to_string())),
        Err(error) => Err(StatusCode::BAD_REQUEST),
    }
};

axum::serve(listener, redirect.into_make_service()).await;
````

`make_https` 把 HTTP URI 改成 HTTPS URI:`http://localhost:7878/foo?a=1` → `https://localhost:3000/foo?a=1`(scheme→https、authority→localhost:3000、path/query 保留)。`Redirect::permanent` 返回永久重定向。

**handler 转 service**:`redirect.into_make_service()` 把 handler closure 变成可交给 `axum::serve` 的 service(见第 30 章 handler/service 互转)。这个 HTTP 服务不需 Router,只做"所有请求重定向到 HTTPS"一件事。

## 常见问题

**浏览器为什么提示不安全?** 示例用自签名证书,不在浏览器信任链里。

**生产能用仓库里的 key.pem 吗?** 不能。私钥必须自己生成并保密,生产证书来自受信任 CA 或基础设施。

**handler 需要知道 HTTPS 吗?** 不需要。TLS 在服务启动层处理,业务 handler 返普通响应。

## 手写任务

1. 准备 cert.pem 和 key.pem。
2. `RustlsConfig::from_pem_file` 加载。
3. `axum_server::bind_rustls` 启动 HTTPS。
4. 再启动一个 HTTP 服务。
5. HTTP URI 改写成 HTTPS URI。
6. 返回 `Redirect::permanent`。

## 小结

- TLS 接入 axum 核心:证书+私钥 → `RustlsConfig` → `axum_server::bind_rustls`;业务 Router 不需改,handler 不感知 TLS。
- `key.pem` 是私钥(保密),`cert.pem` 是证书(公钥+身份+CA 签名);浏览器沿信任链验证到内置根 CA 才认为可信。生产用 Let's Encrypt 等受信任 CA,本地用 mkcert。
- **HTTP/2 实践必须 TLS + ALPN**:TLS 握手协商 h2/http1.1,同 HTTPS 端口服务两种协议;`axum_server` 默认广播,低层 rustls 需手动设 `alpn_protocols`。
- HTTP → HTTPS 重定向可作独立小服务(handler closure + `into_make_service` + `Redirect::permanent`)。

## 源码对照

- `examples/tls-rustls/Cargo.toml`
- `examples/tls-rustls/src/main.rs`
- `examples/tls-rustls/self_signed_certs/cert.pem`
- `examples/tls-rustls/self_signed_certs/key.pem`
