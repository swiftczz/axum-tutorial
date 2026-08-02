# 第 41 章 reqwest-response:把别人的响应转发出去

## 本章你将做出什么

一个**自己也当客户端**的服务:

```
GET /stream    每秒吐一个数字,共 5 个(慢速流)
GET /          用 reqwest 去请求 /stream,把响应原样转发给调用方
```

关键在于:`/` 转发时**不等对方传完**,而是边收边发。用 `curl -N` 访问 `/`,你会看到数字**一个一个蹦出来**,而不是等 5 秒一次性出现。

学完本章你将掌握:

- `reqwest` 的基本用法,以及 `Client` 为什么必须复用
- **`.bytes()` 与 `.bytes_stream()` 的区别**——本章的核心分歧点
- `Body::from_stream`:把任意 `Stream` 变成 HTTP 响应体
- reqwest 的 `Response` 和 axum 的 `Response` 是**两个不同的类型**,以及怎么转换
- `TraceLayer::on_body_chunk`:观测流式响应的每一块
- 转发响应头时有哪些**不能盲抄**的坑

## 前置知识

### 服务端也是客户端

前面 40 章里,响应都是服务端**自己产生**的:从内存、从模板、从数据库、从磁盘。但真实系统里有大量场景是**服务端去问别人**:

- **BFF / 聚合层**:一个请求进来,后端并发调用三五个微服务,拼装结果返回
- **反向代理**:把请求原样转给上游([第 43 章](43-reverse-proxy.md))
- **第三方 API**:调支付、调地图、调对象存储
- **对象下载**:文件存在 S3,服务端取回来转给浏览器

这一章讲的是最后一类的骨架:**取回来,转出去**。

### reqwest:Rust 世界最常用的 HTTP 客户端

```rust
let client = reqwest::Client::new();
let res = client.get("http://example.com").send().await?;
println!("{}", res.status());
```

有两点必须一开始就说清楚:

**1. `Client` 必须复用,不能每次请求 new 一个。**

`Client` 内部持有**连接池**、DNS 缓存和 TLS 配置。每次 `Client::new()` 都是一个全新的池,意味着每个请求都要重新建 TCP 连接、重新握手。高并发下这是灾难。

好在 `Client` 内部是 `Arc`,**克隆非常便宜**(只是引用计数 +1)。所以正确做法是:建一个,放进共享状态,到处 clone。这正好和[第 22 章](22-key-value-store.md)的 `State` 对上——而且比数据库连接池([第 24 章](24-sqlx-postgres.md))更简单,因为不需要 `Arc<...>` 包一层,`Client` 自己就是。

**2. 本章的 reqwest 关掉了默认 feature,所以不支持 HTTPS。**

这里只会请求 `http://127.0.0.1:3000`,用不上 TLS,关掉能少编译一大堆东西。真实项目里你几乎肯定要加回来——「代价与边界」会说怎么加。

### `.bytes()` vs `.bytes_stream()`:本章的分歧点

拿到 `reqwest::Response` 之后,有两种取 body 的方式:

```rust
let body = res.bytes().await?;          // ① 等全部收完,得到一个 Bytes
let stream = res.bytes_stream();        // ② 得到一个 Stream<Item = Result<Bytes, Error>>
```

| | `.bytes()` | `.bytes_stream()` |
|---|---|---|
| 何时返回 | 上游**传完之后** | **立刻** |
| 内存占用 | 整个响应体 | 一小块 |
| 首字节延迟 | = 上游总耗时 | ≈ 上游首字节延迟 |
| 代码复杂度 | 更简单 | 稍多一步 |

这和[第 40 章](40-stream-to-file.md#缓冲-vs-流式)的「缓冲 vs 流式」是同一个矛盾,只是方向反了:上一章是**流进磁盘**,这一章是**流向客户端**。

**转发场景下必须用 `.bytes_stream()`**,理由和上一章一样:

- 上游返回 1GB 文件,`.bytes()` 就是 1GB 内存
- 上游是 SSE([第 33 章](33-sse.md))这种**永不结束**的流,`.bytes()` 永远不返回

第二条尤其致命——用 `.bytes()` 转发一个 SSE 端点,程序会直接挂死。

### `Body::from_stream`:Stream 变成响应体

[第 33 章](33-sse.md)用过 `Sse<S>`,那是 SSE 专用的封装。更底层的通用工具是:

```rust
axum::body::Body::from_stream(stream)
```

它接受任何 `Stream<Item = Result<T, E>>`,其中 `T: Into<Bytes>`、`E: Into<BoxError>`,产出一个 `Body`。

**`Body` 本身实现了 `IntoResponse`**,所以 handler 可以直接返回它:

```rust
async fn handler() -> Body {
    Body::from_stream(some_stream)
}
```

axum 会自动补上 `200 OK`。这是本章 `/stream` 的写法。

> **顺带一提**:`Body::from_stream` 产出的响应**没有 `Content-Length`**(长度事先未知),HTTP/1.1 下会走 `Transfer-Encoding: chunked`。这是流式响应的固有特征,不是 bug。

### 两个 `Response`,不是一回事

这是本章最容易绕晕的地方:

| | `reqwest::Response` | `axum::response::Response` |
|---|---|---|
| 代表 | 我**收到的**响应 | 我**要发出的**响应 |
| body 类型 | reqwest 自己的 | `axum::body::Body` |
| 能做什么 | `.status()` `.headers()` `.bytes_stream()` `.json()` | 被 axum 写回 socket |

两者的 `status` 和 `headers` 其实是**同一套 `http` crate 的类型**(`StatusCode`、`HeaderMap`),可以直接搬。**只有 body 需要转换**。

所以转发的逻辑就是三步:抄状态码、抄响应头、把 body 从流转过去。

### `Response::builder`:手工造响应

前面章节都是靠 `IntoResponse` 自动生成响应。这一章要**精确控制**状态码和全部响应头,得用底层的构造器:

```rust
let mut builder = Response::builder().status(some_status);
*builder.headers_mut().unwrap() = some_header_map;   // 整个 HeaderMap 替换
let response = builder.body(some_body).unwrap();
```

`headers_mut()` 返回 `Option<&mut HeaderMap>`——如果 builder 之前的某一步出过错(比如状态码非法),它会是 `None`。这里刚建出来,一定是 `Some`。

### `tokio_stream` 的组合子

造慢速流要用两个组合子:

```rust
tokio_stream::iter(0..5)              // 把普通迭代器变成 Stream
    .throttle(Duration::from_secs(1)) // 每项之间至少间隔 1 秒
```

`throttle` [第 33 章](33-sse.md#步骤-3配上前端页面)用过。`tokio_stream::iter` 是新的:它把任何 `IntoIterator` 变成一个立刻就绪的 `Stream`,做演示数据源非常方便。

## 项目搭建

```bash
cargo new reqwest-response
cd reqwest-response
```

```bash
cargo add axum
cargo add reqwest --no-default-features --features stream
cargo add tokio --features full
cargo add tokio-stream
cargo add tower-http --features trace
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`--no-default-features --features stream` 这一串的意思是:**只要流式读 body 的能力,不要 TLS、不要 gzip、不要 cookie**。默认 feature 会拉进 `native-tls` 或 `rustls`,编译时间显著增加,而本章用不上。

`tokio-stream` 的默认 feature 里带 `time`,`throttle` 就在里面,不用额外指定。

## 分步构建

### 步骤 1:先做出那个"慢"的数据源

**要做什么**:先不管转发,把 `/stream` 写出来并确认它真的是**一秒一个**。转发能不能保住流式,得先有个流式的东西可保。

```rust
// src/main.rs
use axum::{body::Body, routing::get, Router};
use std::{convert::Infallible, time::Duration};
use tokio_stream::StreamExt;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app = Router::new().route("/stream", get(stream_some_data));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn stream_some_data() -> Body {
    let stream = tokio_stream::iter(0..5)
        .throttle(Duration::from_secs(1))
        .map(|n| n.to_string())
        .map(Ok::<_, Infallible>);
    Body::from_stream(stream)
}
```

**三个 `map` 各干各的:**

```rust
tokio_stream::iter(0..5)              // Stream<Item = i32>
    .throttle(Duration::from_secs(1)) // Stream<Item = i32>,但节流了
    .map(|n| n.to_string())           // Stream<Item = String>
    .map(Ok::<_, Infallible>)         // Stream<Item = Result<String, Infallible>>
```

最后一个 `map` 是为了满足 `Body::from_stream` 的签名——它要 `Result`,因为真实的流(比如读文件、读网络)是可能出错的。这个流不会错,所以错误类型用 `Infallible`(「永不发生」的类型,[第 16 章](16-dependency-injection.md)见过)。

`Ok::<_, Infallible>` 这个写法是把 `Ok` 当函数用,并显式标注错误类型。等价于 `.map(|s| Ok::<String, Infallible>(s))`。

**`String` 能进 `Body::from_stream`**,因为 `String: Into<Bytes>`。

**跑起来看**:

```bash
cargo run
```

另开一个终端:

```bash
curl -N http://127.0.0.1:3000/stream
```

**`-N` 是关键**(等价于 `--no-buffer`)。curl 默认会缓冲输出,加上它才能看到数据实时到达。你应该看到 `0`、`1`、`2`、`3`、`4` **一秒蹦一个**,总共约 4 秒(第一个立刻出来,后面四个各等一秒)。

不加 `-N` 对比一下:

```bash
curl http://127.0.0.1:3000/stream
```

大概率还是逐个出现(curl 对非 tty 输出才强制缓冲),但**别依赖这个**——养成流式测试加 `-N` 的习惯。

看看响应头:

```bash
curl -i -N http://127.0.0.1:3000/stream
```

```
HTTP/1.1 200 OK
transfer-encoding: chunked
date: ...
```

**没有 `content-length`,有 `transfer-encoding: chunked`** —— 前置知识说过的流式特征。

### 步骤 2:把 reqwest 的 Client 放进共享状态

**要做什么**:加上 `Client`,用 `with_state` 共享。

```rust
use axum::{body::Body, extract::State, routing::get, Router};
use reqwest::Client;
```

```rust
    let client = Client::new();

    let app = Router::new()
        .route("/", get(stream_reqwest_response))
        .route("/stream", get(stream_some_data))
        .with_state(client);
```

**`Client` 直接当 state,不用 `Arc` 包。** 前置知识说过它内部已经是 `Arc` 了。这是它和 `Arc<RwLock<HashMap>>`([第 22 章](22-key-value-store.md))那类状态的区别——后者需要你自己加共享和同步,`Client` 自带。

暂时先放一个占位 handler,下一步填实:

```rust
async fn stream_reqwest_response(State(_client): State<Client>) -> Body {
    Body::empty()
}
```

> **更好的写法**:官方示例把上游地址 `http://127.0.0.1:3000/stream` 直接硬编码在 handler 里。生产代码应该把它做成配置(环境变量或配置文件),并连同 `Client` 一起放进一个自定义的 state 结构体——[第 16 章](16-dependency-injection.md)讲的依赖注入正是为此。硬编码地址意味着换环境就得重新编译。

**跑起来看**:

```bash
cargo run
```

```
listening on 127.0.0.1:3000
```

服务照常启动,`/stream` 返回空 body——占位 handler 的预期行为。真正的验证在下一步:填上转发逻辑后,这条路由会原样吐出上游的 JSON。

### 步骤 3:转发响应

**要做什么**:发起请求,把 reqwest 的响应转成 axum 的响应。

```rust
use axum::{
    body::Body,
    extract::State,
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::get,
    Router,
};
```

```rust
async fn stream_reqwest_response(State(client): State<Client>) -> Response {
    let reqwest_response = match client.get("http://127.0.0.1:3000/stream").send().await {
        Ok(res) => res,
        Err(err) => {
            tracing::error!(%err, "request failed");
            return (StatusCode::BAD_REQUEST, Body::empty()).into_response();
        }
    };

    let mut response_builder = Response::builder().status(reqwest_response.status());
    *response_builder.headers_mut().unwrap() = reqwest_response.headers().clone();
    response_builder
        .body(Body::from_stream(reqwest_response.bytes_stream()))
        // This unwrap is fine because the body is empty here
        .unwrap()
}
```

**返回类型从 `Body` 改成了 `Response`**,因为现在有两条分支返回不同的东西(错误分支是状态码 + 空 body,成功分支是完整响应)。[第 04 章](04-templates.md)讲过:分支返回不同类型时,统一到 `Response` 是最直接的办法。

**`.send().await` 返回 `Result`**,失败的情况包括:连不上、DNS 解析失败、超时、TLS 握手失败。注意**上游返回 500 不算失败**——那是一个成功收到的响应,只是状态码是 500,会走 `Ok` 分支被原样转发。这是 reqwest 的设计,和 `curl` 一致。

**三行转发逻辑:**

```rust
Response::builder().status(reqwest_response.status())        // ① 抄状态码
*response_builder.headers_mut().unwrap() = ...headers()...   // ② 抄响应头
.body(Body::from_stream(reqwest_response.bytes_stream()))    // ③ 流式转 body
```

第 ③ 步是全章的核心:**`bytes_stream()` 立刻返回,`Body::from_stream` 立刻包装,handler 立刻返回**。上游的数据还没开始传,响应已经在往客户端发了。

**那个 `.unwrap()` 上的注释值得琢磨。** `builder.body()` 返回 `Result`,因为 builder 可能在之前某一步失败(比如状态码或响应头非法)。官方注释说 "This unwrap is fine because the body is empty here" —— 这句话其实**说的是 builder 的错误状态,不是 body**,措辞有点误导。真正的理由是:`status()` 来自一个合法的 `StatusCode`,`headers` 来自一个已经解析成功的 `HeaderMap`,两者都不可能让 builder 进入错误态。

> **更好的写法**:`return (StatusCode::BAD_REQUEST, Body::empty()).into_response();` 里的 **`400 Bad Request` 用错了**。400 的语义是「**客户端**的请求有问题」,但这里出问题的是**服务与上游之间**——客户端什么都没做错。正确的状态码是 **`502 Bad Gateway`**(上游返回了无效响应)或 **`504 Gateway Timeout`**(上游超时)。这个区分对排障很重要:看到 400 你会去查调用方,看到 502 你才会去查上游。
>
> ```rust
> return (StatusCode::BAD_GATEWAY, "upstream request failed").into_response();
> ```

> **更好的写法**:reqwest 0.12 起提供了 `impl From<reqwest::Response> for http::Response<reqwest::Body>`,配合 `Body::new` 可以少写几行:
>
> ```rust
> let http_response: http::Response<reqwest::Body> = reqwest_response.into();
> let (parts, body) = http_response.into_parts();
> Response::from_parts(parts, Body::new(body))
> ```
>
> 这样连状态码、响应头、HTTP 版本、扩展一起带过来,不会漏。不过手写 builder 的版本更适合教学——它把「转发到底在做什么」摊开了。真实项目里两种都常见。

**跑起来看**:

```bash
cargo run
```

```bash
curl -N http://127.0.0.1:3000/
```

数字仍然一秒一个蹦出来——中间转了一手,节奏没变。下一步再挂上 `TraceLayer`,从日志里把"分块出去"变成看得见的证据。

### 步骤 4:观测每一块数据

**要做什么**:加上 `TraceLayer`,并挂一个 `on_body_chunk` 回调,把响应体的每一块打进日志。

```rust
use axum::body::Bytes;
use tower_http::trace::TraceLayer;
use tracing::Span;
```

```rust
    let app = Router::new()
        .route("/", get(stream_reqwest_response))
        .route("/stream", get(stream_some_data))
        // Add some logging so we can see the streams going through
        .layer(TraceLayer::new_for_http().on_body_chunk(
            |chunk: &Bytes, _latency: Duration, _span: &Span| {
                tracing::debug!("streaming {} bytes", chunk.len());
            },
        ))
        .with_state(client);
```

[第 17 章](17-tracing-aka-logging.md)用 `TraceLayer` 时只用了默认行为。它其实有一整套可定制的回调:

| 回调 | 触发时机 |
|---|---|
| `on_request` | 请求进来 |
| `on_response` | 响应头**已生成**(body 可能还没传完) |
| `on_body_chunk` | 响应体的**每一块**发出去时 |
| `on_eos` | 流结束(trailers) |
| `on_failure` | 出错 |

**`on_body_chunk` 是流式场景的专用观测点。** 普通响应只会触发一次,流式响应会触发很多次——这正好能证明数据是分块出去的。

注意 `on_response` 和 `on_body_chunk` 的时间差:对 `/stream`,`on_response` 在**第 0 秒**就触发了(响应头立刻发出),而最后一个 `on_body_chunk` 在**第 4 秒**。**这个时间差就是流式的证据。**

**跑起来看**:

```bash
cargo run
```

```bash
curl -N http://127.0.0.1:3000/
```

数字照样一秒一个蹦出来。**这就是本章的目标达成了**——中间转了一手,节奏没变。

看服务端日志:

```
DEBUG request{method=GET uri=/ version=HTTP/1.1}: tower_http::trace::on_request: started processing request
DEBUG request{method=GET uri=/stream version=HTTP/1.1}: tower_http::trace::on_request: started processing request
DEBUG request{method=GET uri=/stream ...}: tower_http::trace::on_response: finished processing request latency=0 ms status=200
DEBUG request{method=GET uri=/ ...}: tower_http::trace::on_response: finished processing request latency=1 ms status=200
DEBUG request{method=GET uri=/stream ...}: reqwest_response: streaming 1 bytes
DEBUG request{method=GET uri=/ ...}: reqwest_response: streaming 1 bytes
DEBUG request{method=GET uri=/stream ...}: reqwest_response: streaming 1 bytes
DEBUG request{method=GET uri=/ ...}: reqwest_response: streaming 1 bytes
...
```

**有三处值得注意:**

**1. 一次 curl 产生了两条 `on_request`。** 因为 `/` 的 handler 又去请求了 `/stream`——**这个服务在请求自己**。日志里两条 span 交替出现,清楚地展示了转发链路。

**2. `streaming 1 bytes` 成对出现。** 每个字节先从 `/stream` 的响应流出去(被 `TraceLayer` 记一次),经 reqwest 收回来,再从 `/` 的响应流出去(再记一次)。

**3. 两个 `on_response` 都在 1ms 内完成**,但字节要等 4 秒才传完。前面说的时间差,日志里看得明明白白。

> **更好的写法**:服务请求自己是**演示用的偷懒做法**,真实场景上游是另一个进程。这么写有个隐患:如果 handler 里对上游的请求出于某种原因排在了自己的响应之后(比如运行时只有一个工作线程且被阻塞),就可能死锁。`#[tokio::main]` 默认是多线程运行时,所以本章不会出问题;但**在单线程运行时(`#[tokio::main(flavor = "current_thread")]`)上跑这个例子要当心**。教学之外,永远不要让服务同步地请求自己。

### 步骤 5:故意改坏,看看流式是怎么丢的

**要做什么**:把 `bytes_stream()` 换成 `bytes()`,亲眼看到"转发退化成缓冲"。

把步骤 3 的 body 那行改成:

```rust
    // 故意改坏:先把整个响应读进内存
    let body = reqwest_response.bytes().await.unwrap();
    response_builder.body(Body::from(body)).unwrap()
```

(改完编译器可能提示 `response_builder` 需要 `mut` 或者类型不匹配,按提示调整即可。)

重新运行,然后:

```bash
curl -N http://127.0.0.1:3000/
```

**现象**:前 4 秒**什么都没有**,然后 `01234` 一次性全部出现。

同时看服务端日志,`streaming 1 bytes` 只有 `/stream` 那一侧的五条,`/` 这一侧变成了**一条 `streaming 5 bytes`**——五个字节被攒成了一块。

再对比响应头:

```bash
curl -i http://127.0.0.1:3000/
```

改坏之后多了 `content-length: 5`,`transfer-encoding: chunked` 没了(因为长度现在已知了)。

**这一步证明了三件事:**

1. 上游是流式的,不代表转发也是流式的——**中间任何一环缓冲,整条链路就退化了**
2. 客户端感知到的首字节延迟,从 0 秒变成了 4 秒
3. 内存占用从「一小块」变成了「整个响应体」

把 1GB 换到这个位置想一想:改坏之前内存平稳,改坏之后进程直接吃掉 1GB。

**记得改回 `bytes_stream()`。**

## 完整代码

`src/main.rs`:

```rust
use axum::{
    body::{Body, Bytes},
    extract::State,
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::get,
    Router,
};
use reqwest::Client;
use std::{convert::Infallible, time::Duration};
use tokio_stream::StreamExt;
use tower_http::trace::TraceLayer;
use tracing::Span;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let client = Client::new();

    let app = Router::new()
        .route("/", get(stream_reqwest_response))
        .route("/stream", get(stream_some_data))
        // Add some logging so we can see the streams going through
        .layer(TraceLayer::new_for_http().on_body_chunk(
            |chunk: &Bytes, _latency: Duration, _span: &Span| {
                tracing::debug!("streaming {} bytes", chunk.len());
            },
        ))
        .with_state(client);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn stream_reqwest_response(State(client): State<Client>) -> Response {
    let reqwest_response = match client.get("http://127.0.0.1:3000/stream").send().await {
        Ok(res) => res,
        Err(err) => {
            tracing::error!(%err, "request failed");
            return (StatusCode::BAD_REQUEST, Body::empty()).into_response();
        }
    };

    let mut response_builder = Response::builder().status(reqwest_response.status());
    *response_builder.headers_mut().unwrap() = reqwest_response.headers().clone();
    response_builder
        .body(Body::from_stream(reqwest_response.bytes_stream()))
        // This unwrap is fine because the body is empty here
        .unwrap()
}

async fn stream_some_data() -> Body {
    let stream = tokio_stream::iter(0..5)
        .throttle(Duration::from_secs(1))
        .map(|n| n.to_string())
        .map(Ok::<_, Infallible>);
    Body::from_stream(stream)
}
```

## 代价与边界

**响应头是整个抄过去的,这有风险。** `*response_builder.headers_mut().unwrap() = reqwest_response.headers().clone();` 一行搬走所有响应头。本章上游是自己人,没问题;但转发第三方响应时,有几类头**不该盲抄**:

- **逐跳头(hop-by-hop headers)**:`Connection`、`Keep-Alive`、`Transfer-Encoding`、`Upgrade`、`TE`、`Trailer`、`Proxy-Authenticate`、`Proxy-Authorization`。RFC 9110 规定这些头只对**单个连接**有效,不该被转发。抄过去可能让下游 HTTP 库困惑。
- **`Content-Length` / `Content-Encoding`**:如果 reqwest 开启了 `gzip` feature,它会**自动解压** body,但 `content-encoding: gzip` 和原始的 `content-length` 还留在响应头里。抄过去等于**告诉客户端"这是压缩的、长度是 N"**,而实际发过去的是解压后的、长度不同的数据——客户端会解压失败或读到一半卡死。**本章因为关掉了默认 feature 才侥幸没事。**
- **`Set-Cookie`**:把上游的 cookie 转给客户端,可能造成越权。

生产级的转发要**过滤**响应头,不是全抄。[第 43 章](43-reverse-proxy.md)会再碰到这个问题。

**没有超时。** `client.get(...).send().await` 默认**没有超时限制**。上游卡住,这个请求就永远挂着,连带占用一个连接和一份内存。正确做法是在建 `Client` 时设:

```rust
let client = Client::builder()
    .timeout(Duration::from_secs(30))          // 整个请求的上限
    .connect_timeout(Duration::from_secs(5))   // 建连的上限
    .build()
    .unwrap();
```

注意**流式响应和 `timeout` 有冲突**:`timeout` 管的是「从发请求到 body 读完」的总时长,对一个永不结束的 SSE 流来说这个上限没有意义。这种场景要用 `read_timeout`(两次数据之间的最大间隔)。

**没有重试,也没有熔断。** 上游抖一下,这个请求就失败了。生产环境通常需要:幂等请求的有限重试(带指数退避)、熔断器(上游连续失败时快速失败而不是继续打)、并发限制。tower 生态有 `tower::retry`、`tower::limit` 可用,思路和[第 17 章](17-tracing-aka-logging.md)讲的 `Layer` 一致。

**上游的错误状态码被原样转发。** 上游返回 500,客户端也看到 500,而且是**上游的错误正文**。这可能泄露内部信息(堆栈、内部主机名、SQL 语句)。生产环境应该**把上游的 5xx 转成自己的 502**,并且不透传正文。

**没有 TLS。** `--no-default-features` 关掉了 TLS,只能请求 `http://`。真实项目加回来:

```bash
cargo add reqwest --no-default-features --features stream,rustls-tls
```

`rustls-tls` 是纯 Rust 实现,不依赖系统 OpenSSL,交叉编译友好——和[第 32 章](32-tls-rustls.md)服务端用的是同一套底层库。另一个选择是 `default-tls`(用系统的 TLS)。

**请求头没有转发。** 本章只转发响应,客户端的请求头(比如 `Authorization`、`Accept`)一个都没带给上游。做真正的代理时这些也要处理——[第 43 章](43-reverse-proxy.md)会做。

**`Infallible` 的流不代表真的不会出错。** `stream_some_data` 用了 `Infallible`,但那只是**流本身**不会出错。客户端中途断开时,写 socket 会失败,axum 会丢弃这个响应——错误发生在流的下游,不在流里。

**大量并发转发会放大连接数。** 每个进来的请求都触发一个出去的请求。1000 QPS 进来就是 1000 QPS 出去,而且两边各占一个连接。`Client` 的连接池默认每个 host 保留一定数量的空闲连接,高并发下要调 `pool_max_idle_per_host`。

## 动手练习

**1. 把错误分支的状态码改成 502,并验证。**

提示:先把上游地址改成一个不存在的端口(比如 `http://127.0.0.1:3999/stream`),`curl -i http://127.0.0.1:3000/` 应该看到 400;改成 `StatusCode::BAD_GATEWAY` 后再看,应该是 502。**这是「代价与边界」里最容易犯的错误之一。**

**2. 给 `Client` 加超时,并造一个超时场景。**

提示:`Client::builder().timeout(Duration::from_secs(2)).build().unwrap()`。然后把 `/stream` 的 `throttle` 改成 5 秒,`curl -N http://127.0.0.1:3000/` 观察行为。想一想:**响应头已经发出去了,超时发生在 body 传到一半,客户端会看到什么?**(答案是一个不完整的 chunked 响应——curl 会报 `transfer closed with outstanding read data remaining`。这说明**一旦开始流式,就没法再改状态码了**。)

**3. 过滤逐跳响应头。**

提示:不要整个 `HeaderMap` 替换,改成遍历 + 白名单/黑名单:

```rust
const HOP_BY_HOP: [&str; 8] = [
    "connection", "keep-alive", "proxy-authenticate", "proxy-authorization",
    "te", "trailer", "transfer-encoding", "upgrade",
];
```

逐个 `insert` 不在黑名单里的头。做完用 `curl -i` 确认 `transfer-encoding` 不再是从上游抄来的(而是由 axum 自己根据 body 决定的)。

**4. 开启 reqwest 的 `gzip` feature,亲眼看看响应头骗人的后果。**

提示:`cargo add reqwest --features gzip`,给 `/stream` 加上[第 39 章](39-compression.md)的 `CompressionLayer`,请求时带 `Accept-Encoding: gzip`。reqwest 会自动解压,但 `content-encoding: gzip` 被原样抄给了客户端。用 `curl -i --compressed` 观察客户端的解压失败。**这是「代价与边界」里最隐蔽的坑。**

## 小结与下一章预告

服务端也可以是客户端。**转发一个响应就三步:抄状态码、抄响应头、把 body 从流转过去**——前两步用 `Response::builder`,第三步用 `Body::from_stream`。

全章的胜负手是 **`.bytes_stream()` 而不是 `.bytes()`**。步骤 5 故意改坏证明了:上游是流式的不代表转发也是流式的,**中间任何一环缓冲,整条链路就退化成缓冲**——首字节延迟从 0 秒变成 4 秒,内存从一小块变成整个响应体。上游要是个永不结束的 SSE 流,`.bytes()` 直接挂死。

`TraceLayer::on_body_chunk` 是流式场景的专用观测点。它和 `on_response` 的时间差(1ms vs 4s)就是「响应头先走、body 慢慢流」的直接证据。

`reqwest::Client` 内部是 `Arc`,克隆便宜,**必须复用**(连接池),直接当 axum 的 state 用就行,不需要额外包 `Arc`。

要注意的是,本章的转发是**教学级的**:响应头整个抄过去(逐跳头、`content-encoding` 会出问题)、没有超时、没有重试、错误状态码用了 400 而不是 502、请求头一个没转发。这些在「代价与边界」里逐条列了。

本章转发的是**响应**。[第 42 章 http-proxy](42-http-proxy.md) 走到另一个极端:**连 HTTP 都不解析了**。你会看到 `CONNECT` 方法、连接升级(upgrade)和 `copy_bidirectional`——一个真正的 HTTP 代理是怎么在两个 TCP 连接之间做纯字节搬运的,以及为什么这样才能代理 HTTPS 流量。同时你会第一次**绕开 `axum::serve`**,自己写 accept 循环。
