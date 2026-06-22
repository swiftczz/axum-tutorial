# 10. consume-body-in-extractor-or-middleware

对应示例：`examples/consume-body-in-extractor-or-middleware`

这一章只讲一件事：**请求 body 是一条只能读一次的流**。如果 middleware 或 extractor 提前读了 body，必须把它放回 request，否则后面的 handler 读不到。

分 3 步：先看问题（body 被吃掉了），再写 middleware 的解法，最后写 extractor 的解法。

## Cargo.toml

````toml
[package]
name = "example-consume-body-in-extractor-or-middleware"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
http-body-util = "0.1.0"
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

`http-body-util` 提供 `BodyExt::collect()`，把 body 收集成 bytes。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：先写一个普通 handler 和 middleware 骨架

先搭一个能跑的最小服务：`POST /` 接收 body，middleware 打印"请求来了"，handler 打印"收到 body"。

````rust
use axum::{
    extract::Request,
    middleware::{self, Next},
    response::{IntoResponse, Response},
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

    let app = Router::new()
        .route("/", post(handler))
        .layer(middleware::from_fn(print_request_body));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}

async fn print_request_body(request: Request, next: Next) -> Result<impl IntoResponse, Response> {
    // middleware 先拿到 request，做点什么，再交给 handler
    Ok(next.run(request).await)
}

async fn handler() {
    tracing::debug!("handler called");
}
````

跑起来验证：

````bash
curl -X POST http://127.0.0.1:3000/ -H 'content-type: text/plain' --data-binary 'hello body'
````

服务端日志能看到 "handler called"。但现在 middleware 还没碰 body——下一步加。

---

## 第二步：middleware 读 body——问题与解法

现在让 middleware **打印请求 body**。直觉写法是直接读 body，但这会出问题。

> **新面孔：body 是只能读一次的流**
>
> HTTP 请求体不是存在内存里的字符串，而是**流式**的——像水管一样，读一块、再读一块、读完就没了。
>
> 如果 middleware 把 body 读完了，不把 body 放回 request，后面的 handler 就会看到空 body，或直接无法提取。

解法：**拆开 request → 收集 body 到 bytes → 用 bytes 重建 request**。

````rust
use axum::{
    body::{Body, Bytes},
    http::StatusCode,
    response::IntoResponse,
};
use http_body_util::BodyExt;

async fn print_request_body(request: Request, next: Next) -> Result<impl IntoResponse, Response> {
    let request = buffer_request_body(request).await?;
    Ok(next.run(request).await)
}

async fn buffer_request_body(request: Request) -> Result<Request, Response> {
    // 1. 拆成 parts（请求头等 metadata）和 body
    let (parts, body) = request.into_parts();

    // 2. 把整个 body 收集成 bytes
    let bytes = body
        .collect()
        .await
        .map_err(|err| (StatusCode::INTERNAL_SERVER_ERROR, err.to_string()).into_response())?
        .to_bytes();

    // 3. 对 body 做点事（这里打印日志）
    do_thing_with_request_body(bytes.clone());

    // 4. 用原来的 parts 和同一份 bytes 重建 request
    Ok(Request::from_parts(parts, Body::from(bytes)))
}

fn do_thing_with_request_body(bytes: Bytes) {
    tracing::debug!(body = ?bytes);
}
````

> **新面孔：`into_parts` / `collect` / `from_parts` 三步重建**
>
> - `request.into_parts()`：把 request 拆成 `parts`（method、uri、headers 等）和 `body`。
> - `body.collect().await`：把流式 body 全部收集到内存。注意注释里说 "this won't work if the body is a long-running stream"——如果是无限流或超大文件，这种一次性收集不合适。
> - `Request::from_parts(parts, Body::from(bytes))`：用原 parts 和同一份 bytes 重建 request。
>
> `bytes.clone()` 很便宜——`Bytes` 的 clone 不复制内容，而是共享底层缓冲区。middleware 用 clone 做日志，原始 bytes 放回 request。

---

## 第三步：extractor 读 body——自定义 `BufferRequestBody`

除了 middleware，也可以在 **extractor** 里提前读 body。相比前面章节新引入：自定义 extractor 实现 `FromRequest`（不是 `FromRequestParts`）。

````rust
use axum::{
    body::Bytes,
    extract::{FromRequest, Request},
    response::IntoResponse,
};

async fn handler(BufferRequestBody(body): BufferRequestBody) {
    tracing::debug!(?body, "handler received body");
}

struct BufferRequestBody(Bytes);

impl<S> FromRequest<S> for BufferRequestBody
where
    S: Send + Sync,
{
    type Rejection = Response;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        // 复用 axum 的 Bytes extractor，把 body 读成 Bytes
        let body = Bytes::from_request(req, state)
            .await
            .map_err(|err| err.into_response())?;

        do_thing_with_request_body(body.clone());

        Ok(Self(body))
    }
}
````

> **新面孔：`FromRequest` vs `FromRequestParts`**
>
> 实现了 `FromRequest` 的类型可以从请求里提取自己，包括**消费 body**。
>
> 关键区别：
> - `FromRequest`：能消费 body（Json、Form、Bytes 等）。
> - `FromRequestParts`：只能读 parts（method、uri、headers），**不能**消费 body。
>
> 只要 extractor 要读 body，就必须实现 `FromRequest`。这章的 `BufferRequestBody` 内部复用 axum 的 `Bytes::from_request`，把请求体读成 `Bytes`。

---

## 完整代码

````rust
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

    axum::serve(listener, app).await;
}

async fn print_request_body(request: Request, next: Next) -> Result<impl IntoResponse, Response> {
    let request = buffer_request_body(request).await?;

    Ok(next.run(request).await)
}

async fn buffer_request_body(request: Request) -> Result<Request, Response> {
    let (parts, body) = request.into_parts();

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

## 运行

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

请求成功，服务端日志里 middleware 和 handler 都打印了 body。

## 常见问题

**middleware 和 extractor 都能读 body，怎么选？** 所有接口都需要的逻辑优先 middleware（日志、审计）；只属于某类 handler 参数的逻辑优先 extractor。

**`collect()` 适合大文件吗？** 不适合。它会把整个 body 收进内存。无限流或超大文件应该用流式处理（见 ch08 stream-to-file）。

## 手写任务

1. 在 `do_thing_with_request_body` 里打印 body 长度。
2. 临时删掉 `Request::from_parts(parts, Body::from(bytes))` 的重建逻辑，观察 handler 是否还能收到 body。
3. 把请求体换成 JSON 字符串，观察本章逻辑不关心 content-type，只关心原始 bytes。

## 小结

这章分 3 步讲了"body 只能消费一次"：

1. **骨架**：middleware + handler 的基本结构，先不碰 body。
2. **middleware 解法**：`into_parts` → `collect` → `from_parts` 三步重建 request，body 读完后放回去。
3. **extractor 解法**：自定义 `BufferRequestBody` 实现 `FromRequest`（不是 `FromRequestParts`），内部复用 `Bytes::from_request`。

核心规则：**body 只能读一次。谁先读完，必须放回去，后面的才读得到。**

## 源码对照

- `examples/consume-body-in-extractor-or-middleware/Cargo.toml`
- `examples/consume-body-in-extractor-or-middleware/src/main.rs`
