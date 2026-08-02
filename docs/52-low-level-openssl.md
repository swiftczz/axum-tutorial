# 第 52 章 low-level-openssl:握手拆到最细

## 本章你将做出什么

第三次做同一个服务:

```
https://[::1]:3000/    Hello, World!
```

这次 TLS 交给 **OpenSSL 本尊**。和前两章相比,握手被拆成了三步:创建 `Ssl` 上下文、包装成流、显式触发握手。看起来更啰嗦,但**每一步之间都是你可以插手的地方**。

而且你会撞上一个措手不及的默认值:**这个服务只接受 TLS 1.3 的客户端**。

学完本章你将掌握:

- `SslAcceptor::mozilla_modern_v5`:**一行拿到 Mozilla 推荐的安全配置**
- Mozilla 的三档配置(Modern / Intermediate / Old)分别意味着什么
- 手动握手的三步,以及**为什么需要 `Pin`**
- `check_private_key()`:一个能在启动时就发现问题的检查
- **`set_certificate_file` 和 `set_certificate_chain_file` 的区别**——一个典型的 OpenSSL 陷阱
- 三种 TLS 方案的完整对照

本章是 TLS 三部曲的最后一章,和[第 50](50-low-level-rustls.md)、[第 51 章](51-low-level-native-tls.md)重复的部分不再展开。

## 前置知识

### 直接用 OpenSSL 意味着什么

[第 51 章](51-low-level-native-tls.md)的 `native-tls` 在 Linux 上底层就是 OpenSSL,只是隔着一层门面。**这一章把门面拆了,直接调 OpenSSL 的 API。**

好处和坏处都很明确:

| | 说明 |
|---|---|
|  **能力最全** | OpenSSL 支持的一切都能用:ALPN、mTLS、SNI 回调、加密套件字符串、会话票据、OCSP stapling、自定义 BIO…… |
|  **生态最成熟** | 几十年的部署经验,运维熟悉,`openssl s_client` 之类的排障工具随手就有 |
|  **合规友好** | FIPS 认证的 OpenSSL 版本存在,某些行业只认这个 |
|  **构建最麻烦** | `openssl-sys` 要找系统的 OpenSSL;交叉编译、`FROM scratch` 镜像、静态链接全都要额外折腾 |
|  **API 是 C 风格** | **忘了调某个函数 = 静默地不安全**,类型系统帮不上忙 |
|  **版本差异大** | OpenSSL 1.1.1 / 3.x / LibreSSL / BoringSSL 的行为和 API 都有差别 |

**第二条  值得展开。** 对比一下[第 50 章](50-low-level-rustls.md#步骤-2构造-rustls-的-serverconfig)的 rustls:

```rust
ServerConfig::builder()
    .with_no_client_auth()          // 不选就编译不过
    .with_single_cert(certs, key)   // 不选就编译不过
```

**类型状态模式强制你把必要配置做完。** 而 OpenSSL 是:

```rust
let mut b = SslAcceptor::mozilla_modern_v5(SslMethod::tls()).unwrap();
b.set_certificate_file(...).unwrap();
b.set_private_key_file(...).unwrap();
// 少调一个也能 build(),运行时才出问题
```

**这就是为什么 `mozilla_modern_v5` 这个设计特别有价值** —— 它把"一堆容易漏的安全配置"打包成一行。

### Mozilla 的三档 TLS 配置

Mozilla 维护着一份[服务端 TLS 配置指南](https://wiki.mozilla.org/Security/Server_Side_TLS),把配置分成三档,`rust-openssl` 把它们做成了构造函数:

| 档位 | 构造函数 | 大致含义 | 代价 |
|---|---|---|---|
| **Modern** | `mozilla_modern_v5` | **只允许 TLS 1.3** | 老客户端全部连不上 |
| **Intermediate** | `mozilla_intermediate_v5` | TLS 1.2 + 1.3,现代套件 | **通用推荐** |
| **Old** | (旧版本有 `mozilla_old`) | 兼容很老的客户端 | 安全性打折 |

**本章的示例用的是 Modern。** 注释里没说,但这意味着 **TLS 1.2 的客户端会直接握手失败**。步骤 3 会验证。

> **这是一个应该刻意做的决策,不该照抄。** 面向公网、面向不可控客户端的服务,几乎都该用 **Intermediate**;只在完全可控的内网服务间调用里,Modern 才合适。

**`v5` 是这份指南的版本号**,不是 TLS 版本。指南本身会随时间更新,所以库把版本号带进了函数名——**这个设计很贴心**:指南更新时加一个 `_v6`,你的旧代码行为不会被悄悄改掉。

### 手动握手的三步

前两章都是一步到位:

```rust
let stream = tls_acceptor.accept(cnx).await?;    // 第 50、51 章
```

本章拆成三步:

```rust
let ssl = Ssl::new(tls_acceptor.context())?;             // ① 从上下文造一个连接对象
let mut tls_stream = SslStream::new(ssl, cnx)?;          // ② 把它和 TCP 流绑在一起
SslStream::accept(Pin::new(&mut tls_stream)).await?;     // ③ 真正握手
```

**看起来更啰嗦,但 ① 和 ② 之间是个插手的机会**:`Ssl` 是**per-connection** 的对象,你可以在握手前对这一条连接单独设置——比如按来源 IP 决定要不要验客户端证书、给 SNI 回调塞上下文数据、设置这条连接专用的验证参数。

**这是三种方案里控制粒度最细的一层。** 前两章的 `accept()` 把这三步包死了,给不了这个机会。

**注意 ③ 的返回值是 `Result<(), Error>`,不是一个新的流** —— 握手是**原地**在 `tls_stream` 上完成的,所以后面继续用 `tls_stream` 本身。这和前两章"吃一个流吐一个流"的形态不同。

### 为什么需要 `Pin`

```rust
SslStream::accept(Pin::new(&mut tls_stream)).await
```

`SslStream::accept` 的签名要求 `Pin<&mut Self>`,因为 **`SslStream` 是 `!Unpin`**:它内部把 OpenSSL 的 BIO 结构和底层流绑在一起,**OpenSSL 那边持有指向这块内存的指针**。一旦这个结构被移动,那些指针就悬空了。

`Pin` 就是类型系统对"这个值不能再移动"的表达。[第 40 章](40-stream-to-file.md#步骤-1流式写盘的核心函数)用过 `std::pin::pin!` 宏,那里是把值钉在栈上。

**这里用的是 `Pin::new(&mut tls_stream)`** —— 这个构造函数只对 `Unpin` 类型可用……**但 `Pin::new` 作用在 `&mut T` 上,而 `&mut T` 本身永远是 `Unpin`**,所以这行是合法的:被钉住的是**这个可变引用**,不是 `SslStream` 本身。`tls_stream` 是局部变量,在这个 `async` 块里不会被移动,所以安全。

> 这是 `Pin` 里最绕的一处。**记住结论就行:局部变量 + `Pin::new(&mut x)` 是处理 `!Unpin` 类型的标准写法。**

握手之后 `tls_stream` 被 `TokioIo::new` 拿走,后面的路径和前两章完全一致。

### `check_private_key()`

```rust
tls_builder.check_private_key().unwrap();
```

这一行做的是:**验证已加载的证书和私钥是配对的**。

为什么需要?因为 `set_certificate_file` 和 `set_private_key_file` 是两次独立的调用,**你完全可能把两个不匹配的文件塞进去**——比如换证书时只换了一半,或者两个文件的路径写反了。

**不做这个检查的话,问题要到第一个客户端来握手时才暴露**,而且错误信息在服务端和客户端两侧都很晦涩。

**这正是 C 风格 API 的典型形态**:安全性依赖你记得调用某个检查函数。对比 rustls 的 `with_single_cert(certs, key)` —— **证书和私钥一起传进去,库自己就配对检查了,返回 `Result`**,没有"忘了检查"这个选项。

### 其余部分和前两章相同

以下内容[第 50 章](50-low-level-rustls.md)已经讲透,本章不重复:

- [手写 accept 循环](50-low-level-rustls.md#tls-握手在连接生命周期里的位置)、[握手必须在 spawn 里做](50-low-level-rustls.md#tls-握手在连接生命周期里的位置)
- [ALPN 是什么](50-low-level-rustls.md#alpntls-层的协议协商)
- [`[::1]` 是 IPv6 回环](50-low-level-rustls.md#1ipv6-回环地址)
- [`TokioIo`、`auto::Builder`、hyper 的 `Service`](44-serve-with-hyper.md#axumserve-的五件事)

## 项目搭建

```bash
cargo new low-level-openssl
cd low-level-openssl
```

**先装系统依赖**(和[第 51 章](51-low-level-native-tls.md#项目搭建)一样):

```bash
# Debian / Ubuntu
sudo apt install pkg-config libssl-dev

# Fedora / RHEL
sudo dnf install pkgconf-pkg-config openssl-devel

# macOS(系统自带的是 LibreSSL,建议装 OpenSSL)
brew install openssl@3
```

 **macOS 上可能还要告诉 `openssl-sys` 去哪找**:

```bash
export OPENSSL_DIR=$(brew --prefix openssl@3)
```

```bash
cargo add axum
cargo add hyper --features full
cargo add hyper-util --features tokio,server-auto,http1,http2
cargo add openssl
cargo add tokio --features full
cargo add tokio-openssl
cargo add tower --features make
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

>  和前两章一样,**官方示例的 `hyper-util` feature 写得不全**(靠 workspace feature 统一才编译得过),单独建项目必须显式写全。

> **构建实在过不去的话**,可以让 `openssl` crate 自己编译一份:`cargo add openssl --features vendored`。代价是首次编译很慢(要编译整个 OpenSSL),好处是不依赖系统环境、方便交叉编译。**这也是 Rust 项目里躲开 OpenSSL 构建问题的常用招数。**

### 生成证书

和[第 50 章](50-low-level-rustls.md#生成证书)相同,可直接复用:

```bash
mkdir self_signed_certs
cd self_signed_certs

openssl req -x509 -newkey rsa:4096 \
  -keyout key.pem -out cert.pem \
  -days 365 -nodes \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1,IP:::1"

cd ..
echo 'self_signed_certs/*.pem' >> .gitignore
```

## 分步构建

### 步骤 1:构造 SslAcceptor

**要做什么**:从[第 50 章](50-low-level-rustls.md#步骤-1先跑通不带-tls-的手写循环)那个明文循环出发,先把 OpenSSL 的 acceptor 建出来。

```rust
use openssl::ssl::{SslAcceptor, SslFiletype, SslMethod};
use std::path::PathBuf;
```

```rust
    let mut tls_builder = SslAcceptor::mozilla_modern_v5(SslMethod::tls()).unwrap();

    tls_builder
        .set_certificate_file(
            PathBuf::from(env!("CARGO_MANIFEST_DIR"))
                .join("self_signed_certs")
                .join("cert.pem"),
            SslFiletype::PEM,
        )
        .unwrap();

    tls_builder
        .set_private_key_file(
            PathBuf::from(env!("CARGO_MANIFEST_DIR"))
                .join("self_signed_certs")
                .join("key.pem"),
            SslFiletype::PEM,
        )
        .unwrap();

    tls_builder.check_private_key().unwrap();

    let tls_acceptor = tls_builder.build();
```

**四步:选配置档 → 设证书 → 设私钥 → 校验配对。**

**`SslMethod::tls()`** 表示"用通用的 TLS 方法,具体版本由配置决定"。OpenSSL 历史上还有 `tlsv1()`、`sslv23()` 之类,**现在一律用 `tls()`**,版本范围交给上面的配置档控制。

**`SslFiletype::PEM`** —— 还有 `ASN1`(也就是 DER,二进制格式)。PEM 是那种 `-----BEGIN ...-----` 的文本格式。

**`build()` 消耗 builder,产出 `SslAcceptor`。** 和前两章一样,它内部是引用计数的,**clone 很便宜**。

**验证 `check_private_key` 的价值**:临时把两个文件路径写反(`set_certificate_file` 指向 `key.pem`),`cargo run`:

```
thread 'main' panicked at ...: error:...:PEM routines::no start line
```

启动就失败了,**而不是等第一个用户来访问才出问题**。改回去。

### 步骤 2:手动握手

**要做什么**:在 spawn 出去的任务里,用三步完成握手。

```rust
use openssl::ssl::Ssl;
use std::pin::Pin;
use tokio_openssl::SslStream;
use tracing::error;
```

```rust
        tokio::spawn(async move {
            let ssl = Ssl::new(tls_acceptor.context()).unwrap();
            let mut tls_stream = SslStream::new(ssl, cnx).unwrap();
            if let Err(err) = SslStream::accept(Pin::new(&mut tls_stream)).await {
                error!(
                    "error during tls handshake connection from {}: {}",
                    addr, err
                );
                return;
            }

            // Hyper has its own `AsyncRead` and `AsyncWrite` traits and doesn't use tokio.
            // `TokioIo` converts between them.
            let stream = TokioIo::new(tls_stream);

            // ... 后面和第 50 章完全一样
        });
```

**三步在前置知识里讲过了,这里补充两点:**

**① `tls_acceptor.context()`** 返回的是 `&SslContextRef` —— 那份共享的配置。每条连接从它派生出一个自己的 `Ssl`。**配置共享,连接对象独立。**

**② 错误处理比前两章好。** 注意这里用的是 `if let Err(err) = ...`,**把错误值打出来了**:

```rust
error!("error during tls handshake connection from {}: {}", addr, err);
```

而[第 50](50-low-level-rustls.md#let-else)、[51 章](51-low-level-native-tls.md)用的是 `let ... else`,**错误值被丢掉了**。[第 50 章的「代价与边界」](50-low-level-rustls.md#代价与边界)专门批评过这一点——**本章的写法才是对的**。

**验证**:

```bash
cargo run
```

```bash
curl -ki https://[::1]:3000/
```

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8

Hello, World!
```

**通了。** 顺便试试明文打 HTTPS 端口:

```bash
curl -i http://[::1]:3000/
```

服务端日志:

```
ERROR example_low_level_openssl: error during tls handshake connection from [::1]:54321: error:0A00010B:SSL routines:ssl3_get_record:wrong version number
```

**`wrong version number`** —— 这就是有错误详情和没有的差别。这条信息一眼就能看出"对方发的不是 TLS"。[第 50 章](50-low-level-rustls.md)那个 `let ... else` 版本只会告诉你"握手失败了",什么原因得靠猜。

### 步骤 3:一个意外发现——TLS 1.2 的客户端连不上

**要做什么**:测一下协议版本兼容性。

先看协商结果:

```bash
curl -kv https://[::1]:3000/ 2>&1 | grep -i "SSL connection using"
```

```
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
```

TLS 1.3,很好。**现在强制用 TLS 1.2:**

```bash
curl -kv --tlsv1.2 --tls-max 1.2 https://[::1]:3000/ 2>&1 | tail -8
```

```
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS alert, protocol version (582):
* OpenSSL/3.x.x: error:0A000102:SSL routines::unsupported protocol
curl: (35) ...
```

**握手直接失败。**

服务端日志也会有一条对应的握手错误。

**原因是 `mozilla_modern_v5`。** 前置知识那张表说过:**Modern 档只允许 TLS 1.3**。

**这是本示例里最容易让人措手不及的一个默认值:**

- 代码里**没有任何一行**写着"只支持 TLS 1.3"
- 注释里也没提
- 只有你去查 `mozilla_modern_v5` 是什么意思才知道

**TLS 1.3 是 2018 年的标准。** 到今天绝大多数客户端都支持,但**不是全部**:一些老的 Java 运行时、老的 Android、企业内网里的老设备、某些嵌入式客户端仍然只有 TLS 1.2。**面向公网的服务这么配,会有一小撮用户完全连不上,而且他们收到的是握手失败,连个错误页都看不到。**

> **更好的写法**:面向不可控客户端时用 **Intermediate** 档:
>
> ```rust
> let mut tls_builder = SslAcceptor::mozilla_intermediate_v5(SslMethod::tls()).unwrap();
> ```
>
> 它同时支持 TLS 1.2 和 1.3,并且只启用现代加密套件——**这是 Mozilla 自己推荐的通用默认值**。Modern 档留给完全可控的内网服务间调用。
>
> **不管选哪个,都该在代码里写一行注释说明为什么。** 这种"函数名里藏着重大安全/兼容性决策"的 API,不写注释的话,下一个人根本不会去查。

### 步骤 4:换成 Intermediate,对比一下

**要做什么**:亲手验证档位的差别。

```rust
    let mut tls_builder = SslAcceptor::mozilla_intermediate_v5(SslMethod::tls()).unwrap();
```

```bash
cargo run
```

```bash
curl -kv --tlsv1.2 --tls-max 1.2 https://[::1]:3000/ 2>&1 | grep -i "SSL connection using"
```

```
* SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384
```

**这次通了。** 而不限制版本时:

```bash
curl -kv https://[::1]:3000/ 2>&1 | grep -i "SSL connection using"
```

```
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
```

**仍然优先用 TLS 1.3。**

**再试试更老的:**

```bash
curl -kv --tlsv1.1 --tls-max 1.1 https://[::1]:3000/ 2>&1 | tail -5
```

**失败** —— Intermediate 档也不接受 TLS 1.1,这是对的([第 51 章](51-low-level-native-tls.md#min_protocol_version这里必须显式设)说过 TLS 1.0/1.1 已被 RFC 8996 正式弃用)。

**一行代码换来的差别:**

| | `mozilla_modern_v5` | `mozilla_intermediate_v5` |
|---|---|---|
| TLS 1.3 | 是 | 是 |
| TLS 1.2 | 否 | 是 |
| TLS 1.1 及以下 | 否 | 否 |
| 适合 | 可控的内网 | **面向公网** |

**本章完整代码保留官方示例的 `modern`,但你在自己项目里要按场景选。**

### 步骤 5:ALPN 的现状

**要做什么**:检查一下 HTTP/2 有没有协商上。

```bash
curl -kv https://[::1]:3000/ 2>&1 | grep -i alpn
```

```
* ALPN: curl offers h2,http/1.1
* ALPN: server did not agree on a protocol. Uses default.
```

```bash
curl -ki https://[::1]:3000/ | head -1
```

```
HTTP/1.1 200
```

**和[第 51 章](51-low-level-native-tls.md#步骤-3第二件不对劲的事协议降级)一样,只有 HTTP/1.1。** 因为示例没有配置 ALPN。

**但和[第 51 章](51-low-level-native-tls.md)有一个关键区别:那一章是"想配也没地方配",这一章是"能配,只是没配"。**

> **更好的写法**:OpenSSL 完全支持服务端 ALPN,补上即可:
>
> ```rust
> // ALPN 的线上格式是"长度前缀 + 内容"拼接:
> //   \x02 + "h2"          → 2 字节的 "h2"
> //   \x08 + "http/1.1"    → 8 字节的 "http/1.1"
> const ALPN: &[u8] = b"\x02h2\x08http/1.1";
>
> tls_builder.set_alpn_protos(ALPN).unwrap();
> tls_builder.set_alpn_select_callback(|_ssl, client_protos| {
>     openssl::ssl::select_next_proto(ALPN, client_protos)
>         .ok_or(openssl::ssl::AlpnError::NOACK)
> });
> ```
>
> 补完再 `curl -kv` 就能看到 `server accepted h2`,响应行变成 `HTTP/2 200`。
>
> **注意那个字节串格式。** [第 50 章](50-low-level-rustls.md#步骤-4配置-alpn)用 rustls 时写的是 `vec![b"h2".to_vec(), b"http/1.1".to_vec()]`,库帮你做了长度前缀的编码;OpenSSL 这里**要你自己按线上格式拼**。这就是"能力最全"和"API 是 C 风格"的一体两面——**它把协议的原始形态直接暴露给你,好处是什么都能做,坏处是拼错一个字节就静默失效。**
>
> (具体的函数签名和 `AlpnError` 变体请以你所用的 `openssl` 版本为准,这一块的 API 在不同版本间有过调整。)

## 完整代码

`src/main.rs`:

```rust
use axum::{http::Request, routing::get, Router};
use hyper::body::Incoming;
use hyper_util::rt::{TokioExecutor, TokioIo};
use openssl::ssl::{Ssl, SslAcceptor, SslFiletype, SslMethod};
use std::{path::PathBuf, pin::Pin};
use tokio::net::TcpListener;
use tokio_openssl::SslStream;
use tower::Service;
use tracing::{error, info, warn};
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

    let mut tls_builder = SslAcceptor::mozilla_modern_v5(SslMethod::tls()).unwrap();

    tls_builder
        .set_certificate_file(
            PathBuf::from(env!("CARGO_MANIFEST_DIR"))
                .join("self_signed_certs")
                .join("cert.pem"),
            SslFiletype::PEM,
        )
        .unwrap();

    tls_builder
        .set_private_key_file(
            PathBuf::from(env!("CARGO_MANIFEST_DIR"))
                .join("self_signed_certs")
                .join("key.pem"),
            SslFiletype::PEM,
        )
        .unwrap();

    tls_builder.check_private_key().unwrap();

    let tls_acceptor = tls_builder.build();

    let bind = "[::1]:3000";
    let tcp_listener = TcpListener::bind(bind).await.unwrap();
    info!("HTTPS server listening on {bind}. To contact curl -k https://localhost:3000");
    let app = Router::new().route("/", get(handler));

    loop {
        let tower_service = app.clone();
        let tls_acceptor = tls_acceptor.clone();

        // Wait for new tcp connection
        let (cnx, addr) = tcp_listener.accept().await.unwrap();

        tokio::spawn(async move {
            let ssl = Ssl::new(tls_acceptor.context()).unwrap();
            let mut tls_stream = SslStream::new(ssl, cnx).unwrap();
            if let Err(err) = SslStream::accept(Pin::new(&mut tls_stream)).await {
                error!(
                    "error during tls handshake connection from {}: {}",
                    addr, err
                );
                return;
            }

            // Hyper has its own `AsyncRead` and `AsyncWrite` traits and doesn't use tokio.
            // `TokioIo` converts between them.
            let stream = TokioIo::new(tls_stream);

            // Hyper also has its own `Service` trait and doesn't use tower. We can use
            // `hyper::service::service_fn` to create a hyper `Service` that calls our app through
            // `tower::Service::call`.
            let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
                // We have to clone `tower_service` because hyper's `Service` uses `&self` whereas
                // tower's `Service` requires `&mut self`.
                //
                // We don't need to call `poll_ready` since `Router` is always ready.
                tower_service.clone().call(request)
            });

            let ret = hyper_util::server::conn::auto::Builder::new(TokioExecutor::new())
                .serve_connection_with_upgrades(stream, hyper_service)
                .await;

            if let Err(err) = ret {
                warn!("error serving connection from {}: {}", addr, err);
            }
        });
    }
}

async fn handler() -> &'static str {
    "Hello, World!"
}
```

## 代价与边界

**`set_certificate_file` 只发叶子证书,不发证书链。** 这是本章最该记住的一条,而且**自签名证书完全掩盖了它**。

OpenSSL 有两个函数:

| 函数 | 作用 |
|---|---|
| `set_certificate_file` | 只设置**一张**证书(PEM 文件里的第一张) |
| **`set_certificate_chain_file`** | 设置**整条链**:叶子 + 所有中间 CA |

真实的 CA 签发的证书几乎总是带中间 CA 的:

```
根 CA(客户端信任库里有)
  └─ 中间 CA(必须由服务器发给客户端)
       └─ 你的证书
```

**服务器只发叶子证书的话,客户端拿不到中间 CA,就没法把链接到自己信任的根上。**

麻烦的是**症状很不一致**:

- 浏览器**可能**正常(它们会缓存见过的中间证书,或者用 AIA 字段主动去下载)
- `curl` 和多数编程语言的 HTTP 客户端**会直接报证书校验失败**
- **同一个站点,有的用户正常有的报错** —— 因为浏览器缓存了

**本章用自签名证书,只有一张,所以完全看不出区别。** 一换成真证书就出事。

> **更好的写法**:除非你确定证书文件里只有一张证书,否则用 `set_certificate_chain_file`:
>
> ```rust
> tls_builder.set_certificate_chain_file(cert_path).unwrap();
> ```
>
> **它对单张证书的文件同样工作**,所以直接用它就好,没有需要权衡的地方。[第 50 章](50-low-level-rustls.md#步骤-2构造-rustls-的-serverconfig)的 rustls 用 `pem_file_iter` 收集**所有**证书,天然就是链;`native-tls` 的 `Identity::from_pkcs8` 也接受链。**只有 OpenSSL 把这个坑留在了默认选项上**,这是历史包袱。

**`mozilla_modern_v5` 只支持 TLS 1.3。** 步骤 3、4 说了。这个决策藏在函数名里,代码和注释都没提。

**没有配置 ALPN。** 步骤 5 说了,补法也给了。

**构建依赖 OpenSSL。** 交叉编译、`FROM scratch` 镜像、静态链接都要额外折腾;不同发行版的 OpenSSL 版本差异也可能带来行为差异。`vendored` feature 能绕开,代价是编译时间。

**`Ssl::new` 和 `SslStream::new` 的 `unwrap()` 在连接任务里。** 这两处 panic 会让**那条连接的任务**崩掉。tokio 会捕获 spawn 任务的 panic 不至于让进程退出,但**日志里只有一条 panic 堆栈,没有对端地址**,排查困难。应该和握手一样用 `if let Err`。

**每条连接的 per-connection 定制能力没用上。** 前置知识说了,`Ssl::new` 和 `SslStream::new` 之间是这个方案最独特的插手点(按连接决定验证策略、SNI 回调等)。**示例把这个能力拿到手了,一点也没用** —— 和[第 50 章](50-low-level-rustls.md#代价与边界)的评价一样。

**没有握手超时、`accept().unwrap()` 会 panic、没有连接数上限、没有优雅停机、证书路径绑定 `CARGO_MANIFEST_DIR`、证书不会热重载、绑 `[::1]` 只监听 IPv6 回环** —— [第 50 章](50-low-level-rustls.md#代价与边界)逐条讲过,本章原样存在。

**没有 mTLS、没有 SNI。** 能做但没做。OpenSSL 的 SNI 回调(`set_servername_callback`)是三种方案里最灵活的。

## 动手练习

**1. 补上 ALPN,让 HTTP/2 能协商上。**

提示:按步骤 5 的「更好的写法」。验证:`curl -kv https://[::1]:3000/ 2>&1 | grep -i alpn` 应该看到 `server accepted h2`,响应行变成 `HTTP/2 200`。**顺便试着把 `\x02h2` 里的长度改成 `\x03`,看看会发生什么** —— 这能让你对"自己拼线上格式"的风险有实感。

**2. 改用 `set_certificate_chain_file`,并造一条真正的证书链来验证差别。**

提示:先造两级 CA:

```bash
# 根 CA
openssl req -x509 -newkey rsa:2048 -keyout root-key.pem -out root-cert.pem -days 365 -nodes -subj "/CN=my-root"
# 中间 CA(由根签发)
openssl req -newkey rsa:2048 -keyout inter-key.pem -out inter.csr -nodes -subj "/CN=my-intermediate"
openssl x509 -req -in inter.csr -CA root-cert.pem -CAkey root-key.pem -out inter-cert.pem -days 365 -CAcreateserial \
  -extfile <(echo "basicConstraints=critical,CA:TRUE")
# 服务器证书(由中间 CA 签发)
openssl req -newkey rsa:2048 -keyout key.pem -out server.csr -nodes -subj "/CN=localhost"
openssl x509 -req -in server.csr -CA inter-cert.pem -CAkey inter-key.pem -out server-cert.pem -days 365 -CAcreateserial \
  -extfile <(printf "subjectAltName=DNS:localhost,IP:127.0.0.1,IP:::1")
# 拼成链文件
cat server-cert.pem inter-cert.pem > cert.pem
```

然后**只信任根证书**去访问:`curl --cacert root-cert.pem https://localhost:3000/`。

- 用 `set_certificate_file`:**校验失败**(客户端拿不到中间 CA)
- 用 `set_certificate_chain_file`:**成功**

**这个练习把「代价与边界」里最隐蔽的一条变成了亲眼所见,值得花二十分钟做一遍。**

**3. 换成 Intermediate 档并写上注释。**

提示:步骤 4 做过验证,这次把它固化到代码里,并在旁边写一行注释说明选择理由。**这是本章最实际的一个改动。**

**4. 用上 per-connection 的定制能力。**

提示:在 `Ssl::new` 之后、`SslStream::new` 之前,根据 `addr` 判断来源——比如内网地址要求客户端证书(`ssl.set_verify(SslVerifyMode::PEER | SslVerifyMode::FAIL_IF_NO_PEER_CERT)`),外网地址不要求。**这是三种方案里只有 OpenSSL 能优雅做到的事**,做完你就明白拆成三步的价值了。

**5. 用 `openssl s_client` 排查,而不是 curl。**

提示:

```bash
openssl s_client -connect [::1]:3000 -showcerts
openssl s_client -connect [::1]:3000 -tls1_2      # 验证 modern 档拒绝 TLS 1.2
openssl s_client -connect [::1]:3000 -alpn h2     # 看 ALPN 协商
```

**这是运维排查 TLS 问题的标准工具**,输出比 curl 详细得多(完整证书链、协商细节、会话信息)。花点时间读懂它的输出,比记住任何一个库的 API 都有用。

## 小结与下一章预告

直接用 OpenSSL 的取舍很清晰:**能力最全、生态最熟、合规友好**,代价是**构建最麻烦**、**API 是 C 风格(忘了调某个函数就静默不安全)**、**版本差异大**。

`SslAcceptor::mozilla_modern_v5` 是这个 C 风格 API 里一个很贴心的设计:**一行拿到 Mozilla 推荐的整套安全配置**。但它也藏着一个重大决策——**Modern 档只允许 TLS 1.3**,步骤 3 验证了 TLS 1.2 的客户端会直接握手失败。**面向公网的服务应该用 `mozilla_intermediate_v5`**,而且不管选哪个都该写一行注释说明理由。

握手被拆成三步(`Ssl::new` → `SslStream::new` → `SslStream::accept`),比前两章啰嗦,但**前两步之间是 per-connection 定制的唯一入口**——按连接决定验证策略、SNI 回调都在这里做。`Pin::new(&mut tls_stream)` 是因为 `SslStream` 是 `!Unpin`(OpenSSL 持有指向它的指针),**局部变量 + `Pin::new(&mut x)` 是处理这类类型的标准写法**。

本章的错误处理反而是三章里最好的:用 `if let Err(err)` **把握手错误详情打了出来**,`wrong version number` 这样的信息一眼就能定位问题。前两章的 `let ... else` 把错误值丢了。

最该记住的坑是 **`set_certificate_file` 只发叶子证书,不发证书链**。自签名证书完全掩盖了它,一换成真证书就出事,而且症状不一致(浏览器可能正常、curl 报错)。**直接用 `set_certificate_chain_file` 即可,没有需要权衡的地方。**

### 三种 TLS 方案对照

| | `rustls`([50](50-low-level-rustls.md)) | `native-tls`([51](51-low-level-native-tls.md)) | `openssl`([52](52-low-level-openssl.md)) |
|---|---|---|---|
| 实现 | 纯 Rust | 平台自带 | OpenSSL |
| 系统依赖 | **无** | Linux 上有 | **有** |
| 交叉编译 / 精简镜像 | **容易** | 麻烦 | **最麻烦**(`vendored` 可缓解) |
| 配置安全性 | **类型状态强制** | 门面,选项少 | **靠你记得调** |
| 服务端 ALPN |  一行 |  **不支持** |  要自己拼线上格式 |
| mTLS | 是 | 否 | 是 |
| SNI 选证书 | 是 | 否 |  **最灵活** |
| 加密套件定制 | 细粒度 | **只能设版本上下限** | 最细 |
| per-connection 定制 | 有限 | 无 |  **独有** |
| 各平台行为一致 | 是 | 否 | 基本一致 |
| 排障工具 | 较少 | 视平台 |  `openssl s_client` |

**选型建议:**

1. **绝大多数情况:让反向代理终止 TLS**,应用只讲明文 HTTP
2. **应用内做 TLS:优先 `axum-server`**([第 32](32-tls-rustls.md)、[46 章](46-tls-graceful-shutdown.md)),三行搞定
3. **需要 ALPN / mTLS / SNI / 热重载:手写循环 + `rustls`**([第 50 章](50-low-level-rustls.md))
4. **合规要求系统密码模块:`native-tls`**,但要接受没有 ALPN
5. **需要 OpenSSL 特有能力或 FIPS:直接用 `openssl`**

---

TLS 三部曲到此结束。[第 53 章 unix-domain-socket](53-unix-domain-socket.md) 换一个完全不同的方向:**连 TCP 都不用了**。你会看到 axum 在 Unix 域套接字上跑服务,以及一个 TCP 永远给不了的能力——**内核直接告诉你对端进程的用户 ID、组 ID 和进程 ID**,连接层面的身份认证,业务代码一行不用写。
