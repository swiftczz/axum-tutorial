# 31. simple-router-wasm

对应示例：`examples/simple-router-wasm`

前面所有示例都是 `TcpListener → axum::serve(listener, app)`。这章换角度:不启动端口,把 `Router` 当成一个处理单次 `Request` 的 service——模拟 WASM 或 serverless 风格环境。核心认识:**axum Router 本身是 Tower service,不一定非要配合 TcpListener 使用**。

## Cargo.toml

````toml
[package]
name = "example-simple-router-wasm"
version = "0.1.0"
edition = "2018"
publish = false

[package.metadata.cargo-machete]
ignored = ["axum-extra"]

[dependencies]
# `default-features = false` to not depend on tokio features which don't support wasm
axum = { version = "0.8", default-features = false }
axum-extra = { version = "0.12", default-features = false }
futures-executor = "0.3.21"
http = "1.0.0"
tower-service = "0.3.1"
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

````rust
use axum::{
    response::{Html, Response},
    routing::get,
    Router,
};
use futures_executor::block_on;
use http::Request;
use tower_service::Service;

fn main() {
    let request: Request<String> = Request::get("https://serverless.example/api/")
        .body("Some Body Data".into())
        .unwrap();

    let response: Response = block_on(app(request));

    assert_eq!(200, response.status());
}

#[allow(clippy::let_and_return)]
async fn app(request: Request<String>) -> Response {
    let mut router = Router::new().route("/api/", get(index));
    let response = router.call(request).await.unwrap();
    response
}

async fn index() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
````

## 运行

````bash
cd examples
cargo run -p example-simple-router-wasm
````

没输出且正常退出,说明 `assert_eq!(200, response.status())` 通过。编译到 wasm 目标:

````bash
rustup target add wasm32-unknown-unknown   # 没装先装
cargo build -p example-simple-router-wasm --target wasm32-unknown-unknown
````

## 解读

### 传统服务器 vs 本章模式

```text
传统:程序自己监听端口 → axum::serve(listener, app) → 一直运行
本章:外部运行时负责接收请求 → 你的函数把 Request 变成 Response → router.call(request).await
```

```text
let listener = tokio::net::TcpListener::bind(...).await.unwrap();
axum::serve(listener, app).await.unwrap();
                            ↓ 对比 ↓
let response = router.call(request).await.unwrap();
```

这是理解 serverless 和部分 WASM 场景的关键。

### 为什么关闭 axum 默认 feature

````toml
axum = { version = "0.8", default-features = false }
````

tokio 的 IO 层 `mio` 不支持 `wasm32-unknown-unknown`。axum 默认 feature 会拉入传统服务器运行时需要的能力,但 `wasm32-unknown-unknown` 没有常规操作系统 TCP IO。关闭默认 feature 后只用 routing/extractor/response/tower service 等核心能力。

`axum-extra` 故意加进依赖是为了**验证它在 wasm 目标也能编译**(关闭默认 feature 后),你自己项目里不需要加(用 `[package.metadata.cargo-machete] ignored` 标记)。

### 手动构造 Request

````rust
let request: Request<String> = Request::get("https://serverless.example/api/")
    .body("Some Body Data".into())
    .unwrap();
````

没浏览器也没 curl,代码手动创建 HTTP 请求(method、URI、body)。说明 `Request` 本身就是一个结构体——传统服务器只是帮你从网络连接里构造出它,本章直接手动构造。

URI 是完整 URL 但路由仍能匹配 `/api/`,因为 Router 匹配时只关注路径部分。

### `block_on` 在同步 main 跑 async

````rust
let response: Response = block_on(app(request));
````

`app(request)` 是 async 函数返回 future,普通同步 `main` 不能直接 `.await`,用 `futures_executor::block_on` 运行 future 直到拿结果。真实 WASM/serverless 运行时通常自己负责调 async 函数,这里用 main 模拟运行时。

### `app` 像 serverless handler

````rust
async fn app(request: Request<String>) -> Response {
    let mut router = Router::new().route("/api/", get(index));
    let response = router.call(request).await.unwrap();
    response
}
````

接收 Request 返回 Response,很像 serverless 平台的 `fn handle(request) -> response`。内部仍用 axum Router:**外面是 serverless 风格,里面仍然用 axum 路由和 handler**。

### Router 是 Tower service

````rust
use tower_service::Service;  // 引入才能调 .call()
let mut router = Router::new().route("/api/", get(index));
let response = router.call(request).await.unwrap();
````

`Router` 实现了 Tower 的 `Service` trait——Service 可理解为"接收 Request 异步返回 Response"。这就是为什么 Router 不一定通过 `axum::serve` 使用,你也可以直接把请求交给它。

**为什么 router 要 `mut`?** `Service::call` 需要可变引用,因为 service 可能维护内部状态(readiness、缓存、中间件状态)。

### handler 写法不变

````rust
async fn index() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
````

即使外层运行环境不同,extractor/handler/response/Router 这些 axum 核心能力依然能用。

## 常见问题

**这是浏览器前端 WASM 吗?** 不是。主要展示 axum Router 在 WASM 目标可编译,以及 serverless 风格 request/response 调用方式。

**为什么没有 `tokio::main`?** 不启动 Tokio TCP server,用 `futures_executor::block_on` 在同步 main 中执行 async 函数。

**为什么 axum 要 `default-features = false`?** 默认 feature 引入 Tokio IO,`wasm32-unknown-unknown` 不支持常规 OS IO;关闭后只用 Router/handler/extractor/response 核心能力。

**为什么 `use tower_service::Service`?** `router.call(request)` 来自 `Service` trait,不引入这个 trait 方法无法调用。

## 手写任务

按下面顺序敲:

1. 创建普通 `Router::new().route("/api/", get(index))`。
2. 写 `index` handler 返回 `Html`。
3. 手动构造 `Request::get("https://serverless.example/api/")`。
4. 写 `async fn app(request) -> Response`。
5. 在 `app` 里用 `router.call(request).await`。
6. main 里用 `block_on(app(request))`。
7. 断言状态码 200。

加深练习:

1. 新增 `/api/health` 路由,手动构造请求验证状态码。
2. handler 改成返回 JSON,观察 Response 仍正常返回。
3. 尝试编译 `wasm32-unknown-unknown`,理解哪些依赖不能用于该目标。

## 小结

- axum 核心不只是"监听端口":`Router` 是 Tower Service,接收 Request 返回 Response。
- 传统服务器里 `axum::serve` 从 TCP 连接产生 Request;serverless/WASM 风格环境里外部运行时已给你 Request,你只调用 `router.call(request).await`。
- WASM 目标关闭 axum 默认 feature(`mio` 不支持 wasm),只用核心能力。
- handler 写法不变,extractor/Router/response 等核心能力可在更多运行环境复用。
- `Service::call` 需要 `&mut self`(维护内部状态),所以 router 要 `mut`。

## 源码对照

- `examples/simple-router-wasm/Cargo.toml`
- `examples/simple-router-wasm/src/main.rs`
