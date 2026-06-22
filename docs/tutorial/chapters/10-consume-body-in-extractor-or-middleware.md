# 10. consume-body-in-extractor-or-middleware

对应示例：`examples/consume-body-in-extractor-or-middleware`

本章目标：理解请求体只能消费一次，以及在 middleware 或 extractor 里提前读取 body 后如何继续把请求交给后续逻辑。

这一章比前几章更偏底层。第一次学习先抓住一句话：

```text
请求 body 像一条只能读一次的流。
谁先读完了，后面就没得读。
```

## 这个小项目在做什么

这个 example 演示两种“提前读取请求体”的位置：

- middleware：在 handler 之前读取并打印 request body，然后把 request 重新组装好交给 handler。
- extractor：在 handler 参数里用自定义 `BufferRequestBody` 读取 body。

应用路由是：

```text
POST / -> handler
```

请求主线是：

```text
客户端 POST /
-> print_request_body middleware 先读取 body
-> middleware 用同一份 bytes 重新构造 Request
-> handler 里的 BufferRequestBody 再读取 body
-> handler 打印收到的 body
```

这一章最重要的细节是：middleware 读完 body 后，必须把 body 放回 request，否则 handler 就读不到 body。

## 先理解“body 只能消费一次”

HTTP 请求由几部分组成：

```text
请求行：POST /
请求头：content-type、content-length 等
请求体：真正提交的数据
```

请求头可以反复查看，但请求体通常是流式读取的。  
你可以把它想成水管：

```text
body stream -> 读取一块 -> 读取下一块 -> 读完
```

读完之后，流里就没有数据了。

所以如果 middleware 做了这件事：

```text
读取整个 body
不把 body 放回 request
```

后面的 handler 就会看到一个空 body，或者直接无法再提取。

## 文件和依赖

这个 example 有两个文件：

1. `examples/consume-body-in-extractor-or-middleware/Cargo.toml`：声明 Axum、http-body-util、Tokio、tracing。
2. `examples/consume-body-in-extractor-or-middleware/src/main.rs`：实现 middleware、body buffering、自定义 extractor 和 handler。

关键依赖：

- `axum`：提供 `Body`、`Bytes`、`Request`、`FromRequest`、middleware、`Next`、`Response`。
- `http-body-util`：提供 `BodyExt::collect()`，把 body 收集成 bytes。
- `tokio`：提供异步运行时。
- `tracing` / `tracing-subscriber`：输出调试日志。

## 第一步：注册 handler 和 middleware

源码：

````rust
let app = Router::new()
    .route("/", post(handler))
    .layer(middleware::from_fn(print_request_body));
````

这表示：

```text
POST / 请求
-> 先经过 print_request_body middleware
-> 再进入 handler
```

`middleware::from_fn(...)` 可以把一个 async 函数变成 Axum middleware。

middleware 的位置很重要：  
它包在路由外层，所以会在 handler 之前运行。

## 第二步：middleware 先读取 request body

源码：

````rust
async fn print_request_body(request: Request, next: Next) -> Result<impl IntoResponse, Response> {
    let request = buffer_request_body(request).await?;

    Ok(next.run(request).await)
}
````

middleware 函数接收两个东西：

- `request: Request`：当前请求。
- `next: Next`：后续 middleware 或最终 handler。

逻辑是：

```text
先调用 buffer_request_body 读取 body
拿到重新构造好的 request
调用 next.run(request) 继续往后走
```

如果 `buffer_request_body` 出错，就提前返回 `Response`，不会进入 handler。

## 第三步：拆开 Request，收集 body，再重建 Request

核心源码：

````rust
let (parts, body) = request.into_parts();

let bytes = body
    .collect()
    .await
    .map_err(|err| (StatusCode::INTERNAL_SERVER_ERROR, err.to_string()).into_response())?
    .to_bytes();

do_thing_with_request_body(bytes.clone());

Ok(Request::from_parts(parts, Body::from(bytes)))
````

分成四步看：

1. `request.into_parts()`：把 request 拆成请求头等 metadata 和 body。
2. `body.collect().await`：把整个 body 收集到内存。
3. `do_thing_with_request_body(bytes.clone())`：对 body 做一些事情，这里只是打印。
4. `Request::from_parts(parts, Body::from(bytes))`：用原来的 parts 和同一份 bytes 重建 request。

为什么要 `bytes.clone()`？

`Bytes` 的 clone 很便宜，它不是复制整份内容，而是共享底层缓冲区。  
middleware 用 clone 出来的一份做日志，原始 bytes 继续放回 request 里。

源码注释里有一句很关键：

```text
this won't work if the body is a long-running stream
```

因为 `collect()` 会等待 body 完全结束。  
如果 body 是无限流、长连接或很大的上传文件，这种“一次性收集到内存”的方式就不合适。

## 第四步：自定义 extractor 消费 body

源码：

````rust
struct BufferRequestBody(Bytes);

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
````

`BufferRequestBody` 是一个自定义 extractor。  
它内部直接复用 Axum 已经提供的 `Bytes` extractor：

```text
Bytes::from_request(req, state)
```

这会把请求 body 读成 `Bytes`。

为什么这里实现的是 `FromRequest`，不是 `FromRequestParts`？

因为：

```text
FromRequest 可以消费请求 body。
FromRequestParts 只能读取请求头、路径、扩展等 parts，不能消费 body。
```

只要你的 extractor 要读 body，就必须实现 `FromRequest`。

## 第五步：handler 使用自定义 extractor

源码：

````rust
async fn handler(BufferRequestBody(body): BufferRequestBody) {
    tracing::debug!(?body, "handler received body");
}
````

这里的参数写法是模式匹配：

```text
BufferRequestBody(body): BufferRequestBody
```

意思是：

```text
先运行 BufferRequestBody extractor
再把包装器里的 Bytes 拿出来绑定到 body 变量
```

如果 middleware 没有把 body 重建回 request，这里的 extractor 就拿不到原始 body。

## 第六步：middleware 和 extractor 的区别

两者都能读取 body，但使用场景不同：

| 位置 | 适合做什么 | 注意点 |
| --- | --- | --- |
| middleware | 日志、审计、签名校验、统一处理 | 读完 body 后要重建 request |
| extractor | 某个 handler 专用的输入解析 | 要实现 `FromRequest` 才能读 body |

如果逻辑是所有接口都需要的，优先考虑 middleware。  
如果逻辑只属于某类 handler 参数，优先考虑 extractor。

## 函数职责速查

- `main`：初始化日志，注册 `POST /`，挂载 middleware，启动服务。
- `print_request_body`：middleware 入口，读取并重建 request，再调用后续 handler。
- `buffer_request_body`：拆 request、收集 body、打印 body、重新构造 request。
- `do_thing_with_request_body`：模拟对 body 做处理，这里是日志打印。
- `handler`：使用 `BufferRequestBody` extractor 读取 body。
- `BufferRequestBody`：包装 `Bytes` 的自定义 extractor。
- `from_request`：实现 extractor 逻辑，用 `Bytes::from_request` 消费 body。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-consume-body-in-extractor-or-middleware
//! ```

// 引入 Body、Bytes、FromRequest、Request、状态码、middleware、Next、响应和 Router。
use axum::{
    body::{Body, Bytes},
    extract::{FromRequest, Request},
    http::StatusCode,
    middleware::{self, Next},
    response::{IntoResponse, Response},
    routing::post,
    Router,
};
// BodyExt 提供 collect，用来把 body 收集成 bytes。
use http_body_util::BodyExt;
// tracing_subscriber 用来初始化日志。
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 初始化日志系统。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 注册 POST /，并挂载一个会提前读取请求体的 middleware。
    let app = Router::new()
        .route("/", post(handler))
        .layer(middleware::from_fn(print_request_body));

    // 绑定本机 3000 端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 输出监听地址。
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动 HTTP 服务。
    axum::serve(listener, app).await;
}

// middleware：演示如何在 handler 前提前消费 request body。
async fn print_request_body(request: Request, next: Next) -> Result<impl IntoResponse, Response> {
    // 读取 body，并把 body 重新放回 request。
    let request = buffer_request_body(request).await?;

    // 继续调用后续 middleware 或最终 handler。
    Ok(next.run(request).await)
}

// 拆开 request，读取 body，处理 body，再重新组装 request。
async fn buffer_request_body(request: Request) -> Result<Request, Response> {
    // 拆出请求 parts 和 body。
    let (parts, body) = request.into_parts();

    // 把 body 收集成 Bytes。大文件或长流不适合这样一次性收集。
    let bytes = body
        .collect()
        .await
        .map_err(|err| (StatusCode::INTERNAL_SERVER_ERROR, err.to_string()).into_response())?
        .to_bytes();

    // 对 body 做一些事情，这里只是打印日志。
    do_thing_with_request_body(bytes.clone());

    // 用原来的 parts 和同一份 bytes 重建 request，让后面的 handler 还能读 body。
    Ok(Request::from_parts(parts, Body::from(bytes)))
}

// 模拟对请求体做处理。
fn do_thing_with_request_body(bytes: Bytes) {
    tracing::debug!(body = ?bytes);
}

// handler 通过自定义 extractor 读取 body。
async fn handler(BufferRequestBody(body): BufferRequestBody) {
    tracing::debug!(?body, "handler received body");
}

// 自定义 extractor，内部保存读取出来的 Bytes。
struct BufferRequestBody(Bytes);

// 因为要消费 body，所以必须实现 FromRequest，而不是 FromRequestParts。
impl<S> FromRequest<S> for BufferRequestBody
where
    S: Send + Sync,
{
    // 提取失败时直接返回 Response。
    type Rejection = Response;

    // Axum 看到 handler 参数是 BufferRequestBody 时，会调用这里。
    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        // 复用 Axum 的 Bytes extractor，把 body 读成 Bytes。
        let body = Bytes::from_request(req, state)
            .await
            .map_err(|err| err.into_response())?;

        // 对 body 做处理，这里仍然只是打印日志。
        do_thing_with_request_body(body.clone());

        // 包装成 BufferRequestBody 交给 handler。
        Ok(Self(body))
    }
}
````

## 运行和验证

运行前先确认 `examples/consume-body-in-extractor-or-middleware/Cargo.toml` 里的 `axum = { path = "../../axum" }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-consume-body-in-extractor-or-middleware
````

发送请求：

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'content-type: text/plain' \
  --data-binary 'hello body'
````

预期请求成功，并在服务端日志里看到 middleware 和 handler 都打印了 body。

常见卡点：

- `collect()` 会把整个 body 收进内存，不适合无限流或超大文件。
- middleware 如果读了 body，就要重建 request，否则后续 handler 读不到。
- 想在 extractor 里读 body，要实现 `FromRequest`，不是 `FromRequestParts`。
- `Bytes::clone()` 通常很便宜，不等于复制整份 body 内容。

## 手写任务

完成本章代码后，试着做三个小改动：

1. 在 `do_thing_with_request_body` 里打印 body 长度。
2. 临时删除 `Request::from_parts(parts, Body::from(bytes))` 的重建逻辑，观察 handler 是否还能收到 body。
3. 把请求体换成 JSON 字符串发送，观察本章逻辑并不关心 content-type，只关心原始 bytes。

## 本章真正要记住什么

- 请求 body 是只能消费一次的流。
- middleware 提前读 body 后，必须重新构造 request 才能继续往后传。
- `BodyExt::collect()` 可以把 body 收集成 `Bytes`，但不适合所有场景。
- 需要消费 body 的 extractor 要实现 `FromRequest`。
- `FromRequestParts` 不能读取请求 body。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/consume-body-in-extractor-or-middleware/Cargo.toml`
- `examples/consume-body-in-extractor-or-middleware/src/main.rs`
