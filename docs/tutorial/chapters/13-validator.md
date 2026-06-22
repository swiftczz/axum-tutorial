# 13. validator

对应示例：`examples/validator`

本章目标：把提取和参数校验组合成 `ValidatedForm<T>`，理解如何在 handler 之前统一校验输入。

前面已经学过：

- `Form<T>` 可以把表单或 query 参数提取成结构体。
- 自定义 extractor 可以包装已有 extractor。
- extractor 失败时可以返回自己的错误响应。

这一章把这些能力组合起来：

```text
先用 Form<T> 提取输入
再用 validator 校验字段
校验通过才进入 handler
校验失败返回 400
```

## 这个小项目在做什么

这个 example 只有一个接口：

```text
GET /?name=LT
```

如果 `name` 合法，返回：

```html
<h1>Hello, LT!</h1>
```

如果 `name` 缺失、为空或太短，返回 `400 Bad Request` 和错误文本。

请求主线是：

```text
GET /?name=LT
-> ValidatedForm<NameInput>
-> Form<NameInput> 提取 name
-> name.validate()
-> handler 返回 HTML
```

失败主线是：

```text
GET /?name=
-> Form<NameInput> 提取成功
-> validate 失败
-> ServerError::ValidationError
-> 400 响应
```

## 先理解“提取”和“校验”的区别

提取解决的是：

```text
请求里有没有这个字段？
字段能不能变成 Rust 类型？
```

校验解决的是：

```text
字段虽然能解析，但业务上是否合法？
```

例子：

| 请求 | 提取是否成功 | 校验是否成功 | 原因 |
| --- | --- | --- | --- |
| `/?name=LT` | 成功 | 成功 | name 长度满足要求 |
| `/?name=` | 成功 | 失败 | name 是空字符串 |
| `/?name=X` | 成功 | 失败 | name 太短 |
| `/` | 失败 | 不会进入校验 | 缺少 name 字段 |

所以这一章会同时处理两类错误：

- `FormRejection`：提取失败。
- `ValidationErrors`：校验失败。

## 文件和依赖

这个 example 有两个文件：

1. `examples/validator/Cargo.toml`：声明 Axum、Serde、thiserror、validator 和测试依赖。
2. `examples/validator/src/main.rs`：实现 Router、输入结构体、自定义 extractor、错误响应和测试。

关键依赖：

- `axum`：提供 `Form`、`FromRequest`、`Html`、`IntoResponse`。
- `serde`：把请求参数反序列化成 Rust struct。
- `validator`：提供 `Validate` trait 和校验宏。
- `thiserror`：减少错误类型转换样板代码。
- `tokio`、`tracing`、`tracing-subscriber`：运行时和日志。
- `tower`、`http-body-util`：测试里直接调用 Router 和读取响应体。

## 第一步：把 Router 提成 app 函数

源码：

````rust
fn app() -> Router {
    Router::new().route("/", get(handler))
}
````

这和前面测试章节的思路一样：

```text
main 用 app() 启动服务
测试用 app() 直接发请求
```

这里路由是：

```text
GET / -> handler
```

请求参数放在 query string 里，例如：

```text
/?name=LT
```

### 为什么这里用 `Form<T>` 而不是 `Query<T>`？

你可能会问：参数明明在 query string 里，为什么不直接用 `axum::extract::Query<T>`？

这其实是 Axum 的一个反直觉特性：**`Form<T>` 对 GET/HEAD 请求会从 query string 提取**，对其他方法（POST/PUT...）才从请求 body 提取。所以对 GET 请求来说，`Form<T>` 和 `Query<T>` 行为几乎一样。

两者有个小差别：`Form<T>` 用 `serde_urlencoded` 反序列化，`Query<T>` 也是；但 `Query<T>` 不会消费 body，而 `Form<T>` 在 POST 时会。本例是 GET，所以无所谓。

**那 example 为什么选 `Form<T>`？** 因为 `ValidatedForm<T>` 这个名字暗示了"它既能用于 GET query，也能用于 POST 表单"。如果你想明确只处理 query，用 `Query<T>` 更清晰；如果想做一个同时支持 query 和表单的校验器，用 `Form<T>` 更通用。这里只是 example 的选择，不是唯一正确做法。

记住这条规则就好：

```text
GET/HEAD  + Form<T>  → 从 query string 提取
POST/其他 + Form<T>  → 从请求 body 提取
任何方法 + Query<T>  → 从 query string 提取
```

## 第二步：定义带校验规则的输入结构体

源码：

````rust
#[derive(Debug, Deserialize, Validate)]
pub struct NameInput {
    #[validate(length(min = 2, message = "Can not be empty"))]
    pub name: String,
}
````

逐个看：

- `Deserialize`：让 `Form<NameInput>` 能从请求里解析出 `name`。
- `Validate`：让 `NameInput` 拥有 `.validate()` 方法。
- `#[validate(...)]`：给字段加校验规则。

这里的规则是：

```text
name 长度至少为 2
错误消息是 Can not be empty
```

### ⚠️ 一个反模式：错误消息和校验规则不匹配

注意看，`min = 2` 表示"长度至少为 2"，但 message 却写的是 "Can not be empty"（不能为空）。这会误导用户：

- 用户传 `?name=X`（长度 1，不是空），收到的错误却是"不能为空"，会以为是 bug。
- 真正的"不能为空"对应的是 `name` 字段缺失，而那属于**提取失败**（`FormRejection`），走的是另一条错误路径。

这是官方 example 留下的一个小瑕疵。**真实项目里，错误消息应该准确描述校验规则**，例如：

````rust
#[validate(length(min = 2, message = "name must be at least 2 characters"))]
````

养成习惯：校验规则变了，message 也要跟着变。否则用户（和未来的你）会被错误消息带偏。

## 第三步：handler 使用 ValidatedForm

源码：

````rust
async fn handler(ValidatedForm(input): ValidatedForm<NameInput>) -> Html<String> {
    Html(format!("<h1>Hello, {}!</h1>", input.name))
}
````

handler 不直接写 `Form<NameInput>`，而是写：

```text
ValidatedForm<NameInput>
```

这说明 handler 只接收“已经提取并通过校验”的输入。

如果提取失败或校验失败，handler 不会执行。

这个设计的好处是：

```text
handler 里不用再写 if name.is_empty()
校验逻辑统一放到 extractor
```

## 第四步：定义 ValidatedForm 包装器

源码：

````rust
#[derive(Debug, Clone, Copy, Default)]
pub struct ValidatedForm<T>(pub T);
````

这仍然是前几章反复出现的模式：

```text
创建一个包装类型
让它实现 FromRequest
在 from_request 里复用已有 extractor
```

`ValidatedForm<T>` 的含义是：

```text
已经经过 Form<T> 提取
并且已经通过 Validate 校验的 T
```

## 第五步：实现 FromRequest

核心源码：

````rust
impl<T, S> FromRequest<S> for ValidatedForm<T>
where
    T: DeserializeOwned + Validate,
    S: Send + Sync,
    Form<T>: FromRequest<S, Rejection = FormRejection>,
{
    type Rejection = ServerError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Form(value) = Form::<T>::from_request(req, state).await?;
        value.validate()?;
        Ok(ValidatedForm(value))
    }
}
````

这段做了两步：

1. `Form::<T>::from_request(req, state).await?`：先提取请求参数。
2. `value.validate()?`：再运行校验规则。

泛型约束说明：

- `T: DeserializeOwned`：T 要能从请求参数反序列化出来。
- `T: Validate`：T 要能调用 `.validate()`。
- `Form<T>: FromRequest<S, Rejection = FormRejection>`：内部复用 Axum 的 `Form<T>` 提取器。

这里的两个 `?` 会把错误转成 `ServerError`。  
这是因为 `ServerError` 实现了从这两种错误来的转换。

## 第六步：统一错误类型 ServerError

源码：

````rust
#[derive(Debug, Error)]
pub enum ServerError {
    #[error(transparent)]
    ValidationError(#[from] validator::ValidationErrors),

    #[error(transparent)]
    AxumFormRejection(#[from] FormRejection),
}
````

这个错误类型能接住两类失败：

- `ValidationError`：字段校验失败。
- `AxumFormRejection`：表单/query 提取失败。

`#[from]` 会生成对应的 `From` 实现，所以 `?` 可以自动转换错误类型。

## 第七步：把错误转成 HTTP 响应

源码：

````rust
impl IntoResponse for ServerError {
    fn into_response(self) -> Response {
        match self {
            ServerError::ValidationError(_) => {
                let message = format!("Input validation error: [{self}]").replace('\n', ", ");
                (StatusCode::BAD_REQUEST, message)
            }
            ServerError::AxumFormRejection(_) => (StatusCode::BAD_REQUEST, self.to_string()),
        }
        .into_response()
    }
}
````

不管是提取失败还是校验失败，都返回 400。

区别是：

- 校验失败：统一加上 `Input validation error: [...]`。
- 表单提取失败：直接返回 FormRejection 的错误文本。

## 第八步：理解测试覆盖了什么

测试分四种情况：

| 测试 | 请求 | 预期 |
| --- | --- | --- |
| `test_no_param` | `/` | 400，缺少 `name` |
| `test_with_param_without_value` | `/?name=` | 400，校验失败 |
| `test_with_param_with_short_value` | `/?name=X` | 400，校验失败 |
| `test_with_param_and_value` | `/?name=LT` | 200，返回 HTML |

这很好地覆盖了本章核心：

```text
提取失败
校验失败
校验成功
```

## 函数职责速查

- `main`：初始化日志，调用 `app()`，绑定端口并启动服务。
- `app`：构造 Router，注册 `GET /`。
- `NameInput`：描述输入字段和校验规则。
- `handler`：接收已经校验过的输入，返回问候 HTML。
- `ValidatedForm<T>`：把 `Form<T>` 和 `Validate` 组合成一个 extractor。
- `from_request`：先提取，再校验。
- `ServerError`：统一表示提取失败和校验失败。
- `IntoResponse for ServerError`：把错误转成 400 响应。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-validator
//! ```

// 引入 Form、FromRequest、Request、状态码、Html、响应转换、GET 路由和 Router。
use axum::{
    extract::{rejection::FormRejection, Form, FromRequest, Request},
    http::StatusCode,
    response::{Html, IntoResponse, Response},
    routing::get,
    Router,
};
// serde 用来反序列化请求参数。
use serde::{de::DeserializeOwned, Deserialize};
// thiserror 用来减少错误类型样板代码。
use thiserror::Error;
// TcpListener 用来绑定端口。
use tokio::net::TcpListener;
// tracing_subscriber 用来初始化日志。
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};
// validator 提供 Validate trait 和校验宏。
use validator::Validate;

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 初始化日志系统。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 构造应用路由。
    let app = app();

    // 绑定本机 3000 端口。
    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动 HTTP 服务。
    axum::serve(listener, app).await;
}

// 构造 Router，方便 main 和测试复用。
fn app() -> Router {
    Router::new().route("/", get(handler))
}

// 输入结构体：从请求参数里反序列化，并带有校验规则。
#[derive(Debug, Deserialize, Validate)]
pub struct NameInput {
    // name 长度至少为 2，否则返回自定义消息。
    #[validate(length(min = 2, message = "Can not be empty"))]
    pub name: String,
}

// handler 只接收已经提取并校验通过的 NameInput。
async fn handler(ValidatedForm(input): ValidatedForm<NameInput>) -> Html<String> {
    Html(format!("<h1>Hello, {}!</h1>", input.name))
}

// 自定义 extractor 包装器。
#[derive(Debug, Clone, Copy, Default)]
pub struct ValidatedForm<T>(pub T);

// 让 ValidatedForm<T> 成为 extractor。
impl<T, S> FromRequest<S> for ValidatedForm<T>
where
    // T 要能从请求参数反序列化。
    T: DeserializeOwned + Validate,
    // state 要能安全用于异步环境。
    S: Send + Sync,
    // 内部复用 Axum 的 Form<T>，失败类型是 FormRejection。
    Form<T>: FromRequest<S, Rejection = FormRejection>,
{
    // 提取或校验失败时返回 ServerError。
    type Rejection = ServerError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        // 先用 Form<T> 提取请求参数。
        let Form(value) = Form::<T>::from_request(req, state).await?;
        // 再运行 validator 校验规则。
        value.validate()?;
        // 两步都成功，才包装成 ValidatedForm。
        Ok(ValidatedForm(value))
    }
}

// 统一错误类型：包含校验失败和表单提取失败。
#[derive(Debug, Error)]
pub enum ServerError {
    // validator 校验失败。
    #[error(transparent)]
    ValidationError(#[from] validator::ValidationErrors),

    // Axum Form extractor 提取失败。
    #[error(transparent)]
    AxumFormRejection(#[from] FormRejection),
}

// 把 ServerError 转成 HTTP 响应。
impl IntoResponse for ServerError {
    fn into_response(self) -> Response {
        match self {
            // 校验失败返回 400 和校验错误文本。
            ServerError::ValidationError(_) => {
                let message = format!("Input validation error: [{self}]").replace('\n', ", ");
                (StatusCode::BAD_REQUEST, message)
            }
            // 表单提取失败也返回 400。
            ServerError::AxumFormRejection(_) => (StatusCode::BAD_REQUEST, self.to_string()),
        }
        .into_response()
    }
}
````

## 运行和验证

运行前先确认 `examples/validator/Cargo.toml` 里的 `axum = { path = "../../axum" }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-validator
````

合法请求：

````bash
curl -i 'http://127.0.0.1:3000?name=LT'
````

预期响应体：

````html
<h1>Hello, LT!</h1>
````

缺少参数：

````bash
curl -i 'http://127.0.0.1:3000'
````

预期是 400，响应体类似：

````text
Failed to deserialize form: missing field `name`
````

参数为空：

````bash
curl -i 'http://127.0.0.1:3000?name='
````

预期是 400，响应体：

````text
Input validation error: [name: Can not be empty]
````

运行测试：

````bash
cd examples
cargo test -p example-validator
````

常见卡点：

- `Form<T>` 不只适用于 POST 表单，GET/HEAD 请求时它会从 query string 提取（详见第一步说明）。
- `Deserialize` 只表示"能解析"，不表示"业务合法"。
- `Validate` 负责业务校验，校验失败不会进入 handler。
- `#[from]` 让 `?` 可以把不同错误自动转成 `ServerError`。
- 错误消息要和校验规则匹配，别像 example 那样 `min=2` 却写"Can not be empty"（见第二步警告）。

## 手写任务

完成本章代码后，试着做三个小改动：

1. 把 `min = 2` 改成 `min = 3`，观察 `LT` 是否还能通过。
2. 给 `NameInput` 增加 `email: String` 字段，并使用 validator 的 email 校验规则。
3. 把错误响应改成 JSON，例如 `{"error":"...", "type":"validation"}`。

## 本章真正要记住什么

- 提取和校验是两件事：先提取，再校验。
- `validator::Validate` 可以把校验规则写在结构体字段上。
- 自定义 `ValidatedForm<T>` 可以把 `Form<T>` 和校验组合起来。
- handler 收到的应该是已经通过校验的数据。
- 统一错误类型可以同时处理提取失败和校验失败。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/validator/Cargo.toml`
- `examples/validator/src/main.rs`
