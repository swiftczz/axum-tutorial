# 14. validator

对应示例：`examples/validator`

把"提取"和"校验"组合成一个 extractor `ValidatedForm<T>`:先用 `Form<T>` 提取输入,再用 `validator` 校验字段,校验通过才进入 handler。这样 handler 不用再写 `if name.is_empty()` 这类校验逻辑。

## Cargo.toml

````toml
[package]
edition = "2024"
name = "example-validator"
publish = false
version = "0.1.0"

[dependencies]
axum = "0.8"
serde = { version = "1.0", features = ["derive"] }
thiserror = "2"
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
validator = { version = "0.20.0", features = ["derive"] }

[dev-dependencies]
http-body-util = "0.1.0"
tower = { version = "0.5.2", features = ["util"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

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

    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app()).await;
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
````

## 运行

````bash
cd examples
cargo run -p example-validator
````

合法请求:

````bash
curl -i 'http://127.0.0.1:3000?name=LT'
# 预期: <h1>Hello, LT!</h1>
````

缺参数:

````bash
curl -i 'http://127.0.0.1:3000'
# 预期 400: Failed to deserialize form: missing field `name`
````

参数为空:

````bash
curl -i 'http://127.0.0.1:3000?name='
# 预期 400: Input validation error: [name: Can not be empty]
````

运行测试:

````bash
cargo test -p example-validator
````

## 解读

### 提取 vs 校验

| 请求 | 提取是否成功 | 校验是否成功 | 原因 |
| --- | --- | --- | --- |
| `/?name=LT` | 成功 | 成功 | name 长度满足要求 |
| `/?name=` | 成功 | 失败 | name 是空字符串 |
| `/?name=X` | 成功 | 失败 | name 太短 |
| `/` | 失败 | 不会进入校验 | 缺少 name 字段 |

提取解决"请求里有没有这个字段,能不能变成 Rust 类型";校验解决"字段虽然能解析,但业务上是否合法"。本章同时处理这两类失败:`FormRejection`(提取失败)和 `ValidationErrors`(校验失败)。

### 为什么 GET 用 `Form<T>` 而不是 `Query<T>`

反直觉但重要:**`Form<T>` 对 GET/HEAD 请求会从 query string 提取**,对 POST/PUT 才从 body 提取。规则:

```text
GET/HEAD  + Form<T>  → 从 query string 提取
POST/其他 + Form<T>  → 从 body 提取
任何方法 + Query<T>  → 从 query string 提取
```

example 选 `Form<T>` 是因为 `ValidatedForm<T>` 暗示了"既能处理 GET query,也能处理 POST 表单"。如果你想明确只处理 query,用 `Query<T>` 更清晰。

### 校验规则写在字段上

````rust
#[derive(Debug, Deserialize, Validate)]
pub struct NameInput {
    #[validate(length(min = 2, message = "Can not be empty"))]
    pub name: String,
}
````

- `Deserialize`:让 `Form<NameInput>` 能从请求解析出 `name`。
- `Validate`:让 `NameInput` 拥有 `.validate()` 方法。
- `#[validate(length(min = 2, ...))]`:name 长度至少为 2。

⚠️ **注意 example 的小瑕疵**:`min = 2` 表示"长度至少为 2",但 message 写的是 "Can not be empty"(不能为空)。用户传 `?name=X`(长度 1,不是空)收到的却是"不能为空",会被误导。真实项目里 message 要准确描述校验规则,比如 `message = "name must be at least 2 characters"`。规则变了,message 也要跟着变。

### `ValidatedForm` 两步组合

````rust
async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
    let Form(value) = Form::<T>::from_request(req, state).await?;
    value.validate()?;
    Ok(ValidatedForm(value))
}
````

两步:先用 `Form<T>` 提取,再 `value.validate()` 校验。两个 `?` 会把错误转成 `ServerError`。

### `ServerError` 统一两类错误

````rust
#[derive(Debug, Error)]
pub enum ServerError {
    #[error(transparent)]
    ValidationError(#[from] validator::ValidationErrors),
    #[error(transparent)]
    AxumFormRejection(#[from] FormRejection),
}
````

`#[from]` 生成 `From` 实现,所以 `?` 能自动转换错误类型。`IntoResponse` 把两种错误都返回 400,校验失败额外加 `Input validation error: [...]` 前缀。

## 手写任务

跑通后做三个小改动:

1. 把 `min = 2` 改成 `min = 3`,观察 `LT` 是否还能通过。
2. 给 `NameInput` 加 `email: String` 字段,用 validator 的 email 校验规则。
3. 把错误响应改成 JSON,例如 `{"error":"...", "type":"validation"}`。

## 小结

- 提取和校验是两件事:先提取,再校验。
- `validator::Validate` 把校验规则写在字段上,自定义 `ValidatedForm<T>` 把提取和校验组合成一个 extractor。
- handler 收到的应该是已经通过校验的数据,不用再写校验逻辑。
- `Form<T>` 对 GET/HEAD 请求从 query string 提取,对 POST 从 body 提取。
- `#[from]` 让 `?` 自动转换错误类型,统一错误枚举可同时处理提取失败和校验失败。

## 源码对照

- `examples/validator/Cargo.toml`
- `examples/validator/src/main.rs`
