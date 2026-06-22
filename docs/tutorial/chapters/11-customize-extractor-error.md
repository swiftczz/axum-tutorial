# 11. customize-extractor-error

对应示例：`examples/customize-extractor-error`

extractor 解析请求失败时，axum 会返回默认的 rejection（错误响应）。本章把这个默认 rejection 改成项目自己的 JSON 错误格式。

同一目标有**三种写法**，从简到繁：

1. `WithRejection` 包装器（最简单）
2. `#[derive(FromRequest)]` 派生宏（handler 参数干净）
3. 手写 `FromRequest`（最灵活，能读 request 上下文）

本章用 4 步走完三种写法：第一步搭骨架并观察默认 rejection，后三步依次实现三种方案。

相比前面章节新引入：**`JsonRejection`、`WithRejection`、`#[derive(FromRequest)]`、手写 `FromRequest`、`MatchedPath`、`RequestPartsExt`**。

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

两个 feature 要注意：`derive_from_request.rs` 需要 axum 的 `macros`，`with_rejection.rs` 需要 axum-extra 的 `with-rejection`。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：骨架 + 观察默认 rejection

先搭一个能跑的最小服务：一个 `POST /` 接口接收 JSON，成功就原样返回。这步先看**默认 rejection 长什么样**——这是我们要改掉的东西。

````rust
use axum::{routing::post, Json, Router};
use serde_json::Value;
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

    let app = Router::new().route("/", post(handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler(Json(value): Json<Value>) -> Json<Value> {
    Json(dbg!(value))
}
````

发正确 JSON，正常返回：

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'content-type: application/json' \
  -d '{"hello":"world"}'
````

发**错误** JSON，观察默认 rejection：

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'content-type: application/json' \
  -d '{"hello":'
````

默认响应是**纯文本** `Failed to parse the request body as JSON: ...`，状态码 400。真实项目的 API 规范通常要求 JSON 格式（如 `{"message":"...","code":"INVALID_JSON"}`）——下面三步分别用三种方式实现。

> **新面孔：rejection**
>
> extractor 在 handler 被调用**之前**解析请求。解析失败产生的错误叫 **rejection**，axum 直接把它转成响应返回，handler 根本不会执行。
>
> 每个 extractor 都有自己的 rejection 类型。`Json<T>` 的 rejection 是 `JsonRejection`，自带状态码（400）和错误文本。后面三步都是把 `JsonRejection` 转成自定义错误类型。

---

## 第二步：方案一 `WithRejection`（最简单）

`axum_extra::extract::WithRejection` 是个包装器：把已有 extractor 的 rejection 换成自己的错误类型。只需要写一个 `From<JsonRejection> for ApiError` 转换，`thiserror` 的 `#[from]` 能自动生成。

````rust
use axum::{extract::rejection::JsonRejection, response::IntoResponse, routing::post, Json, Router};
use axum_extra::extract::WithRejection;
use serde_json::{json, Value};
use thiserror::Error;
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

    let app = Router::new().route("/with-rejection", post(handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

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

验证：

````bash
curl -i -X POST http://127.0.0.1:3000/with-rejection \
  -H 'content-type: application/json' \
  -d '{"hello":'
````

现在错误响应是 JSON：`{"message":"...","origin":"with_rejection"}`。

> **新面孔：`WithRejection`**
>
> `axum_extra` 提供的包装器，语法 `WithRejection<Inner, Err>`。它内部调用 `Inner` extractor，成功就返回值，失败就把原 rejection 通过 `From` 转成 `Err` 并作为响应返回。
>
> `WithRejection(Json(value), _)` 的第二个元素 `_` 是占位符——`WithRejection` 的第二个构造参数没有意义，可以忽略。

> **新面孔：`JsonRejection`**
>
> `axum::extract::rejection::JsonRejection` 是 `Json<T>` 提取失败时的错误类型。它提供两个关键方法：
> - `.status()` → `StatusCode`（通常是 400）
> - `.body_text()` → `String`（错误描述）
>
> 自定义错误类型只要能从 `JsonRejection` 转过来（实现 `From<JsonRejection>`），就能复用这些信息。

> **新面孔：`thiserror` 的 `#[from]` 和 `#[error(transparent)]`**
>
> `#[from]` 自动生成 `From<JsonRejection> for ApiError`，管"类型转换"。
> `#[error(transparent)]` 让这个变体的错误消息**透传**给内部错误（显示 `JsonRejection` 自己的文本）。想加自己的内容可改成 `#[error("JSON 解析失败: {0}")]`，`{0}` 指代内部错误。

---

## 第三步：方案二 `#[derive(FromRequest)]`（handler 参数干净）

方案一的缺点：handler 参数 `WithRejection<Json<Value>, ApiError>` 很长。方案二自定义一个 `Json<T>` 类型，内部复用 `axum::Json`，handler 参数就和原生一样干净。

````rust
use axum::{
    extract::rejection::JsonRejection, extract::FromRequest, http::StatusCode,
    response::IntoResponse, routing::post, Router,
};
use serde::Serialize;
use serde_json::{json, Value};
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

    let app = Router::new().route("/derive-from-request", post(handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

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

注意 handler 里的 `Json` 是**本模块自定义的类型**，不是 `axum::Json`。但它用法和原生一模一样。

验证：

````bash
curl -i -X POST http://127.0.0.1:3000/derive-from-request \
  -H 'content-type: application/json' \
  -d '{"hello":'
````

错误响应：`{"message":"...","origin":"derive_from_request"}`。

> **新面孔：`#[derive(FromRequest)]`**
>
> axum `macros` feature 提供的派生宏，自动为你的类型生成 `FromRequest` 实现。属性 `#[from_request(via(axum::Json), rejection(ApiError))]` 表示：
> - `via(axum::Json)`：内部用 `axum::Json` 做实际提取
> - `rejection(ApiError)`：失败时把 rejection 转成 `ApiError`
>
> 需要你自己提供 `From<JsonRejection> for ApiError`（用 `#[from]` 或手写都行）。优点是 handler 参数干净；缺点是每种 rejection 都要定义一个包装类型。

---

## 第四步：方案三 手写 `FromRequest`（最灵活）

前两种方案只能改 rejection 类型，**拿不到 request 上下文**。手写 `FromRequest` 能在提取过程中读 `MatchedPath`（匹配的路径）、HTTP 方法等，把它们塞进错误响应。

````rust
use axum::{
    extract::{rejection::JsonRejection, FromRequest, MatchedPath, Request},
    http::StatusCode,
    response::IntoResponse,
    routing::post,
    RequestPartsExt, Router,
};
use serde_json::{json, Value};
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

    let app = Router::new().route("/custom-extractor", post(handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

pub async fn handler(Json(value): Json<Value>) -> impl IntoResponse {
    Json(dbg!(value))
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

        // parts 类提取要先做（不消费 body）
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

验证：

````bash
curl -i -X POST http://127.0.0.1:3000/custom-extractor \
  -H 'content-type: application/json' \
  -d '{"hello":'
````

错误响应多了 `path` 字段：`{"message":"...","origin":"custom_extractor","path":"/custom-extractor"}`。

> **新面孔：手写 `FromRequest`**
>
> `FromRequest<S>` 是 axum extractor 的核心 trait（`S` 是 State 类型）。实现它要指定：
> - `type Rejection`：提取失败的错误类型（必须实现 `IntoResponse`）
> - `async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection>`
>
> 这是三种方案里唯一能拿到完整 `Request` 的——可以读 parts、改 parts、再交给内部 extractor。

> **新面孔：`MatchedPath`**
>
> `axum::extract::MatchedPath` 是个 parts 类 extractor，提取**路由匹配后的路径**（如 `/custom-extractor`）。常用于日志和错误上下文。
>
> 用法：`parts.extract::<MatchedPath>().await`。注意是在 **parts** 上调用，不消费 body。

> **新面孔：`RequestPartsExt::extract`**
>
> `axum::RequestPartsExt` 给 `Parts` 加了 `.extract::<T>().await` 方法，能在手写 `FromRequest` 里方便地跑其他 parts 类 extractor。`parts.extract::<MatchedPath>().await` 就是它提供的。

### 关键顺序：parts 先，body 后

手写 `FromRequest` 最容易踩的坑是顺序。看源码这两段：

````rust
// 第 1 步：从 parts 读 MatchedPath（不消费 body）
let (mut parts, body) = req.into_parts();
let path = parts.extract::<MatchedPath>().await...;

// 第 2 步：拼回 request，交给 axum::Json<T>（消费 body）
let req = Request::from_parts(parts, body);
axum::Json::<T>::from_request(req, state).await
````

这和第 10 章讲的"body 只能消费一次"一回事。`axum::Json<T>` 会**消费整个 body**，执行完 `req` 就没了。规则：

```text
不消费 body 的提取（parts 类：Method、Uri、Headers、MatchedPath...）先做。
消费 body 的提取（Json、Form、Bytes...）最后做，做完就结束。
```

这也解释了 axum 把 extractor 分两类的原因：

| 类型 | 能拿什么 | 能否多次提取 |
| --- | --- | --- |
| `FromRequestParts` | method、uri、headers、扩展 | 可以，parts 能反复读 |
| `FromRequest` | 上面那些 + 整个 body（会消费） | 一次性的，body 只能消费一次 |

---

## 完整代码

examples 把三种方案放在同一个 binary 里，通过 `mod` 引入三个模块文件。

`src/main.rs`：

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

    axum::serve(listener, app).await.unwrap();
}
````

`src/with_rejection.rs`：

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

`src/derive_from_request.rs`：

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

`src/custom_extractor.rs`：

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

正确 JSON（三个接口都同样测）：

````bash
curl -i -X POST http://127.0.0.1:3000/with-rejection \
  -H 'content-type: application/json' \
  -d '{"hello":"world"}'
````

错误 JSON：

````bash
curl -i -X POST http://127.0.0.1:3000/with-rejection \
  -H 'content-type: application/json' \
  -d '{"hello":'
````

预期响应是 JSON 错误，`origin` 标识当前方案：

````json
{
  "message": "...",
  "origin": "with_rejection"
}
````

## 解读

### 三种方式怎么选

| 方式 | 适合场景 | 代价 |
| --- | --- | --- |
| `WithRejection` | 快速把已有 extractor 的 rejection 换成自己的错误 | handler 类型比较长 |
| derive `FromRequest` | 想要干净的 handler 参数 | 依赖 macro，每种包装器都要定义类型 |
| 手写 `FromRequest` | 需要读 request parts、路径、header 等上下文 | 代码最多，复杂度最高 |

新手建议顺序：先理解 `WithRejection`，再看 derive `FromRequest`，最后看手写。

### rejection 复用：`JsonRejection` 自带状态码和文本

三种方案的错误信息都来自 `JsonRejection` 的两个方法：

- `rejection.status()` → 400
- `rejection.body_text()` → `"Failed to parse the request body as JSON: ..."`

自定义错误类型只是把这些信息**重新包装**成项目统一的 JSON 格式。

## 手写任务

跑通后做三个小改动：

1. 把三个方案的错误 JSON 都加上 `"code": "INVALID_JSON"`。
2. 给 `custom_extractor` 的错误响应再加 `"method"` 字段，从 request parts 读 HTTP 方法。
3. 分别给三个接口发错误 JSON，对比 `origin` 字段是否符合预期。

## 小结

这章用 4 步讲了如何自定义 extractor 错误：

1. **骨架**：观察默认 rejection 是纯文本，不符合 API 规范。
2. **`WithRejection`**：包装器 + `thiserror`，最简单，但 handler 类型长。
3. **derive `FromRequest`**：自定义 `Json<T>` 类型，handler 参数干净。
4. **手写 `FromRequest`**：能读 `MatchedPath` 等上下文，最灵活但代码最多。

核心规则：**手写 `FromRequest` 时，parts 类提取（不消费 body）要先于 body 类提取。**

## 源码对照

- `examples/customize-extractor-error/Cargo.toml`
- `examples/customize-extractor-error/src/main.rs`
- `examples/customize-extractor-error/src/with_rejection.rs`
- `examples/customize-extractor-error/src/derive_from_request.rs`
- `examples/customize-extractor-error/src/custom_extractor.rs`
