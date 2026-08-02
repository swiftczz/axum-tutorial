# 第 50 章 low-level-rustls:自己握手,换来控制权

## 本章你将做出什么

一个 HTTPS 服务,但**这次没有 `axum-server`**:

```
https://[::1]:3000/    Hello, World!
```

TLS 握手是你自己在 accept 循环里显式做的一步。看起来绕远路,回报是**一堆封装拿不到的东西**——其中本章会用上的是 **ALPN**:同一个端口上,浏览器和支持 HTTP/2 的客户端会自动协商到 **HTTP/2**,老客户端仍然走 HTTP/1.1。

这是[第 44 章](44-serve-with-hyper.md)那个手写 accept 循环的续集:那一章拆开了 `axum::serve`,这一章在拆开的循环里**插进 TLS**。

学完本章你将掌握:

- **四种给 axum 加 TLS 的方式**,以及各自适合什么场景
- 下沉到底层能换来哪五种能力(mTLS、SNI、ALPN、握手信息、证书热重载)
- `tokio_rustls::TlsAcceptor`:握手在连接生命周期里的确切位置
- **ALPN**:TLS 层的协议协商,以及它和[第 44 章](44-serve-with-hyper.md)那个"偷看前 24 字节"的关系
- `rustls` 0.23 加载证书和私钥的新 API(`PemObject`)
- `[::1]` 是什么,以及它可能带来的一个坑
- `let ... else` 处理"失败就放弃这条连接"

## 前置知识

### 给 axum 加 TLS 的四种方式

到这一章为止,教程里已经出现过两种,加上本章和后面两章,一共是四种:

| 方式 | 出现在 | 特点 |
|---|---|---|
| **反向代理终止 TLS** | [第 32 章](32-tls-rustls.md#代价与边界)提过 | **生产环境主流**。应用只讲明文 HTTP,证书由 Nginx / ELB / Cloudflare 管 |
| **`axum-server`** | [第 32 章](32-tls-rustls.md)、[第 46 章](46-tls-graceful-shutdown.md) | 三行代码搞定,够用就好 |
| **手写循环 + `rustls`** | **本章** | 纯 Rust,控制力最强 |
| **手写循环 + `native-tls` / `openssl`** | [第 51](51-low-level-native-tls.md)、[第 52 章](52-low-level-openssl.md) | 用系统的 TLS 实现 |

**先说结论:大多数项目应该用前两种。** 这三章讲的是"当你确实需要控制握手时该怎么做",不是推荐做法。

### 下沉到底层能换来什么

具体列一下 `axum-server` 那层封装拿不到、而自己握手能拿到的东西:

| 能力 | 说明 |
|---|---|
| **ALPN 控制** | 决定在 TLS 层宣告支持哪些协议(`h2`、`http/1.1`、甚至自定义)。**本章会用到** |
| **mTLS(双向认证)** | 要求客户端也出示证书。金融、内网服务间调用常用 |
| **SNI 按域名选证书** | 一个端口服务多个域名,握手时根据客户端请求的域名挑证书 |
| **拿到握手结果** | 客户端证书的主体、协商出的协议和加密套件——可以塞进 `ConnectInfo` 给 handler 用 |
| **证书热重载** | 换证书不重启进程 |
| **加密套件 / 协议版本定制** | 合规要求(等保、PCI DSS)常常规定必须禁用某些套件 |

**如果这几条你一条都不需要,就别写这一章的代码。** 复杂度是有代价的——「代价与边界」会列出这个示例丢掉的东西。

### TLS 握手在连接生命周期里的位置

[第 44 章](44-serve-with-hyper.md#axumserve-的五件事)那个循环是这样的:

```
accept TCP  →  TokioIo 包装  →  hyper 跑 HTTP
```

加上 TLS 只是中间插一步:

```
accept TCP  →  【TLS 握手】  →  TokioIo 包装  →  hyper 跑 HTTP
                    ↑
              tls_acceptor.accept(cnx).await
```

**握手完成后拿到的 `TlsStream` 仍然是一个 `AsyncRead + AsyncWrite`** —— 读进来的是解密后的明文,写出去的自动加密。**对上层完全透明**,所以后面那几步和[第 44 章](44-serve-with-hyper.md)逐字相同。

**这一步是异步的,而且可能失败**:客户端可能用了不支持的协议版本、可能证书校验不过(mTLS 时)、可能根本不是 TLS 流量(有人拿 `curl http://` 打你的 HTTPS 端口)。**失败只应该放弃这条连接,不能影响服务器。**

 而且它**必须在 spawn 出去的任务里做**。如果在 accept 循环里直接 `.await` 握手,**一个慢客户端就能把整个服务卡住** —— 循环停在那里,后面的连接全都 accept 不了。这是一个典型的拒绝服务开口。示例的写法是对的:先 spawn,再握手。

### `TlsAcceptor`

```rust
let tls_acceptor = TlsAcceptor::from(rustls_config);   // 从 Arc<ServerConfig> 构造
let stream = tls_acceptor.accept(tcp_stream).await?;   // 吃一个 TcpStream,吐一个 TlsStream
```

`TlsAcceptor` 内部只有一个 `Arc<ServerConfig>`,**克隆极便宜**,和[第 41 章](41-reqwest-response.md)的 `reqwest::Client`、[第 46 章](46-tls-graceful-shutdown.md)的 `axum_server::Handle` 是同一个模式。

它来自 `tokio-rustls`,是 `rustls`(纯 Rust 的 TLS 实现,不依赖 OpenSSL)的 tokio 适配层。[第 32 章](32-tls-rustls.md#rustls-与-native-tls)讲过为什么 rustls 在 Rust 生态里是首选:**没有 C 依赖,交叉编译和容器构建都省心**。

### ALPN:TLS 层的协议协商

这是本章最重要的概念。

[第 44 章](44-serve-with-hyper.md#autobuilder一个端口两种协议)说过,`auto::Builder` 靠**偷看连接的头 24 字节**判断是 HTTP/1.1 还是 HTTP/2。那是**明文连接**的做法。

**加密之后行不通了** —— 服务端得先握完手才能看到明文,而握手时就得知道要用什么协议(HTTP/2 的 TLS 参数要求更严格)。而且更实际的问题是:**客户端不知道服务端支不支持 HTTP/2,不敢贸然发 HTTP/2 前言**。

**ALPN(Application-Layer Protocol Negotiation)就是为此而生的 TLS 扩展**:

```
客户端 ClientHello:  ALPN: ["h2", "http/1.1"]    ← 我会这些,你挑
服务端 ServerHello:  ALPN: "h2"                   ← 就用 h2
                     ---- 握手完成 ----
双方直接开始讲 HTTP/2,不需要试探
```

**协议在握手时就定好了,零额外往返。**

代码里就一行:

```rust
config.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1".to_vec()];
```

**顺序即优先级**:服务端从客户端提供的列表里,按**自己这个列表的顺序**挑第一个双方都支持的。把 `h2` 放前面就是"能用 HTTP/2 就用"。

**不设置 `alpn_protocols` 会怎样?** 服务端在握手时不宣告任何协议,**浏览器和 curl 就只会用 HTTP/1.1**。服务照常工作,只是永远用不上 HTTP/2 —— **没有任何报错,静默降级**。步骤 5 会亲手演示。

**这也解释了 `auto::Builder` 在 TLS 场景下的作用**:ALPN 已经在握手时定了协议,`auto::Builder` 拿到解密后的字节流再确认一次。两者配合,不冲突。

### `[::1]`:IPv6 回环地址

本章绑的是 `[::1]:3000`,不是前面一直用的 `127.0.0.1:3000`。

| | `127.0.0.1` | `::1` |
|---|---|---|
| 协议族 | IPv4 | **IPv6** |
| 含义 | 本机回环 | 本机回环 |
| 写成 socket 地址 | `127.0.0.1:3000` | **`[::1]:3000`**(方括号,因为 IPv6 地址里有冒号) |

**方括号是必需的**,否则 `::1:3000` 里的最后一个冒号会有歧义。

**这可能带来一个坑**:`curl https://localhost:3000` 能不能连上,取决于你系统的 `localhost` 解析到 `::1` 还是 `127.0.0.1`。多数现代系统会优先试 `::1`,但如果你的 `/etc/hosts` 只写了 `127.0.0.1 localhost`,**这条命令会连接被拒**。

**稳妥的验证命令是显式写地址**:

```bash
curl -k https://[::1]:3000/
```

「代价与边界」会说这个选择的取舍。

### `let ... else`

```rust
let Ok(stream) = tls_acceptor.accept(cnx).await else {
    error!("error during tls handshake connection from {}", addr);
    return;
};
```

`let ... else` 是 Rust 1.65 稳定的语法:**模式匹配成功就绑定并继续,失败就执行 `else` 块**,而 `else` 块必须发散(`return` / `break` / `panic!` 之类)。

比 `match` 少一层缩进,比 `if let` 少一次取反,**特别适合"这一步不成功就放弃"的场景**。

 注意这个写法**丢掉了错误值** —— 日志里只有地址,没有失败原因。「代价与边界」会说为什么这是个问题。

## 项目搭建

```bash
cargo new low-level-rustls
cd low-level-rustls
```

```bash
cargo add axum
cargo add hyper --features full
cargo add hyper-util --features tokio,server-auto,http1,http2
cargo add tokio --features full
cargo add tokio-rustls
cargo add tower-service
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

>  **注意 `hyper-util` 的 feature。** 官方示例的 `Cargo.toml` 里写的是 `hyper-util = { version = "0.1", features = ["http2"] }`,少了 `tokio`、`server-auto`、`http1`。它能编译是因为**官方示例都在同一个 cargo workspace 里,别的示例打开了这些 feature,cargo 做了 feature 统一**。
>
> **你单独建的项目没有这个便利,照抄那份 `Cargo.toml` 会编译不过**(找不到 `TokioIo`、`auto::Builder`)。上面这行 `cargo add` 才是正确的。[第 44 章](44-serve-with-hyper.md#项目搭建)用的是同一组 feature。
>
> 这也是本教程坚持用 `cargo add` 而不是抄 `Cargo.toml` 的原因之一。

**`tower-service` 而不是 `tower`。** 这里只需要 `Service` trait 本身,不需要 `tower` 里那一堆中间件。`tower` 只是把 `tower-service` 重导出了,直接依赖更轻。

### 生成证书

和[第 32 章](32-tls-rustls.md#生成证书)完全一样:

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

**注意 SAN 里多了 `IP:::1`** —— 因为本章绑的是 IPv6 回环。如果你打算用 `curl -k` 跳过校验,这一条可有可无;但**一旦你想用 `--cacert` 做真正的校验,证书里没有 `::1` 就会失败**。[第 32 章](32-tls-rustls.md#生成自签名证书)强调过 SAN 的重要性。

## 分步构建

### 步骤 1:先跑通不带 TLS 的手写循环

**要做什么**:从[第 44 章](44-serve-with-hyper.md#步骤-1手写第一个服务)那个循环出发,先确认明文能通。这样加 TLS 时,出问题能立刻定位到握手那一步。

```rust
// src/main.rs
use axum::{extract::Request, routing::get, Router};
use hyper::body::Incoming;
use hyper_util::rt::{TokioExecutor, TokioIo};
use tokio::net::TcpListener;
use tower_service::Service;
use tracing::{info, warn};
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

    let bind = "[::1]:3000";
    let tcp_listener = TcpListener::bind(bind).await.unwrap();
    info!("server listening on {bind}");
    let app = Router::new().route("/", get(handler));

    loop {
        let tower_service = app.clone();

        // Wait for new tcp connection
        let (cnx, addr) = tcp_listener.accept().await.unwrap();

        tokio::spawn(async move {
            let stream = TokioIo::new(cnx);

            let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
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

**这段代码[第 44 章](44-serve-with-hyper.md#步骤-1手写第一个服务)逐行讲过**,只有两处不同:绑的是 `[::1]:3000`,以及日志用 `tracing` 而不是 `eprintln!`。

**跑起来看**:

```bash
cargo run
```

```bash
curl -i http://[::1]:3000/
```

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8

Hello, World!
```

**顺便确认一下 IPv6 的坑**:

```bash
curl -i http://localhost:3000/
```

如果这条也通,说明你的 `localhost` 解析到了 `::1`。如果报 `Connection refused`,就是解析到了 `127.0.0.1` —— **后面的验证一律用 `[::1]` 的写法**。

### 步骤 2:构造 rustls 的 ServerConfig

**要做什么**:把证书和私钥读进来,组装成 `ServerConfig`。**这一步先不接进循环。**

```rust
use std::{
    path::{Path, PathBuf},
    sync::Arc,
};
use tokio_rustls::rustls::{
    pki_types::{pem::PemObject, CertificateDer, PrivateKeyDer},
    ServerConfig,
};
```

```rust
fn rustls_server_config(key: impl AsRef<Path>, cert: impl AsRef<Path>) -> Arc<ServerConfig> {
    let key = PrivateKeyDer::from_pem_file(key).unwrap();

    let certs = CertificateDer::pem_file_iter(cert)
        .unwrap()
        .map(|cert| cert.unwrap())
        .collect();

    let mut config = ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(certs, key)
        .expect("bad certificate/key");

    Arc::new(config)
}
```

在 `main` 里调用:

```rust
    let rustls_config = rustls_server_config(
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("key.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("cert.pem"),
    );
```

(暂时会有 unused 警告,下一步用上。)

**本步不单独用 curl 验证 HTTPS。** `cargo check`(或 `cargo run`,服务仍是明文 HTTP)能过即可——证书已经读进内存,握手还没插进循环。下一步插入后再用 `curl -k https://[::1]:3000/` 测。

**逐块看:**

**① `PrivateKeyDer::from_pem_file` 和 `CertificateDer::pem_file_iter`**

这是 `rustls` 0.23 的新 API,来自 `rustls-pki-types` 的 `PemObject` trait(所以要 `use ...pem::PemObject`,**否则这两个方法不存在**——这是最容易漏的一个 import)。

- **私钥是一个**,所以是 `from_pem_file`
- **证书可能是一串**,所以是 `pem_file_iter`

**为什么证书是一串?** 因为要发**证书链**:服务器证书 + 中间 CA 证书。客户端只信任根 CA,需要中间证书才能把链接起来。[第 32 章](32-tls-rustls.md#证书与信任链)讲过信任链。自签名证书只有一张,但 API 得支持一串。

**`.map(|cert| cert.unwrap()).collect()`** —— 迭代器每一项是 `Result`,这里对每一项 unwrap 再收集。**任何一张证书解析失败就 panic。**

**② `ServerConfig::builder()` 的类型状态机**

```rust
ServerConfig::builder()
    .with_no_client_auth()      // 决定客户端认证策略
    .with_single_cert(certs, key)   // 决定服务端证书
```

rustls 用**类型状态(typestate)模式**强制你按顺序做完所有必要配置:`builder()` 返回的类型上**只有** `with_no_client_auth` / `with_client_cert_verifier` 这两个方法,选完之后返回的新类型上才有 `with_single_cert` 等等。

**忘了配某一项,代码根本编译不过。** 这是 Rust 类型系统在安全关键配置上的一个漂亮应用——对比一下 OpenSSL 那种"忘了调某个函数就静默不安全"的 C 风格 API([第 52 章](52-low-level-openssl.md)会看到)。

**③ `with_no_client_auth()` 就是 mTLS 的开关位置**

```rust
.with_no_client_auth()                  // 不要求客户端证书
.with_client_cert_verifier(verifier)    // ← 要 mTLS 就换成这个
```

**这一行就是双向认证的入口。** 「动手练习」会做。

**④ 返回 `Arc<ServerConfig>`**

因为 `TlsAcceptor::from` 要的就是 `Arc<ServerConfig>`,而且每条连接都要共享同一份配置。

### 步骤 3:把握手插进循环

**要做什么**:在 spawn 出去的任务里,accept 之后、`TokioIo` 之前,加一步 TLS 握手。

```rust
use tokio_rustls::TlsAcceptor;
use tracing::error;
```

```rust
    let tls_acceptor = TlsAcceptor::from(rustls_config);
```

```rust
    loop {
        let tower_service = app.clone();
        let tls_acceptor = tls_acceptor.clone();

        // Wait for new tcp connection
        let (cnx, addr) = tcp_listener.accept().await.unwrap();

        tokio::spawn(async move {
            // Wait for tls handshake to happen
            let Ok(stream) = tls_acceptor.accept(cnx).await else {
                error!("error during tls handshake connection from {}", addr);
                return;
            };

            // Hyper has its own `AsyncRead` and `AsyncWrite` traits and doesn't use tokio.
            // `TokioIo` converts between them.
            let stream = TokioIo::new(stream);

            // ... 后面和步骤 1 完全一样
        });
    }
```

**总共只加了三行**(clone acceptor、let-else 握手、以及 `TokioIo` 现在包的是 `TlsStream` 而不是 `TcpStream`)。

**`TokioIo::new(stream)` 那行看起来没变,但包的东西变了。** 这正是前置知识说的:`TlsStream` 也是 `AsyncRead + AsyncWrite`,**上层代码不需要知道下面是不是加密的**。

**握手在 spawn 里面,不在循环里** —— 前置知识强调过,这是防止慢客户端卡死 accept 循环的关键。你可以试着把 `tls_acceptor.accept(cnx).await` 挪到 `tokio::spawn` 之前,然后用 `nc [::1] 3000` 连上去什么都不发,**整个服务就再也接不了新连接了**。

**跑起来看**:

```bash
cargo run
```

```bash
curl -ki https://[::1]:3000/
```

```
HTTP/2 200
content-type: text/plain; charset=utf-8

Hello, World!
```

**HTTPS 通了。** 但先别高兴太早——

 你看到的可能是 `HTTP/1.1 200`,也可能是 `HTTP/2 200`,**这取决于 rustls 的默认 ALPN 行为**。下一步显式配置它,把这件事变成确定的。

再确认一下握手失败不会拖垮服务:

```bash
curl -i http://[::1]:3000/       # 明文打 HTTPS 端口
```

```
curl: (52) Empty reply from server
```

服务端日志:

```
ERROR low_level_rustls: error during tls handshake connection from [::1]:54321
```

**这条连接被放弃了,服务继续跑。** 再 `curl -k https://...` 一次,一切正常。

### 步骤 4:配置 ALPN

**要做什么**:显式声明支持 `h2` 和 `http/1.1`,并验证协商结果。

在 `rustls_server_config` 里,`Arc::new(config)` 之前加一行:

```rust
    config.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1".to_vec()];
```

(`config` 已经声明为 `mut` 了。)

**`b"h2"` 是字节字符串字面量**,类型是 `&[u8; 2]`。ALPN 的协议标识符在协议里就是字节串,不是 UTF-8 文本,所以 rustls 要 `Vec<Vec<u8>>`。

**这些标识符是 IANA 注册的固定值**:`h2` 是 HTTP/2 over TLS,`http/1.1` 是 HTTP/1.1,`h3` 是 HTTP/3(QUIC)。**不能自己编**,写成 `b"http/2"` 客户端就认不出来了。

**跑起来看**:

```bash
cargo run
```

**看协商到了什么:**

```bash
curl -kiv https://[::1]:3000/ 2>&1 | grep -i alpn
```

```
* ALPN: curl offers h2,http/1.1
* ALPN: server accepted h2
```

**`server accepted h2`** —— 协商成功。响应行:

```bash
curl -ki https://[::1]:3000/ | head -1
```

```
HTTP/2 200
```

**强制用 HTTP/1.1 也应该正常:**

```bash
curl -ki --http1.1 https://[::1]:3000/ | head -1
```

```
HTTP/1.1 200
```

**同一个端口,两种协议,靠 ALPN 在握手时选好。** 这就是 `auto::Builder` 和 ALPN 配合的效果。

浏览器访问 `https://[::1]:3000/`(忽略证书警告),开发者工具的网络面板里 Protocol 那一列会显示 **h2**。

### 步骤 5:故意改坏,看 ALPN 缺失的静默降级

**要做什么**:把那行 `alpn_protocols` 注释掉,观察后果。

```rust
    // config.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1".to_vec()];
```

```bash
cargo run
```

```bash
curl -kiv https://[::1]:3000/ 2>&1 | grep -i alpn
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

**现象:一切正常,只是永远是 HTTP/1.1。**

- **没有错误、没有警告、没有日志**
- 服务照常工作,curl 照常拿到 200
- **浏览器也一样,永远走 HTTP/1.1**

**这就是典型的静默降级。** 你以为自己上了 HTTP/2,实际上多路复用、头部压缩、服务端推送一样都没有。**只有翻网络面板或者用 `-v` 才看得出来。**

**在真实项目里这个 bug 的面貌通常是**:"上了 HTTP/2 之后性能没有任何变化" —— 因为压根没上。

**顺带一个反向的坑**:如果你只写 `vec![b"h2".to_vec()]`(只支持 HTTP/2),**不支持 HTTP/2 的客户端会直接握手失败**,而不是降级。ALPN 列表**必须包含 `http/1.1` 兜底**,除非你确定所有客户端都支持 HTTP/2(比如纯 gRPC 服务)。

**把那行加回去。**

## 完整代码

`src/main.rs`:

```rust
use axum::{extract::Request, routing::get, Router};
use hyper::body::Incoming;
use hyper_util::rt::{TokioExecutor, TokioIo};
use std::{
    path::{Path, PathBuf},
    sync::Arc,
};
use tokio::net::TcpListener;
use tokio_rustls::{
    rustls::pki_types::{pem::PemObject, CertificateDer, PrivateKeyDer},
    rustls::ServerConfig,
    TlsAcceptor,
};
use tower_service::Service;
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

    let rustls_config = rustls_server_config(
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("key.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("cert.pem"),
    );

    let tls_acceptor = TlsAcceptor::from(rustls_config);
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
            // Wait for tls handshake to happen
            let Ok(stream) = tls_acceptor.accept(cnx).await else {
                error!("error during tls handshake connection from {}", addr);
                return;
            };

            // Hyper has its own `AsyncRead` and `AsyncWrite` traits and doesn't use tokio.
            // `TokioIo` converts between them.
            let stream = TokioIo::new(stream);

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

fn rustls_server_config(key: impl AsRef<Path>, cert: impl AsRef<Path>) -> Arc<ServerConfig> {
    let key = PrivateKeyDer::from_pem_file(key).unwrap();

    let certs = CertificateDer::pem_file_iter(cert)
        .unwrap()
        .map(|cert| cert.unwrap())
        .collect();

    let mut config = ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(certs, key)
        .expect("bad certificate/key");

    config.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1".to_vec()];

    Arc::new(config)
}
```

## 代价与边界

**握手错误被丢掉了。** `let Ok(stream) = ... else` 这个写法**拿不到错误值**,日志里只有对端地址。而握手失败的原因非常多样:协议版本不匹配、没有共同的加密套件、客户端证书无效、根本不是 TLS 流量、客户端中途断开……**全都长成同一条日志**,排查时毫无线索。

> **更好的写法**:改用 `match`,把错误打出来:
>
> ```rust
> let stream = match tls_acceptor.accept(cnx).await {
>     Ok(stream) => stream,
>     Err(err) => {
>         error!(%err, %addr, "tls handshake failed");
>         return;
>     }
> };
> ```
>
> `let ... else` 在"不关心失败原因"时很好用,但**在网络边界上,失败原因就是最有价值的信息**。这是本章最该改的一处。

**没有握手超时。** 一个客户端可以完成 TCP 连接、然后在 TLS 握手中途停住,**这个任务就永远挂着**,占一个 fd 和一份内存。攒够几万个就是拒绝服务(TLS 版的 slowloris)。

正确做法是包一层超时:

```rust
let stream = match tokio::time::timeout(Duration::from_secs(10), tls_acceptor.accept(cnx)).await {
    Ok(Ok(stream)) => stream,
    Ok(Err(err)) => { /* 握手失败 */ return; }
    Err(_) => { /* 超时 */ return; }
};
```

注意那个**双层 `Result`** —— 外层是超时,内层是握手结果。[第 26 章](26-diesel-postgres.md#三层错误从哪来)见过同样的形状。

**`accept().await.unwrap()` 会让服务 panic。** 和[第 42 章](42-http-proxy.md#步骤-1不用-axumserve自己写-accept-循环)、[第 44 章](44-serve-with-hyper.md#代价与边界)一样的问题,不重复了。

**没有连接数上限,没有优雅停机。** 同上,而且[第 45 章](45-graceful-shutdown.md)的 `with_graceful_shutdown` 在这里**用不了**——手写循环得自己实现整套停机逻辑。**这是"能用 `axum::serve` 就别自己写"的又一个具体代价。**

**证书路径绑定 `CARGO_MANIFEST_DIR`。** [第 32 章](32-tls-rustls.md#代价与边界)详细说过:编译好的二进制换台机器就找不到证书。

**证书不会热重载。** 加载一次就再也不管。证书通常 90 天过期,**过期后所有连接失败**。而**手写握手其实是做热重载最方便的地方**:把 `Arc<ServerConfig>` 换成 `Arc<ArcSwap<ServerConfig>>`,文件变化时换掉,新连接自动用新证书,老连接不受影响。**这正是下沉到底层的价值之一,但示例没用上。**

**没有用上 mTLS 和 SNI。** `with_no_client_auth()` 和单证书。**也就是说,这个示例把底层的控制权拿到手了,却几乎没花出去** —— 唯一实际用上的是 ALPN。真实项目里如果只是为了 ALPN 而下沉到这一层,不如直接用 `axum-server`(它默认就配了 ALPN)。

**握手结果没有传给 handler。** rustls 的 `TlsStream` 可以通过 `stream.get_ref().1` 拿到 `ServerConnection`,里面有协商出的协议、加密套件、客户端证书(mTLS 时)。配合[第 44 章](44-serve-with-hyper.md#service-与-makeservice多出来的那一层)的 `ConnectInfo`,可以把这些信息送进 handler。**这是底层方案最有价值的能力之一,示例没做**,见「动手练习」。

**没有配置加密套件和协议版本。** rustls 的默认值是安全的(只支持 TLS 1.2+ 和现代套件),**多数情况下不该动**。但有合规要求时需要显式配置——而且**这正是审计员会要求你出示的东西**。

**绑 `[::1]` 只监听 IPv6 回环。** 这意味着:IPv4 客户端连不上、别的机器连不上。开发够用,但换环境时**必须重新决策**。[第 44 章](44-serve-with-hyper.md#代价与边界)、[第 45 章](45-graceful-shutdown.md#代价与边界)说过同样的话——**这个值从来不该照抄**。

**没有 HTTP → HTTPS 重定向。** [第 32 章](32-tls-rustls.md#步骤-2http-重定向到-https)那套在这里同样适用,示例没带。

## 动手练习

**1. 把握手错误打出来,并加上超时。**

提示:按「代价与边界」的两段代码改。验证错误信息:`curl http://[::1]:3000/`(明文打 HTTPS 端口),这次日志里应该有具体原因。验证超时:`nc ::1 3000` 连上后什么都不发,10 秒后服务端应该记一条超时日志并清理连接。**这两处是本章最该修的地方。**

**2. 把协商出的协议传给 handler。**

提示:握手后用 `stream.get_ref().1.alpn_protocol()` 拿到协商结果,做成一个自定义结构体,通过[第 44 章](44-serve-with-hyper.md#步骤-4用-makeservice-注入客户端地址)的方式注入。因为是手写循环,可以直接用 `request.extensions_mut().insert(...)` 塞进请求扩展,更简单。验证:handler 返回 `"you are speaking h2"`,分别用 `curl -k` 和 `curl -k --http1.1` 看结果不同。

**3. 开启 mTLS,要求客户端出示证书。**

提示:先造一个 CA 和一张客户端证书:

```bash
openssl req -x509 -newkey rsa:2048 -keyout ca-key.pem -out ca-cert.pem -days 365 -nodes -subj "/CN=my-ca"
openssl req -newkey rsa:2048 -keyout client-key.pem -out client.csr -nodes -subj "/CN=client-1"
openssl x509 -req -in client.csr -CA ca-cert.pem -CAkey ca-key.pem -out client-cert.pem -days 365 -CAcreateserial
```

然后把 `with_no_client_auth()` 换成 `with_client_cert_verifier(...)`(用 `tokio_rustls::rustls::server::WebPkiClientVerifier::builder(roots).build()`,`roots` 里放 CA 证书)。验证:不带证书的 `curl -k https://[::1]:3000/` **握手失败**,带上 `--cert client-cert.pem --key client-key.pem` 才能通。**做完你就理解了为什么金融和内网服务间调用喜欢 mTLS——身份验证在连接层就完成了,业务代码一行不用写。**

**4. 实现证书热重载。**

提示:`cargo add arc-swap`,把 `Arc<ServerConfig>` 存进 `ArcSwap`,spawn 一个任务定时(或用 `notify` 监听文件)重新加载并 `store`。每次 accept 时 `load_full()` 取当前配置构造 `TlsAcceptor`。验证:服务跑着的时候重新生成证书,新的 `curl -kv` 应该看到新证书的指纹,**而且期间已有连接不受影响**。

**5. 只留 `h2`,看看老客户端会怎样。**

提示:`config.alpn_protocols = vec![b"h2".to_vec()];`,然后 `curl -k --http1.1 https://[::1]:3000/`。**你会看到握手直接失败,而不是降级。** 这验证了步骤 5 结尾那个反向的坑。

## 小结与下一章预告

给 axum 加 TLS 有四种方式,**生产环境多数应该让反向代理终止 TLS,次选 `axum-server`**。自己握手是为了拿到那五种控制能力:**ALPN、mTLS、SNI 选证书、握手结果、证书热重载**。一条都不需要就别写这一章的代码。

加 TLS 的改动本身很小:在[第 44 章](44-serve-with-hyper.md)那个 accept 循环里,spawn 之后插一步 `tls_acceptor.accept(cnx).await`。**握手完成后拿到的 `TlsStream` 仍然是 `AsyncRead + AsyncWrite`,上层完全不用改** —— 这就是分层抽象的价值。

**握手必须在 spawn 里做**,放在循环里的话,一个不完成握手的慢客户端就能让整个服务再也接不了新连接。

`rustls` 的 `ServerConfig::builder()` 用**类型状态模式**强制你配完所有必要项,漏了直接编译不过——`with_no_client_auth()` 那个位置就是 mTLS 的开关。这和 OpenSSL 那种"忘了调某个函数就静默不安全"的风格形成鲜明对比([第 52 章](52-low-level-openssl.md)会看到)。

**全章的实际收益是 ALPN**:一行 `config.alpn_protocols = vec![b"h2", b"http/1.1"]`,让浏览器在同一个端口上协商到 HTTP/2。步骤 5 注释掉这行之后,**服务照常工作,只是永远是 HTTP/1.1,没有任何提示**——"上了 HTTP/2 却没有任何性能变化"这个经典困惑就是这么来的。反过来,只写 `h2` 不写 `http/1.1`,老客户端会**握手失败而不是降级**。

要认清的是,**这个示例把控制权拿到手了,却几乎没花出去**:没有 mTLS、没有 SNI、没有热重载、没有把握手结果传给 handler。同时它丢掉了 `axum::serve` 帮你做的事——优雅停机在这里得自己实现。而且握手错误被 `let ... else` 丢掉了,**网络边界上的失败原因是最有价值的信息**,这是最该改的一处。

[第 51 章 low-level-native-tls](51-low-level-native-tls.md) 换一套 TLS 实现:**用操作系统自带的**。循环结构一模一样,只换握手那一层。你会看到它省掉了什么、也失去了什么——**其中一项恰好是本章的主角 ALPN**。那一章还藏着官方示例里一个货真价实的复制粘贴 bug,会让启动日志凭空消失。
