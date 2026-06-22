# 30. static-file-server

对应示例：`examples/static-file-server`

前两章讲服务端渲染 HTML,这章讲后端直接托管静态文件。用 `tower_http::services::ServeDir` 和 `ServeFile` 提供静态文件服务,理解目录挂载、fallback、SPA 回退和单文件路由。



相比前面章节新引入：**`ServeDir`/`ServeFile` 是 Tower Service（不是 Handler）、`SetStatus`、SPA fallback**。

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
tower = { version = "0.5.2", features = ["util"] }
tower-http = { version = "0.6.1", features = ["fs", "set-status", "trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

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


async fn serve(app: Router, port: u16) {
    let addr = SocketAddr::from(([127, 0, 0, 1], port));
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app.layer(TraceLayer::new_for_http())).await;
}
````

## 运行

````bash
cd examples
cargo run -p example-static-file-server
````

不同端口演示不同写法:

````bash
# 3001 最基础的 /assets 挂载
curl http://127.0.0.1:3001/assets/index.html

# 3002 SPA fallback:找不到文件返回 index.html 但状态码 404
curl -i http://127.0.0.1:3002/script.js

# 3003 从根路径 fallback 到静态目录
curl http://127.0.0.1:3003/script.js

# 3307 单路由返回单文件
curl http://127.0.0.1:3307/foo
````

注意:`ServeDir::new("assets")` 用进程当前工作目录下的 `assets`。运行时一直 404,先确认程序工作目录里有 `assets/index.html`。

## 解读

### Service 是本章核心概念(先读 A0 附录)

本章反复出现 `ServeDir`、`ServeFile`、`nest_service`、`fallback_service`、`route_service`、`oneshot`——背后都是同一个抽象:**Service**。如果你还没读**附录 A0《Tower 基础:Service 与 Layer》**,强烈建议先花 10 分钟读。这里说最关键三点:

1. **Service 是"接收请求、返回响应"的对象**。你的 handler 本质上也是 service,只是多了 extractor 能力。
2. **`ServeDir`/`ServeFile` 天生就是 service,不是 handler**——它们内部处理路径匹配/读文件/返回响应,不依赖 extractor,所以直接实现成 `Service`。
3. **挂 service 和挂 handler 用不同 API**:

| API | 接收什么 | 例子 |
| --- | --- | --- |
| `.route("/foo", get(handler))` | handler(有 extractor) | 业务接口 |
| `.route_service("/foo", service)` | service,匹配精确路径 | `ServeFile::new("...")` |
| `.nest_service("/assets", service)` | service,匹配前缀下所有路径 | `ServeDir::new("assets")` |

为什么 `ServeDir` 不能塞进 `get(handler)`?handler 要满足 `Handler` trait,`ServeDir` 实现的是 `Service` trait,两者是不同抽象。

`route_service` vs `nest_service` 的区别在前缀:`route_service("/foo", svc)` 只匹配精确 `/foo`;`nest_service("/assets", svc)` 匹配 `/assets` 下所有路径,剩余部分(如 `/assets/script.js` 的 `/script.js`)传给 svc。

### ServeDir vs ServeFile

```text
ServeDir  -> 托管一个目录(按请求路径在目录里找文件)
ServeFile -> 托管一个文件(不管请求细节,固定返回这个文件)
```

都用 `nest_service` / `route_service` 挂载,不能用 `get(handler)`。

### 8 种写法对照

**1. 最基础 ServeDir**(3001):

````rust
fn using_serve_dir() -> Router {
    Router::new().nest_service("/assets", ServeDir::new("assets"))
}
````

URL 前缀 `/assets` 对应磁盘目录 `assets`。`/assets/index.html` 读 `assets/index.html`。

**2. SPA fallback**(3002):两层 fallback + 状态码 404:

````rust
let index_html = SetStatus::new(ServeFile::new("assets/index.html"), StatusCode::NOT_FOUND);
let serve_dir = ServeDir::new("assets").not_found_service(index_html.clone());

Router::new()
    .route("/foo", get(|| async { "Hi from /foo" }))
    .nest_service("/assets", serve_dir)
    .fallback_service(index_html)
````

- `/assets/...` 找不到文件 → 返回 `index.html`。
- 其他未知路径 → 也返回 `index.html`。
- `SetStatus` 把状态码设 404:内容返回 index.html(让前端路由接管),状态码告诉客户端路径确实不存在。

**3. 从根路径 fallback**(3003):不挂 `/assets` 前缀,把 `ServeDir` 放整个 Router fallback:

````rust
Router::new()
    .route("/foo", get(|| async { "Hi from /foo" }))
    .fallback_service(serve_dir)
````

`GET /script.js` 尝试读 `assets/script.js`,`GET /xxx` 找不到时返回 `index.html`。适合"少量 API 路由 + 其余路径交给前端构建产物"。

**4. handler 转 service 做 404**(3004):

````rust
let service = handle_404.into_service();  // handler 转 service
let serve_dir = ServeDir::new("assets").not_found_service(service);
````

`not_found_service` 需要 service,普通 handler 用 `.into_service()` 转换。展示了 handler 和 Tower service 可互相配合。

**5. 两个静态目录**(3005):`/assets` 和 `/dist` 分别挂不同目录。

**6. handler 里手动调 ServeDir**(3006):

````rust
.get(|request: Request| async {
    let service = ServeDir::new("assets");
    let result = service.oneshot(request).await;
    result
})
````

`oneshot` 来自 `tower::ServiceExt`,用 service 处理一次请求。大多数项目不需要,但帮你理解 `ServeDir` 本质也是接收 Request 返回 Response 的 service。

**7. 单路由返回单文件**(3307):

````rust
Router::new().route_service("/foo", ServeFile::new("assets/index.html"))
````

适合 `/favicon.ico`、`/robots.txt`、`/download/manual.pdf` 这种固定文件。

### 静态文件路径 vs URL 路径

URL 前缀是 Router 定义的,磁盘目录是 `ServeDir::new(...)` 定义的。`nest_service("/assets", ServeDir::new("assets"))` 让 URL `/assets/script.js` 对应磁盘 `assets/script.js`——两者不必相同。

## 常见问题

**nest_service vs fallback_service?** `nest_service("/assets", ...)` 只处理 `/assets/...` 前缀;`fallback_service(...)` 处理没被其他路由匹配到的请求。

**ServeDir 找不到文件返回什么?** 默认找不到,可用 `.not_found_service(...)` 指定交给另一个 service。

**SPA fallback 为什么返回 index.html 但状态码 404?** 请求路径确实不是存在的静态文件,返回 index.html 让前端路由接管页面,状态码 404 保留后端语义。

**生产环境一定用 axum 托管静态文件吗?** 不一定,很多用 Nginx/CDN/对象存储。axum 适合小型服务、管理后台、内网工具、和 Rust 服务打包的简单页面。

## 手写任务

按下面顺序敲:

1. 准备 `assets/index.html` 和 `assets/script.js`。
2. 写 `using_serve_dir()`,把 `assets` 挂到 `/assets`。
3. curl 访问 `/assets/index.html`。
4. 写 `ServeFile::new("assets/index.html")`。
5. 用 `fallback_service` 做 SPA fallback。
6. `route_service("/foo", ServeFile::new(...))` 返回单文件。
7. `route_service("/foo", ServeFile::new(...))` 返回单文件。

加深练习:

1. 新增 `/favicon.ico` 用 `ServeFile` 返回固定文件。
2. 新增 `/public` 挂载另一个静态目录。
3. 给静态文件服务加 `TraceLayer` 观察请求日志。

## 小结

- 静态文件服务的核心不是 handler 而是 Tower service:`ServeDir` 托管目录,`ServeFile` 托管单文件。
- 是 service 不是 handler 的原因:它们自己实现"接收请求返回响应"能力,不需 extractor 从请求里提参数,用 `nest_service`/`route_service` 挂载,不能用 `get(handler)`。
- `route_service` 匹配精确路径,`nest_service` 匹配前缀下所有路径,`fallback_service` 处理未匹配请求。
- SPA fallback 用 `SetStatus::new(ServeFile, StatusCode::NOT_FOUND)`:返回 index.html 让前端路由接管,状态码保留 404 语义。
- handler 和 service 可互转(`handle.into_service()`),axum 和 Tower 能互相配合。不理解 Service 抽象回附录 A0 查。

## 源码对照

- `examples/static-file-server/Cargo.toml`
- `examples/static-file-server/src/main.rs`
- `examples/static-file-server/assets/index.html`
- `examples/static-file-server/assets/script.js`
