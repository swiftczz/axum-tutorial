# 22. print-request-response

对应示例：`examples/print-request-response`

和第 10 章关系很近。第 10 章讲"请求体只能消费一次",这章把同样的思想扩展到**响应 body**——手写打印请求和响应 body 的 middleware,理解为什么读完 body 后必须重新构造 request/response。

## Cargo.toml

````toml
[package]
name = "example-print-request-response"
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

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

````rust
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
    axum::serve(listener, app).await;
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
````

## 运行

````bash
cd examples
cargo run -p example-print-request-response
````

发送 POST:

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'content-type: text/plain' \
  --data-binary 'hello request'
````

客户端看到响应体:

````text
Hello from `POST /`
````

服务端日志看到 request body 和 response body。

## 解读

### 为什么读完 body 必须重建

body 是流,读出来打印后原来的 body 就被消费了:

- middleware 读 request body 后不重新放回去 → handler 读不到请求体。
- middleware 读 response body 后不重新放回去 → 客户端收不到响应体。

所以本章反复出现这个模式:

```text
into_parts() → collect body → 打印 → from_parts(parts, Body::from(bytes))
```

### middleware 两段对称处理

**请求段:**

````rust
let (parts, body) = req.into_parts();
let bytes = buffer_and_print("request", body).await?;
let req = Request::from_parts(parts, Body::from(bytes));

let res = next.run(req).await;  // 重建后的 request 交给 handler
````

**响应段(完全对称):**

````rust
let (parts, body) = res.into_parts();
let bytes = buffer_and_print("response", body).await?;
let res = Response::from_parts(parts, Body::from(bytes));

Ok(res)
````

### `buffer_and_print` 泛型复用

````rust
async fn buffer_and_print<B>(direction: &str, body: B) -> Result<Bytes, (StatusCode, String)>
where
    B: axum::body::HttpBody<Data = Bytes>,
    B::Error: std::fmt::Display,
{
    let bytes = body.collect().await...;
    if let Ok(body) = std::str::from_utf8(&bytes) {
        tracing::debug!("{direction} body = {body:?}");
    }
    Ok(bytes)
}
````

泛型约束 `B: HttpBody<Data = Bytes>` 表示传入的 body 能产出 `Bytes`——request body 和 response body 都满足,所以一个函数处理两边。

只在 body 是 UTF-8 文本时打印;图片、压缩包等二进制内容不打印文本。

### 这种 middleware 的边界

`body.collect().await` 把整个 body 收集进内存。适合小 JSON 请求、调试环境、本地排查;**不适合**大文件上传、长流式响应、含敏感信息的请求体。真实生产环境打印 body 要考虑脱敏和大小限制。

## 手写任务

跑通后做三个小改动:

1. 在日志里同时打印 body 长度。
2. 让 handler 返回 JSON 字符串,观察 response body 日志。
3. 给 `buffer_and_print` 加最大长度限制,只打印前 100 字节。

## 小结

- request body 和 response body 都是只能消费一次的流。
- middleware 读完 body 后必须用 `Body::from(bytes)` 重建,否则后续 handler/客户端读不到。
- `BodyExt::collect()` 适合调试小 body,不适合大文件和长流。
- 打印 body 是强力调试工具,但生产环境要谨慎(脱敏、大小限制)。

## 源码对照

- `examples/print-request-response/Cargo.toml`
- `examples/print-request-response/src/main.rs`
