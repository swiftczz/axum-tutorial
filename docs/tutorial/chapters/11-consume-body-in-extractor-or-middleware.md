# 11. consume-body-in-extractor-or-middleware

对应示例：`examples/consume-body-in-extractor-or-middleware`

这一章只讲一件事:**请求 body 是一条只能读一次的流**。如果 middleware 或 extractor 提前读了 body,必须把它放回 request,否则后面的 handler 读不到。本章演示 middleware 和 extractor 两个位置提前读取 body 的写法。

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

`http-body-util` 提供 `BodyExt::collect()`,把 body 收集成 bytes。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

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

发送请求:

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'content-type: text/plain' \
  --data-binary 'hello body'
````

请求成功,服务端日志里 middleware 和 handler 都打印了 body。

## 解读

### body 只能消费一次

HTTP 请求由三部分组成:请求行、请求头、请求体。请求头可以反复查看,但请求体是流式读取的——像水管一样,读一块、再读一块、读完就没了。

所以 middleware 如果这样做:

```text
读取整个 body
不把 body 放回 request
```

后面的 handler 就会看到空 body,或直接无法提取。

### middleware 读 body 必须重建 request

````rust
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
````

四步:

1. `request.into_parts()`:把 request 拆成 parts(请求头等 metadata)和 body。
2. `body.collect().await`:把整个 body 收集到内存。
3. `do_thing_with_request_body(bytes.clone())`:对 body 做点事,这里只打印。
4. `Request::from_parts(parts, Body::from(bytes))`:用原 parts 和同一份 bytes 重建 request。

`bytes.clone()` 很便宜——`Bytes` 的 clone 不复制内容,而是共享底层缓冲区。middleware 用 clone 做日志,原始 bytes 放回 request。

源码注释里那句 "this won't work if the body is a long-running stream" 很关键:`collect()` 会等 body 完全结束。如果是无限流、长连接或超大文件,这种"一次性收集到内存"的方式不合适。

### 自定义 extractor 也能读 body

````rust
impl<S> FromRequest<S> for BufferRequestBody
where
    S: Send + Sync,
{
    type Rejection = Response;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let body = Bytes::from_request(req, state).await.map_err(|err| err.into_response())?;
        ...
    }
}
````

复用 axum 的 `Bytes::from_request`,把请求体读成 `Bytes`。

关键区别:实现 **`FromRequest`** 才能消费 body;**`FromRequestParts`** 只能读 parts(请求头、路径、扩展),**不能**消费 body。只要 extractor 要读 body,就必须实现 `FromRequest`。

### middleware vs extractor

| 位置 | 适合做什么 | 注意点 |
| --- | --- | --- |
| middleware | 日志、审计、签名校验、统一处理 | 读完 body 要重建 request |
| extractor | 某个 handler 专用的输入解析 | 要实现 `FromRequest` 才能读 body |

所有接口都需要的逻辑优先 middleware;只属于某类 handler 参数的逻辑优先 extractor。

## 手写任务

跑通后做三个小改动:

1. 在 `do_thing_with_request_body` 里打印 body 长度。
2. 临时删掉 `Request::from_parts(parts, Body::from(bytes))` 的重建逻辑,观察 handler 是否还能收到 body。
3. 把请求体换成 JSON 字符串,观察本章逻辑不关心 content-type,只关心原始 bytes。

## 小结

- 请求 body 是只能消费一次的流。
- middleware 提前读 body 后,必须重建 request 才能继续往后传。
- `BodyExt::collect()` 把 body 收集成 `Bytes`,但不适合无限流或超大文件。
- 需要消费 body 的 extractor 要实现 `FromRequest`;`FromRequestParts` 不能读 body。
- `Bytes::clone()` 很便宜,共享底层缓冲区。

## 源码对照

- `examples/consume-body-in-extractor-or-middleware/Cargo.toml`
- `examples/consume-body-in-extractor-or-middleware/src/main.rs`
