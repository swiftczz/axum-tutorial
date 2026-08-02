# 第 38 章 static-file-server:静态文件服务

## 本章你将做出什么

**七个服务同时跑在七个端口上**,每个演示 `ServeDir` 的一种用法:

| 端口 | 做法 |
|---|---|
| 3001 | 挂在 `/assets` 下 |
| 3002 | 加 SPA 兜底(找不到就返回 `index.html`) |
| 3003 | 直接从根路径提供 |
| 3004 | 用自定义 handler 处理 404 |
| 3005 | 两个目录挂在不同路径 |
| 3006 | 在 handler 内部调用 `ServeDir` |
| 3307 | 单个文件挂在一条路由上 |

从[第 33 章](33-sse.md#步骤-3配上前端页面)起 `ServeDir` 就一直挂着「本章不深入」,这一章还这笔债。

学完本章你将掌握:

- `ServeDir` / `ServeFile`:静态文件服务的两个基本构件
- **`nest_service`、`fallback_service`、`route_service` 三者的区别**
- SPA 前端的兜底路由该怎么配
- `not_found_service` 与 `SetStatus`
- **为什么这一章被排到第 38 位**(而不是紧跟模板那两章)

## 前置知识

### 为什么这一章排得这么靠后

[第 04](04-templates.md#项目搭建)、[05 章](05-templates-minijinja.md#项目搭建)教完渲染 HTML 之后,读者最自然的下一个需求就是"给页面配 CSS 和 JS"。但那两章都只留了一句"见第 38 章"。

原因是 `ServeDir` **依赖两个后面才讲的东西**:

- **它是一个 `Service` 而不是 handler** —— 需要[第 17 章](17-tracing-aka-logging.md#towerservice-与-layer) tower 的 `Service`/`Layer` 概念,以及 `nest_service`/`fallback_service` 这类"接受 Service 而非 handler"的方法
- **官方示例带一个测试模块** —— 需要[第 21 章](21-testing.md)的 `oneshot`

**这是本教程排序原则的一个直接体现**:按知识依赖排,不按"读者什么时候想要"排。代价是[第 04](04-templates.md)、[05 章](05-templates-minijinja.md)的页面一直是裸 HTML。

### 静态文件服务在做什么

**把磁盘上的目录映射成 HTTP 路由。** 请求 `/assets/logo.png` 就读 `assets/logo.png` 这个文件返回。

听起来简单,但一个能用的实现要处理:

| 问题 | `ServeDir` 的处理 |
|---|---|
| **路径穿越**(`../../etc/passwd`) | 规范化路径并拒绝逃出根目录的请求 |
| `Content-Type` | 按扩展名推断 MIME 类型 |
| 大文件 | 流式读取,不全部载入内存 |
| 条件请求 | 处理 `If-Modified-Since`、`If-None-Match`,返回 `304` |
| 范围请求 | 支持 `Range` 头部(断点续传、视频拖动) |
| 压缩 | 可选地优先返回 `.gz`/`.br` 预压缩文件 |
| 目录 | 可配置返回 `index.html` 或 `404` |

**路径穿越是这里面最重要的一条。** [第 13 章](13-multipart-form.md#代价与边界)讲过文件名不可信,静态文件服务面临同样的问题:客户端能控制路径。自己实现很容易漏掉边界情况(URL 编码的 `%2e%2e`、符号链接、Windows 的 `..\`)。

**所以别自己写。** `tower-http` 的 `ServeDir` 已经处理好了这些。

### ServeDir 与 ServeFile

两个类型,一个映射目录、一个映射单文件:

```rust
ServeDir::new("assets")              // assets/ 目录
ServeFile::new("assets/index.html")  // 单个文件
```

**它们都是 `Service`,不是 handler。** 这个区别决定了它们怎么挂到 `Router` 上——下一节详说。

`ServeDir` 的常用配置:

```rust
ServeDir::new("assets")
    .append_index_html_on_directories(true)   // 访问 /dir/ 时返回 /dir/index.html
    .not_found_service(some_service)          // 找不到文件时交给谁
    .precompressed_gzip()                     // 优先返回 .gz 文件
```

`append_index_html_on_directories` 在[第 33](33-sse.md#步骤-3配上前端页面)、[34](34-websockets.md#步骤-3前端页面)、[36 章](36-websockets-http2.md#步骤-1https--广播通道骨架)都用过——它让 `/` 能返回 `index.html`。

### 三种挂载方式:nest_service / fallback_service / route_service

**这是本章最需要理清的一组概念。**

前面章节用的 `route`、`nest`、`fallback` 都接受 **handler 或 Router**。它们各有一个 `_service` 后缀的版本,接受 **`Service`**:

| 方法 | 接受 | 匹配范围 |
|---|---|---|
| `route(path, handler)` | handler | 精确路径 |
| `route_service(path, svc)` | **Service** | 精确路径 |
| `nest(prefix, router)` | Router | 前缀下的所有路径 |
| `nest_service(prefix, svc)` | **Service** | 前缀下的所有路径 |
| `fallback(handler)` | handler | 所有未匹配的路径 |
| `fallback_service(svc)` | **Service** | 所有未匹配的路径 |

**为什么需要 `_service` 版本?** 因为 handler 是"函数",而 `ServeDir` 是"实现了 `Service` trait 的类型"。[第 21 章](21-testing.md#router-是一个-service)讲过 `Router` 自己也是 `Service`,所以 `nest` 内部其实也是 service 的组合——只是 API 上区分开了。

**三者的选择:**

```rust
// 1. nest_service:文件在 /assets 前缀下
Router::new().nest_service("/assets", ServeDir::new("assets"))
// GET /assets/script.js → assets/script.js

// 2. fallback_service:文件直接在根路径,但路由优先
Router::new()
    .route("/api/foo", get(handler))
    .fallback_service(ServeDir::new("assets"))
// GET /api/foo    → handler
// GET /script.js  → assets/script.js

// 3. route_service:单个精确路径
Router::new().route_service("/foo", ServeFile::new("assets/index.html"))
// GET /foo → assets/index.html
```

**`nest_service` 会剥掉前缀。** 请求 `/assets/script.js` 传给 `ServeDir` 时,路径变成了 `/script.js`,所以它去找 `assets/script.js`。这和[第 16 章](16-dependency-injection.md#routernest-与-merge-的区别)讲的 `nest` 行为一致。

**`fallback_service` 不剥前缀**,因为它没有前缀。

### SPA 的兜底问题

**单页应用**(SPA,如 React/Vue)有个特殊需求。它的路由在**客户端**:用户访问 `/users/42`,服务端并没有这个文件,但 SPA 的 JS 会解析这个路径并渲染对应页面。

所以服务端的策略应该是:

- **找得到文件** → 返回文件(`/script.js`、`/logo.png`)
- **找不到文件** → 返回 `index.html`,让 SPA 自己处理路由

`ServeDir::not_found_service` 就是配这个的:

```rust
let index_html = ServeFile::new("assets/index.html");
let serve_dir = ServeDir::new("assets").not_found_service(index_html);
```

**但直接这么写有个问题:状态码是 `200`。** 对真正不存在的路径也返回 `200`,会让爬虫和监控误判。

本章示例用 `SetStatus` 解决:

```rust
let index_html = SetStatus::new(ServeFile::new("assets/index.html"), StatusCode::NOT_FOUND);
```

**`SetStatus` 是个 `Service` 包装器**,把内层服务的响应状态码改成指定值。于是返回的是 `404` + `index.html` 内容。

官方注释解释了这个选择:

> We wrap it in `SetStatus` so the response uses 404 instead of 200 -- the path wasn't found, the SPA just handles routing client-side.

**这个做法有争议。** `404` 对爬虫友好(明确说"这个 URL 不存在"),但**对 SPA 不友好**——某些前端框架或浏览器扩展看到 `404` 会认为加载失败。实际项目里两种做法都常见,「代价与边界」会展开。

### HandlerWithoutStateExt::into_service

`not_found_service` 要一个 `Service`,但你可能想用一个普通的 handler:

```rust
async fn handle_404() -> (StatusCode, &'static str) {
    (StatusCode::NOT_FOUND, "Not found")
}

let service = handle_404.into_service();
```

**`into_service()` 把 handler 变成 `Service`**,来自 `HandlerWithoutStateExt` trait([第 32 章](32-tls-rustls.md#into_make_servicerouter-与-makeservice)用过它的姊妹方法 `into_make_service`)。

"WithoutState" 指的是**这个 handler 不需要应用状态**。需要状态的话用 `with_state(...)`([第 22 章](22-key-value-store.md#步骤-3列出所有-key给单条路由挂中间件)见过)。

## 项目搭建

```bash
cargo new static-file-server
cd static-file-server
```

```bash
cargo add axum
cargo add tokio --features full
cargo add tower --features util
cargo add tower-http --features fs,set-status,trace
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`tower-http` 的三个 feature:`fs` 提供 `ServeDir`/`ServeFile`,`set-status` 提供 `SetStatus`,`trace` 提供 `TraceLayer`。

`tower` 的 `util` feature 提供 `ServiceExt::oneshot`(端口 3006 那个例子要用)。

### 静态资源

```bash
mkdir assets
echo 'Hi from index.html' > assets/index.html
echo 'console.log("Hello, World!");' > assets/script.js
```

**注意 `index.html` 不是合法的 HTML**,就是一行纯文本。这让验证时更容易看清返回的是什么。

## 分步构建

### 步骤 1:最基本的用法

**要做什么**:把 `assets` 目录挂在 `/assets` 前缀下,顺便搭出"多端口并行"的骨架。

```rust
// src/main.rs
use axum::Router;
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

    tokio::join!(serve(using_serve_dir(), 3001));
}

fn using_serve_dir() -> Router {
    // serve the file in the "assets" directory under `/assets`
    Router::new().nest_service("/assets", ServeDir::new("assets"))
}

async fn serve(app: Router, port: u16) {
    let addr = SocketAddr::from(([127, 0, 0, 1], port));
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app.layer(TraceLayer::new_for_http()))
        .await
        .unwrap();
}
```

**`tokio::join!` 并行跑多个服务**,[第 20 章](20-cors.md#步骤-1在一个进程里跑两个服务)用过同样的手法。后面会往里加更多。

**`serve` 辅助函数**统一了绑定和挂 `TraceLayer` 的逻辑。注意 `TraceLayer` 是在这里加的,所以七个服务都有请求日志。

**`ServeDir::new("assets")` 的路径是相对于工作目录的**,不是相对于源文件。所以必须在项目根目录 `cargo run`。**这和[第 32](32-tls-rustls.md#步骤-1跑起-https-服务)、[33 章](33-sse.md#步骤-3配上前端页面)用 `env!("CARGO_MANIFEST_DIR")` 的做法不同**——那个是编译期绝对路径,这个是运行时相对路径。两种都有问题:前者换机器失效,后者换工作目录失效。

> **更好的写法**:`ServeDir::new("assets")` 用相对路径,依赖于**运行时的工作目录**。`cargo run` 时工作目录是项目根,所以能找到;但把编译好的二进制拷到别处运行就会 404,而且**不会有任何报错**——`ServeDir` 找不到目录时和找不到文件一样返回 404。生产环境应该从环境变量读取资源目录路径,或者用相对于可执行文件的路径(`std::env::current_exe()`)。

**跑起来看**:

```bash
cargo run
```

```
DEBUG static_file_server: listening on 127.0.0.1:3001
```

```bash
curl http://127.0.0.1:3001/assets/index.html
curl http://127.0.0.1:3001/assets/script.js
```

```
Hi from index.html
console.log("Hello, World!");
```

**看 `Content-Type`:**

```bash
curl -i -s http://127.0.0.1:3001/assets/script.js | grep -i content-type
```

```
content-type: text/javascript
```

**按扩展名自动推断的** —— 前置知识那张表里的一项。

**不存在的文件:**

```bash
curl -i -s http://127.0.0.1:3001/assets/nope.txt | head -1
```

```
HTTP/1.1 404 Not Found
```

**验证路径穿越防护。** 试着逃出 `assets` 目录:

```bash
curl -i -s 'http://127.0.0.1:3001/assets/../Cargo.toml' | head -1
curl -i -s 'http://127.0.0.1:3001/assets/%2e%2e/Cargo.toml' | head -1
```

```
HTTP/1.1 404 Not Found
HTTP/1.1 404 Not Found
```

**都被拒了。** 注意 curl 可能会在发送前就规范化 `..`,所以第二条用了 URL 编码的 `%2e%2e` 绕过 curl 的处理,直接把编码后的路径发给服务端。**两种都拿不到 `Cargo.toml`** —— 这就是前置知识说的"别自己写"的理由。

**验证目录访问:**

```bash
curl -i -s http://127.0.0.1:3001/assets/ | head -1
```

```
HTTP/1.1 404 Not Found
```

**没配 `append_index_html_on_directories`,所以访问目录是 404。** 前面几章都配了这个选项,本章第一个例子没配。

### 步骤 2:SPA 兜底

**要做什么**:加上端口 3002 的服务,演示 `not_found_service` 和 `SetStatus`。

```rust
// src/main.rs
use axum::{http::StatusCode, routing::get, Router};        // 改动
use std::net::SocketAddr;
use tower_http::{
    services::{ServeDir, ServeFile},                        // 改动
    set_status::SetStatus,                                  // 新增
    trace::TraceLayer,
};
// ...(其余 use 不变)

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    // 改动:加上第二个服务
    tokio::join!(
        serve(using_serve_dir(), 3001),
        serve(using_serve_dir_with_assets_fallback(), 3002),
    );
}

// ...(using_serve_dir 不变)

// 新增
fn using_serve_dir_with_assets_fallback() -> Router {
    // `ServeDir` allows setting a fallback if an asset is not found
    // so with this `GET /assets/doesnt-exist.jpg` will return `index.html`
    // rather than a 404.
    // The `fallback_service` ensures that all other paths (the standard
    // SPA pattern) also return `index.html`. We wrap it in `SetStatus` so
    // the response uses 404 instead of 200 -- the path wasn't found, the SPA
    // just handles routing client-side.
    let index_html = SetStatus::new(ServeFile::new("assets/index.html"), StatusCode::NOT_FOUND);
    let serve_dir = ServeDir::new("assets").not_found_service(index_html.clone());

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .nest_service("/assets", serve_dir)
        .fallback_service(index_html)
}

// ...(serve 不变)
```

**这个函数同时用了三种机制**,值得逐个理清:

1. **`route("/foo", ...)`** —— 普通路由,优先级最高
2. **`nest_service("/assets", serve_dir)`** —— `/assets/*` 交给 `ServeDir`
3. **`fallback_service(index_html)`** —— 其它所有路径返回 `index.html`

而 `serve_dir` 内部还有一层:**`/assets/` 下找不到文件时,也返回 `index_html`**。

**`index_html.clone()`** —— `SetStatus<ServeFile>` 是可克隆的(`ServeFile` 内部只存路径)。两处都要用它,所以克隆一份。

**`SetStatus::new(..., StatusCode::NOT_FOUND)`** 前置知识讲过:返回 `index.html` 的内容但状态码是 `404`。

**跑起来看**:

```bash
cargo run
```

**四种情况依次验证:**

```bash
# 1. 普通路由
curl -s http://127.0.0.1:3002/foo
```

```
Hi from /foo
```

```bash
# 2. 存在的静态文件
curl -s http://127.0.0.1:3002/assets/script.js
```

```
console.log("Hello, World!");
```

```bash
# 3. /assets 下不存在的文件 → not_found_service
curl -i -s http://127.0.0.1:3002/assets/doesnt-exist.jpg | head -1
curl -s http://127.0.0.1:3002/assets/doesnt-exist.jpg
```

```
HTTP/1.1 404 Not Found
Hi from index.html
```

**`404` 状态码 + `index.html` 内容** —— `SetStatus` 的效果。

```bash
# 4. 完全不匹配的路径 → fallback_service
curl -i -s http://127.0.0.1:3002/users/42 | head -1
curl -s http://127.0.0.1:3002/users/42
```

```
HTTP/1.1 404 Not Found
Hi from index.html
```

**这就是 SPA 需要的行为**:`/users/42` 服务端没有这个文件,但返回 `index.html` 让前端 JS 处理路由。

**验证 `SetStatus` 的作用。** 临时去掉它:

```rust
    let index_html = ServeFile::new("assets/index.html");   // 临时:不包 SetStatus
    let serve_dir = ServeDir::new("assets").not_found_service(index_html.clone());
```

```bash
cargo run
curl -i -s http://127.0.0.1:3002/users/42 | head -1
```

```
HTTP/1.1 200 OK
```

**变成 `200` 了。** 对不存在的路径返回 `200`,会让搜索引擎收录一堆重复内容、让监控系统看不到 404 率。改回去。

**注意端口 3002 的 `/script.js`(根路径,不在 `/assets` 下)**:

```bash
curl -i -s http://127.0.0.1:3002/script.js | head -1
curl -s http://127.0.0.1:3002/script.js
```

```
HTTP/1.1 404 Not Found
Hi from index.html
```

**它没有匹配到 `/assets` 前缀,所以走了 `fallback_service`** —— 返回 `index.html` 而不是那个 JS 文件。**这正是官方测试里断言的行为**(完整代码里那个 `assets_not_served_at_root` 测试)。

### 步骤 3:另外五种用法

**要做什么**:一次补完剩下的例子,横向对比它们的差别。

```rust
// src/main.rs
use axum::{
    extract::Request, handler::HandlerWithoutStateExt, http::StatusCode, routing::get, Router,
};                                                       // 改动
use std::net::SocketAddr;
use tower::ServiceExt;                                   // 新增
use tower_http::{
    services::{ServeDir, ServeFile},
    set_status::SetStatus,
    trace::TraceLayer,
};
// ...(其余 use 不变)

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    // 改动:七个服务全上
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

// ...(前两个函数不变)

// 新增
fn using_serve_dir_only_from_root_via_fallback() -> Router {
    // you can also serve the assets directly from the root (not nested under `/assets`)
    // by only setting a `ServeDir` as the fallback
    let serve_dir = ServeDir::new("assets").not_found_service(ServeFile::new("assets/index.html"));

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}

// 新增
fn using_serve_dir_with_handler_as_service() -> Router {
    async fn handle_404() -> (StatusCode, &'static str) {
        (StatusCode::NOT_FOUND, "Not found")
    }

    // you can convert handler function to service
    let service = handle_404.into_service();

    let serve_dir = ServeDir::new("assets").not_found_service(service);

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}

// 新增
fn two_serve_dirs() -> Router {
    // you can also have two `ServeDir`s nested at different paths
    let serve_dir_from_assets = ServeDir::new("assets");
    let serve_dir_from_dist = ServeDir::new("dist");

    Router::new()
        .nest_service("/assets", serve_dir_from_assets)
        .nest_service("/dist", serve_dir_from_dist)
}

// 新增
#[allow(clippy::let_and_return)]
fn calling_serve_dir_from_a_handler() -> Router {
    // via `tower::Service::call`, or more conveniently `tower::ServiceExt::oneshot` you can
    // call `ServeDir` yourself from a handler
    Router::new().nest_service(
        "/foo",
        get(|request: Request| async {
            let service = ServeDir::new("assets");
            let result = service.oneshot(request).await;
            result
        }),
    )
}

// 新增
fn using_serve_file_from_a_route() -> Router {
    Router::new().route_service("/foo", ServeFile::new("assets/index.html"))
}

// ...(serve 不变)
```

逐个说清差别。

**端口 3003 `using_serve_dir_only_from_root_via_fallback`** —— 和 3002 的区别是**没有 `nest_service`**,静态文件直接从根路径提供:

```rust
Router::new()
    .route("/foo", get(...))
    .fallback_service(serve_dir)
```

**`/script.js` 现在能直接访问了**(3002 那边不行,因为它只在 `/assets` 下)。

注意 `not_found_service` 用的是**裸的 `ServeFile`**,没包 `SetStatus`,所以找不到时返回 `200`。**这个例子的行为和 3002 不一致**,官方示例没有统一,是有意展示不同配置。

**端口 3004 `using_serve_dir_with_handler_as_service`** —— 用一个普通 handler 处理 404,而不是返回 `index.html`:

```rust
async fn handle_404() -> (StatusCode, &'static str) {
    (StatusCode::NOT_FOUND, "Not found")
}
let service = handle_404.into_service();
```

前置知识讲过 `into_service()`。**这个做法适合 API 服务或多页应用**(不需要 SPA 兜底),想返回自定义的 404 页面。

**端口 3005 `two_serve_dirs`** —— 两个目录挂在不同前缀:

```rust
.nest_service("/assets", ServeDir::new("assets"))
.nest_service("/dist", ServeDir::new("dist"))
```

典型场景:`assets` 放静态资源,`dist` 放前端构建产物。**注意示例里 `dist` 目录不存在**,访问会 404。

**端口 3006 `calling_serve_dir_from_a_handler`** —— 在 handler 内部调用 `ServeDir`:

```rust
get(|request: Request| async {
    let service = ServeDir::new("assets");
    let result = service.oneshot(request).await;
    result
})
```

**`oneshot` 就是[第 21 章](21-testing.md#oneshot-与-readycall)测试里用的那个方法**,来自 `tower::ServiceExt`。它在这里的用途是"手动调用一个 `Service`"。

**为什么要这么做?** 因为这样可以在提供文件**之前**插入自己的逻辑:检查权限、记录下载、按用户改写路径。`nest_service` 做不到这些(它直接把请求交出去了)。

**代价是每次请求都新建一个 `ServeDir`** —— 这个开销很小(它只存了路径),但确实是浪费。更好的做法是把 `ServeDir` 放进 `State` 或用 `LazyLock` 建一次。

`#[allow(clippy::let_and_return)]` 是压制 clippy 的建议(它认为 `let result = ...; result` 该直接返回)。官方保留这个中间变量大概是为了可读性。

**端口 3307 `using_serve_file_from_a_route`** —— 单个文件挂在一条精确路由上:

```rust
Router::new().route_service("/foo", ServeFile::new("assets/index.html"))
```

**注意端口是 3307 而不是 3007** —— 官方示例里的一个笔误,照抄。

**跑起来看**:

```bash
cargo run
```

七个端口都起来了:

```
DEBUG static_file_server: listening on 127.0.0.1:3001
DEBUG static_file_server: listening on 127.0.0.1:3002
...
DEBUG static_file_server: listening on 127.0.0.1:3307
```

**用一个循环对比七个服务对同一路径的响应:**

```bash
for port in 3001 3002 3003 3004 3005 3006 3307; do
  printf "%s /script.js  → %s\n" "$port" "$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:$port/script.js)"
done
```

```
3001 /script.js  → 404
3002 /script.js  → 404
3003 /script.js  → 200
3004 /script.js  → 200
3005 /script.js  → 404
3006 /script.js  → 404
3307 /script.js  → 404
```

**只有 3003 和 3004 能从根路径拿到文件**,因为只有它们把 `ServeDir` 设成了 `fallback_service` 且没有前缀。

**再对比 `/assets/script.js`:**

```bash
for port in 3001 3002 3003 3004 3005 3006 3307; do
  printf "%s /assets/script.js → %s\n" "$port" "$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:$port/assets/script.js)"
done
```

```
3001 /assets/script.js → 200
3002 /assets/script.js → 200
3003 /assets/script.js → 200
3004 /assets/script.js → 200
3005 /assets/script.js → 200
3006 /assets/script.js → 404
3307 /assets/script.js → 404
```

**3003 和 3004 也能访问** —— 因为 `assets/assets/script.js` 不存在,走了 `not_found_service`,前者返回 `index.html`(200),后者返回 "Not found"(404)。

等等,3003 返回 200 但内容是什么?

```bash
curl -s http://127.0.0.1:3003/assets/script.js
```

```
Hi from index.html
```

**是 `index.html` 的内容!** 因为 `/assets/script.js` 在根目录下对应的是 `assets/assets/script.js`(不存在),于是走了 SPA 兜底。**这说明配置方式直接决定了路径映射关系**,搞混了会出现"返回了内容但不是你想要的文件"这种隐蔽问题。

**测试端口 3006 那个 handler 调用:**

```bash
curl -s http://127.0.0.1:3006/foo/script.js
```

```
console.log("Hello, World!");
```

**`nest_service("/foo", ...)` 剥掉了 `/foo` 前缀**,所以 `ServeDir` 收到的是 `/script.js`。

**测试端口 3307:**

```bash
curl -s http://127.0.0.1:3307/foo
```

```
Hi from index.html
```

**精确路径 `/foo` 映射到一个文件。**

## 完整代码

`assets/index.html`:

```
Hi from index.html
```

`assets/script.js`:

```js
console.log("Hello, World!");
```

`src/main.rs`:

```rust
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
    // `ServeDir` allows setting a fallback if an asset is not found
    // so with this `GET /assets/doesnt-exist.jpg` will return `index.html`
    // rather than a 404.
    // The `fallback_service` ensures that all other paths (the standard
    // SPA pattern) also return `index.html`. We wrap it in `SetStatus` so
    // the response uses 404 instead of 200 -- the path wasn't found, the SPA
    // just handles routing client-side.
    let index_html = SetStatus::new(ServeFile::new("assets/index.html"), StatusCode::NOT_FOUND);
    let serve_dir = ServeDir::new("assets").not_found_service(index_html.clone());

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .nest_service("/assets", serve_dir)
        .fallback_service(index_html)
}

fn using_serve_dir_only_from_root_via_fallback() -> Router {
    // you can also serve the assets directly from the root (not nested under `/assets`)
    // by only setting a `ServeDir` as the fallback
    let serve_dir = ServeDir::new("assets").not_found_service(ServeFile::new("assets/index.html"));

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}

fn using_serve_dir_with_handler_as_service() -> Router {
    async fn handle_404() -> (StatusCode, &'static str) {
        (StatusCode::NOT_FOUND, "Not found")
    }

    // you can convert handler function to service
    let service = handle_404.into_service();

    let serve_dir = ServeDir::new("assets").not_found_service(service);

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}

fn two_serve_dirs() -> Router {
    // you can also have two `ServeDir`s nested at different paths
    let serve_dir_from_assets = ServeDir::new("assets");
    let serve_dir_from_dist = ServeDir::new("dist");

    Router::new()
        .nest_service("/assets", serve_dir_from_assets)
        .nest_service("/dist", serve_dir_from_dist)
}

#[allow(clippy::let_and_return)]
fn calling_serve_dir_from_a_handler() -> Router {
    // via `tower::Service::call`, or more conveniently `tower::ServiceExt::oneshot` you can
    // call `ServeDir` yourself from a handler
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
    axum::serve(listener, app.layer(TraceLayer::new_for_http()))
        .await
        .unwrap();
}
```

## 代价与边界

**相对路径依赖工作目录。** 步骤 1 的「更好的写法」指出了:二进制换位置运行就 404,而且**没有任何报错**——找不到目录和找不到文件的表现完全一样。

**生产环境通常不用应用服务静态文件。** 和[第 32 章](32-tls-rustls.md#代价与边界)那个 TLS 的结论类似:主流做法是让 CDN 或 Nginx 处理静态资源:

- **CDN 能就近分发**,延迟远低于回源
- **专用服务器更高效**(sendfile、内存映射)
- **应用服务器的连接数很宝贵**,不该用来传图片

**应用内服务静态文件的合理场景**:小型项目、内部工具、单体部署、或者需要在提供文件前做权限检查(端口 3006 那个例子)。

**没有配缓存头部。** `ServeDir` 会设 `Last-Modified` 并处理 `If-Modified-Since`(返回 `304`),但**不会设 `Cache-Control`**。这意味着浏览器每次都要发一个请求确认。生产环境应该配合 `SetResponseHeaderLayer` 加上 `Cache-Control: public, max-age=31536000`(配合文件名哈希做缓存失效)。

**`SetStatus(404)` 对 SPA 有争议。** 前置知识提过。返回 `404` + HTML 内容这个组合本身就矛盾:

- **支持方**:URL 确实不存在,`404` 是诚实的,对 SEO 和监控友好
- **反对方**:某些前端路由库、Service Worker、预渲染工具看到 `404` 会认为加载失败

**折中做法**是区分对待:已知的 SPA 路由前缀(`/app/*`)返回 `200`,其余返回真正的 `404`。

**七个服务的配置不一致。** 3002 用了 `SetStatus`,3003 没用;3002 的静态文件在 `/assets` 下,3003 在根路径。**这是示例故意展示不同选项**,但如果照抄时不注意,很容易配出"预期外但不报错"的行为——步骤 3 里 3003 的 `/assets/script.js` 返回 `index.html` 就是例子。

**没有压缩。** `ServeDir` 支持 `precompressed_gzip()` / `precompressed_br()`,优先返回预压缩的 `.gz`/`.br` 文件。示例没配,所有文件都是原始大小传输。[第 39 章](39-compression.md)会讲实时压缩。

**没有限制文件大小或类型。** `ServeDir` 会提供目录下的**任何**文件。如果那个目录里不小心放了 `.env` 或备份文件,就直接暴露了。**部署时要确保静态目录里只有该公开的东西。**

**端口 3307 是笔误。** 步骤 3 提过,官方示例里的 typo。

**`calling_serve_dir_from_a_handler` 每次新建 `ServeDir`。** 步骤 3 提过,开销小但没必要。

## 动手练习

**1. 让资源目录可配置。**

提示:`std::env::var("ASSETS_DIR").unwrap_or_else(|_| "assets".to_string())`,传给 `ServeDir::new(...)`。做完把编译好的二进制拷到 `/tmp` 下运行,用 `ASSETS_DIR=/完整路径/assets` 验证它仍然工作。**这是「代价与边界」里最实际的一处改进。**

**2. 加上缓存头部。**

提示:用 `tower_http::set_header::SetResponseHeaderLayer` 给 `/assets` 下的响应加 `Cache-Control: public, max-age=3600`。注意要挂在 `nest_service` 的外面,或者用 `ServiceBuilder` 包装 `ServeDir`。验证:`curl -i` 看头部,然后带 `If-Modified-Since` 再请求一次,确认返回 `304`。

**3. 在提供文件前检查权限。**

提示:参照端口 3006 那个例子,在 handler 里先检查一个查询参数或头部(比如 `?token=secret`),通过了再 `service.oneshot(request).await`,否则返回 `403`。**这正是"在应用内服务静态文件"少数几个合理理由之一** —— CDN 做不到这种细粒度控制。

## 小结与下一章预告

`ServeDir` / `ServeFile` 把磁盘目录或文件映射成 HTTP 路由,并处理了一堆容易写错的细节:**路径穿越防护**、MIME 推断、流式读取、条件请求(`304`)、范围请求。**别自己实现这个** —— 本章验证过 `../` 和 URL 编码的 `%2e%2e` 都被拒绝。

它们是 `Service` 而不是 handler,所以要用 `_service` 后缀的挂载方法:**`nest_service`(前缀,会剥掉前缀)、`fallback_service`(兜底,不剥)、`route_service`(精确路径)**。选错会导致路径映射关系不对——步骤 3 里端口 3003 的 `/assets/script.js` 返回了 `index.html` 而非 JS 文件,就是这类问题。

**SPA 的兜底模式**是 `not_found_service` + `fallback_service` 双层:找得到文件返回文件,找不到返回 `index.html` 让前端路由处理。用 `SetStatus` 包一层能把状态码从 `200` 改成 `404` —— 这个选择有争议,对 SEO 友好但可能让某些前端工具误判。

`HandlerWithoutStateExt::into_service()` 把普通 handler 变成 `Service`,用于 `not_found_service` 这类要求 `Service` 的地方。

**生产环境通常让 CDN 或 Nginx 处理静态资源** —— 应用内服务的合理场景是小型项目、内部工具、或者需要在提供文件前做权限检查(端口 3006 那个 `oneshot` 用法)。

最后回答开头那个问题:**这一章排在第 38 位,是因为它依赖 tower 的 `Service` 概念(第 17 章)和测试知识(第 21 章)。** 这是本教程"按知识依赖排序"原则的直接代价——读者从第 04 章就想要的功能,要等到这里才讲。

本章的文件都是原样传输的。[第 39 章 compression](39-compression.md) 讲怎么压缩:`CompressionLayer` 如何根据客户端的 `Accept-Encoding` 自动选择算法、`DecompressionLayer` 如何处理压缩的请求体,以及一个和[第 33 章](33-sse.md#代价与边界) SSE 类似的冲突——**压缩中间件会缓冲数据,和流式响应天然不合**。
