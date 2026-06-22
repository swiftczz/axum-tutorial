# 20. anyhow-error-response

对应示例：`examples/anyhow-error-response`

上一章设计了完整的 `AppError`,这章展示一个更小、更常见的技巧:业务函数返回 `anyhow::Error`,handler 返回 `Result<_, AppError>`,用 `From` 自动转换,`AppError` 实现 `IntoResponse`。



相比前面章节新引入：**`anyhow::Error` newtype 包装、blanket `impl From`、orphan rule**。

## Cargo.toml

````toml
[package]
name = "example-anyhow-error-response"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
anyhow = "1.0"
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }

[dev-dependencies]
http-body-util = "0.1.0"
tower = { version = "0.5.2", features = ["util"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 完整代码

````rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::get,
    Router,
};

#[tokio::main]
async fn main() {
    let app = app();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    println!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app).await;
}

async fn handler() -> Result<(), AppError> {
    try_thing()?;
    Ok(())
}

fn try_thing() -> Result<(), anyhow::Error> {
    anyhow::bail!("it failed!")
}

struct AppError(anyhow::Error);

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            format!("Something went wrong: {}", self.0),
        )
            .into_response()
    }
}

fn app() -> Router {
    Router::new().route("/", get(handler))
}

impl<E> From<E> for AppError
where
    E: Into<anyhow::Error>,
{
    fn from(err: E) -> Self {
        Self(err.into())
    }
}
````

## 运行

````bash
cd examples
cargo run -p example-anyhow-error-response
````

请求首页:

````bash
curl -i http://127.0.0.1:3000/
````

预期:

````text
HTTP/1.1 500 Internal Server Error

Something went wrong: it failed!
````

运行测试:

````bash
cargo test -p example-anyhow-error-response
````

## 解读

### anyhow 适合做什么

`anyhow::Error` 适合在应用内部快速包装错误:能装下很多不同来源的错误,方便加上下文,适合应用层但不适合公共库 API 暴露给调用方。

但 axum handler 不能直接返回 `anyhow::Error`——axum 需要知道"这个错误应该变成什么 HTTP 状态码、响应体是什么"。所以用 `AppError` 包一层。

### 为什么用 newtype `AppError(anyhow::Error)`

不能直接给 `anyhow::Error` 实现 `IntoResponse`。Rust 的 **orphan rule**(孤儿规则)不允许给"外部 crate 的类型"实现"外部 crate 的 trait":`anyhow::Error` 来自 anyhow,`IntoResponse` 来自 axum,都不是当前 crate 的,编译器会拒绝:

```text
error[E0117]: only traits defined in the current crate can be implemented
for types defined outside of the crate
```

这条规则保证 trait 实现的唯一性——否则两个 crate 都给 `anyhow::Error` 实现 `IntoResponse`,编译器就不知道用哪个。

绕开 orphan rule 的常见做法是 **newtype**:定义本地类型 `struct AppError(anyhow::Error)`,它是当前 crate 的类型,给它实现 `IntoResponse` 就合法了。这是 Rust 生态最常见的解法。

### `IntoResponse for AppError`

````rust
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            format!("Something went wrong: {}", self.0),
        ).into_response()
    }
}
````

所有 `AppError` 都转成 500 + 错误信息。⚠️ example 直接把内部错误返回给客户端,**生产环境不建议这么做**——更常见的是客户端只看通用消息,完整错误记服务端日志(见第 19 章)。

### `From<E> for AppError` 让 `?` 工作(关键技巧)

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

任何能 `Into<anyhow::Error>` 的错误都能转成 `AppError`。所以 handler 里 `try_thing()?` 自动把 `anyhow::Error` 转成 `AppError`,不用每次写 `try_thing().map_err(AppError)?`。

`E: Into<anyhow::Error>` 看起来抽象,等价于 `E: std::error::Error + Send + Sync + 'static`(任何标准 Rust 错误类型)。

### ⚠️ blanket impl 的副作用

这是个 **blanket implementation**(覆盖式实现),对**所有**能转成 `anyhow::Error` 的类型生效。副作用:以后**很难再单独处理某类错误**。比如想像第 19 章那样区分 `JsonRejection`(400)和 `anyhow::Error`(500):

````rust
// ❌ 和 blanket impl 冲突!
impl From<JsonRejection> for AppError { ... }
````

编译器报 `conflicting implementations`——`JsonRejection` 也满足 `Into<anyhow::Error>`,已被 blanket impl 覆盖。取舍:

```text
想要简单(统一 500)         → blanket impl(本章)
想要精细控制(区分 400/500)→ 手写多个 From(第 19 章)
```

需要区分错误类型返回不同状态码时,别用 blanket impl,改用手写 `From` 逐个实现。

## 手写任务

跑通后做三个小改动:

1. 把响应体改成固定文本 `Something went wrong`,不暴露 `it failed!`。
2. 在 `try_thing` 里改用 `anyhow::anyhow!("database failed")` 返回错误。
3. 增加一个成功分支,让某个 query 参数存在时返回 `Ok(())`。

## 小结

- `anyhow` 适合应用内部快速包装错误,但 axum handler 不能直接返回它。
- 用 newtype `AppError(anyhow::Error)` 绕开 orphan rule,给本地类型实现 `IntoResponse`。
- `impl From<E> for AppError where E: Into<anyhow::Error>` 是 blanket impl,让 `?` 自动转换,但会"吃掉"所有错误——只适合统一 500 的简单场景。
- 需要区分错误类型/状态码时,手写多个 `From`(第 19 章)。
- 生产环境不要把内部错误细节直接返回客户端。

## 源码对照

- `examples/anyhow-error-response/Cargo.toml`
- `examples/anyhow-error-response/src/main.rs`
