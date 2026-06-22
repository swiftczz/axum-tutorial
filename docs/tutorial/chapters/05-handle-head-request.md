# 05. handle-head-request

对应示例：`examples/handle-head-request`

本章目标：理解 GET 与 HEAD 的关系，以及何时特殊处理 HEAD。

这一章比前四章更偏 HTTP 细节和测试技巧。第一次学习可以先重点理解 GET 和 HEAD 的区别；测试代码和 `ServiceExt::oneshot` 可以先粗读，后面学测试章节时再回来看。

## 这个小项目在做什么

HTTP 里有一个容易被新手忽略的方法：`HEAD`。

`HEAD` 的语义是：

```text
和 GET 一样拿响应头，但不需要响应 body。
```

Axum 对 GET 路由有一个默认行为：

```text
GET 路由也会处理 HEAD 请求，但最终响应 body 会被移除。
```

这个 example 演示：

- `GET /get-head`：做计算任务，返回 header 和 body。
- `HEAD /get-head`：不做昂贵计算，只返回 header。
- 测试里验证 GET 有 body，HEAD 没有 body。

## 新手先抓住主线

请求流程：

```text
GET /get-head
-> get_head_handler(method)
-> method 不是 HEAD
-> 执行 do_some_computing_task()
-> 返回 header + body

HEAD /get-head
-> get_head_handler(method)
-> method 是 HEAD
-> 提前返回 header
-> Axum/HTTP 语义保证没有 body
```

这一章的重点是：

```text
有些 GET 的 body 生成成本很高。
HEAD 请求只想要响应头，这时可以跳过昂贵计算。
```

## 文件和依赖

这个 example 有两个主要文件：

1. `examples/handle-head-request/Cargo.toml`：声明运行依赖和测试依赖。
2. `examples/handle-head-request/src/main.rs`：实现 GET/HEAD handler，并用测试验证行为。

关键依赖：

- `axum`：提供 `Router`、`get`、`IntoResponse` 和 `Response`。
- `tokio`：提供异步运行时和异步测试支持。
- `tower`：测试中用 `ServiceExt::oneshot` 直接调用 Router。
- `http-body-util`：测试中收集响应 body。

## 第一步：把 app 单独写成函数

源码：

````rust
fn app() -> Router {
    Router::new().route("/get-head", get(get_head_handler))
}
````

为什么这章把 Router 放进 `app()`，而不是直接写在 `main()`？

因为测试也要复用同一个 Router。

如果只有 `main()` 里有 Router，测试就很难直接调用。  
把 Router 构造提出来以后：

```text
main 可以用 app() 启动服务
test 也可以用 app() 发内部请求
```

这是真实项目很常见的写法。

## 第二步：handler 提取 HTTP 方法

源码：

````rust
async fn get_head_handler(method: http::Method) -> Response {
````

这里的 `method: http::Method` 是一个 extractor。

它告诉 Axum：

```text
把当前请求的 HTTP 方法提取出来，传给 handler。
```

所以 handler 可以判断：

````rust
if method == http::Method::HEAD {
    ...
}
````

## 第三步：特殊处理 HEAD

源码：

````rust
if method == http::Method::HEAD {
    return ([("x-some-header", "header from HEAD")]).into_response();
}
````

这里直接提前返回。

为什么？

因为 HEAD 不需要 body。  
如果 body 的生成很贵，提前返回可以省掉这部分成本。

`[("x-some-header", "header from HEAD")]` 是一个 header 数组。  
调用 `.into_response()` 后，它会变成 HTTP 响应。

这种写法适合 GET 和 HEAD 逻辑大体相同，只是 HEAD 想跳过昂贵 body 生成的场景。如果 HEAD 的业务语义明显不同，也可以显式注册 HEAD 路由，把两个 handler 分开组织。

## 第四步：正常处理 GET

源码：

````rust
do_some_computing_task();

([("x-some-header", "header from GET")], "body from GET").into_response()
````

GET 请求会执行计算任务，然后返回：

- 一个响应头：`x-some-header: header from GET`
- 一个响应体：`body from GET`

为什么最后也调用 `.into_response()`？

因为函数返回类型是 `Response`，不是 `impl IntoResponse`。  
所以这里要明确把元组转换成 `Response`。

## 第五步：理解测试

这个 example 还包含测试。测试没有真的启动端口，而是直接调用 Router：

````rust
let response = app
    .oneshot(Request::get("/get-head").body(Body::empty()).unwrap())
    .await
    .unwrap();
````

这说明 Axum 的 Router 本质上也是 Tower service。  
测试可以直接给它一个 HTTP request，然后拿到 response。

这比启动真实端口更快、更稳定。

## 函数职责速查

- `app`：构造应用 Router，让 `main` 和测试复用同一套路由。
- `main`：绑定端口，把 `app()` 返回的 Router 交给 `axum::serve`。
- `get_head_handler`：同时处理 GET 和隐式 HEAD，通过 `http::Method` 判断是否跳过 body 计算。
- `do_some_computing_task`：模拟生成 GET body 前的昂贵计算。
- `test_get`：验证 GET 返回 200、GET header 和 `body from GET`。
- `test_implicit_head`：验证 HEAD 返回 200、HEAD header，并且 body 为空。

这一章最值得关注的是 `app()` 和测试的关系：Router 构造逻辑提出来以后，运行服务和单元测试不会各写一套入口。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-handle-head-request
//! ```

// 引入 IntoResponse 和 Response；前者用于转换，后者是最终响应类型。
use axum::response::{IntoResponse, Response};
// 引入 http 模块、GET 路由函数和 Router。
use axum::{http, routing::get, Router};

// 单独构造 app，方便 main 和测试复用。
fn app() -> Router {
    // GET /get-head 交给 get_head_handler。
    Router::new().route("/get-head", get(get_head_handler))
}

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 绑定本机 3000 端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 打印监听地址。
    println!("listening on {}", listener.local_addr().unwrap());

    // 启动服务，Router 来自 app()。
    axum::serve(listener, app()).await;
}

// GET 路由也会被 HEAD 请求调用，但 HEAD 响应体会被移除。
// 通过提取 http::Method，可以显式判断当前请求方法。
async fn get_head_handler(method: http::Method) -> Response {
    // 如果是 HEAD，就提前返回 header，避免生成昂贵 body。
    if method == http::Method::HEAD {
        return ([("x-some-header", "header from HEAD")]).into_response();
    }

    // GET 请求才执行模拟计算。
    do_some_computing_task();

    // GET 返回 header 和 body，并转换成 Response。
    ([("x-some-header", "header from GET")], "body from GET").into_response()
}

// 模拟生成响应 body 前的昂贵计算。
fn do_some_computing_task() {
    // TODO
}
````

## 运行和验证

运行前先确认 `examples/handle-head-request/Cargo.toml` 里的 `axum = { path = "../../axum" }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-handle-head-request
````

测试 GET：

````bash
curl -i http://127.0.0.1:3000/get-head
````

预期重点：

````text
HTTP/1.1 200 OK
x-some-header: header from GET

body from GET
````

测试 HEAD：

````bash
curl -I http://127.0.0.1:3000/get-head
````

预期重点：

````text
HTTP/1.1 200 OK
x-some-header: header from HEAD
````

`curl -I` 只显示响应头。HEAD 响应本来也不应该有 body，所以看不到 `body from GET` 是正确行为。

运行测试：

````bash
cd examples
cargo test -p example-handle-head-request
````

## 手写任务

完成本章代码后，试着做三个小观察：

1. 把 GET 返回的 body 改成更长的文本，例如 `this is a long body from GET`。
2. 再运行 `curl -i http://127.0.0.1:3000/get-head`，确认 GET 能看到 body。
3. 再运行 `curl -I http://127.0.0.1:3000/get-head`，确认 HEAD 仍然只看到响应头。

你要得到的结论是：

```text
HEAD 关心响应头。
GET 关心响应头和响应体。
```

## 本章真正要记住什么

- HTTP `HEAD` 要返回响应头，但不返回 body。
- Axum 的 GET 路由可以隐式处理 HEAD。
- 如果生成 body 成本很高，可以提取 `http::Method` 并特殊处理 HEAD。
- GET 和 HEAD 差异很大时，也可以显式注册 HEAD 路由。
- 把 `app()` 单独提出来，可以让 main 和测试复用同一个 Router。
- `ServiceExt::oneshot` 可以不启动端口，直接测试 Router。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/handle-head-request/Cargo.toml`
- `examples/handle-head-request/src/main.rs`
