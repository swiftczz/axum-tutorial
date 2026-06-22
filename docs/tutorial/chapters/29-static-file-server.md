# 29. static-file-server

对应示例：`examples/static-file-server`

本章目标：用 `tower_http::services::ServeDir` 和 `ServeFile` 提供静态文件服务，理解目录挂载、fallback、SPA 回退和单文件路由。

前两章讲的是服务端渲染 HTML。  
这一章讲另一类常见需求：后端直接托管静态文件。

## 这个小项目在做什么

这个 example 一次启动多个服务，每个端口演示一种静态文件写法：

```text
3001 -> /assets 下托管 assets 目录
3002 -> /assets 托管 assets，并带 SPA fallback
3003 -> 从根路径 fallback 到 assets 目录
3004 -> 静态目录找不到文件时，用 handler 返回 404
3005 -> 同时挂载两个静态目录
3006 -> 在 handler 里手动调用 ServeDir
3307 -> 单独路由返回一个文件
```

示例静态文件：

```text
assets/index.html
assets/script.js
```

请求主线之一是：

```text
客户端 GET /assets/index.html
-> Router 匹配 /assets
-> ServeDir 从 assets 目录找 index.html
-> 返回文件内容
```

## 先理解静态文件服务是什么

静态文件就是不需要后端动态生成的文件：

```text
HTML
CSS
JavaScript
图片
字体
下载文件
```

后端返回 JSON 时，响应内容通常来自数据库或业务代码。  
静态文件服务返回的内容直接来自磁盘文件。

例如：

```text
GET /assets/script.js
-> 读取 assets/script.js
-> 返回 console.log("Hello, World!");
```

## 先理解 Service（本章核心概念）

这一章会反复出现 `ServeDir`、`ServeFile`、`nest_service`、`fallback_service`、`route_service`、`oneshot` 这些词。它们背后是同一个抽象：**Service**。

如果你还没读过**附录 A0《Tower 基础：Service 与 Layer》**，强烈建议先花 10 分钟读一遍。这里只说本章最关键的三点：

1. **Service 是"接收请求、返回响应"的对象**。你的 handler 本质上也是 service，只是多了 extractor 能力。
2. **`ServeDir` 和 `ServeFile` 天生就是 service，不是 handler**。它们内部要处理路径匹配、读文件、返回响应，不依赖 extractor，所以直接实现成 `Service`。
3. **挂载 service 和挂 handler 用不同的 API**：

| API | 接收什么 | 本章例子 |
| --- | --- | --- |
| `.route("/foo", get(handler))` | handler（有 extractor） | 业务接口 |
| `.route_service("/foo", service)` | service（无 extractor） | `ServeFile::new("...")` |
| `.nest_service("/assets", service)` | service（无 extractor），匹配前缀下所有路径 | `ServeDir::new("assets")` |

为什么 `ServeDir` 不能塞进 `get(handler)`？因为 handler 需要满足 `Handler` trait，而 `ServeDir` 实现的是 `Service` trait，两者是不同的抽象。

`nest_service` 和 `route_service` 的区别在前缀：

- `route_service("/foo", svc)`：只匹配精确的 `/foo`。
- `nest_service("/assets", svc)`：匹配 `/assets` 下所有路径，剩余部分（如 `/assets/script.js` 的 `/script.js`）传给 svc。

记住这三点，本章的代码就不会是黑盒了。需要更深入理解 Service / Layer 时，回 A0 附录查。

## ServeDir 和 ServeFile 的区别

这一章用到两个核心 service（注意：是 service，不是 handler）：

```text
ServeDir
ServeFile
```

区别很简单：

```text
ServeDir  -> 托管一个目录（按请求路径在目录里找文件）
ServeFile -> 托管一个文件（不管请求细节，固定返回这个文件）
```

它们都实现了 `Service<Request>`，所以用 `nest_service` / `route_service` 挂载，不能用 `get(handler)`。

例如：

````rust
ServeDir::new("assets")
````

表示从 `assets` 目录里按请求路径找文件。

````rust
ServeFile::new("assets/index.html")
````

表示不管路由怎么匹配，最终服务的是这个具体文件。

## 文件和依赖

这个 example 有五个主要文件：

1. `examples/static-file-server/Cargo.toml`：声明 Axum、tower、tower-http fs/set-status/trace。
2. `examples/static-file-server/src/main.rs`：实现多种静态文件服务写法。
3. `examples/static-file-server/src/tests.rs`：测试 SPA fallback 的状态码和内容。
4. `examples/static-file-server/assets/index.html`：示例 HTML 文件。
5. `examples/static-file-server/assets/script.js`：示例 JS 文件。

关键依赖：

- `axum`：提供 Router、route、nest_service、fallback_service。
- `tower`：提供 `ServiceExt::oneshot`。
- `tower-http`：提供 `ServeDir`、`ServeFile`、`SetStatus`、`TraceLayer`。
- `tokio`：同时运行多个端口上的服务。
- `tracing` / `tracing-subscriber`：输出请求日志。

`tower-http` 需要这些 feature：

````toml
tower-http = { version = "0.6.1", features = ["fs", "set-status", "trace"] }
````

## 第一步：同时启动多个示例服务

源码：

````rust
tokio::join!(
    serve(using_serve_dir(), 3001),
    serve(using_serve_dir_with_assets_fallback(), 3002),
    serve(using_serve_dir_only_from_root_via_fallback(), 3003),
    serve(using_serve_dir_with_handler_as_service(), 3004),
    serve(two_serve_dirs(), 3005),
    serve(calling_serve_dir_from_a_handler(), 3006),
    serve(using_serve_file_from_a_route(), 3307),
);
````

每个 `using_...` 函数都会返回一个 Router。  
`serve(router, port)` 把它启动到不同端口。

这样你可以一次运行程序，然后分别访问不同端口对比行为。

## 第二步：最基础的 ServeDir

源码：

````rust
fn using_serve_dir() -> Router {
    Router::new().nest_service("/assets", ServeDir::new("assets"))
}
````

这表示：

```text
URL 前缀 /assets
对应磁盘目录 assets
```

访问：

```text
http://127.0.0.1:3001/assets/index.html
```

会读取：

```text
assets/index.html
```

访问：

```text
http://127.0.0.1:3001/assets/script.js
```

会读取：

```text
assets/script.js
```

`nest_service` 的意思是把一个 service 挂到某个路径前缀下。  
这里挂的 service 就是 `ServeDir`。

## 第三步：给 assets 目录加 not_found fallback

源码：

````rust
let index_html = SetStatus::new(ServeFile::new("assets/index.html"), StatusCode::NOT_FOUND);
let serve_dir = ServeDir::new("assets").not_found_service(index_html.clone());

Router::new()
    .route("/foo", get(|| async { "Hi from /foo" }))
    .nest_service("/assets", serve_dir)
    .fallback_service(index_html)
````

这段是最接近前端 SPA 部署的写法。

它做了两层 fallback：

1. `/assets/...` 下找不到文件时，返回 `index.html`。
2. 其他未知路径也返回 `index.html`。

但这里用 `SetStatus` 把状态码设置成 404：

````rust
SetStatus::new(ServeFile::new("assets/index.html"), StatusCode::NOT_FOUND)
````

意思是：

```text
内容返回 index.html
状态码仍然告诉客户端路径没找到
```

为什么要这样？

对于 SPA，前端路由可能需要接管未知路径。  
但是从后端角度看，这个具体文件路径确实不存在，所以用 404 更准确。

## 第四步：从根路径 fallback 到 ServeDir

源码：

````rust
fn using_serve_dir_only_from_root_via_fallback() -> Router {
    let serve_dir = ServeDir::new("assets").not_found_service(ServeFile::new("assets/index.html"));

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}
````

这一种没有把静态文件挂到 `/assets` 前缀。  
而是把 `ServeDir` 放到整个 Router 的 fallback 上。

效果是：

```text
GET /script.js -> 尝试读取 assets/script.js
GET /xxx       -> 找不到时返回 assets/index.html
```

这种更像“整个站点都由静态目录兜底”。

常见于：

```text
后端少量 API 路由
其余路径交给前端构建产物
```

## 第五步：用 handler 作为 404 service

源码：

````rust
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
````

`not_found_service` 需要的是 service。  
普通 Axum handler 不是 service，但可以通过：

````rust
handle_404.into_service()
````

转换成 service。

这样静态文件找不到时，就会调用这个 handler，返回：

```text
404 Not found
```

这展示了 Axum handler 和 Tower service 之间可以互相配合。

## 第六步：同时挂载两个静态目录

源码：

````rust
fn two_serve_dirs() -> Router {
    let serve_dir_from_assets = ServeDir::new("assets");
    let serve_dir_from_dist = ServeDir::new("dist");

    Router::new()
        .nest_service("/assets", serve_dir_from_assets)
        .nest_service("/dist", serve_dir_from_dist)
}
````

这表示：

```text
/assets/... -> assets 目录
/dist/...   -> dist 目录
```

真实项目里可能会这样拆：

```text
/assets -> 公共静态资源
/dist   -> 前端构建产物
```

如果对应目录不存在，访问时会返回找不到文件。  
这不是 Router 错，而是磁盘目录本身没有对应文件。

## 第七步：在 handler 里手动调用 ServeDir

源码：

````rust
Router::new().nest_service(
    "/foo",
    get(|request: Request| async {
        let service = ServeDir::new("assets");
        let result = service.oneshot(request).await;
        result
    }),
)
````

这是一种更底层的写法。  
handler 收到 `Request` 后，手动创建 `ServeDir`，然后调用：

````rust
service.oneshot(request).await
````

`oneshot` 来自 `tower::ServiceExt`，表示用这个 service 处理一次请求。

大多数项目不需要这样写。  
但它能帮助你理解：

```text
ServeDir 本质上也是一个能接收 Request 并返回 Response 的 service
```

## 第八步：单路由返回单文件

源码：

````rust
fn using_serve_file_from_a_route() -> Router {
    Router::new().route_service("/foo", ServeFile::new("assets/index.html"))
}
````

这表示：

```text
GET /foo -> 返回 assets/index.html
```

如果你只想让某个 URL 返回固定文件，用 `ServeFile` 比 `ServeDir` 更直接。

典型场景：

```text
/favicon.ico
/robots.txt
/download/manual.pdf
```

## 第九步：统一启动服务并加 TraceLayer

源码：

````rust
async fn serve(app: Router, port: u16) {
    let addr = SocketAddr::from(([127, 0, 0, 1], port));
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app.layer(TraceLayer::new_for_http())).await;
}
````

这个函数做了两件事：

1. 把 Router 启动到指定端口。
2. 给 Router 加 `TraceLayer`，方便观察静态文件请求日志。

虽然静态文件不是业务 handler，但它仍然是 HTTP 请求。  
加 trace 日志后，你能看到哪些文件被访问、哪些请求返回 404。

## 第十步：测试验证 SPA fallback

测试代码重点：

````rust
let app = using_serve_dir_with_assets_fallback();
let response = app
    .oneshot(
        Request::builder()
            .uri("/script.js")
            .body(Body::empty())
            .unwrap(),
    )
    .await
    .unwrap();

assert_eq!(response.status(), StatusCode::NOT_FOUND);
...
assert_eq!(body_str, INDEX_HTML_CONTENT);
````

这个测试请求的是：

```text
/script.js
```

注意它不是：

```text
/assets/script.js
```

所以它不会命中 `/assets` 下的静态文件。  
请求会落到 SPA fallback，返回 `index.html`，但状态码是 404。

测试验证了两点：

```text
状态码是 404
响应内容是 index.html
```

## 函数职责速查

- `main`：初始化日志，同时启动多个静态文件服务示例。
- `using_serve_dir`：演示最基础的 `/assets` 目录挂载。
- `using_serve_dir_with_assets_fallback`：演示 assets fallback 和 SPA fallback。
- `using_serve_dir_only_from_root_via_fallback`：演示从根路径 fallback 到静态目录。
- `using_serve_dir_with_handler_as_service`：演示 handler 转 service 作为 404。
- `two_serve_dirs`：演示同时挂载两个目录。
- `calling_serve_dir_from_a_handler`：演示在 handler 中手动调用 `ServeDir`。
- `using_serve_file_from_a_route`：演示单路由返回单文件。
- `serve`：绑定端口、加 TraceLayer、启动服务。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//!
//! ```not_rust
//! cargo run -p example-static-file-server
//! ```

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
    // 初始化日志。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 同时启动多个服务，每个端口演示一种静态文件写法。
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

// 最基础写法：把 assets 目录挂到 /assets URL 前缀下。
fn using_serve_dir() -> Router {
    Router::new().nest_service("/assets", ServeDir::new("assets"))
}

// 给 /assets 静态目录加 fallback，同时给整个 Router 加 SPA fallback。
fn using_serve_dir_with_assets_fallback() -> Router {
    // 返回 index.html，但状态码设置为 404。
    let index_html = SetStatus::new(ServeFile::new("assets/index.html"), StatusCode::NOT_FOUND);

    // /assets 下文件找不到时，也返回 index.html。
    let serve_dir = ServeDir::new("assets").not_found_service(index_html.clone());

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .nest_service("/assets", serve_dir)
        .fallback_service(index_html)
}

// 不挂 /assets 前缀，而是让整个 Router 的 fallback 都从 assets 目录找文件。
fn using_serve_dir_only_from_root_via_fallback() -> Router {
    let serve_dir = ServeDir::new("assets").not_found_service(ServeFile::new("assets/index.html"));

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}

// 静态目录找不到文件时，调用一个普通 handler 返回 404。
fn using_serve_dir_with_handler_as_service() -> Router {
    async fn handle_404() -> (StatusCode, &'static str) {
        (StatusCode::NOT_FOUND, "Not found")
    }

    // 把 Axum handler 转成 Tower service。
    let service = handle_404.into_service();

    let serve_dir = ServeDir::new("assets").not_found_service(service);

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}

// 同时挂载两个静态目录。
fn two_serve_dirs() -> Router {
    let serve_dir_from_assets = ServeDir::new("assets");
    let serve_dir_from_dist = ServeDir::new("dist");

    Router::new()
        .nest_service("/assets", serve_dir_from_assets)
        .nest_service("/dist", serve_dir_from_dist)
}

#[allow(clippy::let_and_return)]
fn calling_serve_dir_from_a_handler() -> Router {
    // 在 handler 里手动调用 ServeDir。
    Router::new().nest_service(
        "/foo",
        get(|request: Request| async {
            let service = ServeDir::new("assets");
            let result = service.oneshot(request).await;
            result
        }),
    )
}

// 单个路由直接返回单个文件。
fn using_serve_file_from_a_route() -> Router {
    Router::new().route_service("/foo", ServeFile::new("assets/index.html"))
}

#[cfg(test)]
mod tests;

// 启动一个 Router 到指定端口，并加上 HTTP trace 日志。
async fn serve(app: Router, port: u16) {
    let addr = SocketAddr::from(([127, 0, 0, 1], port));
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app.layer(TraceLayer::new_for_http())).await;
}
````

示例静态文件：

````html
Hi from index.html
````

````javascript
console.log("Hello, World!");
````

## 运行和验证

运行：

````bash
cargo run -p example-static-file-server
````

访问最基础的 `/assets` 挂载：

````bash
curl http://127.0.0.1:3001/assets/index.html
curl http://127.0.0.1:3001/assets/script.js
````

验证 SPA fallback：

````bash
curl -i http://127.0.0.1:3002/script.js
````

预期行为：

```text
状态码：404
body：assets/index.html 的内容
```

验证根路径 fallback：

````bash
curl http://127.0.0.1:3003/script.js
````

验证单文件路由：

````bash
curl http://127.0.0.1:3307/foo
````

注意：`ServeDir::new("assets")` 使用的是进程当前工作目录下的 `assets`。  
如果运行时一直 404，先确认程序的当前工作目录里是否能找到 `assets/index.html`。

## 常见卡点

### 1. nest_service 和 fallback_service 有什么区别？

`nest_service("/assets", ...)` 只处理 `/assets/...` 前缀的请求。  
`fallback_service(...)` 处理没有被其他路由匹配到的请求。

### 2. ServeDir 找不到文件时返回什么？

默认是找不到文件。  
你可以用：

````rust
.not_found_service(...)
````

指定找不到文件时交给另一个 service 处理。

### 3. SPA fallback 为什么返回 index.html 但状态码是 404？

因为请求的路径确实不是一个存在的静态文件。  
返回 `index.html` 是为了让前端路由接管页面，状态码 404 则保留后端语义。

### 4. 为什么静态文件路径和 URL 路径不一样？

因为 URL 前缀是 Router 定义的，磁盘目录是 `ServeDir::new(...)` 定义的。

例如：

````rust
nest_service("/assets", ServeDir::new("assets"))
````

URL `/assets/script.js` 对应磁盘文件 `assets/script.js`。

### 5. 生产环境一定要 Axum 托管静态文件吗？

不一定。  
很多生产环境会用 Nginx、CDN、对象存储托管静态资源。

Axum 托管静态文件适合：

- 小型服务。
- 管理后台。
- 内网工具。
- 需要和 Rust 服务打包在一起的简单页面。

## 手写任务

建议按下面顺序自己敲一遍：

1. 准备 `assets/index.html` 和 `assets/script.js`。
2. 写 `using_serve_dir()`，把 `assets` 挂到 `/assets`。
3. 用 curl 访问 `/assets/index.html`。
4. 写一个 `ServeFile::new("assets/index.html")`。
5. 用 `fallback_service` 做 SPA fallback。
6. 用 `SetStatus` 把 fallback 响应状态码改成 404。
7. 尝试用 `route_service("/foo", ServeFile::new(...))` 返回单文件。

加深练习：

1. 新增 `/favicon.ico`，用 `ServeFile` 返回固定文件。
2. 新增 `/public`，挂载另一个静态目录。
3. 给静态文件服务加上 `TraceLayer`，观察请求日志。

## 本章真正要记住什么

静态文件服务的核心不是 handler，而是 Tower service：

```text
ServeDir  -> 托管目录
ServeFile -> 托管单个文件
```

为什么是 service 不是 handler？因为 `ServeDir`/`ServeFile` 自己实现了"接收请求、返回响应"的能力（`Service` trait），不需要 extractor 从请求里提参数。挂载它们要用 `nest_service` / `route_service`，不能用 `get(handler)`。不理解 Service 抽象的话，回**附录 A0** 查。

在 Axum 里常见组合是：

````rust
Router::new()
    .nest_service("/assets", ServeDir::new("assets"))
    .fallback_service(ServeFile::new("assets/index.html"))
````

如果你在做前后端分离或 SPA 部署，重点理解：

```text
静态目录挂载
fallback
not_found_service
状态码是否应该是 200 或 404
```

## 源码对照

本章手写版对应源码：

- `examples/static-file-server/src/main.rs`
- `examples/static-file-server/src/tests.rs`
- `examples/static-file-server/assets/index.html`
- `examples/static-file-server/assets/script.js`
- `examples/static-file-server/Cargo.toml`
