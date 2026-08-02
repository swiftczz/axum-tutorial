# 第 06 章 global-404-handler:兜底处理器

## 本章你将做出什么

一个和[第 01 章](01-hello-world.md)几乎一样的服务,但访问不存在的路径时,返回的不再是 axum 那个空 body 的 `404`,而是你自己写的内容。

改动只有一行,但这一行背后是 `Router` 的**兜底**(fallback)机制。理解它顺带能看清 `Router` 匹配失败时到底发生了什么。

学完本章你将掌握:

- `Router::fallback` 的作用和覆盖范围
- 元组 `(StatusCode, &str)` 作为返回类型如何同时设定状态码和 body
- `fallback` 和 `method_not_allowed_fallback` 的分工,以及 `404` 与 `405` 的边界

## 前置知识

### 默认 404 的问题

[第 01 章](01-hello-world.md#跑起来看)见过 axum 默认的 `404`:

```
HTTP/1.1 404 Not Found
content-length: 0
date: Thu, 30 Jul 2026 06:13:02 GMT
```

**body 是空的。** 对浏览器用户来说,看到的是一个纯白页面;对调用 API 的程序来说,拿到一个 `404` 但没有任何说明信息。

真实服务通常希望:

- 网页服务返回一个带导航的友好错误页
- API 服务返回结构化的 JSON,如 `{"error": "not found", "path": "/nope"}`
- 至少记一条日志,便于发现死链或探测行为

axum 把这件事交给 `fallback`。

### fallback:匹配不上时交给谁

`Router` 的路由表是一棵前缀树,请求进来先查这棵树。查不到时,axum 需要一个"默认去处"——这就是 fallback。

不设 fallback 时,axum 用内置的默认实现,直接返回空 body 的 `404`。调用 `Router::fallback(handler)` 就把它换成你的处理器:

```rust
let app = Router::new()
    .route("/", get(handler))
    .fallback(handler_404);
```

**fallback 处理器就是一个普通 handler。** 它同样满足[第 01 章](01-hello-world.md#handler为什么普通函数能当处理器)讲的 `Handler` trait 要求——可以有提取器参数,可以返回任何实现 `IntoResponse` 的东西。区别只是它在"路径匹配失败"时被调用,而不是绑定到某个具体路径。

这意味着 fallback 里能拿到请求信息。想在错误页里显示用户访问的路径,提取 `Uri` 就行:

```rust
async fn handler_404(uri: Uri) -> impl IntoResponse {
    (StatusCode::NOT_FOUND, format!("no route for {uri}"))
}
```

`Uri` 是 `FromRequestParts` 类别的提取器,axum 为一批常用的请求信息类型(`Uri`、`Method`、`HeaderMap`)直接实现了它,不需要包装类型。

### fallback 不管 405

这是最容易搞错的一点。[第 01 章](01-hello-world.md#跑起来看)和[第 02 章](02-routes-and-handlers-close-together.md#router-与-methodrouter-的分工)讲过 axum 的两层路由:`Router` 按路径分发给出 `404`,`MethodRouter` 按方法分发给出 `405`。

`fallback` 只接管**第一层**的失败。请求 `POST /` 时路径匹配成功了,失败发生在第二层,所以 fallback **不会**被调用,客户端仍然收到默认的 `405`。

想同时接管 `405`,要用另一个方法:

```rust
let app = Router::new()
    .route("/", get(handler))
    .fallback(handler_404)                        // 路径不匹配
    .method_not_allowed_fallback(handler_405);    // 路径匹配但方法不对
```

两个方法各管一层,职责清晰。本章示例只用了前者,`405` 的部分留到练习题。

### 元组作为返回类型

[第 01 章](01-hello-world.md#intoresponse返回值如何变成-http-响应)那张表里有一行 `(StatusCode, T)`。本章的 fallback 就用它:

```rust
(StatusCode::NOT_FOUND, "nothing to see here")
```

这是一个二元元组,axum 为它实现了 `IntoResponse`:第一个元素设定状态码,第二个元素生成 body 和相应的 `Content-Type`。所以这行代码等价于"状态码 `404`,body 是纯文本 `nothing to see here`,`Content-Type: text/plain; charset=utf-8`"。

元组的形式不止这一种,axum 还实现了:

| 元组形式 | 效果 |
|---|---|
| `(StatusCode, T)` | 状态码 + body |
| `(StatusCode, HeaderMap, T)` | 状态码 + 自定义头部 + body |
| `([(&str, &str); N], T)` | 自定义头部 + body,状态码默认 `200` |
| `(HeaderMap, T)` | 同上,用 `HeaderMap` 形式 |

`T` 本身也得实现 `IntoResponse`,所以可以嵌套——`(StatusCode::NOT_FOUND, Html("<h1>404</h1>"))` 就是"`404` + HTML 页面"。第 07 章会用到带头部的那种形式。

这个设计让"改一下状态码"这种小需求不需要手工构造 `Response`,只要在返回值外面套个元组。

## 项目搭建

```bash
cargo new global-404-handler
cd global-404-handler
```

```bash
cargo add axum
cargo add tokio --features full
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

依赖和[第 03 章](03-form.md#项目搭建)相同,少了 `serde`(本章不需要反序列化)。

## 分步构建

### 步骤 1:一个正常的服务

**要做什么**:先把第 01 章那个服务重建一遍,确认默认 `404` 的行为。

```rust
// src/main.rs
use axum::{response::Html, routing::get, Router};
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

    // build our application with a route
    let app = Router::new().route("/", get(handler));

    // run it
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
```

除了日志样板,这和[第 01 章](01-hello-world.md#完整代码)的完整代码一模一样。

**跑起来看**:

```bash
cargo run
```

```bash
curl -i http://127.0.0.1:3000/
```

```
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 22

<h1>Hello, World!</h1>
```

不存在的路径:

```bash
curl -i http://127.0.0.1:3000/nope
```

```
HTTP/1.1 404 Not Found
content-length: 0
date: Thu, 30 Jul 2026 07:02:11 GMT
```

`content-length: 0`——就是前置知识里说的空 body。用 `-v` 能更清楚地看到 curl 什么都没收到:

```bash
curl -v http://127.0.0.1:3000/nope 2>&1 | tail -5
```

### 步骤 2:加上 fallback

**要做什么**:注册一个兜底处理器,让 `404` 带上内容。

```rust
// src/main.rs
use axum::{
    http::StatusCode,                          // 新增
    response::{Html, IntoResponse},            // 改动:加上 IntoResponse
    routing::get,
    Router,
};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    // build our application with a route
    let app = Router::new().route("/", get(handler));

    // 新增:add a fallback service for handling routes to unknown paths
    let app = app.fallback(handler_404);

    // ...(listener 和 serve 不变)
}

// ...(handler 不变)

// 新增:路径匹配失败时的兜底处理器
async fn handler_404() -> impl IntoResponse {
    (StatusCode::NOT_FOUND, "nothing to see here")
}
```

**注意 `let app = app.fallback(handler_404);` 这个写法。** 它用了 Rust 的变量遮蔽(shadowing):重新声明一个同名变量 `app`,值是在旧 `app` 上调用 `fallback` 的结果。

官方示例这么写是为了把注释分开——上面一行讲"构建路由",下面一行讲"加兜底"。功能上和链式调用完全等价:

```rust
let app = Router::new()
    .route("/", get(handler))
    .fallback(handler_404);
```

两种写法选哪个纯属风格问题。`Router` 的所有方法都是消耗 `self` 返回新 `Router` 的形式(所谓 builder 模式),所以既能链式串起来,也能分步赋值。

**`handler_404` 的返回类型是 `impl IntoResponse`。** [第 04 章](04-templates.md#步骤-3为自己的类型实现-intoresponse)讲过这个写法:具体类型由编译器推断,这里实际是 `(StatusCode, &'static str)`。

写成具体类型也完全可以,而且更直白:

```rust
async fn handler_404() -> (StatusCode, &'static str) {
    (StatusCode::NOT_FOUND, "nothing to see here")
}
```

官方示例用 `impl IntoResponse` 是为了将来改返回内容时不必改签名。

**跑起来看**:

```bash
cargo run
```

正常路径不受影响:

```bash
curl -i http://127.0.0.1:3000/
```

```
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 22

<h1>Hello, World!</h1>
```

不存在的路径现在有内容了:

```bash
curl -i http://127.0.0.1:3000/nope
```

```
HTTP/1.1 404 Not Found
content-type: text/plain; charset=utf-8
content-length: 20

nothing to see here
```

**状态码仍然是 `404`,但多了 `content-type` 和 body。** 元组的第一个元素设定状态码,第二个元素既生成了 body 也决定了 `Content-Type`——前置知识里说的那个机制。

fallback 覆盖所有未匹配的路径,层级多深都算:

```bash
curl -s http://127.0.0.1:3000/a/b/c/d
curl -s http://127.0.0.1:3000/admin
curl -s http://127.0.0.1:3000/.env
```

```
nothing to see here
nothing to see here
nothing to see here
```

### 步骤 3:验证 fallback 管不到 405

**要做什么**:不写新代码,确认前置知识里那个容易踩的坑。

```bash
curl -i -X POST http://127.0.0.1:3000/
```

```
HTTP/1.1 405 Method Not Allowed
allow: GET
content-length: 0
date: Thu, 30 Jul 2026 07:05:43 GMT
```

**`405`,而且 body 又是空的。** `handler_404` 一次都没被调用。

原因就是前置知识讲的两层结构:路径 `/` 在第一层匹配成功了,请求被交给它的 `MethodRouter`;`MethodRouter` 发现没有 `POST` 分支,自己返回 `405`。整个过程压根没走到 fallback 那一层。

对比一下两种失败:

| 请求 | 第一层(路径) | 第二层(方法) | 结果 | 走 fallback? |
|---|---|---|---|---|
| `GET /` | 匹配 | 匹配 | `200` + HTML | 否 |
| `GET /nope` | **不匹配** | — | `404` + 你的内容 | **是** |
| `POST /` | 匹配 | **不匹配** | `405` + 空 body | 否 |

要接管第三种情况,得用 `method_not_allowed_fallback`。这是练习题 2 的内容。

值得说明的是,axum 把这两件事分开而不是合并成一个 fallback,是有道理的:`404` 和 `405` 对客户端的含义完全不同。`404` 说"这个资源不存在,别再找了";`405` 说"资源在,但你用错了方法,换个方法再试"。前者通常该给用户看友好页面,后者更多是给程序看的提示,还得带上 `allow` 头部告诉对方该用什么方法。合并处理反而容易写出语义混乱的响应。

## 完整代码

`src/main.rs`:

```rust
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

    // build our application with a route
    let app = Router::new().route("/", get(handler));

    // add a fallback service for handling routes to unknown paths
    let app = app.fallback(handler_404);

    // run it
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}

async fn handler_404() -> impl IntoResponse {
    (StatusCode::NOT_FOUND, "nothing to see here")
}
```

## 动手练习

**1. 让 404 页面显示用户访问的路径,并记一条日志。**

提示:给 `handler_404` 加一个 `uri: Uri` 参数(`axum::http::Uri`),返回 `(StatusCode::NOT_FOUND, format!("no route for {uri}"))`。顺手加一行 `tracing::warn!(%uri, "404")`——`%uri` 是 tracing 的语法,表示按 `Display` 记录这个字段。验证:`curl http://127.0.0.1:3000/nope` 看响应,同时看服务端终端的日志。

**2. 用 `method_not_allowed_fallback` 接管 405。**

提示:再写一个 `handler_405`,返回 `(StatusCode::METHOD_NOT_ALLOWED, "wrong method")`,用 `.method_not_allowed_fallback(handler_405)` 注册。验证:`curl -i -X POST http://127.0.0.1:3000/` 应该看到你的内容,但要注意 **`allow` 头部还在不在**?如果不在,想想这对客户端意味着什么(提示:HTTP 规范要求 `405` 响应必须带 `Allow`)。

**3. 把 404 改成 JSON 响应。**

提示:`(StatusCode::NOT_FOUND, Json(json!({"error": "not found"})))`,需要 `cargo add serde_json`。这是 API 服务更常见的做法。对比一下 `content-type` 的变化。

## 小结与下一章预告

`Router::fallback` 接管所有路径匹配失败的请求,fallback 处理器就是普通 handler,可以用提取器拿到请求信息。它只管第一层路由的失败——路径存在但方法不对时走的是 `MethodRouter`,归 `method_not_allowed_fallback` 管。

元组 `(StatusCode, T)` 作为返回类型能同时设定状态码和 body,不必手工构造 `Response`。这是 axum 里调整响应最轻量的手段。

本章的 fallback 是"路径不存在"的处理。下一章换个角度:[第 07 章 handle-head-request](07-handle-head-request.md) 讲一个容易被忽略的 HTTP 方法——`HEAD`。axum 默认会用 `GET` 的 handler 处理 `HEAD` 请求并自动丢掉 body,但有时你想显式区分它们。那一章你会看到如何用 `Method` 提取器在 handler 里判断方法,以及带自定义头部的元组返回类型。
