# 04. global-404-handler

对应示例：`examples/global-404-handler`

前三章的服务，访问不存在的路径会返回 axum 默认的 404。本章用 `fallback` 自定义 404：任何没匹配上的路径都交给一个兜底 handler。

## Cargo.toml

````toml
[package]
name = "example-global-404-handler"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

````rust
use axum::{
    http::StatusCode,
    response::{Html, IntoResponse},
    routing::get,
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

    let app = Router::new().route("/", get(handler));
    let app = app.fallback(handler_404);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app).await;
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}

async fn handler_404() -> impl IntoResponse {
    (StatusCode::NOT_FOUND, "nothing to see here")
}
````

## 运行

````bash
cd examples
cargo run -p example-global-404-handler
````

正常路径：

````bash
curl -i http://127.0.0.1:3000/
````

未知路径：

````bash
curl -i http://127.0.0.1:3000/unknown
````

预期状态码 `HTTP/1.1 404 Not Found`，响应体：

````text
nothing to see here
````

再测一个容易混淆的情况——方法不匹配：

````bash
curl -i -X POST http://127.0.0.1:3000/
````

`/` 路径存在，但只有 GET。这用来观察"方法不匹配"和"路径不存在"是两个不同的问题。

## 解读

### `fallback` 是什么

请求进入 Router 后，分两条路：

```text
GET /         → 匹配成功 → handler 返回 HTML
GET /missing  → 没有 route 匹配 → fallback(handler_404) → 返回 404
```

`fallback` 就是路由表最后的兜底处理器，**只在没有任何路由匹配时**执行。

### `let app = app.fallback(...)`

````rust
let app = app.fallback(handler_404);
````

很多 builder API 会消费旧值返回新值，所以重新 `let`。这行在原 Router 基础上加一个兜底 handler。

### `fallback` 不处理"方法不匹配"

这点容易混淆，用一张表区分三类常见错误：

| 情况 | 示例 | 含义 | 本章是否处理 |
| --- | --- | --- | --- |
| 路径不存在 | `GET /missing` | Router 找不到这个路径 | 是，交给 `fallback` |
| 路径存在但方法不对 | `POST /` | 有 `/`，但没有对应的 POST 处理 | 不是 fallback 的重点 |
| handler 内部失败 | 数据库错误 | 路由匹配成功，但处理逻辑出错 | 后续错误处理章节 |

### `handler_404`

````rust
async fn handler_404() -> impl IntoResponse {
    (StatusCode::NOT_FOUND, "nothing to see here")
}
````

404 handler 也是普通 handler，返回 `(StatusCode, body)` 元组，axum 自动转成 HTTP 响应。真实项目里 JSON API 通常返回 JSON 错误（见手写任务）。

### tracing 初始化

````rust
tracing_subscriber::registry()
    .with(tracing_subscriber::EnvFilter::try_from_default_env()...)
    .with(tracing_subscriber::fmt::layer())
    .init();
````

这不是 404 的核心，但真实后端一般都初始化日志。`EnvFilter` 让你通过 `RUST_LOG` 环境变量控制日志级别，`fmt::layer()` 负责格式化输出到终端。

## 手写任务

跑通后做两个小改动：

1. 把 404 响应文本改成 `page not found`。
2. 把 404 handler 改成返回 JSON，例如 `{"error":"not found"}`：

   ````rust
   (StatusCode::NOT_FOUND, Json(serde_json::json!({ "error": "not found" })))
   ````

   这需要给 `Cargo.toml` 加 `serde_json` 依赖，并 `use axum::Json`。

## 小结

- `fallback` 是 Router 的兜底处理器，只在没有任何路由匹配时执行。
- 路径匹配但方法不匹配，不是 fallback 404，要分开理解。
- 404 handler 也是普通 handler，`(StatusCode, body)` 可以直接作为响应。
- 真实项目应该统一 404 响应格式，JSON API 通常返回 JSON 错误。

## 源码对照

- `examples/global-404-handler/Cargo.toml`
- `examples/global-404-handler/src/main.rs`
