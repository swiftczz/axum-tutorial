# 第 51 章 low-level-native-tls:用系统自带的 TLS

## 本章你将做出什么

和[第 50 章](50-low-level-rustls.md)**一模一样的服务**:

```
https://[::1]:3000/    Hello, World!
```

区别只在于 TLS 由谁来做:上一章是纯 Rust 的 `rustls`,这一章交给**操作系统自带的 TLS 实现**。

accept 循环、`TokioIo`、`auto::Builder`、`service_fn` —— **一个字都没变**。变的只有握手那一层的两个函数。

学完本章你将掌握:

- `native-tls` 到底是什么:一个 crate,三套底层实现
- **用系统 TLS 换来什么、失去什么**——其中失去的一项是[第 50 章](50-low-level-rustls.md)的主角 **ALPN**
- `Identity::from_pkcs8`,以及 PKCS#8 和 PKCS#1 私钥格式的区别
- `min_protocol_version`:为什么这里必须显式设置
- 官方示例里一个**货真价实的复制粘贴 bug**,它会让启动日志凭空消失

本章大部分代码和[第 50 章](50-low-level-rustls.md)重复,**只讲不同的部分**。

## 前置知识

### 一个 crate,三套实现

`native-tls` 不自己实现 TLS,它是一层**统一的门面**,底下按平台分发:

| 平台 | 实际用的 TLS 实现 |
|---|---|
| **Windows** | SChannel(系统组件) |
| **macOS / iOS** | Security.framework(系统组件) |
| **Linux / 其他** | **OpenSSL**(需要系统安装) |

**注意最后一行:在 Linux 上,`native-tls` 并没有帮你省掉 OpenSSL 依赖**,只是帮你屏蔽了 API 差异。所以"用 native-tls 就不用装 OpenSSL 了"这个印象是错的——**只有 Windows 和 macOS 才真的不用**。

### rustls vs native-tls:该选哪个

| | `rustls`([第 50 章](50-low-level-rustls.md)) | `native-tls`(本章) |
|---|---|---|
| 实现语言 | **纯 Rust** | C(平台的) |
| 系统依赖 | **无** | Linux 上要 OpenSSL 开发库 |
| 交叉编译 / 静态链接 | **容易** | 麻烦 |
| 容器镜像 | **可以用 `scratch`/`distroless`** | 要带 OpenSSL 运行时 |
| 安全更新 | 跟着你的 `cargo update` | **跟着系统更新** |
| 信任的根证书 | 需要显式提供(`webpki-roots` 或系统的) | **自动用系统信任库** |
| 各平台行为 | **完全一致** | **按平台不同** |
| 服务端 ALPN |  支持 |  **builder 没有暴露** |
| 加密套件定制 | 细粒度 | 粗粒度(只能设协议版本上下限) |
| 企业合规场景 | 有时不被接受 | **有时是硬性要求**(必须用系统认证过的密码模块) |

**Rust 生态的默认选择是 `rustls`**,理由主要是前四行:没有 C 依赖,构建和分发省心太多。

**那什么时候该用 `native-tls`?** 两种情况:

1. **合规要求**:某些行业规定必须使用操作系统提供的、经过认证的密码学模块(FIPS 之类),这时候 `rustls` 直接出局
2. **需要跟随系统的信任策略**:企业里常见的场景是安全软件往系统信任库里装了自己的根证书(用于流量检查),用 `rustls` 的话你得自己去读系统信任库

 **"各平台行为按平台不同"这一行值得单独强调。** 同一份代码,在 macOS 上握手成功、在某台 Linux 上失败,原因可能只是两边的 OpenSSL 版本默认策略不同。**这类问题非常难查**,因为你的开发机和服务器往往不是同一个系统。

### `Identity::from_pkcs8` 和私钥格式

`native-tls` 用 `Identity` 表示"证书 + 私钥"这一对:

```rust
let id = Identity::from_pkcs8(cert_pem.as_bytes(), key_pem.as_bytes()).unwrap();
```

**注意函数名里的 `pkcs8`** —— 它要求私钥是 **PKCS#8** 格式。PEM 文件里的头一行能看出来:

| 头一行 | 格式 | `from_pkcs8` 能不能读 |
|---|---|---|
| `-----BEGIN PRIVATE KEY-----` | **PKCS#8** | 是 |
| `-----BEGIN RSA PRIVATE KEY-----` | PKCS#1(仅 RSA) | 否 |
| `-----BEGIN EC PRIVATE KEY-----` | SEC1(仅 ECDSA) | 否 |

[第 32 章](32-tls-rustls.md#生成自签名证书)那条 `openssl req -newkey ... -nodes` 生成的**就是 PKCS#8**,所以能直接用。但如果你手上的私钥是 PKCS#1(老工具生成的常见),要先转:

```bash
openssl pkcs8 -topk8 -nocrypt -in old-key.pem -out key.pem
```

**这是换 TLS 库时最常见的一个绊脚石。** [第 50 章](50-low-level-rustls.md)的 rustls 用 `PrivateKeyDer::from_pem_file`,它**三种格式都认**,所以那一章不需要操心这件事。

`native-tls` 还有 `Identity::from_pkcs12`(读 `.p12`/`.pfx` 文件),Windows 世界更常见。

### `min_protocol_version`:这里必须显式设

```rust
NativeTlsAcceptor::builder(id)
    // let's be modern
    .min_protocol_version(Some(Protocol::Tlsv12))
    .build()
```

**为什么 `rustls` 不需要这一行,这里需要?**

因为 **`rustls` 从设计上就只实现 TLS 1.2 和 1.3**,压根没有旧协议的代码。而 `native-tls` 底下是系统的实现,**旧协议(TLS 1.0 / 1.1、甚至 SSLv3)是不是启用,取决于系统和版本**。

TLS 1.0 和 1.1 已经在 2021 年被 IETF 正式弃用(RFC 8996),PCI DSS 之类的合规标准也早就禁止了。**所以这一行不是可选的美化,是必需的安全配置。**

> 官方示例的注释写的是 `// let's be modern`,轻描淡写了。**这一行如果漏掉,你可能在某些系统上开着一个允许 TLS 1.0 的服务而不自知** —— 又一个静默的安全问题。

**这正好体现了两种方案的哲学差异**:`rustls` 是"默认安全,不给你不安全的选项";`native-tls` 是"默认跟随系统,安全与否你自己负责"。

### 其余部分和[第 50 章](50-low-level-rustls.md)相同

以下概念[第 50 章](50-low-level-rustls.md)已经讲透,本章不重复:

- [手写 accept 循环的五件事](50-low-level-rustls.md#tls-握手在连接生命周期里的位置)
- [握手必须在 spawn 里做](50-low-level-rustls.md#tls-握手在连接生命周期里的位置)
- [`let ... else` 处理握手失败](50-low-level-rustls.md#let-else)
- [`[::1]` 是 IPv6 回环](50-low-level-rustls.md#1ipv6-回环地址)
- [`TokioIo`、`auto::Builder`、hyper 的 `Service`](44-serve-with-hyper.md#axumserve-的五件事)

## 项目搭建

```bash
cargo new low-level-native-tls
cd low-level-native-tls
```

**Linux 上先装系统依赖**(macOS 和 Windows 跳过):

```bash
# Debian / Ubuntu
sudo apt install pkg-config libssl-dev

# Fedora / RHEL
sudo dnf install pkgconf-pkg-config openssl-devel
```

**没装的话 `cargo build` 会在编译 `openssl-sys` 时报错**,错误信息是找不到 `openssl.pc`。这就是上面那张表里"系统依赖"那一行的实际含义。

```bash
cargo add axum
cargo add hyper --features full
cargo add hyper-util --features tokio,server-auto,http1,http2
cargo add tokio --features full
cargo add tokio-native-tls
cargo add tower-service
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

>  和[第 50 章](50-low-level-rustls.md#项目搭建)一样,**官方示例的 `hyper-util` 没写 feature**(`hyper-util = { version = "0.1" }`),那是靠 workspace 的 feature 统一才能编译的。**你单独建的项目必须显式写全**,否则找不到 `TokioIo` 和 `auto::Builder`。

### 生成证书

和[第 50 章](50-low-level-rustls.md#生成证书)完全相同,可以直接把 `self_signed_certs/` 目录拷过来:

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

**确认私钥是 PKCS#8**:

```bash
head -1 self_signed_certs/key.pem
```

```
-----BEGIN PRIVATE KEY-----
```

**看到 `RSA PRIVATE KEY` 就要按前置知识那条命令转一下。**

## 分步构建

### 步骤 1:换掉 TLS 那一层

**要做什么**:把[第 50 章](50-low-level-rustls.md#完整代码)的代码拿过来,只改两处——证书加载函数,和 acceptor 的构造。

**改动一:`rustls_server_config` 换成 `native_tls_acceptor`**

```rust
use std::path::PathBuf;
use tokio_native_tls::{
    native_tls::{Identity, Protocol, TlsAcceptor as NativeTlsAcceptor},
    TlsAcceptor,
};
```

```rust
fn native_tls_acceptor(key_file: PathBuf, cert_file: PathBuf) -> NativeTlsAcceptor {
    let key_pem = std::fs::read_to_string(&key_file).unwrap();
    let cert_pem = std::fs::read_to_string(&cert_file).unwrap();

    let id = Identity::from_pkcs8(cert_pem.as_bytes(), key_pem.as_bytes()).unwrap();
    NativeTlsAcceptor::builder(id)
        // let's be modern
        .min_protocol_version(Some(Protocol::Tlsv12))
        .build()
        .unwrap()
}
```

**注意那个 `as NativeTlsAcceptor` 重命名。** 这里有两个 `TlsAcceptor`:

| 类型 | 来自 | 作用 |
|---|---|---|
| `native_tls::TlsAcceptor` → 重命名为 `NativeTlsAcceptor` | `native-tls` | **同步的**,不能直接用在 async 代码里 |
| `tokio_native_tls::TlsAcceptor` | `tokio-native-tls` | **异步包装**,提供 `.accept(stream).await` |

**分层和 rustls 那边一模一样**:`rustls::ServerConfig` 是配置,`tokio_rustls::TlsAcceptor` 是异步包装。**只是 native-tls 把"配置"和"acceptor"合成了一个类型,所以撞名了。**

**`std::fs::read_to_string` 是同步 IO** —— 在 `main` 里、服务启动前调用,阻塞一下没关系。[第 26 章](26-diesel-postgres.md)讲的"别在异步上下文里做阻塞 IO"针对的是请求处理路径,启动阶段不受此限。

**改动二:`main` 里的构造**

```rust
    let tls_acceptor = native_tls_acceptor(
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("key.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("cert.pem"),
    );

    let tls_acceptor = TlsAcceptor::from(tls_acceptor);
```

**两行 `let tls_acceptor`,后一行遮蔽(shadow)前一行** —— 先构造同步的,再包成异步的。Rust 的变量遮蔽在这种"同名不同类型的转换"场景下很常用。

**循环内部一个字都不用改。** `tls_acceptor.accept(cnx).await` 的签名和 rustls 版本完全一致,返回的 `TlsStream` 同样实现了 `AsyncRead + AsyncWrite`。

> **这就是分层抽象的价值**:换了整套 TLS 实现,上层的 HTTP 处理代码零改动。[第 50 章](50-low-level-rustls.md#tls-握手在连接生命周期里的位置)说过"上层完全不用知道下面是不是加密的",这一章进一步证明了"也不用知道是哪家实现的加密"。

**跑起来看**:

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

**HTTPS 通了。但有两件事不对劲**,下面两步分别追查。

### 步骤 2:启动日志去哪了

**要做什么**:上一步 `cargo run` 之后,你应该注意到一件事——**那条 `info!("HTTPS server listening on ...")` 没有出现。**

服务明明起来了(curl 能通),但终端上除了 cargo 的编译输出什么都没有。

**先确认不是错觉:**

```bash
RUST_LOG=debug cargo run
```

```
 INFO example_low_level_native_tls: HTTPS server listening on [::1]:3000. ...
```

**加了 `RUST_LOG` 就有了。** 说明日志本身没问题,是**默认的过滤规则**有问题。

**看一眼那段初始化代码:**

```rust
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "example_low_level_rustls=debug".into()),
        )
```

**`example_low_level_rustls`。**

而本项目的 crate 叫 **`example_low_level_native_tls`**(或者你 `cargo new` 时起的名字)。

**这是一个从[第 50 章](50-low-level-rustls.md)那个示例复制过来时没改的名字。** `EnvFilter` 的指令说的是"把 `example_low_level_rustls` 这个目标开到 debug",而本 crate 的日志目标和它对不上,于是**全被过滤掉了**。

**对比一下[第 50 章](50-low-level-rustls.md)的写法:**

```rust
.unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
```

**`env!("CARGO_CRATE_NAME")` 是编译期宏,自动展开成当前 crate 的名字**,永远不会写错。[第 03 章](03-form.md)第一次介绍这段样板时就是这个写法,教程里其他 40 多章也都是。

> **更好的写法**:把硬编码的字符串换回 `env!("CARGO_CRATE_NAME")`:
>
> ```rust
> .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
> ```
>
> **这类"复制粘贴时忘了改名字"的 bug,失败方式是完全静默的**——程序照常运行,只是日志凭空消失。而且它很容易被误判成"日志配置太复杂我搞不定"，然后被人用 `RUST_LOG=debug` 绕过去,根因一直留着。
>
> **凡是"字符串里写了自己的名字"的地方,都应该问一句:能不能让编译器来填?**

**改完再跑:**

```bash
cargo run
```

```
 INFO example_low_level_native_tls: HTTPS server listening on [::1]:3000. To contact curl -k https://localhost:3000
```

**日志回来了。**

### 步骤 3:第二件不对劲的事——协议降级

**要做什么**:对比[第 50 章](50-low-level-rustls.md#步骤-4配置-alpn)的验证结果,看看 HTTP/2 去哪了。

**回顾[第 50 章](50-low-level-rustls.md)的结果:**

```bash
# 第 50 章(rustls + ALPN)
curl -ki https://[::1]:3000/ | head -1
```

```
HTTP/2 200
```

**本章的结果:**

```bash
curl -ki https://[::1]:3000/ | head -1
```

```
HTTP/1.1 200
```

**看看 ALPN 协商过程:**

```bash
curl -kv https://[::1]:3000/ 2>&1 | grep -i alpn
```

```
* ALPN: curl offers h2,http/1.1
* ALPN: server did not agree on a protocol. Uses default.
```

**服务端没有选任何协议。**

**原因**:`native_tls::TlsAcceptor::builder` **没有暴露服务端 ALPN 的配置**。builder 上只有 `identity`、`min_protocol_version`、`max_protocol_version` 这几项——**想设 ALPN 也没有地方设**。

(`native-tls` 的**客户端**侧有 `TlsConnector::builder().request_alpns(...)`,但服务端侧没有对应的。这是这个 crate 为了统一三个平台的 API 而取的最小公约数。)

**后果很明确:用 `native-tls` 做服务端,浏览器只会走 HTTP/1.1。**

[第 50 章](50-low-level-rustls.md#步骤-5故意改坏看-alpn-缺失的静默降级)那个"故意注释掉 ALPN"的实验,在这一章是**默认状态且改不了**。

 **这一条应该直接影响你的选型。** 如果你的服务面向浏览器、或者需要 gRPC(gRPC 强制要求 HTTP/2),**`native-tls` 直接出局**——除非你能在前面挂一个终止 TLS 的反向代理。

**顺便验证一下协议版本下限生效了:**

```bash
curl -kv --tlsv1.1 --tls-max 1.1 https://[::1]:3000/ 2>&1 | tail -5
```

握手应该失败。(如果你的 curl 版本已经完全移除了 TLS 1.1 支持,会在客户端侧就报错,同样说明问题。)

```bash
curl -kv https://[::1]:3000/ 2>&1 | grep -i "SSL connection using"
```

```
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
```

**实际协商出的是 TLS 1.3**,`min_protocol_version` 只是划了个下限。

## 完整代码

`src/main.rs`:

```rust
use axum::{extract::Request, routing::get, Router};
use hyper::body::Incoming;
use hyper_util::rt::{TokioExecutor, TokioIo};
use std::path::PathBuf;
use tokio::net::TcpListener;
use tokio_native_tls::{
    native_tls::{Identity, Protocol, TlsAcceptor as NativeTlsAcceptor},
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
                .unwrap_or_else(|_| "example_low_level_rustls=debug".into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let tls_acceptor = native_tls_acceptor(
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("key.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("cert.pem"),
    );

    let tls_acceptor = TlsAcceptor::from(tls_acceptor);
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
                warn!("error serving connection from {addr}: {err}");
            }
        });
    }
}

async fn handler() -> &'static str {
    "Hello, World!"
}

fn native_tls_acceptor(key_file: PathBuf, cert_file: PathBuf) -> NativeTlsAcceptor {
    let key_pem = std::fs::read_to_string(&key_file).unwrap();
    let cert_pem = std::fs::read_to_string(&cert_file).unwrap();

    let id = Identity::from_pkcs8(cert_pem.as_bytes(), key_pem.as_bytes()).unwrap();
    NativeTlsAcceptor::builder(id)
        // let's be modern
        .min_protocol_version(Some(Protocol::Tlsv12))
        .build()
        .unwrap()
}
```

>  **上面的 `"example_low_level_rustls=debug"` 是官方示例里的原样,是一个 bug**(步骤 2 分析过)。照抄的话启动日志不会出现。**建议改成 `format!("{}=debug", env!("CARGO_CRATE_NAME"))`。** 这里保留原样是为了和官方示例对照。

## 代价与边界

**服务端不支持 ALPN,浏览器只能走 HTTP/1.1。** 步骤 3 说了。**这是本章方案最实质的缺陷**,而且没有绕过的办法。

**日志过滤器写死了别的 crate 名。** 步骤 2 说了,`env!("CARGO_CRATE_NAME")` 能彻底避免。

**Linux 上并没有省掉 OpenSSL。** 前置知识那张表的第一行。构建镜像时仍然要带 OpenSSL 运行时,`FROM scratch` 用不了。

**跨平台行为不一致。** 同一份代码在 macOS 和 Linux 上底层是两套完全不同的实现,支持的加密套件、证书校验的严格程度、错误信息全都不同。**"本地好好的,服务器上握手失败"是这个方案的典型故障。** 排查时要先确认两边的底层实现和版本。

**加密套件无法定制。** builder 只能设协议版本上下限。有"必须禁用某某套件"这类合规要求时,`native-tls` 做不到,得直接用 [第 52 章](52-low-level-openssl.md)的 `openssl` 或[第 50 章](50-low-level-rustls.md)的 `rustls`。

**没有 mTLS。** `native-tls` 的服务端 builder 也没有客户端证书校验的配置。[第 50 章](50-low-level-rustls.md#动手练习)那个 mTLS 练习在这里做不了。

**握手错误被 `let ... else` 丢掉、没有握手超时、`accept().unwrap()` 会 panic、没有连接数上限、没有优雅停机、证书路径绑定 `CARGO_MANIFEST_DIR`、证书不会热重载、绑 `[::1]` 只监听 IPv6 回环** —— 这些[第 50 章](50-low-level-rustls.md#代价与边界)逐条讲过,本章原样存在,不再重复。

**`Identity::from_pkcs8` 对私钥格式挑剔。** 前置知识说过,PKCS#1 和 SEC1 格式的私钥直接失败。而失败信息通常只是一句笼统的解析错误,**不会告诉你"格式不对,请转成 PKCS#8"**。

**每条连接都重新构造了 `TlsStream`,但 `Identity` 只解析了一次。** 这一点是对的,值得指出:证书解析在启动时做一次,`TlsAcceptor` 的 clone 很便宜。**如果你写成每次 accept 都重读文件、重新解析证书,会是一个严重的性能问题**——RSA 4096 的密钥解析不便宜。

## 动手练习

**1. 修掉日志过滤器的 bug。**

提示:换成 `env!("CARGO_CRATE_NAME")`。验证:不设 `RUST_LOG` 直接 `cargo run`,启动日志应该出现。**顺手在你自己的项目里搜一遍有没有同类的硬编码字符串。**

**2. 把私钥转成 PKCS#1,亲眼看看失败长什么样。**

提示:

```bash
openssl rsa -in self_signed_certs/key.pem -out self_signed_certs/key-pkcs1.pem
head -1 self_signed_certs/key-pkcs1.pem     # -----BEGIN RSA PRIVATE KEY-----
```

把代码指向这个文件,观察 `Identity::from_pkcs8` 的 `unwrap()` panic 信息。**看看它有没有告诉你"格式不对"** —— 大概率没有。然后用前置知识那条命令转回来。**这个练习让你以后遇到这个错误时能一眼认出。**

**3. 加上握手错误详情和超时。**

提示:和[第 50 章](50-low-level-rustls.md#动手练习)的练习 1 一样。**做完对比一下两个库的握手错误信息谁更有用** —— 这也是选型时值得考虑的一点。

**4. 在两种系统上分别跑一遍,对比协商结果。**

提示:如果你有 macOS 和 Linux(或者用 Docker 起一个 Linux 容器),在两边都跑 `curl -kv https://[::1]:3000/ 2>&1 | grep "SSL connection using"`,对比协商出的 TLS 版本和加密套件。**大概率不一样。** 这就是"跨平台行为不一致"的实感。

**5. 用反向代理补上 HTTP/2。**

提示:既然 `native-tls` 服务端没有 ALPN,那就让前面的代理来做。用[第 43 章](43-reverse-proxy.md)的反向代理(或者直接用 Nginx)监听 443、开启 HTTP/2 和 ALPN,把请求转到这个明文/TLS 服务上。**做完你会发现:这正是「代价与边界」里反复说的"生产环境通常在代理层做 TLS"的又一个理由。**

## 小结与下一章预告

`native-tls` 是一个门面:Windows 上是 SChannel,macOS 上是 Security.framework,**Linux 上仍然是 OpenSSL** —— 所以"用它就不用装 OpenSSL 了"只对前两个平台成立。

换掉整套 TLS 实现,**accept 循环和 HTTP 处理代码一个字没改**,只换了两个函数。这再次印证了[第 50 章](50-low-level-rustls.md)那句话:`TlsStream` 也是 `AsyncRead + AsyncWrite`,上层不需要知道下面是谁在加密。

选型上,**Rust 生态的默认答案是 `rustls`**:纯 Rust、无系统依赖、交叉编译和容器构建省心、各平台行为一致。`native-tls` 的适用场景很窄——**合规要求必须用系统的密码模块,或者需要跟随系统的信任策略**。

而它有一条硬伤:**服务端 builder 不暴露 ALPN**。步骤 3 验证了后果——浏览器只能走 HTTP/1.1,`ALPN: server did not agree on a protocol`。[第 50 章](50-low-level-rustls.md)那个"故意注释掉 ALPN"的实验,在这一章是默认状态且改不了。**面向浏览器或需要 gRPC 的服务,这一条足以让 `native-tls` 出局。**

`min_protocol_version(Some(Protocol::Tlsv12))` 那一行不是可选的美化:`rustls` 压根没实现旧协议,而 `native-tls` 底下是系统的实现,**旧协议是否启用取决于系统**。这体现了两者的哲学差异——一个是"默认安全,不给不安全的选项",一个是"默认跟随系统,你自己负责"。

步骤 2 那个 bug 值得单独记住:官方示例的日志过滤器写死了 `example_low_level_rustls`,而本 crate 叫别的名字,**导致启动日志完全消失,却没有任何报错**。修法是 `env!("CARGO_CRATE_NAME")` 让编译器来填。**凡是"字符串里写了自己的名字"的地方,都该问一句能不能让编译器填。**

[第 52 章 low-level-openssl](52-low-level-openssl.md) 是这个三部曲的最后一章:**直接用 OpenSSL**。你会看到一个和前两章都不同的握手写法(需要手动 `Pin`),一个一行搞定安全配置的贴心设计(Mozilla 推荐配置档),以及一个可能让你措手不及的默认值——**它默认只允许 TLS 1.3**。章末会给出三种方案的完整对照表。
