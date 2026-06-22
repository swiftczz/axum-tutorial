# 23. print-request-response

对应示例：`examples/print-request-response`

第 22 章用 `TraceLayer` 给请求加日志，但那是结构化 tracing span，body 内容看不到。这章写一个**自定义 middleware**，打印**完整的请求 body 和响应 body**——开发调试时超有用（"客户端到底发了什么？handler 返回了什么？"）。

分 3 步：先写 middleware 骨架（不碰 body，只打一行日志），再加"读请求 body → 放回 → 打印"逻辑，最后加响应 body 同样处理。

相比前面章节新引入：**`middleware::from_fn`、`Next::run`、`into_parts` / `from_parts` 重建 request/response、`HttpBody::collect`**。

## Cargo.toml

````toml
[package]
name = "example-print-request-response"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
http-body-util = "0.1"
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：middleware 骨架（不碰 body）

先写最简 middleware：拿到 request，调 `next.run(req)` 交给 handler，handler 返回响应后原样返回。这步还不打印 body，只验证 middleware 基本结构。

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
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app = Router::new()
        .route("/", post(|| async move { "Hello from `POST /`" }))
        .layer(middleware::from_fn(print_request_response));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn print_request_response(
    req: Request,
    next: Next,
) -> Result<impl IntoResponse, (StatusCode, String)> {
    // 暂时不打印 body，先看 middleware 是否接进来
    tracing::debug!("got request!");
    let res = next.run(req).await;
    tracing::debug!("got response!");
    Ok(res)
}

use axum::http::StatusCode;
````

验证：

````bash
cd examples
cargo run -p example-print-request-response

curl -X POST http://127.0.0.1:3000/ -d "hello body"
# Hello from `POST /`
````

服务端日志看到 `got request!` 和 `got response!`。

> **新面孔：`middleware::from_fn`**
>
> axum 内置的 middleware 工厂——把 async fn 包装成 layer。函数签名 `(req: Request, next: Next) -> Result<impl IntoResponse, _>`：
>
> - `req: Request`：完整请求（包含 body）
> - `next: Next`：调用 `next.run(req).await` 交给下一个 layer/handler
> - 返回值：响应（可改）
>
> 和 ch10 同款 API。`from_fn` 比 tower 的手写 Service 简单很多。

> **新面孔：`Next::run`**
>
> `next.run(req).await` 把 request 传给后续 middleware/handler，返回 Response。**调用 `next.run` 前不能消费 body**（handler 还要用）；调用后 request 就消费了，handler 返回的 Response 可以重新组装。

这步 middleware 还没碰 body，所以 handler 能正常收到 body。下一步加打印逻辑。

---

## 第二步：打印请求 body

要打印请求 body 必须先**读出来**——但 body 是只能消费一次的流（ch10 讲过），读完不还回去 handler 就收不到。这步用 `into_parts → collect → from_parts` 三步重建 request。

````rust
use axum::{
    body::{Body, Bytes},
    http::StatusCode,
};
use http_body_util::BodyExt;

async fn print_request_response(
    req: Request,
    next: Next,
) -> Result<impl IntoResponse, (StatusCode, String)> {
    // 1. 拆开 request
    let (parts, body) = req.into_parts();

    // 2. 把 body 读成 bytes（顺便打印）
    let bytes = buffer_and_print("request", body).await?;

    // 3. 用原 parts + 读到的 bytes 重建 request
    let req = Request::from_parts(parts, Body::from(bytes));

    // 4. 交给 handler
    let res = next.run(req).await;

    // 下一步加响应 body 打印
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

> **新面孔：`into_parts → collect → from_parts` 重建 request**
>
> ch10 详细讲过这个模式。这里复习：
>
> - `req.into_parts()`：拆成 `parts`（method/uri/headers）和 `body`
> - `body.collect().await`：把流式 body 读成完整 bytes
> - `Request::from_parts(parts, Body::from(bytes))`：用原 parts 和 bytes 重建 request
>
> `bytes.clone()` 很便宜（Bytes 用 Arc 共享缓冲区），但这里直接消费 bytes 因为重建需要 owned body。

> **新面孔：`buffer_and_print` 泛型函数**
>
> 接收任何 `HttpBody<Data = Bytes>` 的 body，返回 `Bytes`。同时把 UTF-8 可解码的 body 打印出来。函数参数 `direction: &str` 区分"request"和"response"，日志里能看出来。

> **新面孔：`HttpBody` trait + `collect`**
>
> `axum::body::HttpBody` 是 axum 的 body trait（来自 hyper）。`.collect().await` 把流式 body 全部收进内存，返回 `Collected`，`.to_bytes()` 转成 `Bytes`。
>
> 这是 ch10 用的 `BodyExt::collect` 的 trait-bound 版本——这章因为函数是泛型的，必须显式约束 `B: HttpBody<Data = Bytes>`。

验证：

````bash
curl -X POST http://127.0.0.1:3000/ -d "hello body"
````

服务端日志现在有 `request body = "hello body"`。

---

## 第三步：打印响应 body

响应 body 同样是流式的——`next.run(req).await` 返回的 Response 的 body 也是 stream。这步对 response 做同样的"读 → 打印 → 重建"。

````rust
async fn print_request_response(
    req: Request,
    next: Next,
) -> Result<impl IntoResponse, (StatusCode, String)> {
    // 请求 body
    let (parts, body) = req.into_parts();
    let bytes = buffer_and_print("request", body).await?;
    let req = Request::from_parts(parts, Body::from(bytes));

    // 调 handler
    let res = next.run(req).await;

    // 响应 body：同样的模式
    let (parts, body) = res.into_parts();
    let bytes = buffer_and_print("response", body).await?;
    let res = Response::from_parts(parts, Body::from(bytes));

    Ok(res)
}
````

验证：

````bash
curl -X POST http://127.0.0.1:3000/ -d "hello body"
# Hello from `POST /`
````

服务端日志：

```text
DEBUG request body = "hello body"
DEBUG response body = "Hello from `POST /`"
```

完整请求和响应都打印出来了。

---

## 完整代码

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

curl -X POST http://127.0.0.1:3000/ -d "hello body"
````

服务端日志：

```text
DEBUG request body = "hello body"
DEBUG response body = "Hello from `POST /`"
```

## 解读

### 适用场景

- **开发调试**：看客户端到底发了什么、handler 返回了什么
- **协议逆向**：分析第三方 SDK 的请求格式
- **审计日志**：合规要求记录所有请求/响应内容
- **错误复现**：线上 bug 时看完整 body 帮助复现

### 注意事项

**生产慎用**：

1. **性能**：每个请求 body 全部缓冲进内存，大文件上传会爆内存
2. **隐私**：body 可能含密码、token、个人信息，日志泄露风险大
3. **存储**：日志量爆炸（每个请求都打 body）

生产用的话：

- 限制 body 大小（超过 10KB 不打印）
- 脱敏敏感字段（password、token 替换成 `***`）
- 用采样（1% 请求打 body）

## 常见问题

**为什么不能直接 `println!("{:?}", req.body())`？** body 是 stream 不是 buffer，`Debug` 实现打印不出内容。必须 `.collect().await` 读出来。

**为什么 body 必须"读 → 还回去"？** HTTP body 只能消费一次。middleware 读完了不还，handler 收到空 body 或解析失败。这是 ch10 的核心点。

**middleware 能改 body 吗？** 能——读完 bytes 后改写，再用 `Body::from(new_bytes)` 重建。比如自动解压 gzip body。

## 手写任务

1. 加 body 大小限制：超过 1MB 不打印（防止日志爆炸）。
2. 脱敏 password 字段：用正则把 `"password":"xxx"` 替换成 `"password":"***"`。
3. 改成只对 `/api/*` 路径生效（用 `Router::route_layer` 而不是全局 `.layer`）。
4. 同时打印请求 headers（注意 Authorization 也要脱敏）。

## 小结

这章用 3 步讲了请求/响应 body 日志：

1. **middleware 骨架**：`middleware::from_fn` + `Next::run`，先不碰 body。
2. **请求 body**：`req.into_parts()` → `body.collect()` → `Request::from_parts(parts, Body::from(bytes))` 重建，顺便打印。
3. **响应 body**：同样的"读 → 打印 → 重建"模式。

核心：**body 是只能消费一次的流**（ch10 的核心概念），打印必须读出来再放回去。这个 middleware 是开发调试的瑞士军刀，但生产慎用（性能、隐私、存储）。

## 源码对照

- `examples/print-request-response/Cargo.toml`
- `examples/print-request-response/src/main.rs`
