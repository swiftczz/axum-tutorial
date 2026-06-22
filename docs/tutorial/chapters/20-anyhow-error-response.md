# 20. anyhow-error-response

对应示例：`examples/anyhow-error-response`

第 19 章用 `thiserror` 定义强类型错误枚举。这章用 **`anyhow`**——通用错误类型，适合"错误种类太多懒得枚举"的场景。问题是 anyhow 的 `Error` 不实现 `IntoResponse`，需要写个 wrapper 把它接进 axum。

分 2 步：先写 `AppError(anyhow::Error)` + `IntoResponse` impl，再用 blanket `From<E>` 让 `?` 自动转换。

相比前面章节新引入：**`anyhow` crate（动态错误聚合）、`AppError` wrapper 模式、blanket `impl From<E> for AppError` 让 `?` 自动转换**。

## Cargo.toml

````toml
[package]
name = "example-anyhow-error-response"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
anyhow = "1"
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：`AppError` 包装 anyhow + `IntoResponse`

写 `AppError(anyhow::Error)` 新类型，实现 `IntoResponse` 把它转成 500 响应。

````rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::get,
    Router,
};

// 包装 anyhow::Error 的新类型
struct AppError(anyhow::Error);

// 让 axum 知道怎么把 AppError 转成响应
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            format!("Something went wrong: {}", self.0),
        )
            .into_response()
    }
}

# fn app() -> Router { Router::new().route("/", get(handler)) }
# async fn handler() -> Result<(), AppError> { /* 下一步填 */ Ok(()) }
````

> **新面孔：`anyhow::Error`**
>
> anyhow 是动态错误类型——可以装任何实现了 `std::error::Error` 的具体错误。适合"我不在乎具体错误类型，只想用 `?` 往上抛"。
>
> ```rust
> fn try_thing() -> Result<(), anyhow::Error> {
>     let x: i32 = "abc".parse()?;     // ParseIntError 自动转 anyhow
>     let file = std::fs::read("x")?;  // io::Error 自动转 anyhow
>     Ok(())
> }
> ```
>
> 多种不同错误都能 `?` 进 anyhow——不用每个写 `From` impl。

> **新面孔：`AppError(anyhow::Error)` 新类型模式**
>
> anyhow 的 `Error` **不实现 `IntoResponse`**——axum 不能直接当响应返回。包一层 `AppError(anyhow::Error)` 新类型，给它实现 `IntoResponse`，就能当 handler 错误返回。
>
> 为什么不直接 `impl IntoResponse for anyhow::Error`？因为 anyhow 是外部 crate，给它加 trait impl 违反孤儿规则（orphan rule）。新类型绕过这个限制。

---

## 第二步：blanket `From` 让 `?` 自动转换

handler 用 `try_thing()?` 调返回 `anyhow::Error` 的函数，需要自动转成 `AppError`。加一个 blanket `impl From<E> for AppError`。

````rust
# struct AppError(anyhow::Error);

// blanket impl：任何能转 anyhow::Error 的类型都能转 AppError
// 这让 Result<_, anyhow::Error> 上的 ? 自动转成 AppError
impl<E> From<E> for AppError
where
    E: Into<anyhow::Error>,
{
    fn from(err: E) -> Self {
        Self(err.into())
    }
}

async fn handler() -> Result<(), AppError> {
    // try_thing 返 Result<(), anyhow::Error>
    // ? 自动把 anyhow::Error 转成 AppError（via 上面的 From impl）
    try_thing()?;
    Ok(())
}

fn try_thing() -> Result<(), anyhow::Error> {
    anyhow::bail!("it failed!");
}

#[tokio::main]
async fn main() {
    let app = app();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}

# fn app() -> Router { Router::new().route("/", get(handler)) }
````

验证：

````bash
cd examples
cargo run -p example-anyhow-error-response

curl -i http://127.0.0.1:3000/
# HTTP/1.1 500 Internal Server Error
# Something went wrong: it failed!
````

> **新面孔：blanket `impl From<E> for AppError where E: Into<anyhow::Error>`**
>
> 这是关键技巧。`From` impl 覆盖**所有能转 anyhow::Error 的类型**——包括 `anyhow::Error` 自己、`io::Error`、`ParseIntError`、`reqwest::Error` 等任意 `std::error::Error`。
>
> `?` 操作符的行为：`expr?` 等价于 `match expr { Ok(v) => v, Err(e) => return Err(E::from(e)) }`。所以只要 `From<E> for AppError` 存在，`?` 自动转换。

> **新面孔：`anyhow::bail!` 宏**
>
> `anyhow::bail!("msg")` 等价于 `return Err(anyhow::anyhow!("msg"))`——快速返回错误。`anyhow::anyhow!("msg")` 用 `format!` 语法构造错误。

> **新面孔：anyhow vs thiserror（参考第 19 章）**
>
> | 维度 | anyhow | thiserror（ch19） |
> | --- | --- | --- |
> | 错误类型 | 动态（装任意错误） | 静态（枚举变体） |
> | 适合 | 应用层、错误种类多 | 库、错误种类明确 |
> | 类型安全 | 弱（运行时才知道） | 强（编译期匹配） |
> | `match` 错误 | 不行（要做 downcast） | 直接 match 变体 |
>
> 应用层用 anyhow 简单；库用 thiserror 给用户清晰 API。

---

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

// Make our own error that wraps `anyhow::Error`.
struct AppError(anyhow::Error);

// Tell axum how to convert `AppError` into a response.
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

// This enables using `?` on functions that return `Result<_, anyhow::Error>` to turn them into
// `Result<_, AppError>`. That way you don't need to do that manually.
impl<E> From<E> for AppError
where
    E: Into<anyhow::Error>,
{
    fn from(err: E) -> Self {
        Self(err.into())
    }
}

#[cfg(test)]
mod tests;
````

## 运行

````bash
cd examples
cargo run -p example-anyhow-error-response

curl -i http://127.0.0.1:3000/
# HTTP/1.1 500 Internal Server Error
# Something went wrong: it failed!
````

## 解读

### 三种错误处理方式对比

```text
方式 1：字符串错误
    handler 返回 Result<_, String>
    简单但丢失错误类型信息

方式 2：thiserror（ch19）
    enum AppError { NotFound, BadRequest(String), Io(io::Error) }
    强类型，但每加一种错误要改 enum

方式 3：anyhow（这章）
    handler 返回 Result<_, AppError>（AppError 包装 anyhow::Error）
    动态，任意错误 ? 进来，简单
```

应用层选 3，库选 2。

### 生产改进

这章所有错误都返回 500。生产环境通常要区分：

- 4xx（客户端错误）：参数错、未授权、找不到
- 5xx（服务端错误）：数据库挂了、内部 bug

anyhow 不直接支持区分——可以扩展 `AppError` 加 status code：

````rust
struct AppError {
    inner: anyhow::Error,
    status: StatusCode,
}
````

但这就开始往 thiserror 靠了，不如直接用 thiserror（ch19）。

## 常见问题

**为什么 anyhow 不能直接 `IntoResponse`？** 孤儿规则——`anyhow::Error` 和 `IntoResponse` 都不在我们 crate 里，不能给它加 impl。新类型 `AppError` 在我们 crate 里，可以加 impl。

**blanket `From` impl 会不会太宽泛？** 不会——`E: Into<anyhow::Error>` 约束保证只有真正能转 anyhow 的类型才走这个 impl。Rust 编译器保证不冲突。

**anyhow 错误怎么打印 stacktrace？** `format!("{:?}", err)` 打印 backtrace（如果启用 `RUST_BACKTRACE=1`）。anyhow 自动捕获调用栈。

## 手写任务

1. 把 `try_thing` 改成真的可能成功（随机成功/失败），handler 成功返回 "ok"。
2. 改错误响应成 JSON 格式：`{"error": "it failed!"}`。
3. 加 tracing log：`into_response` 里 `tracing::error!(?self.0)`。
4. 对比 ch19 thiserror 实现，理解两种方式的取舍。

## 小结

这章用 2 步讲了 anyhow 错误接入 axum：

1. **`AppError(anyhow::Error)` 新类型 + `IntoResponse`**：包装 anyhow，实现 trait 接入 axum。
2. **blanket `From<E> for AppError`**：让任何 `Result<_, anyhow::Error>` 上的 `?` 自动转 AppError。

核心模式：**外部类型 + 新类型 wrapper + IntoResponse impl**——任何不能直接实现 `IntoResponse` 的错误类型都能这么接入。anyhow 适合应用层（错误种类多），thiserror 适合库（强类型）。

## 源码对照

- `examples/anyhow-error-response/Cargo.toml`
- `examples/anyhow-error-response/src/main.rs`
- `examples/anyhow-error-response/tests/tests.rs`
