# 第 18 章 print-request-response:把两个方向的 body 都打出来

## 本章你将做出什么

一个中间件,把每个请求和响应的 body 完整打印到日志:

```
DEBUG request body = "{\"name\":\"zhangsan\"}"
DEBUG response body = "Hello from `POST /`"
```

[第 17 章](17-tracing-aka-logging.md)的 `TraceLayer` 记录的是元信息——方法、路径、状态码、耗时。有些场景需要看到**内容本身**:对接第三方时确认"我到底发了什么"、排查"客户端说发了但我没收到"、还原一个复现不了的 bug。

这一章是[第 09 章](09-consume-body-in-extractor-or-middleware.md)那个"拆开—读取—重组"手法的完整版:**这次响应体也要同样处理**。

学完本章你将掌握:

- 响应体的拆装:`Response::into_parts` / `from_parts`,以及它和请求的对称性
- 用泛型函数同时处理请求体和响应体
- `HttpBody` trait 的作用,以及为什么中间件要对它泛型
- 为什么这类中间件几乎不该出现在生产环境

## 前置知识

### 请求与响应的对称性

[第 00 章](00-web-basics.md#http-是一层文本约定)讲过 HTTP 报文的结构:**请求和响应几乎完全对称**——都是"起始行 + 头部 + 空行 + body",只有第一行不同。

这个对称性在 `http` crate 的类型设计里同样成立:

| | 请求 | 响应 |
|---|---|---|
| 类型 | `Request<B>` | `Response<B>` |
| 拆开 | `.into_parts()` → `(Parts, B)` | `.into_parts()` → `(Parts, B)` |
| 组装 | `Request::from_parts(parts, body)` | `Response::from_parts(parts, body)` |
| Parts 里装什么 | 方法、URI、版本、头部、扩展 | 状态码、版本、头部、扩展 |

**所以[第 09 章](09-consume-body-in-extractor-or-middleware.md#步骤-2在中间件里也读一遍-body)对请求用过的手法,原样搬给响应就行**:拆开 → 读完 body → 用读到的字节造一个新 body → 重组。

这一章的价值就在于把这个对称性用足:写**一个**泛型函数,请求和响应都用它。

### HttpBody:为什么函数要泛型

想写一个同时接受请求体和响应体的函数,第一反应可能是:

```rust
async fn buffer_and_print(direction: &str, body: Body) -> Result<Bytes, ...>
```

但请求体和响应体的类型不一定相同。`axum::body::Body` 是 axum 统一后的类型,而中间件在链条的不同位置可能拿到不同的 body 实现——比如被压缩中间件包过的响应体,类型就变了。

所以示例把它写成泛型:

```rust
async fn buffer_and_print<B>(direction: &str, body: B) -> Result<Bytes, (StatusCode, String)>
where
    B: axum::body::HttpBody<Data = Bytes>,
    B::Error: std::fmt::Display,
```

**`HttpBody` 是 `http-body` crate 定义的 trait**,表示"一个可以流式读取的 HTTP 消息体"。它是 hyper、axum、tower-http 之间关于 body 的公共约定——任何实现了它的类型都能被这套生态处理。

两个约束的含义:

- **`Data = Bytes`** — 每次读出来的数据块是 `Bytes` 类型。这是关联类型,不同实现可以用不同的块类型,这里限定成最常见的 `Bytes`。
- **`B::Error: std::fmt::Display`** — 错误能被格式化成字符串,这样出错时才能拼进错误消息。

[第 09 章](09-consume-body-in-extractor-or-middleware.md#collect-与-bodyext)用的 `BodyExt::collect` 正是 `HttpBody` 的扩展方法,所以泛型参数满足这个约束就能调用它。

**这是 Rust 里"面向 trait 编程"的典型场景**:函数不关心具体是什么 body,只要求它能被流式读取。

### 中间件的返回类型

本章中间件的签名和第 09 章略有不同:

```rust
async fn print_request_response(
    req: Request,
    next: Next,
) -> Result<impl IntoResponse, (StatusCode, String)>
```

错误类型是 `(StatusCode, String)` 而不是第 09 章的 `Response`。两种都可以——[第 08 章](08-parse-body-based-on-content-type.md#rejection-该选什么类型)那张表讲过取舍:错误形态单一时用具体类型更清楚,形态多样时才需要擦除成 `Response`。

这里读 body 失败只有一种处理方式,所以用元组就够了。

## 项目搭建

```bash
cargo new print-request-response
cd print-request-response
```

```bash
cargo add axum
cargo add tokio --features full
cargo add http-body-util
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`http-body-util` 提供 `BodyExt::collect`,[第 09 章](09-consume-body-in-extractor-or-middleware.md#collect-与-bodyext)讲过为什么要单独加它。

## 分步构建

### 步骤 1:只处理请求体

**要做什么**:先把第 09 章那套搬过来,只打印请求体。这一步的代码你应该很熟悉。

```rust
// src/main.rs
use axum::{
    body::{Body, Bytes},
    extract::Request,
    http::StatusCode,
    middleware::{self, Next},
    response::IntoResponse,
    routing::post,
    Router,
};
use http_body_util::BodyExt;
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

    let app = Router::new()
        .route("/", post(|| async move { "Hello from `POST /`" }))
        .layer(middleware::from_fn(print_request_response));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn print_request_response(
    req: Request,
    next: Next,
) -> Result<impl IntoResponse, (StatusCode, String)> {
    let (parts, body) = req.into_parts();
    let bytes = buffer_and_print("request", body).await?;
    let req = Request::from_parts(parts, Body::from(bytes));

    Ok(next.run(req).await)
}

async fn buffer_and_print<B>(direction: &str, body: B) -> Result<Bytes, (StatusCode, String)>
where
    B: axum::body::HttpBody<Data = Bytes>,
    B::Error: std::fmt::Display,
{
    let bytes = match body.collect().await {
        Ok(collected) => collected.to_bytes(),
        Err(err) => {
            return Err((
                StatusCode::BAD_REQUEST,
                format!("failed to read {direction} body: {err}"),
            ));
        }
    };

    if let Ok(body) = std::str::from_utf8(&bytes) {
        tracing::debug!("{direction} body = {body:?}");
    }

    Ok(bytes)
}
```

**`post(|| async move { "Hello from `POST /`" })`** 用了闭包做 handler。[第 01 章](01-hello-world.md#handler为什么普通函数能当处理器)讲的 `Handler` trait 对闭包同样成立——只要参数可提取、返回值可转成响应。路由逻辑很简单时,闭包比单独定义函数省事。

**`buffer_and_print` 里的 `direction` 参数**是为了让同一个函数在日志里区分请求和响应。步骤 2 会传 `"response"`。

**`if let Ok(body) = std::str::from_utf8(&bytes)`** 只在 body 是合法 UTF-8 时打印。二进制内容(图片、压缩数据)转不成字符串,这里就静默跳过——**注意这意味着上传图片时你什么日志都看不到**,不是中间件没生效。

`{body:?}` 用 `Debug` 格式打印字符串,好处是换行、引号这些会被转义显示,不会打乱日志的行结构。

**跑起来看**:

```bash
cargo run
```

```bash
curl -s -X POST http://127.0.0.1:3000/ -d '{"name":"zhangsan"}'
```

```
Hello from `POST /`
```

服务端终端:

```
2026-07-30T11:31:02.117294Z DEBUG print_request_response: request body = "{\"name\":\"zhangsan\"}"
```

请求体打出来了,响应体还没有。

### 步骤 2:响应体也来一遍

**要做什么**:用同一个泛型函数处理响应体。

```rust
// src/main.rs
use axum::{
    body::{Body, Bytes},
    extract::Request,
    http::StatusCode,
    middleware::{self, Next},
    response::{IntoResponse, Response},        // 改动:加上 Response
    routing::post,
    Router,
};
// ...(其余 use 不变)

// ...(main 不变)

// 改动:响应也拆开处理
async fn print_request_response(
    req: Request,
    next: Next,
) -> Result<impl IntoResponse, (StatusCode, String)> {
    let (parts, body) = req.into_parts();
    let bytes = buffer_and_print("request", body).await?;
    let req = Request::from_parts(parts, Body::from(bytes));

    let res = next.run(req).await;

    // 新增:对响应做同样的三步
    let (parts, body) = res.into_parts();
    let bytes = buffer_and_print("response", body).await?;
    let res = Response::from_parts(parts, Body::from(bytes));

    Ok(res)
}

// ...(buffer_and_print 不变)
```

**新增的三行和上面处理请求的三行几乎一字不差**,只是把 `Request` 换成了 `Response`、`"request"` 换成了 `"response"`。这就是前置知识说的对称性。

`buffer_and_print` 一个字都没改——泛型让它同时适用于两种 body。

**注意 `next.run(req).await` 的位置。** 它把请求交给内层,拿回响应。这一行之前的代码在**请求路上**执行,之后的在**响应路上**执行——[第 09 章](09-consume-body-in-extractor-or-middleware.md#中间件在请求前后插入逻辑)讲的洋葱结构。

**跑起来看**:

```bash
cargo run
```

```bash
curl -s -X POST http://127.0.0.1:3000/ -d '{"name":"zhangsan"}'
```

```
Hello from `POST /`
```

终端现在有两条:

```
2026-07-30T11:35:41.203817Z DEBUG print_request_response: request body = "{\"name\":\"zhangsan\"}"
2026-07-30T11:35:41.204092Z DEBUG print_request_response: response body = "Hello from `POST /`"
```

**两个方向都记录下来了。**

试试空请求体:

```bash
curl -s -X POST http://127.0.0.1:3000/
```

```
DEBUG print_request_response: request body = ""
DEBUG print_request_response: response body = "Hello from `POST /`"
```

请求体是空字符串,仍然会打印一条。

### 步骤 3:验证"不重组会怎样"

**要做什么**:故意不把响应体放回去,看客户端收到什么。

[第 09 章](09-consume-body-in-extractor-or-middleware.md#步骤-3验证不归还会发生什么)对请求体做过这个验证,响应体的后果同样值得亲眼看一次。

临时把重组那一行改成用空 body:

```rust
    let res = Response::from_parts(parts, Body::empty());   // 临时改动
```

重新运行:

```bash
cargo run
```

```bash
curl -i -s -X POST http://127.0.0.1:3000/ -d 'hi'
```

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 19
date: Thu, 30 Jul 2026 11:39:12 GMT

```

**状态码 `200`、`content-length: 19`,但 body 是空的。** 服务端日志显示响应体是 `"Hello from `POST /`"`(19 字节),客户端却什么都没收到。

**而且 `content-length` 还停留在 19。** 因为头部保存在 `parts` 里,原样搬了过来;body 却被换成了空的。**响应头和实际内容对不上了**——严格来说这是一个违反 HTTP 规范的响应,某些客户端会一直等那 19 个字节直到超时。

这比第 09 章那个"下游拿到空 body"更隐蔽:那次至少服务端逻辑还在正常跑,这次是**响应本身自相矛盾**,而 curl 只是安静地显示了个空行。

改回 `Body::from(bytes)`。

**顺带说明一个容易忽略的点**:即使正确重组,`content-length` 也可能需要重新计算。本章只是读出来再放回去,长度没变所以没问题;但如果中间件**修改**了 body(比如脱敏、压缩),就必须同步更新 `content-length` 头部,否则同样会产生自相矛盾的响应。

## 完整代码

`src/main.rs`:

```rust
use axum::{
    body::{Body, Bytes},
    extract::Request,
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
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app = Router::new()
        .route("/", post(|| async move { "Hello from `POST /`" }))
        .layer(middleware::from_fn(print_request_response));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn print_request_response(
    req: Request,
    next: Next,
) -> Result<impl IntoResponse, (StatusCode, String)> {
    let (parts, body) = req.into_parts();
    let bytes = buffer_and_print("request", body).await?;
    let req = Request::from_parts(parts, Body::from(bytes));

    let res = next.run(req).await;

    let (parts, body) = res.into_parts();
    let bytes = buffer_and_print("response", body).await?;
    let res = Response::from_parts(parts, Body::from(bytes));

    Ok(res)
}

async fn buffer_and_print<B>(direction: &str, body: B) -> Result<Bytes, (StatusCode, String)>
where
    B: axum::body::HttpBody<Data = Bytes>,
    B::Error: std::fmt::Display,
{
    let bytes = match body.collect().await {
        Ok(collected) => collected.to_bytes(),
        Err(err) => {
            return Err((
                StatusCode::BAD_REQUEST,
                format!("failed to read {direction} body: {err}"),
            ));
        }
    };

    if let Ok(body) = std::str::from_utf8(&bytes) {
        tracing::debug!("{direction} body = {body:?}");
    }

    Ok(bytes)
}
```

## 代价与边界

这个中间件很有用,但**几乎不该出现在生产环境**。理由比[第 09 章](09-consume-body-in-extractor-or-middleware.md#缓冲整个-body-的代价)更严重,因为它现在影响两个方向。

**破坏流式响应。** 这是最要命的一条。`collect()` 会等到 body 全部产生完毕才返回,于是:

- **SSE**([第 33 章](33-sse.md))本该持续推送,现在会一直缓冲直到连接断开——**功能直接失效**
- **大文件下载**会先在内存里攒完整个文件再发出去,10 个并发下载 1GB 文件 = 10GB 内存
- **首字节时间(TTFB)**从"handler 一开始产出就发"变成"handler 完全结束才发",用户感知的延迟显著增加

**内存翻倍。** 请求体和响应体都要完整驻留内存。第 09 章那个只有请求体一份。

**日志会记录敏感数据。** 这是最容易被忽视的风险:请求体里有密码、身份证号、支付信息,响应体里有用户隐私、token。这些全被原样写进日志,而日志通常会被转发到集中式系统、长期保留、被很多人访问。**这可能直接构成合规问题**(GDPR、个人信息保护法)。

**日志量爆炸。** 每个请求两条完整 body。一个中等流量的服务一天能产生几十 GB 日志。

**那什么时候可以用:**

- **本地开发调试**,尤其是对接第三方 API 时确认报文格式
- **临时排查**特定问题,而且**只对特定路由启用**——`.layer()` 可以挂在单条路由上,不必全局
- **测试环境**,配合脱敏和路由白名单

**生产环境如果确实需要,至少要做这几件事**:只对白名单路由启用、限制 body 大小(超过就截断)、按内容类型跳过(二进制、`text/event-stream` 一律不碰)、对敏感字段脱敏、用采样而非全量。

一句话:**这是个调试工具,不是可观测性方案。** 生产环境的可观测性应该靠[第 17 章](17-tracing-aka-logging.md)的结构化日志和[第 47 章](47-prometheus-metrics.md)的指标,它们不需要读 body。

## 动手练习

**1. 加上截断,超过 1KB 只打印前 1KB。**

提示:在 `buffer_and_print` 里判断 `bytes.len()`,超限时用 `&bytes[..1024]` 打印并加个 `...(truncated)` 标记。注意**返回的仍然是完整的 `bytes`**——截断只影响日志,不能影响实际传输的内容。这个区别很关键,写错就变成了数据损坏。

**2. 按 `Content-Type` 跳过二进制内容。**

提示:`buffer_and_print` 现在拿不到头部信息,需要多传一个参数。在调用处从 `parts.headers` 里取 `content-type`,如果不是 `text/*`、`application/json`、`application/x-www-form-urlencoded` 就跳过打印(但仍要正常返回 bytes)。做完试试上传一张图片,确认日志里不再出现乱码。

**3. 只对特定路由启用这个中间件。**

提示:`.layer()` 不仅能挂在 `Router` 上,也能挂在单条路由的 `MethodRouter` 上——`post(handler).layer(middleware::from_fn(print_request_response))`。加两条路由,一条挂中间件一条不挂,验证只有前者产生日志。这是「代价与边界」里说的"只对白名单路由启用"的具体做法。

## 小结与下一章预告

`Request` 和 `Response` 在类型设计上是对称的:都能 `into_parts()` 拆成"元信息 + body",都能 `from_parts()` 组装回去。所以[第 09 章](09-consume-body-in-extractor-or-middleware.md)对请求用过的"拆开—读取—重组"手法,原样搬给响应就行——本章那个 `buffer_and_print` 靠 `HttpBody` 泛型同时服务两个方向,一个字都不用改。

不重组的后果在响应侧更严重:**头部还留在 `parts` 里,body 却被换掉了**,于是 `content-length` 和实际内容对不上,产生一个自相矛盾的响应。如果中间件修改了 body,还必须同步更新 `content-length`。

这个中间件是**调试工具而非可观测性方案**。它会破坏流式响应(SSE 直接失效)、让内存翻倍、把密码和 token 原样写进日志。用它的正确姿势是:本地开发、临时排查、只挂在特定路由上。

本章和第 17 章的日志都缺一样东西:**把同一个请求的多条日志串起来的标识**。服务一多,一次用户操作会横跨好几个服务的日志,没有共同标识就没法追踪。[第 19 章 request-id](19-request-id.md) 解决这个问题:为每个请求生成唯一 ID、写进 span、并透传到响应头。你还会第一次用上[第 17 章](17-tracing-aka-logging.md#洋葱结构与包装顺序)讲的 `ServiceBuilder`——因为"生成 ID"必须排在"记录 ID"前面,顺序不能错。
