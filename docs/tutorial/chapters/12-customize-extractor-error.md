# 12. customize-extractor-error

对应示例：`examples/customize-extractor-error`

extractor 解析请求失败时,axum 会返回默认的 rejection(错误响应)。本章把这个默认 rejection 改成项目自己的 JSON 错误格式。本章有三个源码文件,演示同一目标的**三种写法**,先看共同目标,再对比差异。

## Cargo.toml

````toml
[package]
name = "example-customize-extractor-error"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["macros"] }
axum-extra = { version = "0.12", features = ["with-rejection"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
thiserror = "2"
tokio = { version = "1.20", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

两个 feature 要注意:`derive_from_request.rs` 需要 axum 的 `macros`,`with_rejection.rs` 需要 axum-extra 的 `with-rejection`。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

````rust
mod custom_extractor;
mod derive_from_request;
mod with_rejection;

use axum::{routing::post, Router};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=trace", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app = Router::new()
        .route("/with-rejection", post(with_rejection::handler))
        .route("/custom-extractor", post(custom_extractor::handler))
        .route("/derive-from-request", post(derive_from_request::handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app).await;
}
````

三个接口对应三种实现方式。每个接口都接收 JSON——JSON 正确就原样返回,错误就返回自定义 JSON 错误。

> 本章多文件,手写建议顺序:`main.rs` → `with_rejection.rs` → `derive_from_request.rs` → `custom_extractor.rs`。下方三个模块的核心版按此顺序排列,完整代码见源码对照。

### `with_rejection.rs`(核心版)

````rust
use axum::{extract::rejection::JsonRejection, response::IntoResponse, Json};
use axum_extra::extract::WithRejection;
use serde_json::{json, Value};
use thiserror::Error;

pub async fn handler(
    WithRejection(Json(value), _): WithRejection<Json<Value>, ApiError>,
) -> impl IntoResponse {
    Json(dbg!(value))
}

#[derive(Debug, Error)]
pub enum ApiError {
    #[error(transparent)]
    JsonExtractorRejection(#[from] JsonRejection),
}

impl IntoResponse for ApiError {
    fn into_response(self) -> axum::response::Response {
        let (status, message) = match self {
            ApiError::JsonExtractorRejection(json_rejection) => {
                (json_rejection.status(), json_rejection.body_text())
            }
        };

        let payload = json!({
            "message": message,
            "origin": "with_rejection"
        });

        (status, Json(payload)).into_response()
    }
}
````

### `derive_from_request.rs`(核心版)

````rust
use axum::{
    extract::rejection::JsonRejection, extract::FromRequest, http::StatusCode,
    response::IntoResponse,
};
use serde::Serialize;
use serde_json::{json, Value};

pub async fn handler(Json(value): Json<Value>) -> impl IntoResponse {
    Json(dbg!(value))
}

#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(ApiError))]
pub struct Json<T>(T);

impl<T: Serialize> IntoResponse for Json<T> {
    fn into_response(self) -> axum::response::Response {
        let Self(value) = self;
        axum::Json(value).into_response()
    }
}

#[derive(Debug)]
pub struct ApiError {
    status: StatusCode,
    message: String,
}

impl From<JsonRejection> for ApiError {
    fn from(rejection: JsonRejection) -> Self {
        Self {
            status: rejection.status(),
            message: rejection.body_text(),
        }
    }
}

impl IntoResponse for ApiError {
    fn into_response(self) -> axum::response::Response {
        let payload = json!({
            "message": self.message,
            "origin": "derive_from_request"
        });

        (self.status, axum::Json(payload)).into_response()
    }
}
````

### `custom_extractor.rs`(核心版)

````rust
use axum::{
    extract::{rejection::JsonRejection, FromRequest, MatchedPath, Request},
    http::StatusCode,
    response::IntoResponse,
    RequestPartsExt,
};
use serde_json::{json, Value};

pub async fn handler(Json(value): Json<Value>) -> impl IntoResponse {
    Json(dbg!(value));
}

pub struct Json<T>(pub T);

impl<S, T> FromRequest<S> for Json<T>
where
    axum::Json<T>: FromRequest<S, Rejection = JsonRejection>,
    S: Send + Sync,
{
    type Rejection = (StatusCode, axum::Json<Value>);

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let (mut parts, body) = req.into_parts();

        let path = parts
            .extract::<MatchedPath>()
            .await
            .map(|path| path.as_str().to_owned())
            .ok();

        let req = Request::from_parts(parts, body);

        match axum::Json::<T>::from_request(req, state).await {
            Ok(value) => Ok(Self(value.0)),
            Err(rejection) => {
                let payload = json!({
                    "message": rejection.body_text(),
                    "origin": "custom_extractor",
                    "path": path,
                });

                Err((rejection.status(), axum::Json(payload)))
            }
        }
    }
}
````

## 运行

````bash
cd examples
cargo run -p example-customize-extractor-error
````

正确 JSON(三个接口都同样测):

````bash
curl -i -X POST http://127.0.0.1:3000/with-rejection \
  -H 'content-type: application/json' \
  -d '{"hello":"world"}'
````

错误 JSON:

````bash
curl -i -X POST http://127.0.0.1:3000/with-rejection \
  -H 'content-type: application/json' \
  -d '{"hello":'
````

预期响应是 JSON 错误,`origin` 标识当前方案:

````json
{
  "message": "...",
  "origin": "with_rejection"
}
````

## 解读

### rejection 是什么

handler 参数里的 extractor 解析请求失败时,产生的错误叫 **rejection**。例如客户端发非法 JSON,`Json<T>` 在进入 handler 前就解析失败,handler 根本不会被调用。默认 rejection 不一定符合你的 API 规范,真实项目常统一成 `{"message": "...", "code": "INVALID_JSON"}` 这种格式。

### 方案一:`WithRejection`(最简单)

````rust
pub async fn handler(
    WithRejection(Json(value), _): WithRejection<Json<Value>, ApiError>,
) -> impl IntoResponse {
    Json(dbg!(value))
}
````

用 `axum_extra::extract::WithRejection` 包装已有 extractor,失败时转成 `ApiError`。`ApiError` 关键:

````rust
#[derive(Debug, Error)]
pub enum ApiError {
    #[error(transparent)]
    JsonExtractorRejection(#[from] JsonRejection),
}
````

两个 thiserror 属性分工不同:

- `#[from]`:自动生成 `From<JsonRejection> for ApiError` 实现,管"类型转换"。
- `#[error(transparent)]`:这个变体的错误消息透传给内部错误,管"显示文本"。想加自己的内容可改成 `#[error("JSON 解析失败: {0}")]`,`{0}` 指代内部错误的显示。

优点:学习成本低,不用手写 `FromRequest`。缺点:handler 参数类型比较长。

### 方案二:derive `FromRequest`

````rust
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(ApiError))]
pub struct Json<T>(T);
````

创建自己的 `Json<T>`,内部实际用 `axum::Json<T>` 提取,失败时转 `ApiError`。handler 参数看起来和原生 `Json<T>` 一样干净:

````rust
pub async fn handler(Json(value): Json<Value>) -> impl IntoResponse {
    Json(dbg!(value))
}
````

这里的 `Json` 是本模块自定义类型,不是 `axum::Json`。需要 axum 的 `macros` feature。优点:handler 参数干净;缺点:每种 rejection 都要写一个包装类型。

### 方案三:手写 `FromRequest`(最灵活)

````rust
async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
    let (mut parts, body) = req.into_parts();

    let path = parts.extract::<MatchedPath>().await
        .map(|path| path.as_str().to_owned())
        .ok();

    let req = Request::from_parts(parts, body);

    match axum::Json::<T>::from_request(req, state).await {
        ...
    }
}
````

自己写完整提取流程,可以把 `MatchedPath`、HTTP 方法等额外上下文塞进错误响应。优点:最灵活;缺点:样板代码最多。

### 关键顺序:parts 先,body 后

手写 `FromRequest` 最容易踩的坑是顺序。看源码这两段:

````rust
// 第 1 步:从 parts 读 MatchedPath(不消费 body)
let (mut parts, body) = req.into_parts();
let path = parts.extract::<MatchedPath>().await...;

// 第 2 步:拼回 request,交给 axum::Json<T>(消费 body)
let req = Request::from_parts(parts, body);
axum::Json::<T>::from_request(req, state).await
````

这和第 11 章讲的"body 只能消费一次"一回事。`axum::Json<T>` 会**消费整个 body**,执行完 `req` 就没了。所以规则是:

```text
不消费 body 的提取(parts 类:Method、Uri、Headers、MatchedPath...)先做。
消费 body 的提取(Json、Form、Bytes...)最后做,做完就结束。
```

这也解释了为什么 axum 把 extractor 分两类:

| 类型 | 能拿什么 | 能否多次提取 |
| --- | --- | --- |
| `FromRequestParts` | method、uri、headers、扩展 | 可以,parts 能反复读 |
| `FromRequest` | 上面那些 + 整个 body(会消费) | 一次性的,body 只能消费一次 |

### 三种方式怎么选

| 方式 | 适合场景 | 代价 |
| --- | --- | --- |
| `WithRejection` | 快速把已有 extractor 的 rejection 换成自己的错误 | handler 类型比较长 |
| derive `FromRequest` | 想要干净的 handler 参数 | 依赖 macro,每种包装器都要定义类型 |
| 手写 `FromRequest` | 需要读 request parts、路径、header 等上下文 | 代码最多,复杂度最高 |

新手建议顺序:先理解 `WithRejection`,再看 derive `FromRequest`,最后看手写。

## 手写任务

跑通后做三个小改动:

1. 把三个方案的错误 JSON 都加上 `"code": "INVALID_JSON"`。
2. 给 `custom_extractor` 的错误响应再加 `"method"` 字段,从 request parts 读 HTTP 方法。
3. 分别给三个接口发错误 JSON,对比 `origin` 字段是否符合预期。

## 小结

- extractor 失败时产生 rejection,handler 不会被调用。
- `JsonRejection` 已包含状态码和错误文本,可以复用。
- `WithRejection` 最简单,derive `FromRequest` 更整洁,手写 `FromRequest` 最灵活。
- 手写 `FromRequest` 时,parts 类提取要在 body 类提取之前。
- `IntoResponse` 是错误类型接入 HTTP 响应的关键。

## 源码对照

- `examples/customize-extractor-error/Cargo.toml`
- `examples/customize-extractor-error/src/main.rs`
- `examples/customize-extractor-error/src/with_rejection.rs`
- `examples/customize-extractor-error/src/derive_from_request.rs`
- `examples/customize-extractor-error/src/custom_extractor.rs`
