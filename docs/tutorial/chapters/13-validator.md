# 13. validator

对应示例：`examples/validator`

第 11 章讲了自定义 extractor 错误。这章进一步：用 `validator` crate + 自定义 extractor **自动验证输入**——比如"name 至少 2 字符"、"email 格式"、"age 在 0-150 之间"。handler 拿到的永远是验证过的数据，无需手写 if。

分 3 步：先用 `validator` 给 model 加验证规则，再写自定义 `ValidatedForm` extractor 把"解析 + 验证"封装起来，最后写 `ServerError` 把验证错误转成 400 响应。

相比前面章节新引入：**`validator` crate、`#[validate(...)]` 属性、自定义 `FromRequest` extractor 包装内部 extractor、`thiserror` 多变体错误枚举**。

## Cargo.toml

````toml
[package]
name = "example-validator"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["macros"] }
serde = { version = "1", features = ["derive"] }
thiserror = "2"
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
validator = { version = "0.20", features = ["derive"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：给 model 加 `#[validate(...)]` 规则

先定义带验证规则的输入类型。`validator` 提供很多内置规则（length、email、range、contains...），用属性标注。

````rust
use axum::{
    response::Html,
    routing::get,
    Router,
};
use serde::Deserialize;
use tokio::net::TcpListener;
use validator::Validate;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[derive(Debug, Deserialize, Validate)]
pub struct NameInput {
    #[validate(length(min = 2, message = "Can not be empty"))]
    pub name: String,
}

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(/* tracing 初始化 */)
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app = Router::new().route("/", get(handler));

    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

// handler 暂时直接收 NameInput（这步还没接 validator extractor）
async fn handler(input: axum::Form<NameInput>) -> Html<String> {
    Html(format!("<h1>Hello, {}!</h1>", input.name))
}
````

> **新面孔：`validator` crate**
>
> Rust 生态的输入验证库，提供几十种内置验证规则。在 struct 字段加 `#[validate(rule)]` 属性，调 `.validate()` 时检查所有规则。
>
> 这章用 `length(min = 2)`——字符串长度 >= 2。其他常用规则：
> - `email`：必须是合法 email
> - `range(min = 0, max = 150)`：数字范围
> - `url`：必须是合法 URL
> - `contains("xxx")`：必须包含子串
> - `custom(func)`：自定义验证函数

> **新面孔：`#[derive(Validate)]`**
>
> 给 struct derive `Validate` trait，提供 `.validate()` 方法。必须同时 derive 这个和 serde 的 Deserialize（解析 JSON/Form 后才能验证）。

> **新面孔：`#[validate(...)]` 属性**
>
> 标注每个字段的验证规则。`#[validate(length(min = 2, message = "Can not be empty"))]`：
> - `length(min = 2)`：字符串长度至少 2
> - `message = "..."`：验证失败的错误消息（可读）
>
> 一个字段可叠加多个规则：`#[validate(length(min = 2))] #[validate(custom = "must_be_alphanumeric")]`。

验证（这步 handler 还没自动验证，只解析）：

````bash
cd examples
cargo run -p example-validator

curl 'http://127.0.0.1:3000/?name=LT'
# 返回 <h1>Hello, LT!</h1>

curl 'http://127.0.0.1:3000/?name='
# 也返回 <h1>Hello, !</h1>，没验证（这步还没接）
````

下一步接 validator extractor。

---

## 第二步：`ValidatedForm` extractor——解析 + 验证

写一个泛型 extractor `ValidatedForm<T>`，封装 `Form<T>` 并在解析后自动 `.validate()`。

````rust
use axum::{
    extract::{rejection::FormRejection, Form, FromRequest, Request},
};
use serde::de::DeserializeOwned;
use validator::Validate;

#[derive(Debug, Clone, Copy, Default)]
pub struct ValidatedForm<T>(pub T);

impl<T, S> FromRequest<S> for ValidatedForm<T>
where
    T: DeserializeOwned + Validate,
    S: Send + Sync,
    Form<T>: FromRequest<S, Rejection = FormRejection>,
{
    type Rejection = ServerError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        // 1. 用 axum::Form<T> 解析 query string/form
        let Form(value) = Form::<T>::from_request(req, state).await?;
        // 2. 验证
        value.validate()?;
        Ok(ValidatedForm(value))
    }
}

async fn handler(ValidatedForm(input): ValidatedForm<NameInput>) -> Html<String> {
    Html(format!("<h1>Hello, {}!</h1>", input.name))
}
````

> **新面孔：自定义 `FromRequest` 包装内部 extractor**
>
> 第 11 章讲过自定义 extractor。这里更进一步：内部复用 `axum::Form<T>` 做解析（不重新发明轮子），再加 `.validate()`。
>
> 关键约束 `Form<T>: FromRequest<S, Rejection = FormRejection>`：要求内部 `Form<T>` extractor 的 rejection 类型是 `FormRejection`。这样 `Form::<T>::from_request(...)?` 的错误能被 `ServerError` 接收。

> **新面孔：泛型 extractor 模式**
>
> `ValidatedForm<T>` 是泛型的——任何 `T: DeserializeOwned + Validate` 都能用。给不同 model（`NameInput`、`UserInput`、`OrderInput`）都自动加验证，不用每个写一遍。
>
> axum 内置的 `Json<T>`、`Form<T>` 都是泛型 extractor——你可以仿造这个模式写自己的（如 `ValidatedJson<T>`、`AuthedUser<S>` 等）。

> **新面孔：`?` 跨错误类型**
>
> `Form::<T>::from_request(...).await?` 返回 `Result<Form<T>, FormRejection>`，`value.validate()?` 返回 `Result<_, ValidationErrors>`。两种不同错误类型都用 `?`——因为下面 `ServerError` 实现了 `From<FormRejection>` 和 `From<ValidationErrors>`（用 thiserror `#[from]`）。

---

## 第三步：`ServerError` 把验证错误转 400 响应

最后定义 `ServerError` 枚举把两种错误（验证失败、Form 解析失败）转成可读 400 响应。

````rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
};
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ServerError {
    #[error(transparent)]
    ValidationError(#[from] validator::ValidationErrors),

    #[error(transparent)]
    AxumFormRejection(#[from] FormRejection),
}

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

验证：

````bash
cd examples
cargo run -p example-validator

curl 'http://127.0.0.1:3000/?name='
# Input validation error: [name: Can not be empty]

curl 'http://127.0.0.1:3000/?name=L'
# Input validation error: [name: Can not be empty]

curl 'http://127.0.0.1:3000/?name=LT'
# <h1>Hello, LT!</h1>

curl 'http://127.0.0.1:3000/'
# Failed to deserialize form: missing field `name`
````

> **新面孔：`thiserror` 多变体错误枚举**
>
> 第 11 章用过 `#[from]` 自动生成 `From` impl。这里演示更复杂的多变体错误：
>
> - `ValidationError(#[from] validator::ValidationErrors)`：验证失败时 `?` 自动转
> - `AxumFormRejection(#[from] FormRejection)`：Form 解析失败时 `?` 自动转
>
> `#[error(transparent)]` 表示这个变体的错误消息透传给内部错误（不额外加前缀）。

> **新面孔：`IntoResponse` 实现错误类型**
>
> 让 axum 把 `ServerError` 直接当响应返回。两种变体都映射到 400 Bad Request，但消息不同：验证错误时把 `ValidationErrors` 的多行格式（`name: Can not be empty\nemail: ...`）替换成单行。

---

## 完整代码

````rust
use axum::{
    extract::{rejection::FormRejection, Form, FromRequest, Request},
    http::StatusCode,
    response::{Html, IntoResponse, Response},
    routing::get,
    Router,
};
use serde::{de::DeserializeOwned, Deserialize};
use thiserror::Error;
use tokio::net::TcpListener;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};
use validator::Validate;

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
    let app = app();

    // run it
    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}

fn app() -> Router {
    Router::new().route("/", get(handler))
}

#[derive(Debug, Deserialize, Validate)]
pub struct NameInput {
    #[validate(length(min = 2, message = "Can not be empty"))]
    pub name: String,
}

async fn handler(ValidatedForm(input): ValidatedForm<NameInput>) -> Html<String> {
    Html(format!("<h1>Hello, {}!</h1>", input.name))
}

#[derive(Debug, Clone, Copy, Default)]
pub struct ValidatedForm<T>(pub T);

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

#[derive(Debug, Error)]
pub enum ServerError {
    #[error(transparent)]
    ValidationError(#[from] validator::ValidationErrors),

    #[error(transparent)]
    AxumFormRejection(#[from] FormRejection),
}

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

#[cfg(test)]
mod tests;
````

## 运行

````bash
cd examples
cargo run -p example-validator

curl 'http://127.0.0.1:3000/?name=LT'    # <h1>Hello, LT!</h1>
curl 'http://127.0.0.1:3000/?name='      # Input validation error: [name: Can not be empty]
curl 'http://127.0.0.1:3000/'            # Failed to deserialize form: missing field `name`
````

## 解读

### validator 内置规则速查

```rust
#[validate(length(min = 2, max = 50))]                       // 长度
#[validate(email)]                                            // email 格式
#[validate(url)]                                              // URL 格式
#[validate(range(min = 0, max = 150))]                       // 数字范围
#[validate(contains("xxx"))]                                  // 包含子串
#[validate(regex = "RE_PATTERN")]                            // 正则
#[validate(custom(function = "my_validate_fn"))]             // 自定义函数
#[validate(must_match(other = "password_confirm"))]          // 两字段相等
```

完整列表见 `validator` crate 文档。

### 自定义 extractor 的复用模式

`ValidatedForm<T>` 是个**通用模式**：定义 wrapper struct + 实现 `FromRequest` + 内部复用已有 extractor。同样的模式可以写：

- `ValidatedJson<T>`：JSON + validator
- `AuthedUser<S>`：从 header 提 token + 验证 + 反序列化
- `Cached<T>`：解析 + 查缓存

## 常见问题

**`validator` 和 `garde` 区别？** 两个都是验证库，API 类似。`validator` 更老更流行，`garde` 是较新的替代（更好的错误消息、async 支持）。任选其一。

**验证失败应该返回 400 还是 422？** HTTP 语义上 422 Unprocessable Entity 更准确（语法正确但语义错误），但 400 也常用。这章用 400。

**怎么测自定义验证？** validator 是同步的，单元测试直接构造 struct 调 `.validate()`，不用启 server。

## 手写任务

1. 加 `email` 字段用 `#[validate(email)]` 规则。
2. 加 `age` 字段用 `#[validate(range(min = 0, max = 150))]`。
3. 写 `ValidatedJson<T>` 泛型 extractor（同样的模式，内部换 `axum::Json<T>`）。
4. 验证错误改成 JSON 格式：`{"errors": [{"field": "name", "message": "..."}]}`。

## 小结

这章用 3 步讲了输入验证：

1. **`#[validate(...)]` 规则**：用 validator crate 的属性标注每个字段规则。
2. **`ValidatedForm<T>` extractor**：泛型 wrapper，内部复用 `Form<T>` + 自动 `.validate()`。
3. **`ServerError` 转响应**：thiserror 多变体错误 + `IntoResponse` 实现。

核心模式：**自定义泛型 extractor 复用内部 extractor + 加额外逻辑**。这套模式适用于验证、缓存、认证等任意"装饰"现有 extractor 的场景。

## 源码对照

- `examples/validator/Cargo.toml`
- `examples/validator/src/main.rs`
- `examples/validator/tests/tests.rs`
