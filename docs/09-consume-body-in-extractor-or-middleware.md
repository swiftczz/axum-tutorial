# 第 09 章 consume-body-in-extractor-or-middleware:body 只能读一次

## 本章你将做出什么

一个 `POST /` 服务,请求体会被**读取两次**——一次在中间件里,一次在提取器里——但两次都拿到完整内容。

听起来像悖论:[第 08 章](08-parse-body-based-on-content-type.md#fromrequest-与-fromrequestparts-的实现细节)刚讲过 body 是一次性的,`FromRequest` 拿走 `Request` 的所有权后别人就没得用了。这一章讲清那个限制的真正边界,以及"读完再放回去"这个通用手法。

顺带这是本教程第一个**中间件**(middleware)章节。

学完本章你将掌握:

- 为什么 HTTP body 天然只能读一次,这个限制来自哪里
- `Request::into_parts` / `from_parts` 拆装请求的手法,以及如何"消费后归还"
- `middleware::from_fn` 写中间件:`Next` 是什么、控制权如何传递
- 缓冲整个 body 的代价,以及什么时候绝对不该这么做

## 前置知识

### body 为什么只能读一次

前面几章说 body "只能消费一次",一直没解释原因。这不是 axum 的人为限制,而是从底层一路传导上来的。

HTTP body 在网络上是**流式**到达的。服务端收到请求头之后,body 的字节可能还在路上——客户端上传一个 1GB 的文件,服务端可能要等几分钟才收完。所以底层类型(hyper 的 `Incoming`)是一个**流**(stream):你每次轮询它,它给你下一块数据(`Frame`),取完为止。

流的本质是"取一次就没了"。已经从 socket 读出来的字节,如果你不自己存着,就再也拿不回来——除非让客户端重发一遍整个请求,那显然不现实。

这个特性沿着类型系统一路体现出来:

- hyper 的 body 是 `Stream`,轮询会推进内部状态
- axum 的 `Body` 包装了它,`collect()` 之类的方法消耗 `self`
- `FromRequest::from_request` 接收 `Request` 的**所有权**而不是引用
- 所以 handler 参数里只能有一个 `FromRequest` 提取器,而且必须放最后

**但"只能读一次"和"只能被一个地方使用"是两件事。** 只要有人把流读完并**存进内存**,那份内存数据就可以任意复制、传递、重复读取。这就是本章的核心手法。

### Bytes:可以廉价克隆的字节缓冲

存 body 用 `bytes::Bytes`(axum 从 `axum::body::Bytes` 重新导出)。

它是一段不可变的字节缓冲,关键特性是 **`clone()` 极其廉价**——内部是引用计数 + 偏移量,克隆只增加计数,不复制底层内存。多个 `Bytes` 可以指向同一块缓冲的不同片段。

这和 `Vec<u8>` 的区别很实际:克隆 `Vec<u8>` 会真的复制全部字节,而克隆 `Bytes` 几乎零成本。网络库到处用 `Bytes` 就是因为这个——一份数据要给日志、给业务、给转发,复制多次代价太高。

本章的代码里会看到 `bytes.clone()` 出现两次,理解了这一点就知道它们不是性能问题。

### collect() 与 BodyExt

把流读完成内存,用 `http_body_util::BodyExt::collect`:

```rust
let bytes = body.collect().await?.to_bytes();
```

`BodyExt` 是 `http-body-util` crate 提供的扩展 trait,给所有实现了 `http_body::Body` 的类型加上便利方法。要用它必须显式 `use http_body_util::BodyExt;`——这是 Rust 扩展 trait 的规矩,不导入 trait 就调不到方法。

`collect()` 返回一个 `Collected<Bytes>`,再调 `.to_bytes()` 得到 `Bytes`。中间多一层是因为 `Collected` 还保留了 trailers(HTTP/2 的尾部头部)信息,大多数场景用不上。

注意本章要单独 `cargo add http-body-util`——axum 内部用了它,但没有重新导出 `BodyExt`。

### 中间件:在请求前后插入逻辑

前面所有章节的代码都是"请求 → handler → 响应"。但很多需求不属于任何单个 handler:

- 记录每个请求的方法、路径、耗时
- 校验 `Authorization` 头部,没登录直接拒绝
- 给所有响应加上 CORS 头部
- 生成请求 ID 并贯穿整条链路

这些逻辑横切所有路由,写进每个 handler 显然荒谬。**中间件**就是把它们提取出来,包在 handler 外面。

axum 里最简单的写法是 `middleware::from_fn`:

```rust
async fn my_middleware(request: Request, next: Next) -> Response {
    // 请求到达 handler 之前:这里能改 request
    let response = next.run(request).await;
    // handler 返回之后:这里能改 response
    response
}

let app = Router::new()
    .route("/", get(handler))
    .layer(middleware::from_fn(my_middleware));
```

签名的两个参数是关键:

- **`request: Request`** — 完整的请求,所有权归你。这意味着中间件也能消耗 body,和提取器面临同样的问题。
- **`next: Next`** — 代表"链条上剩下的部分"(后续中间件 + 最终的 handler)。调用 `next.run(request).await` 把控制权交下去,拿到响应。

所以中间件是个**洋葱结构**:`next.run` 之前的代码在请求路上执行,之后的代码在响应路上执行。**你也可以不调用 `next.run`**——那就是短路,直接返回响应,请求永远到不了 handler。认证中间件就靠这个拒绝未授权请求。

返回类型可以是 `Response`,也可以是 `Result<impl IntoResponse, E>`(`E: IntoResponse`),后者让中间件能用 `?` 提前失败。本章示例用的是后者。

`.layer(...)` 把中间件挂到 `Router` 上。多个 `.layer` 的叠加顺序、以及它背后 tower 的 `Layer`/`Service` 抽象,是更大的话题。

> **本章不深入**:`.layer()` 接受的是 tower 的 `Layer`,`middleware::from_fn` 只是把一个函数包装成 `Layer` 的便利入口。tower 的 `Service`/`Layer` trait 定义、多个 layer 的叠加顺序(为什么是"后注册的在外层")、以及 `tower-http` 提供的现成中间件,见[第 17 章](17-tracing-aka-logging.md)。本章只用一个中间件,顺序问题不会出现。

## 项目搭建

```bash
cargo new consume-body-in-extractor-or-middleware
cd consume-body-in-extractor-or-middleware
```

```bash
cargo add axum
cargo add tokio --features full
cargo add http-body-util
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`http-body-util` 提供 `BodyExt::collect`,理由见前置知识。

## 分步构建

### 步骤 1:一个缓冲 body 的提取器

**要做什么**:写一个自定义提取器,把整个 body 读成 `Bytes` 并顺手打条日志。

```rust
// src/main.rs
use axum::{
    body::Bytes,
    extract::{FromRequest, Request},
    response::Response,
    routing::post,
    Router,
};
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

    let app = Router::new().route("/", post(handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler(BufferRequestBody(body): BufferRequestBody) {
    tracing::debug!(?body, "handler received body");
}

// extractor that shows how to consume the request body upfront
struct BufferRequestBody(Bytes);

// we must implement `FromRequest` (and not `FromRequestParts`) to consume the body
impl<S> FromRequest<S> for BufferRequestBody
where
    S: Send + Sync,
{
    type Rejection = Response;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let body = Bytes::from_request(req, state)
            .await
            .map_err(|err| err.into_response())?;

        do_thing_with_request_body(body.clone());

        Ok(Self(body))
    }
}

fn do_thing_with_request_body(bytes: Bytes) {
    tracing::debug!(body = ?bytes);
}
```

自定义提取器的套路[第 08 章](08-parse-body-based-on-content-type.md#步骤-2实现-fromrequest)讲过,这里只说不同的地方。

**`Bytes::from_request(req, state)`** —— `Bytes` 本身就是个提取器!axum 为它实现了 `FromRequest`,语义是"把整个 body 读成字节,不做任何解析"。所以这个自定义提取器实际上是"委托给 `Bytes`,顺手做点别的"。

这个委托模式值得记住:**自定义提取器很少从零开始读 body,通常是包一层已有提取器**。第 10、11 章的三种做法全都是这个思路。

**`do_thing_with_request_body(body.clone())`** —— 这里 `clone()` 是廉价的引用计数操作,不复制字节。函数拿到一份,`Ok(Self(body))` 里还留着一份,两份指向同一块内存。

**handler 返回 `()`** —— [第 01 章](01-hello-world.md#intoresponse返回值如何变成-http-响应)那张表里有这一行:`()` 转成 `200` + 空 body。示例只关心日志,不关心响应内容。

**`tracing::debug!(?body, ...)`** 里的 `?body` 是 tracing 的字段语法,表示"按 `Debug` 格式记录名为 `body` 的字段"。前面章节用过 `%uri`(按 `Display`)。

**跑起来看**:

```bash
cargo run
```

```bash
curl -X POST http://127.0.0.1:3000/ -d 'hello world'
```

curl 没有输出(响应 body 是空的),但服务端终端有两行:

```
2026-07-30T08:12:03.117294Z DEBUG consume_body_in_extractor_or_middleware: body=b"hello world"
2026-07-30T08:12:03.117382Z DEBUG consume_body_in_extractor_or_middleware: handler received body body=b"hello world"
```

第一行来自提取器里的 `do_thing_with_request_body`,第二行来自 handler。**同一份 body 被两处代码读到了**——因为它已经在内存里,`Bytes` 的克隆是廉价的。

用 `-i` 确认状态码:

```bash
curl -i -X POST http://127.0.0.1:3000/ -d 'hello world'
```

```
HTTP/1.1 200 OK
content-length: 0
date: Thu, 30 Jul 2026 08:12:20 GMT
```

### 步骤 2:在中间件里也读一遍 body

**要做什么**:加一个中间件,在请求到达提取器**之前**就把 body 读出来打日志,然后让提取器仍然能拿到完整 body。

这是本章的核心难点。中间件拿到的是 `Request` 的所有权,读 body 就会消耗掉它——之后提取器只能拿到一个空 body。解决办法是**读完再放回去**。

```rust
// src/main.rs
use axum::{
    body::{Body, Bytes},                                   // 改动:加上 Body
    extract::{FromRequest, Request},
    http::StatusCode,                                      // 新增
    middleware::{self, Next},                              // 新增
    response::{IntoResponse, Response},                    // 改动:加上 IntoResponse
    routing::post,
    Router,
};
use http_body_util::BodyExt;                               // 新增
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    // 改动:挂上中间件
    let app = Router::new()
        .route("/", post(handler))
        .layer(middleware::from_fn(print_request_body));

    // ...(listener 和 serve 不变)
}

// 新增:middleware that shows how to consume the request body upfront
async fn print_request_body(request: Request, next: Next) -> Result<impl IntoResponse, Response> {
    let request = buffer_request_body(request).await?;

    Ok(next.run(request).await)
}

// 新增:the trick is to take the request apart, buffer the body, do what you need to do, then put
// the request back together
async fn buffer_request_body(request: Request) -> Result<Request, Response> {
    let (parts, body) = request.into_parts();

    // this won't work if the body is a long-running stream
    let bytes = body
        .collect()
        .await
        .map_err(|err| (StatusCode::INTERNAL_SERVER_ERROR, err.to_string()).into_response())?
        .to_bytes();

    do_thing_with_request_body(bytes.clone());

    Ok(Request::from_parts(parts, Body::from(bytes)))
}

// ...(handler、BufferRequestBody、do_thing_with_request_body 不变)
```

`buffer_request_body` 是整章最值得理解的十行,拆开看。

**`request.into_parts()`** 把 `Request` 拆成两半:`parts`(`http::request::Parts`,装着方法、URI、版本、头部、扩展)和 `body`。这是 `http` crate 的设计——请求的元信息和 body 是可分离的。

拆开的意义在于:**元信息是可以随便搬动的,只有 body 是一次性的。** 把它们分开处理,问题就只剩下 body。

**`body.collect().await?.to_bytes()`** 把流读完成 `Bytes`。前置知识讲过这两步。

`map_err` 那一行处理读取失败——客户端中途断连、超时都可能触发。转成 `500` 加错误文本,`?` 直接返回。注意错误类型是 `Response`,和函数签名对应,这是[第 08 章](08-parse-body-based-on-content-type.md#rejection-该选什么类型)讲过的"擦除成 `Response`"手法。

> **更好的写法**:这里有两个问题。一是 `err.to_string()` 被直接发给了客户端,`5xx` 不该暴露内部错误细节(理由见[第 14 章](14-error-handling.md#4xx-与-5xx-决定了两件事))。二是状态码用 `500` 不够准确——客户端中途断连、发送超时是**客户端**的问题,更贴切的是 `400`;真正该用 `500` 的只有服务端自身出错的情况。`http_body_util` 的错误类型无法直接区分这两者,所以稳妥做法是记日志 + 统一回一个固定文本的 `400`。示例选择"一律 500 + 原始错误文本"是为了少写分支、也方便你在 curl 里看到出了什么事。

**`Request::from_parts(parts, Body::from(bytes))`** 是关键的"归还"动作。用原来的 `parts` 和一个**新造的 body** 组装出一个新 `Request`。

`Body::from(bytes)` 把内存里的字节包成一个 `Body`。这个 body 不再是网络流,而是一段已知长度的内存数据——读它不需要等网络,而且**内容和原始 body 完全一致**。对下游(提取器、handler)来说,它和原始请求毫无区别。

**这就是"消费后归还"手法的全部。** 不是绕过了"body 只能读一次"的限制,而是接受它:读一次,存下来,再用存下来的内容造一个新的。

`print_request_body` 本身很薄:调用 `buffer_request_body` 拿到重组后的请求,交给 `next.run`。`?` 让缓冲失败时直接短路,请求到不了 handler。

**跑起来看**:

```bash
cargo run
```

```bash
curl -X POST http://127.0.0.1:3000/ -d 'hello world'
```

终端现在有三行日志:

```
2026-07-30T08:20:41.552033Z DEBUG consume_body_in_extractor_or_middleware: body=b"hello world"
2026-07-30T08:20:41.552198Z DEBUG consume_body_in_extractor_or_middleware: body=b"hello world"
2026-07-30T08:20:41.552241Z DEBUG consume_body_in_extractor_or_middleware: handler received body body=b"hello world"
```

**三行,内容都是 `hello world`。** 第一行来自中间件(经由 `buffer_request_body`),第二行来自提取器,第三行来自 handler。前两行调用的是同一个 `do_thing_with_request_body` 函数,所以格式一样。

body 被完整读了两次(中间件一次、提取器一次),两次都拿到了全部内容——这就是本章开头说的那个"悖论"。

试试空 body 和长一点的内容:

```bash
curl -X POST http://127.0.0.1:3000/ -d ''
curl -X POST http://127.0.0.1:3000/ -d 'a much longer body with 中文 characters'
```

都能正常工作,日志里三行始终一致。

### 步骤 3:验证"不归还"会发生什么

**要做什么**:故意破坏归还逻辑,看下游拿到什么。这一步能让你彻底记住为什么必须重组请求。

把 `buffer_request_body` 最后一行临时改成用一个空 body:

```rust
    // 临时改动:不把读出来的字节放回去
    Ok(Request::from_parts(parts, Body::empty()))
```

重新运行:

```bash
cargo run
```

```bash
curl -X POST http://127.0.0.1:3000/ -d 'hello world'
```

```
2026-07-30T08:25:11.334072Z DEBUG consume_body_in_extractor_or_middleware: body=b"hello world"
2026-07-30T08:25:11.334198Z DEBUG consume_body_in_extractor_or_middleware: body=b""
2026-07-30T08:25:11.334241Z DEBUG consume_body_in_extractor_or_middleware: handler received body body=b""
```

**中间件看到了完整内容,提取器和 handler 拿到的是空的。** 而且**没有任何报错**——请求正常返回 `200`,只是数据悄悄丢了。

这类 bug 极其难查:中间件的日志显示数据收到了,业务逻辑却说没数据。如果这个 handler 用的是 `Json<T>` 提取器,你会看到"请求明明发了 JSON,却报 body 为空"这种令人困惑的错误。

**推论:任何消耗 body 的中间件都必须归还。** 这条规则在写日志中间件、审计中间件、签名校验中间件时都要遵守。第 18 章 print-request-response 会展示一个更完整的版本,连响应 body 也一并处理。

改回 `Body::from(bytes)`。

### 缓冲整个 body 的代价

这个手法有个明确的适用边界,官方示例的注释直接写了:`this won't work if the body is a long-running stream`。

**代价是内存。** `collect()` 会把整个 body 读进内存才返回。客户端上传 1GB 文件,你的进程就吃掉 1GB。更糟的是并发场景:100 个并发上传,内存瞬间到 100GB,进程被 OOM killer 干掉。

**这是一个可被利用的攻击面。** 攻击者只要并发发送大量超大请求体,就能耗尽服务器内存。所以生产环境必须限制请求体大小。axum 的 `Json`、`Form`、`Bytes` 等内置提取器**默认有 2MB 上限**(由 `DefaultBodyLimit` 控制),超过直接返回 `413 Payload Too Large`。但注意:**本章中间件里的 `body.collect()` 绕过了这个限制**——它直接操作原始 body,不经过提取器。

所以真实项目里,消耗 body 的中间件应该显式限制大小,或者用 `tower-http` 的 `RequestBodyLimitLayer` 在更外层拦住——**它作用于整条处理链,不管下游用不用提取器,正好堵住这个洞**。用法和它与 `DefaultBodyLimit` 的分工见[第 13 章](13-multipart-form.md#两层请求体限制)。

什么时候**不该**缓冲整个 body:

- 文件上传 —— 应该边收边写磁盘,见第 40 章 stream-to-file
- 大文件代理转发 —— 应该流式转发,见第 42、43 章
- 长连接流式协议 —— WebSocket、SSE 根本不该走这条路

什么时候可以放心缓冲:

- JSON API 请求(通常几 KB)
- 表单提交(不含文件)
- 需要完整内容才能处理的场景,比如验签(签名覆盖整个 body,不看完算不出来)

## 完整代码

`src/main.rs`:

```rust
use axum::{
    body::{Body, Bytes},
    extract::{FromRequest, Request},
    http::StatusCode,
    middleware::{self, Next},
    response::{IntoResponse, Response},
    routing::post,
    Router,
};
use http_body_util::BodyExt;
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

    let app = Router::new()
        .route("/", post(handler))
        .layer(middleware::from_fn(print_request_body));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

// middleware that shows how to consume the request body upfront
async fn print_request_body(request: Request, next: Next) -> Result<impl IntoResponse, Response> {
    let request = buffer_request_body(request).await?;

    Ok(next.run(request).await)
}

// the trick is to take the request apart, buffer the body, do what you need to do, then put
// the request back together
async fn buffer_request_body(request: Request) -> Result<Request, Response> {
    let (parts, body) = request.into_parts();

    // this won't work if the body is a long-running stream
    let bytes = body
        .collect()
        .await
        .map_err(|err| (StatusCode::INTERNAL_SERVER_ERROR, err.to_string()).into_response())?
        .to_bytes();

    do_thing_with_request_body(bytes.clone());

    Ok(Request::from_parts(parts, Body::from(bytes)))
}

fn do_thing_with_request_body(bytes: Bytes) {
    tracing::debug!(body = ?bytes);
}

async fn handler(BufferRequestBody(body): BufferRequestBody) {
    tracing::debug!(?body, "handler received body");
}

// extractor that shows how to consume the request body upfront
struct BufferRequestBody(Bytes);

// we must implement `FromRequest` (and not `FromRequestParts`) to consume the body
impl<S> FromRequest<S> for BufferRequestBody
where
    S: Send + Sync,
{
    type Rejection = Response;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let body = Bytes::from_request(req, state)
            .await
            .map_err(|err| err.into_response())?;

        do_thing_with_request_body(body.clone());

        Ok(Self(body))
    }
}
```

## 动手练习

**1. 给中间件加上大小限制,超过 100 字节直接拒绝。**

提示:`collect()` 之后检查 `bytes.len()`,超限返回 `(StatusCode::PAYLOAD_TOO_LARGE, "body too large").into_response()`。验证:`curl -X POST http://127.0.0.1:3000/ -d "$(printf 'a%.0s' {1..200})"` 应该得到 `413`。想想这个检查放在 `collect()` **之后**有什么问题——字节已经进内存了,内存已经被占用了。真正的防护该怎么做?(提示:看看 `http_body::Body::size_hint` 和 `content-length` 头部。)

**2. 让中间件把 body 内容作为响应头返回。**

提示:在 `print_request_body` 里拿到 `bytes` 之后,先 `next.run(request).await` 得到响应,再用 `response.headers_mut().insert(...)` 加一个头部。需要把 `buffer_request_body` 改成同时返回 `Request` 和 `Bytes`。这个练习能让你体会"洋葱结构"——同一个中间件函数里,既能处理请求也能处理响应。验证:`curl -i -X POST http://127.0.0.1:3000/ -d 'hi'` 应该在响应头里看到 `hi`。

## 小结与下一章预告

HTTP body 只能读一次,这个限制来自它的流式本质,一路体现在 hyper 的 `Stream`、axum 的 `FromRequest` 取所有权、以及"只能有一个 body 提取器且必须放最后"这些规则上。

绕开它的办法不是打破限制,而是**读一次存下来**:`request.into_parts()` 把元信息和 body 分开,`collect()` 把 body 读成 `Bytes`,处理完用 `Request::from_parts(parts, Body::from(bytes))` 重组一个新请求交给下游。忘记归还不会报错,只会让下游静默拿到空 body——这类 bug 很难查,步骤 3 亲手复现过。

`middleware::from_fn` 是写中间件最简单的方式:`request` 给你所有权,`next.run(request)` 把控制权交给链条剩下的部分,不调用它就是短路。`next.run` 之前的代码在请求路上跑,之后的代码在响应路上跑。

缓冲整个 body 的代价是内存,而且绕过了提取器的默认 2MB 限制,大文件和流式场景绝对不能这么做。

本章的自定义提取器返回的拒绝信息还是内置提取器那套纯文本。写 API 时通常需要结构化的 JSON 错误。[第 10 章 customize-extractor-error](10-customize-extractor-error.md) 专门解决这个问题,而且是本教程第一个**多文件示例**——它用三个独立模块展示三种不同的做法,从手写 `FromRequest` 到派生宏到 `axum-extra` 的现成工具,并横向对比各自的取舍。
