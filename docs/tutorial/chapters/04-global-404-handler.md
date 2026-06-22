# 04. global-404-handler

对应示例：`examples/global-404-handler`

本章目标：学会用 `fallback` 处理未知路由。

## 这个小项目在做什么

前三章的服务都有一个问题：如果用户访问不存在的路径，例如 `/abc`，默认 404 响应不一定是你想要的格式。

这一章演示：

```text
已知路径：GET / -> 返回 Hello 页面
未知路径：任何没有匹配上的路径 -> 返回自定义 404
```

这就是全局 404 处理。

## 新手先抓住主线

请求进入 Router 后有两种情况：

```text
GET /
-> 匹配成功
-> handler 返回 HTML

GET /missing
-> 没有任何 route 匹配
-> fallback(handler_404)
-> 返回 404 状态码和文本
```

所以 `fallback` 可以理解成：

```text
路由表最后的兜底处理器。
```

## 文件和依赖

这个 example 有两个文件：

1. `examples/global-404-handler/Cargo.toml`：声明 Axum、Tokio 和 tracing 相关依赖。
2. `examples/global-404-handler/src/main.rs`：注册正常路由、fallback，并启动服务。

关键依赖：

- `axum`：提供 `Router`、`fallback`、`StatusCode` 和响应转换能力。
- `tokio`：提供异步运行时。
- `tracing` / `tracing-subscriber`：初始化日志，方便观察服务启动和请求处理。

## 第一步：正常注册首页路由

源码：

````rust
let app = Router::new().route("/", get(handler));
````

这和第一章一样：

```text
GET / -> handler
```

## 第二步：添加 fallback

源码：

````rust
let app = app.fallback(handler_404);
````

为什么这里重新 `let app = ...`？

因为 Rust 里很多 builder 风格 API 会消费旧值并返回新值。  
这行的意思是：

```text
在原来的 app 基础上，增加一个兜底 handler。
```

`fallback` 只有在没有路由匹配时才会执行。

这里要和另一种情况区分开：

```text
路径匹配了，但 HTTP 方法不匹配
```

例如应用里只有 `GET /`，你发 `POST /`，这通常不是“未知路径”，而是“这个路径不支持 POST”。这类响应和 fallback 404 不是同一个概念。

可以先用这张表区分三类常见错误：

| 情况 | 示例 | 含义 | 本章是否处理 |
| --- | --- | --- | --- |
| 路径不存在 | `GET /missing` | Router 找不到这个路径 | 是，交给 `fallback` |
| 路径存在但方法不对 | `POST /` | 有 `/`，但没有对应的 POST 处理 | 不是 fallback 的重点 |
| handler 内部失败 | 数据库错误、业务校验失败 | 路由匹配成功，但处理逻辑出错 | 后续错误处理章节再讲 |

## 第三步：写 404 handler

源码：

````rust
async fn handler_404() -> impl IntoResponse {
    (StatusCode::NOT_FOUND, "nothing to see here")
}
````

这段返回一个二元组：

```text
(状态码, 响应体)
```

Axum 知道如何把这个二元组转换成 HTTP 响应。

为什么返回 `impl IntoResponse`？

因为我们不想手动构造底层 `Response`，只要返回能变成响应的东西即可。

## 第四步：加入 tracing 日志

这个 example 比第一章多了日志初始化：

````rust
tracing_subscriber::registry()
    .with(...)
    .with(tracing_subscriber::fmt::layer())
    .init();
````

新手可以先这样理解：

```text
tracing_subscriber 用来配置日志。
EnvFilter 让你可以通过环境变量控制日志级别。
fmt::layer() 负责把日志格式化输出到终端。
```

这不是 404 的核心，但是真实后端一般都会初始化日志。

## 函数职责速查

- `main`：初始化日志，注册首页路由，添加 `fallback`，绑定端口并启动服务。
- `handler`：处理 `GET /`，返回首页 HTML。
- `handler_404`：处理未匹配路径，返回 `404 Not Found` 和文本响应。

这一章真正需要盯住的是 `app.fallback(handler_404)`：它把未知路径统一交给 `handler_404`，而不是让每个 route 自己处理 404。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-global-404-handler
//! ```

// 引入状态码、Html 响应、IntoResponse、GET 路由函数和 Router。
use axum::{
    http::StatusCode,
    response::{Html, IntoResponse},
    routing::get,
    Router,
};

// 引入 tracing subscriber 的扩展 trait，用来组装日志层。
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 初始化 tracing 日志系统。
    tracing_subscriber::registry()
        .with(
            // 尝试从环境变量读取日志级别；没有配置时默认当前 crate 为 debug。
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        // 把日志格式化输出到终端。
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 注册正常路由：GET / 交给 handler。
    let app = Router::new().route("/", get(handler));

    // 给所有未匹配路径添加兜底 handler。
    let app = app.fallback(handler_404);

    // 绑定监听端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 打印监听地址。
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动服务。
    axum::serve(listener, app).await;
}

// 首页 handler，返回 HTML。
async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}

// 404 handler，返回 404 状态码和文本响应。
async fn handler_404() -> impl IntoResponse {
    (StatusCode::NOT_FOUND, "nothing to see here")
}
````

## 运行和验证

运行前先确认 `examples/global-404-handler/Cargo.toml` 里的 `axum = { path = "../../axum" }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-global-404-handler
````

访问正常路径：

````bash
curl -i http://127.0.0.1:3000/
````

访问未知路径：

````bash
curl -i http://127.0.0.1:3000/unknown
````

你应该看到状态码是：

````text
HTTP/1.1 404 Not Found
````

响应体是：

````text
nothing to see here
````

再测试一个容易混淆的情况：

````bash
curl -i -X POST http://127.0.0.1:3000/
````

这个请求路径 `/` 是存在的，但方法不是 GET。它主要用来观察“方法不匹配”和“路径不存在”不是同一个问题。

## 手写任务

完成本章代码后，试着做两个小改动：

1. 把 404 响应文本改成 `page not found`。
2. 把 404 handler 改成返回 JSON，例如 `{"error":"not found"}`。

第二个任务可以先写成这种形式：

````rust
(StatusCode::NOT_FOUND, Json(serde_json::json!({ "error": "not found" })))
````

如果你这样写，需要给 `Cargo.toml` 增加 `serde_json` 依赖，并在代码里引入 `axum::Json`。

## 本章真正要记住什么

- `fallback` 是 Router 的兜底处理器。
- 它只在没有路由匹配时执行。
- 路径匹配但方法不匹配，不要简单理解成 fallback 404。
- 404 handler 也只是普通 handler。
- `(StatusCode, body)` 可以直接变成响应。
- 真实项目应该统一设计 404 响应格式，例如 JSON API 通常返回 JSON 错误。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/global-404-handler/Cargo.toml`
- `examples/global-404-handler/src/main.rs`
