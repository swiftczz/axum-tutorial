# 22. print-request-response

对应示例：`examples/print-request-response`

本章目标：手写打印请求和响应 body 的 middleware，理解为什么读完 body 后必须重新构造 request/response。

这章和第 10 章关系很近。第 10 章讲“请求体只能消费一次”。  
这一章把同样的思想扩展到：

```text
请求 body
响应 body
```

## 这个小项目在做什么

应用只有一个接口：

```text
POST /
```

handler 返回固定文本：

```text
Hello from `POST /`
```

middleware 做两件事：

1. 在 handler 前读取并打印 request body。
2. 在 handler 后读取并打印 response body。

请求主线是：

```text
客户端 POST /
-> print_request_response 读取 request body
-> 重新构造 Request
-> next.run(req) 调用 handler
-> 读取 response body
-> 重新构造 Response
-> 返回给客户端
```

## 先理解为什么要重建 body

body 是流。你把它读出来打印之后，原来的 body 就被消费掉了。

如果 middleware 只做：

```text
读取 request body
不重新放回去
```

后面的 handler 就读不到请求体。

如果 middleware 只做：

```text
读取 response body
不重新放回去
```

客户端就收不到响应体。

所以本章反复出现这个模式：

```text
into_parts()
-> collect body
-> 打印
-> from_parts(parts, Body::from(bytes))
```

## 文件和依赖

这个 example 有两个文件：

1. `examples/print-request-response/Cargo.toml`：声明 Axum、http-body-util、Tokio、tracing。
2. `examples/print-request-response/src/main.rs`：实现 middleware、body buffering 和简单 POST handler。

关键依赖：

- `axum`：提供 `Body`、`Bytes`、`Request`、`Response`、middleware。
- `http-body-util`：提供 `BodyExt::collect()`。
- `tokio`：异步运行时。
- `tracing` / `tracing-subscriber`：输出日志。

## 第一步：注册路由和 middleware

源码：

````rust
let app = Router::new()
    .route("/", post(|| async move { "Hello from `POST /`" }))
    .layer(middleware::from_fn(print_request_response));
````

路由很简单：

```text
POST / -> 返回 Hello from `POST /`
```

重点是：

```text
middleware::from_fn(print_request_response)
```

它把 `print_request_response` 函数变成 middleware，包在 handler 外面。

## 第二步：读取并重建 request

源码：

````rust
let (parts, body) = req.into_parts();
let bytes = buffer_and_print("request", body).await?;
let req = Request::from_parts(parts, Body::from(bytes));
````

这三行做了完整的“读 body 后放回去”：

1. `req.into_parts()`：拆成 request parts 和 body。
2. `buffer_and_print("request", body)`：读取 body 并打印。
3. `Request::from_parts(parts, Body::from(bytes))`：用 bytes 重建 request。

如果不做第三步，后面的 handler 拿不到 body。

## 第三步：调用后续 handler

源码：

````rust
let res = next.run(req).await;
````

`next.run(req)` 表示：

```text
把重建后的 request 交给后面的 middleware 或 handler
```

返回值是 response。  
接下来 middleware 还要读取这个 response 的 body。

## 第四步：读取并重建 response

源码：

````rust
let (parts, body) = res.into_parts();
let bytes = buffer_and_print("response", body).await?;
let res = Response::from_parts(parts, Body::from(bytes));

Ok(res)
````

这和 request 的处理完全对称：

1. 拆开 response。
2. 读取 response body 并打印。
3. 用原来的 parts 和 bytes 重建 response。
4. 返回给客户端。

如果不重建 response，客户端就拿不到原始响应体。

## 第五步：通用的 buffer_and_print

源码：

````rust
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

这个函数可以处理 request body，也可以处理 response body。

泛型约束：

```rust
B: axum::body::HttpBody<Data = Bytes>
```

表示传进来的 body 能产出 `Bytes`。

它只在 body 是 UTF-8 文本时打印：

````rust
if let Ok(body) = std::str::from_utf8(&bytes) {
    tracing::debug!("{direction} body = {body:?}");
}
````

如果是图片、压缩包等二进制内容，就不打印文本。

## 第六步：这种 middleware 的边界

这个 example 很适合学习，但真实项目要谨慎使用。

因为：

```text
body.collect().await
```

会把整个 body 收集进内存。

适合：

- 小 JSON 请求。
- 调试环境。
- 本地排查问题。

不适合：

- 大文件上传。
- 长流式响应。
- 包含敏感信息的请求体。

真实生产环境里打印 body 时要考虑脱敏和大小限制。

## 函数职责速查

- `main`：初始化日志，注册 POST 路由和打印 middleware，启动服务。
- `print_request_response`：读取并打印 request body，再读取并打印 response body。
- `buffer_and_print`：把 body 收集成 `Bytes`，如果是 UTF-8 就打印。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-print-request-response
//! ```

// 引入 Body、Bytes、Request、状态码、middleware、Next、响应转换、路由和 Router。
use axum::{
    body::{Body, Bytes},
    extract::Request,
    http::StatusCode,
    middleware::{self, Next},
    response::{IntoResponse, Response},
    routing::post,
    Router,
};
// BodyExt 提供 collect。
use http_body_util::BodyExt;
// tracing_subscriber 用来初始化日志。
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 初始化日志系统。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 注册 POST /，并挂载打印 request/response body 的 middleware。
    let app = Router::new()
        .route("/", post(|| async move { "Hello from `POST /`" }))
        .layer(middleware::from_fn(print_request_response));

    // 绑定本机 3000 端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}

// middleware：打印 request body 和 response body。
async fn print_request_response(
    req: Request,
    next: Next,
) -> Result<impl IntoResponse, (StatusCode, String)> {
    // 拆开 request，读取并打印 body，然后重建 request。
    let (parts, body) = req.into_parts();
    let bytes = buffer_and_print("request", body).await?;
    let req = Request::from_parts(parts, Body::from(bytes));

    // 调用后续 handler。
    let res = next.run(req).await;

    // 拆开 response，读取并打印 body，然后重建 response。
    let (parts, body) = res.into_parts();
    let bytes = buffer_and_print("response", body).await?;
    let res = Response::from_parts(parts, Body::from(bytes));

    Ok(res)
}

// 通用函数：把 body 收集成 Bytes，并在它是 UTF-8 时打印。
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

    // 只打印 UTF-8 文本 body，二进制内容不打印。
    if let Ok(body) = std::str::from_utf8(&bytes) {
        tracing::debug!("{direction} body = {body:?}");
    }

    Ok(bytes)
}
````

## 运行和验证

运行前先确认 `examples/print-request-response/Cargo.toml` 里的 `axum = { path = "../../axum" }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-print-request-response
````

发送 POST：

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'content-type: text/plain' \
  --data-binary 'hello request'
````

客户端应看到响应体：

````text
Hello from `POST /`
````

服务端日志应能看到 request body 和 response body。

常见卡点：

- 打印 body 会消费 body，所以必须重建 request/response。
- `collect()` 会把整个 body 收进内存，不适合大文件或长流。
- 只打印 UTF-8 body，二进制 body 不会显示成文本。
- 生产环境打印 body 要注意隐私、密码、token 等敏感信息。

## 手写任务

完成本章代码后，试着做三个小改动：

1. 在日志里同时打印 body 长度。
2. 让 handler 返回 JSON 字符串，观察 response body 日志。
3. 给 `buffer_and_print` 加一个最大长度限制，只打印前 100 字节。

## 本章真正要记住什么

- request body 和 response body 都是只能消费一次的流。
- middleware 读完 body 后，必须用 `Body::from(bytes)` 重建。
- `BodyExt::collect()` 适合调试小 body，不适合大文件和长流。
- 打印 body 是强力调试工具，但生产环境要谨慎。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/print-request-response/Cargo.toml`
- `examples/print-request-response/src/main.rs`
