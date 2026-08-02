# 第 32 章 tls-rustls:给服务启用 HTTPS

## 本章你将做出什么

一个跑 HTTPS 的服务,外加一个**把 HTTP 请求重定向到 HTTPS** 的辅助服务:

```
https://localhost:3000/   正常服务
http://localhost:7878/    → 301 重定向到 https://localhost:3000/
```

[第 30 章](30-jwt.md)的 token 和[第 31 章](31-oauth.md)的 session cookie 都在头部明文传输——**HTTP 下任何中间人都能截获并冒用**。这一章补上加密这一环。

学完本章你将掌握:

- TLS 握手在做什么,证书解决的是什么问题
- 自签名证书怎么生成,以及浏览器为什么会警告
- **为什么 `axum::serve` 不支持 TLS,要换成 `axum_server`**
- HTTP → HTTPS 重定向的实现,以及为什么用 `301` 而非 `302`
- `HandlerWithoutStateExt::into_make_service`

## 前置知识

### TLS 解决三个问题

[第 00 章](00-web-basics.md#tcp端口与-socket)讲过 HTTP 建立在 TCP 之上,而 TCP 传的是明文字节。这意味着链路上任何一环——路由器、Wi-Fi 热点、ISP、代理——都能看到甚至改写内容。

**TLS**(Transport Layer Security,SSL 的继任者)在 TCP 和 HTTP 之间插了一层,解决三件事:

**机密性(加密)** —— 中间人看不到内容。你的 token、密码、用户数据都被加密。

**完整性(防篡改)** —— 中间人改了内容会被发现。没有 TLS 时,运营商往页面里插广告是常见操作。

**身份认证(证书)** —— **确认你连的确实是 `example.com` 而不是冒充者。** 这一点最容易被忽略,但没有它,前两点毫无意义:如果你加密后把数据发给了中间人,加密只是让别人看不到,中间人自己看得一清二楚。

**HTTPS 就是 HTTP over TLS**,默认端口 443。

### 证书与信任链

服务端的证书里有两样关键信息:**域名**和**公钥**,再加上某个**证书颁发机构**(CA)的签名。

浏览器验证时:

1. 检查证书里的域名和你访问的域名是否一致
2. 检查证书是否在有效期内
3. **用 CA 的公钥验证签名** —— 确认这张证书确实是 CA 签发的,不是伪造的
4. 如果签发 CA 不是根 CA,继续往上验证,直到某个**操作系统/浏览器内置信任的根 CA**

这条链就是**信任链**。你信任的其实是几十个内置根 CA,它们背书了中间 CA,中间 CA 背书了具体网站。

**自签名证书**跳过了 CA——自己给自己签。加密照常工作,但**没有第三方背书**,浏览器无法确认"这张证书对应的服务器就是你要访问的那个",所以会弹出醒目的警告。

**自签名证书的用途**:本地开发、内网服务、测试环境。**公网服务绝不能用**——用户会看到吓人的警告,而且这个警告一旦被训练成"点忽略就好",真正的中间人攻击也会被忽略掉。

公网服务应该用 [Let's Encrypt](https://letsencrypt.org/) 这类免费 CA,配合 certbot 自动签发和续期。

### 生成自签名证书

用 `openssl` 一条命令:

```bash
openssl req -x509 -newkey rsa:4096 \
  -keyout key.pem -out cert.pem \
  -days 365 -nodes \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

参数逐个说:

| 参数 | 含义 |
|---|---|
| `-x509` | 直接生成自签名证书,而非签发请求(CSR) |
| `-newkey rsa:4096` | 同时生成一把新的 4096 位 RSA 私钥 |
| `-keyout key.pem` | 私钥输出文件 |
| `-out cert.pem` | 证书输出文件 |
| `-days 365` | 有效期一年 |
| `-nodes` | **不给私钥加密码**(否则服务启动时要人工输入) |
| `-subj "/CN=localhost"` | 证书的通用名 |
| `-addext "subjectAltName=..."` | **主题备用名称(SAN)** |

**`subjectAltName` 是必需的,不是可选项。** 现代浏览器**只看 SAN,完全忽略 `CN`** —— 只写 `-subj "/CN=localhost"` 生成的证书,浏览器会报 `ERR_CERT_COMMON_NAME_INVALID`,即使域名对得上。这是很多人第一次生成证书就踩的坑。

**两个文件的区别很重要:**

- **`cert.pem`(证书)** —— 公开的,发给每个连接的客户端
- **`key.pem`(私钥)** —— **绝密**,泄露等于身份被完全冒用

**私钥绝不能进 git。** 项目里应该有 `.gitignore` 排除 `*.pem`,证书由部署流程提供。

> 本教程仓库的 `examples/tls-rustls/self_signed_certs/` 里有现成的 `cert.pem` 和 `key.pem`,那是 axum 官方为了让示例开箱即用而提交的。**这个做法只适合示例仓库**——真实项目里私钥进版本控制是严重的安全事故。

### rustls 与 native-tls

Rust 生态有两条 TLS 路线:

**`rustls`** —— 纯 Rust 实现。不依赖系统库,交叉编译简单,Docker 镜像小,内存安全由 Rust 保证。缺点是不支持某些老旧的加密套件(这通常是好事)。

**`native-tls`** —— 调用操作系统的 TLS 实现(Linux 上是 OpenSSL,macOS 是 Security.framework,Windows 是 SChannel)。兼容性好,能用系统的证书策略,但引入了系统依赖。

**默认选 `rustls`。** 前面章节反复提到"纯 Rust 实现省掉系统依赖"的好处——[第 27 章](27-diesel-async-postgres.md#diesel-async-改变了什么)的 diesel-async、[第 31 章](31-oauth.md#创建项目)的 reqwest 都是这个考虑。

[第 51 章](51-low-level-native-tls.md)会讲 native-tls 的用法,[第 50](50-low-level-rustls.md)、[52 章](52-low-level-openssl.md)则是更底层的手动控制。

### 为什么不能用 axum::serve

**这是本章在 axum 层面最重要的一点。**

前面 31 章一直用的 `axum::serve(listener, app)` **不支持 TLS**。它的文档写得很直白:

> This method of running a service is intentionally simple and doesn't support any configuration.

`axum::serve` 接受一个 `TcpListener`,里面流动的是明文字节。要加密就得在 TCP 之上、HTTP 之下插入 TLS 层——这需要对连接建立过程有更多控制,超出了 `axum::serve` 的设计范围。

**本章用 `axum-server`**,一个独立的 crate:

```rust
axum_server::bind_rustls(addr, config)
    .serve(app.into_make_service())
    .await
```

它替换掉 `axum::serve`,内部完成 TLS 握手,再把解密后的字节交给 axum 的 `Router`。

**注意 `axum-server` 不是 axum 官方维护的**,但它是社区里最常用的 TLS 方案,axum 官方示例也用它。

另外两条路:

- **反向代理终止 TLS** —— Nginx/Caddy/云负载均衡器处理 HTTPS,后端服务跑 HTTP。**这是生产环境最常见的做法**,「代价与边界」会展开。
- **手动用 hyper + tokio-rustls** —— 完全控制,见[第 50 章](50-low-level-rustls.md)。

### into_make_service:Router 与 MakeService

`axum_server` 的 `serve` 要求的不是 `Router`,而是一个 **`MakeService`**:

```rust
.serve(app.into_make_service())
```

区别在于:

- **`Service`** —— 处理**一个请求**(`Router` 就是,[第 21 章](21-testing.md#router-是一个-service)讲过)
- **`MakeService`** —— 每来**一个连接**就产出一个 `Service`

`axum::serve` 内部会自动做这个转换,所以前面章节没见过。`axum_server` 的 API 层次更低,需要显式调用。

本章的重定向服务用的是另一个变体:

```rust
axum::serve(listener, redirect.into_make_service()).await;
```

这里 `redirect` 是个**闭包 handler**,`into_make_service` 来自 `HandlerWithoutStateExt` —— 它把单个 handler 直接变成服务,**不需要 `Router`**。

适用于"整个服务只有一个处理逻辑、不区分路径"的场景,重定向正是如此:**不管访问什么路径,一律转到 HTTPS**。

### 重定向用 301 还是 302

本章用的是 `Redirect::permanent`,也就是 **`301 Moved Permanently`**。

| 状态码 | 含义 | 浏览器行为 |
|---|---|---|
| `301` | 永久移动 | **会缓存**,下次直接访问新地址 |
| `302`/`307` | 临时移动 | 不缓存,每次都先问原地址 |

HTTP → HTTPS 用 `301` 是对的:这是个**永久**的策略,而且被缓存意味着**用户下次直接走 HTTPS,连那次明文请求都省了**——少一次被中间人截获的机会。

但要注意 **`301` 的缓存很难撤销**。浏览器会长期记住,即使你后来改了策略。所以在还不确定要不要全站 HTTPS 时,先用 `302` 更安全。

**更彻底的方案是 HSTS**(HTTP Strict Transport Security)响应头:

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

它告诉浏览器"接下来一年内,访问本站一律用 HTTPS,不要发明文请求"。**连第一次的重定向都省了。** 本章示例没有实现它,「代价与边界」会提。

## 项目搭建

```bash
cargo new tls-rustls
cd tls-rustls
```

```bash
cargo add axum
cargo add axum-extra
cargo add axum-server --features tls-rustls
cargo add tokio --features full
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

### 生成证书

```bash
mkdir self_signed_certs
cd self_signed_certs

openssl req -x509 -newkey rsa:4096 \
  -keyout key.pem -out cert.pem \
  -days 365 -nodes \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

cd ..
```

检查生成的证书:

```bash
openssl x509 -in self_signed_certs/cert.pem -noout -text | grep -A1 'Subject Alternative Name'
```

```
X509v3 Subject Alternative Name:
    DNS:localhost, IP Address:127.0.0.1
```

**确认 SAN 存在** —— 前置知识强调过它是必需的。

顺手加上 `.gitignore`:

```bash
echo 'self_signed_certs/*.pem' >> .gitignore
```

## 分步构建

### 步骤 1:跑起 HTTPS 服务

**要做什么**:加载证书,用 `axum_server` 启动 HTTPS。

```rust
// src/main.rs
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
```

**`RustlsConfig::from_pem_file(cert, key)`** 加载证书和私钥。注意顺序:**证书在前,私钥在后**,写反了会报解析错误。

**`env!("CARGO_MANIFEST_DIR")`** 是编译期宏,展开成项目根目录的绝对路径。用它而不是相对路径,是因为**相对路径取决于运行时的工作目录** —— `cargo run` 时是项目根,但直接跑二进制时可能是别处。

> **更好的写法**:证书路径写死在代码里,而且绑定到 `CARGO_MANIFEST_DIR`(**编译时**的项目目录)。这意味着编译好的二进制拷到别的机器上就找不到证书了——路径指向的是构建机器上的目录。生产环境应该从环境变量或配置文件读路径(`std::env::var("TLS_CERT_PATH")`),或者用相对于可执行文件的路径。示例这么写是为了 `cargo run` 就能跑通。

**`axum_server::bind_rustls(addr, config)`** 替代了 `axum::serve`,前置知识讲过原因。

**`app.into_make_service()`** 把 `Router` 转成 `MakeService`。

**跑起来看**:

```bash
cargo run
```

```
DEBUG tls_rustls: listening on 127.0.0.1:3000
```

用 curl 访问——**注意是 `https://`**:

```bash
curl https://localhost:3000/
```

```
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html
```

**被拒了。** curl 默认会验证证书,而自签名证书没有 CA 背书——**这正是前置知识讲的"身份认证"在起作用**。

加 `-k`(或 `--insecure`)跳过验证:

```bash
curl -k https://localhost:3000/
```

```
Hello, World!
```

**通了。** 加密仍然生效,只是跳过了身份验证。

**看看握手过程:**

```bash
curl -kv https://localhost:3000/ 2>&1 | grep -E 'TLS|SSL|subject|issuer' | head -10
```

```
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
*  subject: CN=localhost
*  issuer: CN=localhost
```

**`subject` 和 `issuer` 都是 `CN=localhost`** —— 自己给自己签名,这就是"自签名"的字面含义。正规证书的 `issuer` 会是某个 CA。

**用 HTTP 访问 HTTPS 端口会怎样?**

```bash
curl -s http://localhost:3000/ | head -c 100
```

浏览器/curl 会收到一堆二进制乱码或直接报错——**服务端把明文 HTTP 请求当成了 TLS 握手数据**,解析失败。这也是为什么需要下一步那个重定向服务。

浏览器访问 `https://localhost:3000/` 会看到醒目的警告页("您的连接不是私密连接"),点"高级"→"继续前往"才能看到 `Hello, World!`。

### 步骤 2:HTTP 重定向到 HTTPS

**要做什么**:再起一个 HTTP 服务,把所有请求 `301` 到 HTTPS。

```rust
// src/main.rs
use axum::{
    handler::HandlerWithoutStateExt,                          // 新增
    http::{uri::Authority, StatusCode, Uri},                  // 新增
    response::Redirect,                                       // 新增
    routing::get,
    BoxError, Router,                                         // 改动:加上 BoxError
};
use axum_server::tls_rustls::RustlsConfig;
use std::{net::SocketAddr, path::PathBuf};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 新增
#[derive(Clone, Copy)]
struct Ports {
    http: u16,
    https: u16,
}

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    // 新增:两个端口
    let ports = Ports {
        http: 7878,
        https: 3000,
    };
    // optional: spawn a second server to redirect http requests to this server
    tokio::spawn(redirect_http_to_https(ports));

    // ...(证书加载不变)

    let app = Router::new().route("/", get(handler));

    // 改动:端口从 ports 取
    let addr = SocketAddr::from(([127, 0, 0, 1], ports.https));
    tracing::debug!("listening on {}", addr);
    axum_server::bind_rustls(addr, config)
        .serve(app.into_make_service())
        .await
        .unwrap();
}

// ...(handler 不变)

// 新增
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
    axum::serve(listener, redirect.into_make_service()).await.unwrap();
}
```

**`tokio::spawn(redirect_http_to_https(ports))`** 把重定向服务丢到后台任务,主任务继续跑 HTTPS 服务。[第 21 章](21-testing.md#什么时候仍需真实服务器)用过同样的手法。

注意这里**不需要 `join!`**([第 20 章](20-cors.md#步骤-1在一个进程里跑两个服务)那样),因为主任务的 `axum_server` 永不返回。**如果重定向服务崩了,主服务仍在跑但没人重定向了** —— 这是 `spawn` 的隐患,「代价与边界」会提。

**`make_https` 改写 URI 的三步:**

1. `parts.scheme = HTTPS` —— 协议从 `http` 换成 `https`
2. `parts.authority = "localhost:3000"` —— 主机和端口换成 HTTPS 服务的
3. `path_and_query` 为空时补一个 `/`

`Uri::into_parts()` / `from_parts()` 这个拆装模式,和[第 09 章](09-consume-body-in-extractor-or-middleware.md#步骤-2在中间件里也读一遍-body)对 `Request`、[第 18 章](18-print-request-response.md#请求与响应的对称性)对 `Response` 做的是同一件事。

> **更好的写法**:`parts.authority` 被硬编码成了 `localhost`,**丢弃了客户端原本请求的主机名**。这意味着用户访问 `http://example.com/page` 会被重定向到 `https://localhost:3000/page` —— 完全错误。正确做法是从请求的 `Host` 头部取出主机名,只替换端口:提取 `TypedHeader<headers::Host>`([第 30 章](30-jwt.md#typedheader类型化的头部)讲过),把其中的主机部分和 HTTPS 端口组合。示例硬编码是因为它只在本地跑,但这个 bug 在真实部署里会立刻暴露。

**`redirect` 是个闭包 handler**,参数是 `Uri` 提取器——拿到完整的请求 URI。

**`redirect.into_make_service()`** 用的是 `HandlerWithoutStateExt`,前置知识讲过:单个 handler 直接变服务,不需要 `Router`。

**`Redirect::permanent`** 产生 `301`,前置知识讲过为什么用它。

**跑起来看**:

```bash
cargo run
```

```
DEBUG tls_rustls: listening on 127.0.0.1:7878
DEBUG tls_rustls: listening on 127.0.0.1:3000
```

**两个端口都起来了。**

访问 HTTP 端口:

```bash
curl -i -s http://localhost:7878/ | head -3
```

```
HTTP/1.1 301 Moved Permanently
location: https://localhost:3000/
content-length: 0
```

**`301` 加 `location` 头部。** 让 curl 自动跟随(`-L`),并跳过证书验证:

```bash
curl -kL http://localhost:7878/
```

```
Hello, World!
```

**完整链路通了**:HTTP → 301 → HTTPS → 内容。

带路径和查询参数也能正确保留:

```bash
curl -i -s 'http://localhost:7878/some/path?a=1&b=2' | grep -i location
```

```
location: https://localhost:3000/some/path?a=1&b=2
```

**路径和查询字符串都跟过去了** —— `make_https` 只替换了 scheme 和 authority,`path_and_query` 原样保留。

**验证那个硬编码 `localhost` 的问题。** 用别的 Host 头部请求:

```bash
curl -i -s -H 'Host: example.com' http://localhost:7878/page | grep -i location
```

```
location: https://localhost:3000/page
```

**客户端请求的是 `example.com`,却被重定向到了 `localhost`。** 这就是上面「更好的写法」指出的 bug——本地测试看不出来,一旦部署到真实域名就会把用户导到错误的地方。

## 完整代码

`src/main.rs`:

```rust
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
    axum::serve(listener, redirect.into_make_service()).await.unwrap();
}
```

## 代价与边界

**生产环境通常不在应用里做 TLS。** 这是本章最重要的一条。主流做法是**反向代理终止 TLS**:

```
用户 --HTTPS--> Nginx/Caddy/ALB --HTTP--> 你的 axum 服务
```

理由很实际:

- **证书管理集中化** —— 一处配置、自动续期(Caddy 内置 Let's Encrypt),不用每个服务都配
- **应用不用重启** —— 换证书只重载代理
- **性能** —— 代理层可以做 TLS 会话复用、硬件加速
- **多服务共享** —— 一个域名下的多个后端服务共用一套证书

**那什么时候在应用里做 TLS:** 没有代理层的简单部署、需要端到端加密(内网也不信任)、mTLS 双向认证、或者像本章这样的学习场景。

**私钥进了版本控制。** 前置知识提过,官方示例仓库里有现成的 `.pem` 文件。真实项目必须排除,证书由部署流程注入(Kubernetes Secret、密钥管理服务等)。

**证书路径绑定到 `CARGO_MANIFEST_DIR`。** 步骤 1 的「更好的写法」指出了——编译好的二进制换机器就跑不了。

**重定向硬编码 `localhost`。** 步骤 2 的「更好的写法」指出了,这是个会在真实部署中立刻暴露的 bug。

**没有 HSTS 头部。** 前置知识讲过,加上 `Strict-Transport-Security` 能让浏览器**连第一次明文请求都不发**。但要谨慎:`max-age` 一旦设置,在有效期内浏览器**拒绝**用 HTTP 访问你的站点,配置错误会导致站点无法访问且难以撤销。建议先用小的 `max-age`(比如 300 秒)验证无误再逐步加长。

**证书过期无感知。** 示例加载一次证书就再也不管了。证书通常 90 天或 1 年过期,**过期后所有客户端连接失败**。`axum-server` 支持 `RustlsConfig::reload_from_pem_file` 热重载,配合文件监听可以做到不重启换证书。生产环境还应该有证书到期的监控告警。

**重定向服务用 `spawn` 且无监督。** 步骤 2 提过:如果它 panic 了,主服务照常运行但 HTTP 端口不再响应,而且**没有任何告警**。稳妥的做法是用 `tokio::select!` 让任一服务退出时整体退出,或者监控这个任务的 `JoinHandle`。

**没有配置 TLS 版本和加密套件。** `axum-server` 的默认配置是合理的(rustls 默认只支持 TLS 1.2+ 和现代套件),但有合规要求时需要显式配置。

**没有处理 SNI。** 一个 IP 上跑多个域名时需要根据 SNI 选择不同证书。`rustls` 支持,但示例只有一张证书。

## 动手练习

**1. 修掉重定向硬编码 `localhost` 的 bug。**

提示:给 `redirect` 闭包加一个 `TypedHeader<headers::Host>` 参数(需要 `axum-extra` 的 `typed-header` feature),从中取出主机名,和 HTTPS 端口组合成新的 authority。验证:`curl -i -H 'Host: example.com' http://localhost:7878/page` 应该重定向到 `https://example.com:3000/page`。**这是「代价与边界」里最该修的一处。**

**2. 加上 HSTS 头部。**

提示:用[第 17 章](17-tracing-aka-logging.md)的中间件或 `tower_http::set_header::SetResponseHeaderLayer`,给所有 HTTPS 响应加 `Strict-Transport-Security: max-age=300`。**先用 300 秒验证**,别一上来就设一年。用 `curl -kv` 确认头部出现了。

**3. 试试证书不匹配会怎样。**

提示:重新生成一张 SAN 只有 `DNS:example.com` 的证书替换掉,然后 `curl https://localhost:3000/`(不加 `-k`)。你会看到域名不匹配的错误,和步骤 1 那个"自签名"的错误不同——**这两种失败原因不同**:一个是没有 CA 背书,一个是证书上的域名对不上。理解这个区别有助于排查线上的证书问题。

## 小结与下一章预告

TLS 解决三个问题:加密(中间人看不到)、完整性(改了会被发现)、**身份认证(确认对方是谁)**。第三点最容易被忽略,但没有它前两点毫无意义——加密后发给冒充者,对方照样看得一清二楚。

自签名证书跳过了 CA 背书,加密照常但浏览器会警告。生成时**`subjectAltName` 是必需的**——现代浏览器只看 SAN、完全忽略 `CN`,这是第一次生成证书最常踩的坑。私钥绝不能进版本控制(官方示例仓库那样做只适合示例)。

**`axum::serve` 不支持 TLS**,它的设计目标就是"简单、无配置"。本章换成 `axum-server` 的 `bind_rustls`,并且要显式调 `into_make_service()` 把 `Router` 转成 `MakeService`。重定向服务用的是另一个变体——`HandlerWithoutStateExt::into_make_service` 让单个 handler 直接变服务,不需要 `Router`。

HTTP → HTTPS 重定向用 `301` 而非 `302`,因为浏览器会缓存,**用户下次直接走 HTTPS,连那次明文请求都省了**。更彻底的方案是 HSTS 响应头。

**但生产环境通常不在应用里做 TLS** —— 反向代理终止 TLS 是主流做法,证书管理集中、换证书不用重启应用。本章的技能适用于没有代理层的部署、端到端加密、或 mTLS 场景。

示例里有个会在真实部署中立刻暴露的 bug:重定向把 authority 硬编码成了 `localhost`,丢弃了客户端请求的主机名。本章验证过 `Host: example.com` 会被错误地导向 `localhost`。

---

「认证与安全」阶段(30~32)到此结束。你现在能自己签发 token、能对接第三方登录、能给服务加上 HTTPS。

接下来是「实时通信」阶段。前面所有章节都是"客户端问一句、服务端答一句"的请求-响应模式,但很多场景需要**服务端主动推送**:通知、进度、实时数据。[第 33 章 sse](33-sse.md) 讲最简单的一种方案——Server-Sent Events:它就是一个不结束的 HTTP 响应,服务端可以持续往里写数据。你会看到流式响应在 axum 里怎么表达,以及它和[第 09 章](09-consume-body-in-extractor-or-middleware.md#缓冲整个-body-的代价)那个"缓冲整个 body"的中间件为什么天然冲突。
