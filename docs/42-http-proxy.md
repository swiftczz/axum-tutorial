# 第 42 章 http-proxy:CONNECT 隧道与连接升级

## 本章你将做出什么

一个**能被 curl 当代理用**的服务:

```bash
curl -v -x "127.0.0.1:3000" https://tokio.rs
```

这条命令会通过你写的服务访问 tokio.rs,而**你的服务从头到尾看不懂里面传的是什么**——因为那是加密的 TLS 流量,它只负责在两个 TCP 连接之间搬字节。

同一个端口还挂着一个普通的 axum 路由,`curl http://127.0.0.1:3000/` 返回 `Hello, World!`。

学完本章你将掌握:

- **正向代理与反向代理的区别**
- HTTP 请求行里的三种「请求目标」形式,以及代理为什么需要它们
- **`CONNECT` 方法与隧道(tunnel)**:代理是怎么处理 HTTPS 的
- **连接升级(upgrade)**:一个 HTTP 连接怎么变成裸 TCP 流
- `tokio::io::copy_bidirectional`:双向字节搬运
- **绕开 `axum::serve`,自己写 accept 循环**——顺带看清 `axum::serve` 到底封装了什么
- hyper 的 `Service` 和 tower 的 `Service` 是**两个不同的 trait**

>  **本章的代码是一个开放代理(open proxy),绝对不要部署到公网。** 「代价与边界」会详细说为什么。

## 前置知识

### 正向代理 vs 反向代理

这两个词长得像,做的事完全不同:

| | 正向代理(forward proxy) | 反向代理(reverse proxy) |
|---|---|---|
| 站在谁那边 | **客户端** | **服务端** |
| 谁知道它存在 | 客户端(要显式配置) | 客户端**不知道**,以为它就是服务器 |
| 目标地址谁决定 | 客户端(在请求里给出) | 代理自己(配置里写死) |
| 典型用途 | 翻墙、公司出网管控、抓包工具 | 负载均衡、TLS 卸载、API 网关 |
| 典型软件 | Squid、Charles、mitmproxy | Nginx、Envoy、HAProxy |

**本章做的是正向代理**,[第 43 章](43-reverse-proxy.md)做反向代理。

一句话记忆:**正向代理替客户端出门,反向代理替服务端开门。**

### 请求行里的「请求目标」有三种形式

平时抓包看到的请求行长这样:

```
GET /users/1 HTTP/1.1
Host: example.com
```

`/users/1` 这种只有路径的叫 **origin-form**。但 HTTP 规定的请求目标一共有四种形式,和代理有关的是这三种:

| 形式 | 长什么样 | 谁用 |
|---|---|---|
| **origin-form** | `GET /users/1 HTTP/1.1` | 直连服务器 |
| **absolute-form** | `GET http://example.com/users/1 HTTP/1.1` | **明文 HTTP 走正向代理** |
| **authority-form** | `CONNECT example.com:443 HTTP/1.1` | **HTTPS 走正向代理** |

为什么代理需要不一样的形式?因为**代理得知道你要去哪**。直连时目标就是"我自己",路径足够了;走代理时代理必须从请求里读出目标主机。

`Host` 头虽然也带主机名,但它是 HTTP/1.1 才有的,而且语义上是"这个请求想访问的虚拟主机",不是"请把这个请求转给谁"。所以标准专门规定了 absolute-form。

### CONNECT:代理怎么处理 HTTPS

这是本章的核心。明文 HTTP 走代理很简单——代理能看懂请求,改改转发就行。**但 HTTPS 不行**:

- 内容是加密的,代理看不懂,也就没法解析出「要转给谁」
- 就算能解密,TLS 证书是签给 `tokio.rs` 的,代理冒充不了

所以 HTTPS 走代理用的是**完全不同的机制**:

```
客户端 → 代理:  CONNECT tokio.rs:443 HTTP/1.1
代理:           (去连 tokio.rs:443)
代理 → 客户端:  HTTP/1.1 200 OK
                ---- 从这里开始,这条连接不再是 HTTP ----
客户端 ⇄ 代理 ⇄ tokio.rs:  纯字节双向搬运(里面是 TLS 握手和加密数据)
```

**`CONNECT` 请求说的是:"帮我连到这个地址,然后闭嘴当根管子。"**

代理回 `200` 之后,这条 TCP 连接上**再也没有 HTTP 了**。客户端在这根管子里做 TLS 握手、发 HTTP 请求、收响应,代理**一个字节都看不懂**,也不需要看懂。

这就是为什么公司代理能记录你访问了哪些域名(CONNECT 请求行里有),但看不到你在 HTTPS 网站上具体做了什么。

> 抓包工具(Charles、mitmproxy)能看到 HTTPS 内容,是因为它们**不做隧道**,而是自己伪造证书跟你握手、再以客户端身份跟真实服务器握手——这需要你在系统里信任它的根证书。那是另一套机制,叫 MITM(中间人)。

### 连接升级(upgrade)

HTTP 里有一类操作是「**这条连接不再讲 HTTP 了**」:

- WebSocket([第 34 章](34-websockets.md)):`Upgrade: websocket` + `101 Switching Protocols`
- HTTP/2 明文升级(h2c)
- **本章的 `CONNECT` 隧道**:`2xx` 之后连接变成裸流

hyper 把这件事抽象成 `hyper::upgrade`:

```rust
let upgraded = hyper::upgrade::on(req).await?;
// upgraded 是这条连接的裸读写句柄
```

**关键在于时序**:`hyper::upgrade::on(req)` 返回的 future,**必须等响应发出去之后才会完成**。因为「升级」的定义就是「响应发完,协议切换」。

这带来一个必然的写法:**你不能在 handler 里 `.await` 它,必须 `spawn` 出去**。

```rust
tokio::task::spawn(async move {
    let upgraded = hyper::upgrade::on(req).await?;   // 等响应发出去
    tunnel(upgraded, addr).await                     // 然后开始搬字节
});
Ok(Response::new(Body::empty()))                     // 立刻返回响应
```

在 handler 里直接 `.await` 会**死锁**:你在等响应发出去,而响应要等你的 handler 返回才能发。[第 34 章](34-websockets.md)的 `ws.on_upgrade(...)` 其实就是把这个 spawn 封装好了,只是当时没点破。

### hyper 的 Service ≠ tower 的 Service

本章会同时出现两个 `service_fn`,长得一样,来自不同的库:

| | `tower::service_fn` | `hyper::service::service_fn` |
|---|---|---|
| 产出的 trait | `tower::Service` | `hyper::service::HttpService` |
| `call` 的接收者 | **`&mut self`** | **`&self`** |
| axum 的 `Router` 实现了 |  这个 | 否 |
| hyper 的 `serve_connection` 要 | 否 |  这个 |

因为 hyper 的 `call` 只拿 `&self`,而 tower 的要 `&mut self`,所以中间那层永远要写一次 `clone()`:

```rust
let hyper_service = hyper::service::service_fn(move |request| {
    tower_service.clone().call(request)   // clone 出一个可变的
});
```

**这个 clone 不贵**——`Router` 的 clone 是内部 `Arc` 的引用计数 +1,和[第 41 章](41-reqwest-response.md)的 `reqwest::Client` 一个道理。

### `axum::serve` 到底是什么

前面 41 章一直在写:

```rust
axum::serve(listener, app).await.unwrap();
```

它做的事其实是一个循环:

```
loop {
    accept 一个 TCP 连接
    spawn 一个任务
        把 socket 包成 hyper 的 IO
        把 Router 包成 hyper 的 Service
        hyper 在这条连接上处理请求(自动协商 HTTP/1.1 和 HTTP/2,支持升级)
}
```

**本章要把这个循环自己写一遍。** 不是为了炫技,而是因为有些事 `axum::serve` 没暴露:

1. **`CONNECT` 请求得在进 `Router` 之前就分流掉。** authority-form 的 URI(`tokio.rs:443`)根本没有路径,`Router` 匹配不上任何路由。
2. **hyper 的连接级配置** ——本章用到的 `preserve_header_case`、`title_case_headers`,`axum::serve` 没有对应开关。

第 1 点严格说也能用 `Router::fallback` 绕过去,但**在路由之前分流**更能表达意图。[第 44 章](44-serve-with-hyper.md)会专门把这个 accept 循环讲透,本章先用起来。

### `match err {}`:消灭 Infallible

代码里会出现这么一行:

```rust
router_svc.oneshot(req).await.map_err(|err| match err {})
```

`Router` 的错误类型是 `Infallible`(永不出错)。`Infallible` 是一个**没有任何变体的枚举**,所以 `match err {}` —— 零个分支的 match —— 是合法的:编译器知道这里不可能到达,于是这个表达式的类型是「任意类型」。

**效果是把 `Result<T, Infallible>` 转成 `Result<T, 任何你需要的错误类型>`。** [第 41 章](41-reqwest-response.md)用 `Infallible` 是在构造流,这里是在消费它。

## 项目搭建

```bash
cargo new http-proxy
cd http-proxy
```

```bash
cargo add axum
cargo add hyper --features full
cargo add hyper-util
cargo add tokio --features full
cargo add tower --features make,util
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

这是本教程第一次**直接依赖 hyper**。前面 41 章里 hyper 一直是 axum 的间接依赖,轮不到我们直接调用。

- `hyper` 的 `full` feature 带上 `server`、`http1`、`client` 等全部能力
- `hyper-util` 提供 `TokioIo`(桥接 hyper 和 tokio 的 IO trait)
- `tower` 的 `util` feature 提供 `ServiceExt::oneshot`([第 21 章](21-testing.md)用过)

## 分步构建

### 步骤 1:不用 `axum::serve`,自己写 accept 循环

**要做什么**:先把一个最普通的 axum 路由跑起来,但**手写服务循环**。这一步的产出和 `axum::serve` 完全一样,目的是把封装拆开。

```rust
// src/main.rs
use axum::{body::Body, extract::Request, routing::get, Router};
use hyper::body::Incoming;
use hyper::server::conn::http1;
use hyper_util::rt::TokioIo;
use std::net::SocketAddr;
use tokio::net::TcpListener;
use tower::Service;
use tower::ServiceExt;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=trace,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let router_svc = Router::new().route("/", get(|| async { "Hello, World!" }));

    let tower_service = tower::service_fn(move |req: Request<_>| {
        let router_svc = router_svc.clone();
        let req = req.map(Body::new);
        async move { router_svc.oneshot(req).await.map_err(|err| match err {}) }
    });

    let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
        tower_service.clone().call(request)
    });

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {}", addr);

    let listener = TcpListener::bind(addr).await.unwrap();
    loop {
        let (stream, _) = listener.accept().await.unwrap();
        let io = TokioIo::new(stream);
        let hyper_service = hyper_service.clone();
        tokio::task::spawn(async move {
            if let Err(err) = http1::Builder::new()
                .preserve_header_case(true)
                .title_case_headers(true)
                .serve_connection(io, hyper_service)
                .await
            {
                println!("Failed to serve connection: {err:?}");
            }
        });
    }
}
```

代码不长,但每一行都在做一件事,拆开看:

**① 三层 Service 套娃**

```
hyper::service::service_fn   ← hyper 认这个
    └─ tower::service_fn     ← 我们的分流逻辑写在这里
           └─ Router         ← axum 的路由
```

中间那层 `tower::service_fn` 现在只是原样转发,**下一步就要在这里加 CONNECT 分流**。

**② `req.map(Body::new)`**

hyper 交给我们的是 `Request<Incoming>`(`Incoming` 是 hyper 的 body 类型),而 `Router` 要的是 `Request<axum::body::Body>`。`Request::map` 只换 body、保留其他部分,`Body::new` 做类型包装。

**③ `router_svc.oneshot(req)`**

`oneshot` 来自 `tower::ServiceExt`,意思是「用完就扔:等就绪、调一次、拿结果」。[第 21 章](21-testing.md)测试时用过同一个方法。这里每个请求 clone 一个 `Router` 再 `oneshot`,是把 `&mut self` 的问题绕过去的标准写法。

**④ `TokioIo::new(stream)`**

hyper 1.0 定义了自己的 `hyper::rt::Read` / `hyper::rt::Write`,**不直接用 tokio 的 `AsyncRead` / `AsyncWrite`**。这么做是为了让 hyper 不绑定任何一个运行时。`TokioIo` 就是两者之间的转接头,来自 `hyper-util`。

**⑤ `http1::Builder::new()`**

这里显式选了 **HTTP/1.1 only**。代理隧道本来就只在 HTTP/1.1 上讲得通(`CONNECT` 在 HTTP/2 里语义不同)。[第 44 章](44-serve-with-hyper.md)会用 `auto::Builder`,那个能同时支持 1 和 2。

- `preserve_header_case(true)`:hyper 默认把响应头名归一化成小写,这个开关让它**保留原始大小写**
- `title_case_headers(true)`:发出去的头写成 `Content-Type` 而不是 `content-type`

HTTP 头名本来就是大小写不敏感的,但**代理场景要尽量原样转发**——有些老旧的客户端或服务端会错误地依赖大小写。这两个开关就是为此存在的,`axum::serve` 没有暴露它们。

**⑥ `loop { accept; spawn }`**

每来一条连接就 spawn 一个任务,这样多个连接能并发处理。**这就是 `axum::serve` 内部做的事。**

**跑起来看**:

```bash
cargo run
```

```bash
curl -i http://127.0.0.1:3000/
```

```
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 13

Hello, World!
```

**注意响应头是 `Content-Type` 而不是 `content-type`** —— `title_case_headers(true)` 生效了。用 `axum::serve` 跑出来的是小写。这是本步骤唯一肉眼可见的差别。

> **更好的写法**:`println!("Failed to serve connection: {err:?}")` 和文件开头的 `tracing_subscriber` 初始化自相矛盾——既然配了结构化日志,这里就该用 `tracing::error!`。用 `println!` 的话,这条信息不带 span 上下文、不受 `RUST_LOG` 控制、也进不了日志收集系统。[第 17 章](17-tracing-aka-logging.md)讲过为什么这很重要。

> **更好的写法**:`listener.accept().await.unwrap()` 会在**任何** accept 错误上让整个服务 panic 退出。而 accept 的错误里有一大类是**可恢复的**(比如 `EMFILE`「文件描述符耗尽」、客户端在 accept 前就断开)。生产代码应该记日志并继续循环,只在真正致命的错误上退出:
>
> ```rust
> let (stream, _) = match listener.accept().await {
>     Ok(v) => v,
>     Err(err) => {
>         tracing::warn!(%err, "accept failed");
>         continue;
>     }
> };
> ```
>
> `axum::serve` 内部就是这么做的(它还会在连续出错时退避重试),自己写循环就得自己负责。

### 步骤 2:把 CONNECT 请求分流出去

**要做什么**:在中间那层 `tower::service_fn` 里加判断——`CONNECT` 走隧道逻辑,其他走 `Router`。

```rust
use axum::http::Method;
```

```rust
    let tower_service = tower::service_fn(move |req: Request<_>| {
        let router_svc = router_svc.clone();
        let req = req.map(Body::new);
        async move {
            if req.method() == Method::CONNECT {
                proxy(req).await
            } else {
                router_svc.oneshot(req).await.map_err(|err| match err {})
            }
        }
    });
```

**这就是"三层套娃"中间那层存在的理由。** 分流必须发生在 `Router` 之前:

```
CONNECT tokio.rs:443 HTTP/1.1
        └── 这个 URI 是 authority-form,没有路径
```

`Router` 匹配路由靠的是 `req.uri().path()`,而 authority-form 的 URI 取 path 会得到空字符串或 `/`——要么 404,要么误匹配到 `/` 路由返回 `Hello, World!`。**必须在进 Router 之前拦下来。**

先放一个占位:

```rust
use axum::response::Response;

async fn proxy(req: Request) -> Result<Response, hyper::Error> {
    tracing::trace!(?req);
    Ok(Response::new(Body::empty()))
}
```

注意两个分支的返回类型要一致:`proxy` 返回 `Result<Response, hyper::Error>`,所以另一分支的 `map_err(|err| match err {})` 就被推断成产出 `hyper::Error`——前置知识讲的 `Infallible` 消除在这里派上用场。

**跑起来看**(现在还没有真正的隧道,但能看到分流生效):

```bash
cargo run
```

```bash
curl -v -x "127.0.0.1:3000" https://tokio.rs
```

curl 的输出里能看到:

```
* CONNECT tunnel: HTTP/1.1 negotiated
> CONNECT tokio.rs:443 HTTP/1.1
> Host: tokio.rs:443
>
< HTTP/1.1 200 OK
< content-length: 0
* CONNECT phase completed
* CONNECT tunnel established, response 200
* ... TLS handshake ...
```

然后**卡住或者报 TLS 错误**——因为我们回了 200 说隧道建好了,实际上什么都没建。

服务端日志里能看到 `tracing::trace!(?req)` 打出的请求,URI 是 `tokio.rs:443`,method 是 `CONNECT`。**分流成功了。**

### 步骤 3:发起升级

**要做什么**:从 CONNECT 请求里取出目标地址,spawn 一个任务等连接升级。

```rust
use axum::{http::StatusCode, response::IntoResponse};
```

```rust
async fn proxy(req: Request) -> Result<Response, hyper::Error> {
    tracing::trace!(?req);

    if let Some(host_addr) = req.uri().authority().map(|auth| auth.to_string()) {
        tokio::task::spawn(async move {
            match hyper::upgrade::on(req).await {
                Ok(upgraded) => {
                    if let Err(e) = tunnel(upgraded, host_addr).await {
                        tracing::warn!("server io error: {}", e);
                    };
                }
                Err(e) => tracing::warn!("upgrade error: {}", e),
            }
        });

        Ok(Response::new(Body::empty()))
    } else {
        tracing::warn!("CONNECT host is not socket addr: {:?}", req.uri());
        Ok((
            StatusCode::BAD_REQUEST,
            "CONNECT must be to a socket address",
        )
            .into_response())
    }
}
```

**`req.uri().authority()`** —— authority-form 的 URI(`tokio.rs:443`)整个就是一个 authority,所以这里能拿到 `Some("tokio.rs:443")`。如果客户端发了一个 origin-form 的 CONNECT(不合规),就是 `None`,回 400。

**`tokio::task::spawn` 是必须的,不是优化。** 前置知识讲过:`hyper::upgrade::on(req)` 要等响应发出去才完成,而响应要等 `proxy` 返回才发。**在这里直接 `.await` 就是死锁**,请求会永远挂着。

写成 spawn 之后时序是:

```
proxy 返回 200        →  hyper 把 200 写进 socket
                      →  连接进入 upgraded 状态
spawn 的任务被唤醒     →  拿到 upgraded 句柄
                      →  开始搬字节
```

**`Ok(Response::new(Body::empty()))` 回的是 200。** RFC 9110 规定:CONNECT 的 **2xx 响应表示隧道已建立**,响应不该有 body。这里用空 body 的默认 200,符合规范。

> **更好的写法**:**200 回得太早了。** 现在的顺序是「先回 200,再去连上游」。如果上游根本连不上(域名不存在、端口不通、被防火墙拦),客户端已经收到「隧道建好了」,然后开始 TLS 握手,却发现管子那头是空的——客户端看到的是一个莫名其妙的连接重置,而不是清晰的错误。
>
> 正确顺序是**先 `TcpStream::connect` 成功,再回 200;连不上就回 `502 Bad Gateway`**:
>
> ```rust
> let server = match TcpStream::connect(&host_addr).await {
>     Ok(s) => s,
>     Err(err) => {
>         tracing::warn!(%err, "upstream connect failed");
>         return Ok((StatusCode::BAD_GATEWAY, "upstream unreachable").into_response());
>     }
> };
> tokio::task::spawn(async move { /* upgrade + copy_bidirectional(server) */ });
> Ok(Response::new(Body::empty()))
> ```
>
> 代价是客户端要多等一个建连的往返时间才收到 200。真实代理(Squid、Envoy)都是这么做的——**清晰的错误比快 50 毫秒重要**。这和[第 41 章](41-reqwest-response.md)那个「上游失败该回 502 而不是 400」是同一类问题。

### 步骤 4:搬字节

**要做什么**:实现 `tunnel`,连上游 + 双向拷贝。

```rust
use hyper::upgrade::Upgraded;
use tokio::net::TcpStream;
```

```rust
async fn tunnel(upgraded: Upgraded, addr: String) -> std::io::Result<()> {
    let mut server = TcpStream::connect(addr).await?;
    let mut upgraded = TokioIo::new(upgraded);

    let (from_client, from_server) =
        tokio::io::copy_bidirectional(&mut upgraded, &mut server).await?;

    tracing::debug!(
        "client wrote {} bytes and received {} bytes",
        from_client,
        from_server
    );

    Ok(())
}
```

**整个隧道就三行。** 这是本章最反差的地方——前面铺垫了那么多,真正"代理"的部分短得不像话。因为**隧道本来就不需要理解内容**。

**`TokioIo::new(upgraded)` 又出现了,但方向反了。** 步骤 1 里是把 tokio 的 socket 转成 hyper 的 IO;这里是把 hyper 的 `Upgraded` 转回 tokio 的 `AsyncRead + AsyncWrite`,因为 `copy_bidirectional` 是 tokio 的函数。**同一个转接头,两个方向都能用。**

**`tokio::io::copy_bidirectional(a, b)`** 做的事:

```
a 读到的 → 写给 b
b 读到的 → 写给 a
```

两个方向**并发**进行,直到**两边都 EOF**。返回 `(a→b 的字节数, b→a 的字节数)`。

它比自己写两个 `tokio::io::copy` 再 `join` 好在:正确处理了半关闭(一方 `shutdown` 写端但还在读)、内部有缓冲、并且会在拷贝结束时 flush 并 shutdown 对端写端。

**注意 `upgraded` 里流动的是什么**:TLS 握手报文、加密的 HTTP 请求和响应。**我们的代码完全不 care**,它看到的只是 `&[u8]`。这就是隧道的本质。

**跑起来看**:

```bash
cargo run
```

```bash
curl -v -x "127.0.0.1:3000" https://tokio.rs
```

这次应该**真的把 tokio.rs 的首页拉下来了**。curl 的详细输出里能看到完整链路:

```
* Connected to 127.0.0.1 (127.0.0.1) port 3000
* CONNECT tunnel: HTTP/1.1 negotiated
> CONNECT tokio.rs:443 HTTP/1.1
> Host: tokio.rs:443
> Proxy-Connection: Keep-Alive
>
< HTTP/1.1 200 OK
* CONNECT phase completed
* CONNECT tunnel established, response 200
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
...
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* Server certificate:
*  subject: CN=tokio.rs
...
> GET / HTTP/2
> Host: tokio.rs
<
< HTTP/2 200
<
<!DOCTYPE html>...
```

**逐段读一下这个输出,整章的知识点全在里面:**

- `CONNECT tokio.rs:443` —— authority-form 请求目标
- `HTTP/1.1 200 OK` —— 隧道建立(不是 tokio.rs 返回的,是**我们的服务**返回的)
- `TLS handshake` —— **在隧道里**握手,我们的代理看不懂
- `Server certificate: CN=tokio.rs` —— **证书是 tokio.rs 的**,证明没有中间人
- `GET / HTTP/2` —— 隧道里跑的是 HTTP/2,我们的代理更看不懂了

服务端日志:

```
DEBUG example_http_proxy: client wrote 1234 bytes and received 56789 bytes
```

这是 `copy_bidirectional` 的返回值——**代理对这次通信的全部认知,就是两个字节数**。

### 步骤 5:故意改坏,理解 `.with_upgrades()`

**要做什么**:步骤 1 到 4 的代码里少了一样东西,现在补上并解释它。

如果你严格照着前面写,`curl -x ... https://tokio.rs` 大概率**还是不通**——`hyper::upgrade::on(req)` 会一直挂着,日志里既没有 `upgrade error` 也没有 `client wrote ... bytes`。

原因是 `serve_connection` **不处理升级**。要换成:

```rust
            if let Err(err) = http1::Builder::new()
                .preserve_header_case(true)
                .title_case_headers(true)
                .serve_connection(io, hyper_service)
                .with_upgrades()          // ← 加上这一行
                .await
```

**`.with_upgrades()` 的作用**:让 hyper 在响应完成后**不要关闭连接、也不要继续读下一个 HTTP 请求**,而是把这条连接的所有权交给 `hyper::upgrade::on` 拿到的那个句柄。

不加的话,hyper 处理完 200 响应就认为这个请求结束了,连接被它自己接管着等下一个请求,`upgrade::on` 的 future 永远不会完成。

**加回去再试一次,隧道就通了。**

**把它删掉再跑一次**,亲眼确认现象:

- curl 卡在 `CONNECT tunnel established` 之后,最终 TLS 握手超时
- 服务端**没有任何错误日志**——这是最坑的地方,**它是静默失败**

这类"少一个方法调用,功能静默消失"的问题,和[第 36 章](36-websockets-http2.md)那个漏掉 `enable_connect_protocol()` 的坑是一模一样的病:**协议层的开关没打开,表现是挂起而不是报错**。

> `axum::serve` 内部用的是 `serve_connection_with_upgrades`(`auto::Builder` 上的对应方法),**默认就支持升级**——所以[第 34 章](34-websockets.md)的 WebSocket 不用你操心这个。自己写循环就要自己记得。

### 步骤 6:一个意外发现——明文 HTTP 并没有被代理

**要做什么**:试试用这个代理访问一个 `http://` 地址,看看会发生什么。

```bash
curl -v -x "127.0.0.1:3000" http://tokio.rs
```

**你会看到**:

```
> GET http://tokio.rs/ HTTP/1.1
> Host: tokio.rs
>
< HTTP/1.1 200 OK
< Content-Type: text/plain; charset=utf-8
<
Hello, World!
```

**返回的是 `Hello, World!`,不是 tokio.rs 的首页。**

为什么?curl 对明文 HTTP 走代理用的是 **absolute-form**(`GET http://tokio.rs/ HTTP/1.1`),不是 CONNECT。所以它没被分流,进了 `Router`。而 `Router` 匹配路由只看 `uri().path()`,absolute-form 的 path 就是 `/`——**正好命中了那条 `Hello, World!` 路由**。

**这不是 bug,是这个示例的边界:它只代理 HTTPS(CONNECT),不代理明文 HTTP。** 官方示例的注释里只给了 `https://tokio.rs` 的命令,也印证了这一点。

要支持明文 HTTP 代理,得再加一层判断:**URI 是不是 absolute-form(有 scheme 和 authority)?是的话就用 HTTP 客户端转发出去,而不是交给 `Router`**。这正好是[第 43 章](43-reverse-proxy.md)要做的事(虽然那一章的目标地址是写死的)。

这个发现值得记住:**「能当代理用」和「能代理所有流量」是两回事。** 读别人的代理代码时,先问一句"它处理 absolute-form 吗"。

## 完整代码

`src/main.rs`:

```rust
use axum::{
    body::Body,
    extract::Request,
    http::{Method, StatusCode},
    response::{IntoResponse, Response},
    routing::get,
    Router,
};

use hyper::body::Incoming;
use hyper::server::conn::http1;
use hyper::upgrade::Upgraded;
use std::net::SocketAddr;
use tokio::net::{TcpListener, TcpStream};
use tower::Service;
use tower::ServiceExt;

use hyper_util::rt::TokioIo;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=trace,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let router_svc = Router::new().route("/", get(|| async { "Hello, World!" }));

    let tower_service = tower::service_fn(move |req: Request<_>| {
        let router_svc = router_svc.clone();
        let req = req.map(Body::new);
        async move {
            if req.method() == Method::CONNECT {
                proxy(req).await
            } else {
                router_svc.oneshot(req).await.map_err(|err| match err {})
            }
        }
    });

    let hyper_service = hyper::service::service_fn(move |request: Request<Incoming>| {
        tower_service.clone().call(request)
    });

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {}", addr);

    let listener = TcpListener::bind(addr).await.unwrap();
    loop {
        let (stream, _) = listener.accept().await.unwrap();
        let io = TokioIo::new(stream);
        let hyper_service = hyper_service.clone();
        tokio::task::spawn(async move {
            if let Err(err) = http1::Builder::new()
                .preserve_header_case(true)
                .title_case_headers(true)
                .serve_connection(io, hyper_service)
                .with_upgrades()
                .await
            {
                println!("Failed to serve connection: {err:?}");
            }
        });
    }
}

async fn proxy(req: Request) -> Result<Response, hyper::Error> {
    tracing::trace!(?req);

    if let Some(host_addr) = req.uri().authority().map(|auth| auth.to_string()) {
        tokio::task::spawn(async move {
            match hyper::upgrade::on(req).await {
                Ok(upgraded) => {
                    if let Err(e) = tunnel(upgraded, host_addr).await {
                        tracing::warn!("server io error: {}", e);
                    };
                }
                Err(e) => tracing::warn!("upgrade error: {}", e),
            }
        });

        Ok(Response::new(Body::empty()))
    } else {
        tracing::warn!("CONNECT host is not socket addr: {:?}", req.uri());
        Ok((
            StatusCode::BAD_REQUEST,
            "CONNECT must be to a socket address",
        )
            .into_response())
    }
}

async fn tunnel(upgraded: Upgraded, addr: String) -> std::io::Result<()> {
    let mut server = TcpStream::connect(addr).await?;
    let mut upgraded = TokioIo::new(upgraded);

    let (from_client, from_server) =
        tokio::io::copy_bidirectional(&mut upgraded, &mut server).await?;

    tracing::debug!(
        "client wrote {} bytes and received {} bytes",
        from_client,
        from_server
    );

    Ok(())
}
```

## 代价与边界

** 这是一个开放代理,绝对不能部署到公网。** 这条要单独列在最前面,因为它的危害远超其他所有问题:

- **任何人都能用它连任何地址。** `tunnel()` 拿到 `host_addr` 就直接 `TcpStream::connect`,**没有任何白名单、认证、限流**。
- **SSRF(服务端请求伪造)。** 攻击者可以 `CONNECT 169.254.169.254:80`(云厂商的元数据服务)偷取你的实例凭证,或者 `CONNECT 10.0.0.5:6379` 直连你内网的 Redis。**代理在你的内网里,而攻击者在外面。**
- **被用作攻击跳板。** 别人拿你的 IP 去扫描、爆破、发垃圾邮件(`CONNECT smtp.xxx:25`),追查到的是**你**。
- **成为流量中转站。** 带宽账单是你的。

公网上的扫描器**几分钟内**就会发现一个开放代理。真实的正向代理必须有:身份认证(`Proxy-Authorization` 头)、目标地址白名单或黑名单(至少屏蔽私有网段和 link-local 地址)、端口限制(通常只允许 443)、并发和带宽限制。

**没有超时。** `TcpStream::connect` 没有超时,`copy_bidirectional` 也没有空闲超时。一个客户端可以建立隧道然后什么都不做,**永久占用两个 TCP 连接和一个 task**。攒够几万个就是拒绝服务。

**没有连接数上限。** accept 循环无脑 spawn,来多少收多少。文件描述符耗尽时 `accept().await.unwrap()` 直接 panic(步骤 1 的「更好的写法」提过)。

**目标地址没有校验。** `req.uri().authority()` 拿到什么就连什么。除了上面说的内网地址,还有一个细节:**没有检查端口**。`CONNECT tokio.rs:22` 会给你开一条 SSH 隧道。

**没有代理认证。** RFC 9110 定义了 `407 Proxy Authentication Required` 和 `Proxy-Authorization` 头,这个示例完全没实现。

**HTTP/1.1 only。** `http1::Builder` 写死了。`CONNECT` 在 HTTP/2 里有不同的语义(扩展 CONNECT,[第 36 章](36-websockets-http2.md)见过),这个实现不支持。

**明文 HTTP 不被代理。** 步骤 6 详细说了。

**日志用了 `println!`。** 步骤 1 的「更好的写法」提过。

**`tracing::trace!(?req)` 会打出完整请求,包括头。** 正向代理的请求头里可能有 `Proxy-Authorization`(将来加了认证的话),**打进日志就是凭证泄露**。[第 18 章](18-print-request-response.md)讲过同样的问题。

**没有优雅停机。** `loop` 里 spawn 出去的连接任务,进程退出时会被直接砍掉,正在传输的隧道全部中断。[第 45 章](45-graceful-shutdown.md)会补上这一块。

## 动手练习

**1. 先连上游、再回 200。**

提示:按步骤 3 的「更好的写法」重构,把 `TcpStream::connect` 挪到 `proxy` 里,连不上就回 502。验证:`curl -v -x "127.0.0.1:3000" https://does-not-exist.invalid` 改造前会在 TLS 握手阶段莫名失败,改造后应该直接看到 `HTTP/1.1 502 Bad Gateway`。**对比一下两种错误哪个更好排查。**

**2. 加一个目标地址黑名单,堵住 SSRF。**

提示:解析 `host_addr` 里的主机部分,如果是 IP 且落在私有网段(`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`、`127.0.0.0/8`、`169.254.0.0/16`)就回 403。验证:`curl -x "127.0.0.1:3000" https://169.254.169.254` 应该被拒。**想一想这个防护有什么绕过方式**(答案:域名解析到内网 IP——所以真正的防护要在 `connect` 之后检查对端地址,或者用受控的 DNS 解析器)。

**3. 给隧道加空闲超时。**

提示:用 `tokio::time::timeout` 包住 `copy_bidirectional`,或者更精细一点,自己写两个 `copy` 并在每次读之间打点。做完试着建立一条隧道后什么都不发,确认服务端会在超时后清理掉。

**4. 支持明文 HTTP 代理(absolute-form)。**

提示:在分流逻辑里加一个分支——`req.uri().scheme().is_some() && req.uri().authority().is_some()` 时,用 `hyper_util` 的客户端([第 43 章](43-reverse-proxy.md)会讲)把请求转发出去。做完 `curl -x "127.0.0.1:3000" http://tokio.rs` 应该返回真正的 tokio.rs 首页,而不是 `Hello, World!`。**这个练习把步骤 6 的边界补上了。**

**5. 加代理认证。**

提示:CONNECT 分支里先检查 `Proxy-Authorization` 头,没有或不对就返回 `407 Proxy Authentication Required` 并带上 `Proxy-Authenticate: Basic realm="proxy"`。用 `curl -x "user:pass@127.0.0.1:3000" https://tokio.rs` 验证。**注意别把这个头打进日志。**

## 小结与下一章预告

正向代理替客户端出门,反向代理替服务端开门。**明文 HTTP 走代理用 absolute-form 请求目标,HTTPS 走代理用 `CONNECT` + 隧道**——后者是本章的主题。

`CONNECT` 的语义是「帮我连到这个地址,然后闭嘴当根管子」。代理回 2xx 之后,**这条连接上就没有 HTTP 了**,只有裸字节。所以代理能记录你访问了哪个域名(CONNECT 请求行里有),但看不到里面的内容——`tunnel` 函数只知道两个字节数。

实现上有三个必须记住的点:

1. **`hyper::upgrade::on(req)` 必须 spawn,不能在 handler 里 `.await`** ——它要等响应发出去才完成,而响应要等 handler 返回。直接 await 就是死锁。
2. **`serve_connection` 后面要跟 `.with_upgrades()`** ——步骤 5 证明了少这一行会**静默挂起**,没有任何错误日志。
3. **`TokioIo` 是 hyper IO 和 tokio IO 之间的转接头**,两个方向都要用到。

本章第一次绕开 `axum::serve` 自己写 accept 循环,顺带看清了它封装的东西:accept、spawn、`TokioIo`、`hyper::service::service_fn`、连接级 builder。也看到了 hyper 的 `Service`(`&self`)和 tower 的 `Service`(`&mut self`)是两个 trait,中间那次 `clone()` 就是为它们的差异买单。

步骤 6 那个意外发现值得记住:**这个代理只代理 HTTPS,明文 HTTP 请求会直接命中本地路由返回 `Hello, World!`。** 「能当代理用」不等于「能代理所有流量」。

最后再强调一次:**这是开放代理,不要部署到公网。** SSRF、攻击跳板、带宽盗用,「代价与边界」里逐条列了。

本章的代理**不解析内容**。[第 43 章 reverse-proxy](43-reverse-proxy.md) 走向另一端:**目标地址写死,但每个请求都要被解析、改写、再转发**。你会看到怎么改写请求的 URI、怎么用 `hyper-util` 的客户端把 `Request` 打出去,以及一个和[第 41 章](41-reqwest-response.md)呼应的老问题——**响应头哪些能抄、哪些不能**。
