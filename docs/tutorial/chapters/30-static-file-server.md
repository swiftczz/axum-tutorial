# 30. static-file-server

对应示例：`examples/static-file-server`

前面章节 handler 都是动态生成响应。这章讲怎么用 `tower-http` 的 `ServeDir` / `ServeFile` 把本地文件目录暴露成 HTTP 接口——这是搭建静态站点（HTML/CSS/JS）、SPA 前端托管、文件下载服务的基础。

示例一口气展示了**七种**静态文件服务写法。本章按由简到繁拆成 3 步：先用最基础的 `ServeDir`，再加 fallback（404 → `index.html`，SPA 路由模式），最后看几种进阶变体（`ServeFile`、`oneshot` 在 handler 内调用）。

相比前面章节新引入：**`tower-http` crate、`ServeDir`/`ServeFile` service、`nest_service` 挂载 service、`fallback_service`、`SetStatus`、`HandlerWithoutStateExt::into_service`**。

## Cargo.toml

````toml
[package]
name = "example-static-file-server"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }
tower = { version = "0.5", features = ["util"] }
tower-http = { version = "0.6", features = ["fs", "trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

`tower-http` 启用 `fs`（`ServeDir`/`ServeFile`）和 `trace`（日志）feature。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：最基础的 `ServeDir`——把目录挂到路径

`ServeDir::new("assets")` 是一个 service，把 `assets/` 目录暴露成 HTTP。用 `nest_service` 挂到 `/assets` 路径前缀，访问 `/assets/foo.html` 就会找 `assets/foo.html`。

````rust
use axum::{routing::get, Router};
use std::net::SocketAddr;
use tower_http::{services::ServeDir, trace::TraceLayer};
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

    serve(using_serve_dir(), 3001).await;
}

fn using_serve_dir() -> Router {
    // serve the file in the "assets" directory under `/assets`
    Router::new().nest_service("/assets", ServeDir::new("assets"))
}

async fn serve(app: Router, port: u16) {
    let addr = SocketAddr::from(([127, 0, 0, 1], port));
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app.layer(TraceLayer::new_for_http())).await.unwrap();
}
````

准备测试文件（参考 `examples/static-file-server/assets/` 目录）：

````bash
mkdir -p assets
echo '<h1>Hello</h1>' > assets/index.html
echo 'body { color: red; }' > assets/style.css
````

验证：

````bash
cd examples
cargo run -p example-static-file-server
````

````bash
curl http://127.0.0.1:3001/assets/index.html   # 返回 <h1>Hello</h1>
curl http://127.0.0.1:3001/assets/style.css    # 返回 css
curl -i http://127.0.0.1:3001/assets/nope.html # 404
````

> **新面孔：`tower-http` crate**
>
> `tower` 生态的 HTTP middleware 和 service 库。axum 本身不内置静态文件服务——`tower-http::services::{ServeDir, ServeFile}` 提供这两个 service。还有 `TraceLayer`（日志）、`CompressionLayer`（压缩）、`CorsLayer`（CORS）等，后面章节陆续见到。

> **新面孔：`ServeDir`**
>
> 一个 `tower::Service`：把某个目录暴露成 HTTP。`ServeDir::new("assets")` 服务 `assets/` 目录，访问 `/assets/x.html` 自动找 `assets/x.html` 文件返回。文件不存在返回 404。

> **新面孔：`nest_service` vs `route`**
>
> `Router::route("/path", handler)` 把 handler 挂到精确路径。`nest_service("/prefix", service)` 把一个 service 挂到路径前缀——后续路径任意。`ServeDir` 是 service 不是 handler，必须用 `nest_service`（或 `fallback_service`）挂载。

---

## 第二步：SPA fallback——404 返回 `index.html`

单页应用（SPA）的客户端路由（如 React Router）要求所有未匹配路径都返回 `index.html`，让前端路由接管。这步用 `not_found_service` + `fallback_service` 实现这个模式。

````rust
use axum::http::StatusCode;
use tower_http::{
    services::{ServeDir, ServeFile},
    set_status::SetStatus,
};

fn using_serve_dir_with_assets_fallback() -> Router {
    // 关键：用 404 状态码返回 index.html
    let index_html = SetStatus::new(ServeFile::new("assets/index.html"), StatusCode::NOT_FOUND);
    let serve_dir = ServeDir::new("assets").not_found_service(index_html.clone());

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .nest_service("/assets", serve_dir)
        .fallback_service(index_html)
}
````

> **新面孔：`SetStatus`**
>
> 包装一个 service，强制覆盖响应状态码。这里 `ServeFile::new("assets/index.html")` 默认返回 200，包装成 `SetStatus::new(..., StatusCode::NOT_FOUND)` 后返回 404——表示"文件没找到，但前端可以拿 index.html 接管路由"。
>
> 为什么不直接 200？SEO/客户端逻辑可能依赖状态码。404 + 内容 让客户端能区分"路径不存在但 SPA 可处理" vs "真正命中"。

> **新面孔：`ServeFile`**
>
> 单文件版的 `ServeDir`，只服务一个文件（不是目录）。常配合 `SetStatus` 做自定义 404 页面。

> **新面孔：`not_found_service` + `fallback_service`**
>
> 两个相关但不同的 API：
> - `ServeDir::new(...).not_found_service(svc)`：`ServeDir` 找不到文件时交给 `svc` 处理（**仅影响 `/assets/` 下的查找**）
> - `Router::fallback_service(svc)`：所有路径都没匹配时交给 `svc`（**影响整个 app**）
>
> SPA 模式同时用两个：`/assets/doesnt-exist` 走 `not_found_service`（返回 404 index.html），`/random/path` 走 `fallback_service`（同样返回 404 index.html）。

### 只用 fallback（不 nest）的变体

````rust
fn using_serve_dir_only_from_root_via_fallback() -> Router {
    // 不挂 /assets 前缀，直接从根路径服务
    let serve_dir = ServeDir::new("assets").not_found_service(ServeFile::new("assets/index.html"));

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}
````

这版把 `ServeDir` 作为全局 fallback：访问 `/foo` 走正常 handler，访问 `/index.html` 或 `/style.css` 走 fallback 找文件。

### handler 作为 fallback service

````rust
use axum::handler::HandlerWithoutStateExt;

fn using_serve_dir_with_handler_as_service() -> Router {
    async fn handle_404() -> (StatusCode, &'static str) {
        (StatusCode::NOT_FOUND, "Not found")
    }

    // handler 函数转成 service
    let service = handle_404.into_service();

    let serve_dir = ServeDir::new("assets").not_found_service(service);

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}
````

> **新面孔：`HandlerWithoutStateExt::into_service`**
>
> `not_found_service` 接受的是 `Service`，不是 handler。`handle_404.into_service()` 把 handler 函数转成 service——这样能传给 `ServeDir::not_found_service` 或 `fallback_service`。
>
> `into_service` vs `into_make_service`（第 50 章见过）：`into_service` 转**单个** service 适合做 fallback；`into_make_service` 转**service 工厂**适合 `axum::serve`。

---

## 第三步：进阶变体——多个 ServeDir、handler 内调用、ServeFile

最后几种写法展示了 `ServeDir`/`ServeFile` 的灵活性。

### 两个 `ServeDir` 挂不同路径

````rust
fn two_serve_dirs() -> Router {
    let serve_dir_from_assets = ServeDir::new("assets");
    let serve_dir_from_dist = ServeDir::new("dist");

    Router::new()
        .nest_service("/assets", serve_dir_from_assets)
        .nest_service("/dist", serve_dir_from_dist)
}
````

`/assets/*` 查 `assets/` 目录，`/dist/*` 查 `dist/` 目录。可用于同时托管多套前端构建产物。

### 在 handler 内手动调 `ServeDir`（`oneshot`）

````rust
use axum::extract::Request;
use tower::ServiceExt;

#[allow(clippy::let_and_return)]
fn calling_serve_dir_from_a_handler() -> Router {
    Router::new().nest_service(
        "/foo",
        get(|request: Request| async {
            let service = ServeDir::new("assets");
            // 用 oneshot 手动调一次 service
            let result = service.oneshot(request).await;
            result
        }),
    )
}
````

> **新面孔：`ServiceExt::oneshot`**
>
> `tower::ServiceExt` 提供的便捷方法：**只调用一次** service 并取结果。正常 service 要先 `.ready().await`（等就绪）再 `.call(req)`，`oneshot` 把这两步合起来。
>
> 用途：handler 内动态决定要不要走 `ServeDir`，比如根据用户权限决定要不要返回静态文件。

### `ServeFile` 作为单文件 route

````rust
use tower_http::services::ServeFile;

fn using_serve_file_from_a_route() -> Router {
    Router::new().route_service("/foo", ServeFile::new("assets/index.html"))
}
````

> **新面孔：`route_service`**
>
> `Router::route` 挂 handler，`Router::route_service` 挂 service。`ServeFile` 是 service 不是 handler，所以用 `route_service`。
>
> 用途：固定路径返回固定文件（如 `/favicon.ico` → favicon 文件）。

---

## 完整代码

````rust
use axum::{
    extract::Request, handler::HandlerWithoutStateExt, http::StatusCode, routing::get, Router,
};
use std::net::SocketAddr;
use tower::ServiceExt;
use tower_http::{
    services::{ServeDir, ServeFile},
    set_status::SetStatus,
    trace::TraceLayer,
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

    tokio::join!(
        serve(using_serve_dir(), 3001),
        serve(using_serve_dir_with_assets_fallback(), 3002),
        serve(using_serve_dir_only_from_root_via_fallback(), 3003),
        serve(using_serve_dir_with_handler_as_service(), 3004),
        serve(two_serve_dirs(), 3005),
        serve(calling_serve_dir_from_a_handler(), 3006),
        serve(using_serve_file_from_a_route(), 3307),
    );
}

fn using_serve_dir() -> Router {
    // serve the file in the "assets" directory under `/assets`
    Router::new().nest_service("/assets", ServeDir::new("assets"))
}

fn using_serve_dir_with_assets_fallback() -> Router {
    let index_html = SetStatus::new(ServeFile::new("assets/index.html"), StatusCode::NOT_FOUND);
    let serve_dir = ServeDir::new("assets").not_found_service(index_html.clone());

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .nest_service("/assets", serve_dir)
        .fallback_service(index_html)
}

fn using_serve_dir_only_from_root_via_fallback() -> Router {
    let serve_dir = ServeDir::new("assets").not_found_service(ServeFile::new("assets/index.html"));

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}

fn using_serve_dir_with_handler_as_service() -> Router {
    async fn handle_404() -> (StatusCode, &'static str) {
        (StatusCode::NOT_FOUND, "Not found")
    }

    let service = handle_404.into_service();

    let serve_dir = ServeDir::new("assets").not_found_service(service);

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}

fn two_serve_dirs() -> Router {
    let serve_dir_from_assets = ServeDir::new("assets");
    let serve_dir_from_dist = ServeDir::new("dist");

    Router::new()
        .nest_service("/assets", serve_dir_from_assets)
        .nest_service("/dist", serve_dir_from_dist)
}

#[allow(clippy::let_and_return)]
fn calling_serve_dir_from_a_handler() -> Router {
    Router::new().nest_service(
        "/foo",
        get(|request: Request| async {
            let service = ServeDir::new("assets");
            let result = service.oneshot(request).await;
            result
        }),
    )
}

fn using_serve_file_from_a_route() -> Router {
    Router::new().route_service("/foo", ServeFile::new("assets/index.html"))
}

#[cfg(test)]
mod tests;

async fn serve(app: Router, port: u16) {
    let addr = SocketAddr::from(([127, 0, 0, 1], port));
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app.layer(TraceLayer::new_for_http())).await.unwrap();
}
````

## 运行

````bash
cd examples
cargo run -p example-static-file-server
````

七个端口同时跑（3001-3006 + 3307）。准备 `assets/index.html` 测试：

````bash
curl http://127.0.0.1:3001/assets/index.html            # ServeDir 基础
curl -i http://127.0.0.1:3002/assets/doesnt-exist       # 404 + index.html 内容（SPA 模式）
curl -i http://127.0.0.1:3002/random/path               # 404 + index.html（fallback）
curl http://127.0.0.1:3003/style.css                    # 不带 /assets 前缀直接访问
curl -i http://127.0.0.1:3004/assets/nope               # 404 "Not found"
curl http://127.0.0.1:3307/foo                          # ServeFile 单文件
````

## 解读

### Service vs Handler

这章反复出现 service 和 handler 的转换，理清概念：

| 概念 | 接口 | axum 挂载方式 |
| --- | --- | --- |
| **Handler** | async fn 返回 `IntoResponse` | `.route("/path", get(handler))` |
| **Service** | `tower::Service<Request>` | `.nest_service(...)` / `.route_service(...)` / `.fallback_service(...)` |

`ServeDir`/`ServeFile` 是 service 不是 handler。要把 handler 当 service 用，调 `.into_service()`。

### SPA 路由模式

单页应用（React/Vue）的客户端路由要求：所有未匹配路径都返回 `index.html`，让前端 JS 接管路由。模式：

```text
请求 /foo（不存在）
  → fallback_service 命中
  → 返回 index.html（状态码 404，表示"路径没找到但 SPA 可处理"）
  → 浏览器加载 SPA，前端路由渲染 /foo 对应的组件
```

## 常见问题

**`nest_service` vs `fallback_service`？** `nest_service` 挂到指定前缀，`fallback_service` 兜底所有未匹配路径。SPA 模式常用 `fallback_service(ServeDir)` 让静态文件接管全部未匹配路径。

**为什么要 `SetStatus` 改成 404？** 200 会让浏览器/CDN 误以为路径真的存在，404 表示"路径不存在但给你 SPA 入口让前端处理"。

**怎么防止目录穿越（`/assets/../etc/passwd`）？** `ServeDir` 内置防穿越，自动拒绝 `..` 路径。

## 手写任务

1. 加 `tower-http` 的 `CompressionLayer`，对比压缩前后 response size。
2. 写个自定义 fallback service，返回 JSON 错误而不是 HTML。
3. 改成 `ServeDir::new("assets").append_index_html_on_directories(true)`（默认行为），访问 `/assets/` 自动返回 `index.html`。
4. 在 handler 内 `oneshot` ServeDir 时加权限检查（用户未登录返回 403）。

## 小结

这章用 3 步讲了 axum 静态文件服务：

1. **基础 ServeDir**：`nest_service("/assets", ServeDir::new("assets"))` 把目录挂到路径前缀。
2. **SPA fallback**：`not_found_service` + `fallback_service` 让 404 返回 `index.html`，支持前端路由；`SetStatus` 控制状态码。
3. **进阶变体**：多 `ServeDir`、`oneshot` 在 handler 内调用、`ServeFile` 单文件、`into_service` 把 handler 转 service。

核心概念：**Service vs Handler**——`ServeDir`/`ServeFile` 是 service，挂载用 `nest_service`/`route_service`/`fallback_service`，转换用 `into_service` / `oneshot`。

## 源码对照

- `examples/static-file-server/Cargo.toml`
- `examples/static-file-server/src/main.rs`
- `examples/static-file-server/tests/tests.rs`
