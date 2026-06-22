# 20. anyhow-error-response

对应示例：`examples/anyhow-error-response`

本章目标：用 `anyhow` 包装内部错误，并通过自定义 `AppError` 接入 Axum 的响应系统。

上一章设计了比较完整的 `AppError`。  
这一章展示一个更小、更常见的技巧：

```text
业务函数返回 anyhow::Error
handler 返回 Result<_, AppError>
用 From 自动把 anyhow::Error 转成 AppError
AppError 实现 IntoResponse
```

## 这个小项目在做什么

应用只有一个接口：

```text
GET /
```

handler 会调用一个必定失败的函数：

````rust
try_thing()?;
````

`try_thing()` 返回 `anyhow::Error`。  
handler 返回 `Result<(), AppError>`。  
中间靠 `From` 实现自动转换。

请求主线是：

```text
GET /
-> handler
-> try_thing() 返回 anyhow::Error
-> ? 自动转成 AppError
-> AppError::into_response
-> 500 Something went wrong: it failed!
```

## 先理解 anyhow 适合做什么

`anyhow::Error` 适合在应用内部快速包装错误。  
它的优势是：

- 可以装下很多不同来源的错误。
- 方便给错误加上下文。
- 适合应用层，不适合公共库 API 暴露给调用方。

但 Axum handler 不能直接随便返回 `anyhow::Error`。  
Axum 需要知道：

```text
这个错误应该变成什么 HTTP 状态码？
响应体是什么？
```

所以本章用 `AppError` 包一层。

## 文件和依赖

这个 example 有两个文件：

1. `examples/anyhow-error-response/Cargo.toml`：声明 `anyhow`、Axum、Tokio 和测试依赖。
2. `examples/anyhow-error-response/src/main.rs`：实现 handler、`AppError`、错误转换和测试。

关键依赖：

- `anyhow`：应用内部错误包装。
- `axum`：提供 `IntoResponse`、`Response`、`Router`。
- `tokio`：异步运行时和测试。
- `tower`、`http-body-util`：测试里直接调用 Router 并读取响应体。

## 第一步：handler 返回 Result

源码：

````rust
async fn handler() -> Result<(), AppError> {
    try_thing()?;
    Ok(())
}
````

handler 的成功类型是 `()`，失败类型是 `AppError`。

`try_thing()?` 表示：

```text
如果 try_thing 成功，继续往下
如果 try_thing 失败，把错误转换后提前返回
```

关键问题是：

```text
try_thing 返回 anyhow::Error
handler 要求 AppError
为什么 ? 能工作？
```

答案在后面的 `From` 实现。

## 第二步：业务函数返回 anyhow::Error

源码：

````rust
fn try_thing() -> Result<(), anyhow::Error> {
    anyhow::bail!("it failed!")
}
````

`anyhow::bail!` 是一个方便宏，相当于：

```text
立刻返回 Err(anyhow::Error)
```

这里故意让它失败，是为了演示错误如何变成 HTTP 响应。

## 第三步：定义 AppError 包装 anyhow::Error

源码：

````rust
struct AppError(anyhow::Error);
````

这是一个 tuple struct。  
它只做一件事：

```text
把 anyhow::Error 包起来
```

为什么不直接给 `anyhow::Error` 实现 `IntoResponse`？

因为 Rust 的 **orphan rule**（孤儿规则）不允许你给"外部 crate 的类型"实现"外部 crate 的 trait"。`anyhow::Error` 来自 anyhow，`IntoResponse` 来自 axum，两者都不是你当前 crate 的，所以编译器会拒绝：

```text
error[E0117]: only traits defined in the current crate can be implemented
for types defined outside of the crate
```

这条规则的初衷是**保证 trait 实现的唯一性**。如果没有这个限制，两个不同的 crate 可能都给 `anyhow::Error` 实现 `IntoResponse`，编译器就不知道用哪一个了。

绕开 orphan rule 有两种典型做法：

1. **newtype 包装**（本章做法）：定义自己的类型 `struct AppError(anyhow::Error)`，它是当前 crate 的类型，给它实现 `IntoResponse` 就合法了。
2. **在自己的 crate 里重新定义 trait**：比如定义一个 `MyIntoResponse`，给它实现也合法，但需要到处用新 trait，不太实际。

newtype 是 Rust 生态里最常见的解法，记住它就够了。

## 第四步：让 AppError 变成 HTTP 响应

源码：

````rust
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            format!("Something went wrong: {}", self.0),
        )
            .into_response()
    }
}
````

这里把所有 `AppError` 都转成：

```text
500 Internal Server Error
Something went wrong: 原始错误信息
```

注意：示例直接把内部错误信息返回给客户端。  
真实生产环境通常不建议这么做，更常见的是：

```text
客户端：Something went wrong
服务端日志：记录完整 anyhow error
```

上一章的 `error-handling` 就展示了更完整的生产型模式。

## 第五步：实现 From，让 ? 自动转换

源码：

````rust
impl<E> From<E> for AppError
where
    E: Into<anyhow::Error>,
{
    fn from(err: E) -> Self {
        Self(err.into())
    }
}
````

这段是本章最关键的技巧。

它表示：

```text
任何能 Into<anyhow::Error> 的错误
都可以转换成 AppError
```

所以当 handler 里写：

````rust
try_thing()?;
````

编译器会做：

```text
anyhow::Error -> AppError
```

这样你就不用每次手动写：

````rust
try_thing().map_err(AppError)?;
````

### ⚠️ blanket impl 的副作用：它会"吃掉"所有错误

这个 `impl<E> From<E> for AppError where E: Into<anyhow::Error>` 是一个 **blanket implementation**（覆盖式实现）——它对**所有**能转成 `anyhow::Error` 的类型都生效。

这有个重要副作用：以后你**很难再给 `AppError` 单独处理某类错误**了。比如你想像第 19 章那样，区分 `JsonRejection`（返回 400）和 `anyhow::Error`（返回 500）：

````rust
// ❌ 这会和 blanket impl 冲突！
impl From<JsonRejection> for AppError {
    fn from(r: JsonRejection) -> Self { ... }
}
````

编译器会报错：`conflicting implementations`。因为 `JsonRejection` 也满足 `Into<anyhow::Error>`，已经被上面的 blanket impl 覆盖了。

所以**这个写法只适合"所有错误都统一处理成 500"的简单场景**。如果你需要区分错误类型、返回不同状态码，就别用 blanket impl，改用手写的 `From` 逐个实现（像第 19 章那样）。

记住这条取舍：

```text
想要简单（统一 500）        → blanket impl（本章）
想要精细控制（区分 400/500）→ 手写多个 From（第 19 章）
```

另外补充一句：`E: Into<anyhow::Error>` 这个 bound 看起来抽象，但它等价于 `E: std::error::Error + Send + Sync + 'static`——也就是"任何标准的 Rust 错误类型"。anyhow 为这类类型提供了一个 blanket impl 转成 `anyhow::Error`，这里只是借用了它。

## 第六步：测试错误响应

测试里直接调用 Router：

````rust
let response = app()
    .oneshot(Request::get("/").body(Body::empty()).unwrap())
    .await
    .unwrap();
````

然后验证：

- 状态码是 `500 Internal Server Error`。
- 响应体是 `Something went wrong: it failed!`。

这说明 `anyhow::Error` 已经成功经过 `AppError` 转成了 Axum 响应。

## 函数职责速查

- `main`：调用 `app()`，绑定端口并启动服务。
- `app`：注册 `GET /`。
- `handler`：调用可能失败的业务函数，用 `?` 传播错误。
- `try_thing`：模拟返回 `anyhow::Error` 的业务函数。
- `AppError`：包装 `anyhow::Error` 的本地错误类型。
- `IntoResponse for AppError`：把错误转成 HTTP 500 响应。
- `From<E> for AppError`：让 `?` 自动把错误转换成 `AppError`。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-anyhow-error-response
//! ```

// 引入状态码、响应转换、GET 路由和 Router。
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::get,
    Router,
};

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 构造应用路由。
    let app = app();

    // 绑定本机 3000 端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 输出监听地址。
    println!("listening on {}", listener.local_addr().unwrap());

    // 启动 HTTP 服务。
    axum::serve(listener, app).await;
}

// handler 返回 Result，失败类型是 AppError。
async fn handler() -> Result<(), AppError> {
    // try_thing 返回 anyhow::Error；? 会自动转成 AppError。
    try_thing()?;
    Ok(())
}

// 模拟一个会失败的业务函数。
fn try_thing() -> Result<(), anyhow::Error> {
    anyhow::bail!("it failed!")
}

// 自定义错误类型，包装 anyhow::Error。
struct AppError(anyhow::Error);

// 告诉 Axum 如何把 AppError 转成 HTTP 响应。
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            format!("Something went wrong: {}", self.0),
        )
            .into_response()
    }
}

// 构造 Router，方便 main 和测试复用。
fn app() -> Router {
    Router::new().route("/", get(handler))
}

// 让任何能 Into<anyhow::Error> 的错误都能转成 AppError。
// 这样 handler 里就可以直接使用 ?。
impl<E> From<E> for AppError
where
    E: Into<anyhow::Error>,
{
    fn from(err: E) -> Self {
        Self(err.into())
    }
}
````

## 运行和验证

运行前先确认 `examples/anyhow-error-response/Cargo.toml` 里的 `axum = { path = "../../axum" }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-anyhow-error-response
````

请求首页：

````bash
curl -i http://127.0.0.1:3000/
````

预期状态码：

````text
HTTP/1.1 500 Internal Server Error
````

预期响应体：

````text
Something went wrong: it failed!
````

运行测试：

````bash
cd examples
cargo test -p example-anyhow-error-response
````

常见卡点：

- `anyhow::Error` 是内部错误包装，不会自动变成 HTTP 响应。
- 需要用本地类型 `AppError` 包一层，再实现 `IntoResponse`。
- `From<E> for AppError` 是让 `?` 工作的关键。
- 生产环境通常不要把内部错误细节直接返回给客户端。

## 手写任务

完成本章代码后，试着做三个小改动：

1. 把响应体改成固定文本 `Something went wrong`，不要暴露 `it failed!`。
2. 在 `try_thing` 里改用 `anyhow::anyhow!("database failed")` 返回错误。
3. 增加一个成功分支，让某个 query 参数存在时返回 `Ok(())`。

## 本章真正要记住什么

- `anyhow` 适合应用内部快速包装错误。
- Axum 需要错误类型实现 `IntoResponse` 才能返回给客户端。
- 用本地 `AppError(anyhow::Error)` 可以绕开 orphan rule。
- 实现 `From<E> for AppError` 后，handler 里可以自然使用 `?`。
- 简单项目可以这样写，生产项目应更谨慎地区分公开错误和内部错误。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/anyhow-error-response/Cargo.toml`
- `examples/anyhow-error-response/src/main.rs`
