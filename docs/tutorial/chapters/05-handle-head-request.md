# 05. handle-head-request

对应示例：`examples/handle-head-request`

HTTP 里有一个容易被忽略的方法：`HEAD`——和 GET 一样拿响应头，但不需要响应 body。axum 对 GET 路由有默认行为：GET 路由也会处理 HEAD 请求，但响应 body 会被移除。本章演示当 GET 的 body 生成成本很高时，如何在 HEAD 请求里跳过这部分计算。这章比前四章偏 HTTP 细节，测试代码（`ServiceExt::oneshot`）可以第一次粗读，后面测试章节再回头看。

## Cargo.toml

````toml
[package]
name = "example-handle-head-request"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }

[dev-dependencies]
http-body-util = "0.1.0"
hyper = { version = "1.0.0", features = ["full"] }
tower = { version = "0.5.2", features = ["util"] }
````

- `tower`：测试里用 `ServiceExt::oneshot` 直接调用 Router。
- `http-body-util`：测试里收集响应 body。
- 这两个是 `[dev-dependencies]`，只在测试时编译，不进运行时。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

````rust
use axum::response::{IntoResponse, Response};
use axum::{http, routing::get, Router};

fn app() -> Router {
    Router::new().route("/get-head", get(get_head_handler))
}

#[tokio::main]
async fn main() {
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    println!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app()).await;
}

async fn get_head_handler(method: http::Method) -> Response {
    if method == http::Method::HEAD {
        return ([("x-some-header", "header from HEAD")]).into_response();
    }

    do_some_computing_task();

    ([("x-some-header", "header from GET")], "body from GET").into_response()
}

fn do_some_computing_task() {
    // TODO
}
````

注意 `app()` 单独写成函数返回 Router——这让 `main` 和测试可以复用同一套路由，是真实项目很常见的写法。

## 运行

````bash
cd examples
cargo run -p example-handle-head-request
````

GET：

````bash
curl -i http://127.0.0.1:3000/get-head
````

预期：

````text
HTTP/1.1 200 OK
x-some-header: header from GET

body from GET
````

HEAD（`curl -I` 只显示响应头）：

````bash
curl -I http://127.0.0.1:3000/get-head
````

预期：

````text
HTTP/1.1 200 OK
x-some-header: header from HEAD
````

HEAD 响应本来就不该有 body，所以看不到 `body from GET` 是正确行为。

运行测试：

````bash
cargo test -p example-handle-head-request
````

## 解读

### GET 和 HEAD 的关系

```text
GET /get-head  → 执行 do_some_computing_task() → 返回 header + body
HEAD /get-head → 提前返回 header → axum/HTTP 语义保证没有 body
```

axum 的 GET 路由会**隐式处理 HEAD**：HEAD 请求也进入同一个 handler，但响应 body 会被 axum 移除。这意味着如果不做特殊处理，HEAD 请求也会执行 `do_some_computing_task()`，白白浪费计算。

### 提取 `http::Method`

````rust
async fn get_head_handler(method: http::Method) -> Response {
    if method == http::Method::HEAD {
        return ([("x-some-header", "header from HEAD")]).into_response();
    }
    ...
}
````

`method: http::Method` 是一个 extractor，让 handler 拿到当前请求的 HTTP 方法。HEAD 时提前返回 header，跳过昂贵的 body 计算。

这种写法适合 GET 和 HEAD 逻辑大体相同、只是 HEAD 想跳过昂贵 body 的场景。如果 HEAD 业务语义明显不同，也可以显式注册 HEAD 路由，把两个 handler 分开。

### `.into_response()`

返回类型是 `Response`（具体类型，不是 `impl IntoResponse`），所以末尾要明确调用 `.into_response()` 把元组转换成 `Response`。`[("x-some-header", "...")]` 是 header 数组，`(headers, body)` 元组会被 axum 转成 HTTP 响应。

### 测试不启动端口

````rust
let response = app
    .oneshot(Request::get("/get-head").body(Body::empty()).unwrap())
    .await
    .unwrap();
````

测试直接给 Router 一个 HTTP request 拿到 response——这说明 axum 的 Router 本质上也是 Tower service（见附录 A0）。这比启动真实端口更快、更稳定，也是 `app()` 单独提出来的原因。

## 手写任务

跑通后做三个小观察：

1. 把 GET 返回的 body 改长，例如 `this is a long body from GET`。
2. `curl -i` 确认 GET 能看到 body。
3. `curl -I` 确认 HEAD 仍然只有响应头。

结论：HEAD 关心响应头，GET 关心响应头和响应体。

## 小结

- HTTP `HEAD` 返回响应头但不返回 body。
- axum 的 GET 路由隐式处理 HEAD，body 会被移除。
- body 生成昂贵时，提取 `http::Method` 并在 HEAD 时提前返回，可以省掉这部分计算。
- GET 和 HEAD 差异大时，可以显式注册 HEAD 路由分开处理。
- 把 `app()` 单独提出，让 main 和测试复用同一个 Router。
- `ServiceExt::oneshot` 可以不启动端口直接测试 Router。

## 源码对照

- `examples/handle-head-request/Cargo.toml`
- `examples/handle-head-request/src/main.rs`
