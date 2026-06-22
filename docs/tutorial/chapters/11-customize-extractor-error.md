# 11. customize-extractor-error

对应示例：`examples/customize-extractor-error`

本章目标：自定义 JSON 提取失败时的错误响应，理解 extractor rejection 如何转换成项目自己的 API 错误格式。

这一章有三个源码文件，演示同一个目标的三种写法。第一次学习不需要全部背下来，先抓住共同目标：

```text
客户端发错 JSON
-> axum::Json 提取失败
-> 不直接返回默认错误
-> 转成我们自己的 JSON 错误响应
```

## 这个小项目在做什么

前面用过很多 extractor：

- `Json<T>`
- `Form<T>`
- `Multipart`
- `Path<T>`
- 自定义 `JsonOrForm<T>`

extractor 可能失败。比如 `Json<T>` 解析失败时，Axum 默认会返回一个 rejection。

这一章演示如何把 `JsonRejection` 变成统一的 API 错误响应：

```json
{
  "message": "...",
  "origin": "..."
}
```

应用提供三个接口：

| 路径 | 文件 | 做法 |
| --- | --- | --- |
| `POST /with-rejection` | `with_rejection.rs` | 用 `axum_extra::extract::WithRejection` 包装已有 extractor |
| `POST /derive-from-request` | `derive_from_request.rs` | 用 derive 宏生成 `FromRequest` |
| `POST /custom-extractor` | `custom_extractor.rs` | 手写 `FromRequest` |

三种方式最终都在做一件事：

```text
JsonRejection -> 自定义错误响应
```

## 先理解 rejection

在 Axum 里，extractor 失败时产生的错误通常叫 rejection。

例如 handler 写成：

````rust
async fn handler(Json(payload): Json<Payload>) {
}
````

客户端却发送了非法 JSON：

```text
content-type: application/json
body: {"foo":
```

这时 handler 不会被调用。  
因为请求在进入 handler 前，`Json<Payload>` 就已经解析失败了。

默认错误不一定符合你的 API 规范，所以真实项目常常会统一成：

```json
{
  "message": "具体错误信息",
  "code": "INVALID_JSON"
}
```

本章就是在学这个转换点。

## 文件和依赖

这个 example 有六个文件：

1. `examples/customize-extractor-error/Cargo.toml`：声明 Axum macros、axum-extra、serde、thiserror 等依赖。
2. `examples/customize-extractor-error/README.md`：说明三种方式的差异。
3. `examples/customize-extractor-error/src/main.rs`：注册三个路由。
4. `examples/customize-extractor-error/src/with_rejection.rs`：用 `WithRejection` 转换错误。
5. `examples/customize-extractor-error/src/derive_from_request.rs`：用 derive 宏包装 `Json`。
6. `examples/customize-extractor-error/src/custom_extractor.rs`：手写 `FromRequest`。

关键依赖：

- `axum`：提供 `Json`、`FromRequest`、`JsonRejection`、`IntoResponse`。
- `axum-extra`：提供 `WithRejection`。
- `serde` / `serde_json`：构造 JSON 响应。
- `thiserror`：减少错误类型转换样板代码。
- `tokio`、`tracing`、`tracing-subscriber`：运行时和日志。

这里的依赖有两个 feature 要注意：

````toml
axum = { path = "../../axum", features = ["macros"] }
axum-extra = { path = "../../axum-extra", features = ["with-rejection"] }
````

`derive_from_request` 需要 Axum macros。  
`with_rejection` 需要 axum-extra 的 `with-rejection` feature。

## 第一步：main 注册三个对比接口

源码：

````rust
mod custom_extractor;
mod derive_from_request;
mod with_rejection;

let app = Router::new()
    .route("/with-rejection", post(with_rejection::handler))
    .route("/custom-extractor", post(custom_extractor::handler))
    .route("/derive-from-request", post(derive_from_request::handler));
````

这章不是三个业务功能，而是三个实现方案的对比。

每个接口都接收 JSON。  
如果 JSON 正确，就返回收到的 JSON。  
如果 JSON 错误，就返回自定义错误格式。

## 第二步：方案一，用 WithRejection 包装已有 extractor

核心 handler：

````rust
pub async fn handler(
    WithRejection(Json(value), _): WithRejection<Json<Value>, ApiError>,
) -> impl IntoResponse {
    Json(dbg!(value))
}
````

读法是：

```text
先按 Json<Value> 提取请求体
如果 Json<Value> 失败
把 JsonRejection 转成 ApiError
再把 ApiError 转成 HTTP 响应
```

`ApiError` 的关键代码：

````rust
#[derive(Debug, Error)]
pub enum ApiError {
    #[error(transparent)]
    JsonExtractorRejection(#[from] JsonRejection),
}
````

这里有两个 thiserror 宏属性，第一次见需要解释：

- `#[from]`：自动生成 `From<JsonRejection> for ApiError` 的实现。所以 `WithRejection` 知道怎么把原始 rejection 转成 `ApiError`。
- `#[error(transparent)]`：告诉 thiserror，这个变体的错误消息"透传"给内部错误。也就是说 `ApiError` 自己不写新消息，直接用 `JsonRejection` 原本的 `Display` 输出。如果你想在错误消息里加点自己的内容，可以改成 `#[error("JSON 解析失败: {0}")]`，`{0}` 指代内部错误的显示。

注意两者分工不同：`#[from]` 管"类型转换"，`#[error(...)]` 管"显示文本"。

最后实现 `IntoResponse`：

````rust
impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
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

这个方案的特点：

- 优点：学习成本低，不用手写 `FromRequest`。
- 缺点：类型会变长，handler 参数看起来比较重。

## 第三步：方案二，用 derive FromRequest 包装 Json

核心代码：

````rust
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(ApiError))]
pub struct Json<T>(T);
````

这段意思是：

```text
创建自己的 Json<T>
内部实际用 axum::Json<T> 提取请求
如果失败，把 JsonRejection 转成 ApiError
```

handler 看起来更干净：

````rust
pub async fn handler(Json(value): Json<Value>) -> impl IntoResponse {
    Json(dbg!(value))
}
````

因为这个文件里自己定义了 `Json<T>`，所以这里的 `Json` 不是 `axum::Json`，而是自定义包装器。

为了成功时还能直接返回 JSON，它还实现了：

````rust
impl<T: Serialize> IntoResponse for Json<T> {
    fn into_response(self) -> Response {
        let Self(value) = self;
        axum::Json(value).into_response()
    }
}
````

失败时的转换逻辑：

````rust
impl From<JsonRejection> for ApiError {
    fn from(rejection: JsonRejection) -> Self {
        Self {
            status: rejection.status(),
            message: rejection.body_text(),
        }
    }
}
````

这个方案的特点：

- 优点：handler 参数比较干净，适合包装已有 extractor。
- 缺点：需要 Axum macros，并且每种自定义 rejection 都要写一个包装类型。

## 第四步：方案三，手写 FromRequest

手写版最灵活。核心结构：

````rust
pub struct Json<T>(pub T);

impl<S, T> FromRequest<S> for Json<T>
where
    axum::Json<T>: FromRequest<S, Rejection = JsonRejection>,
    S: Send + Sync,
{
    type Rejection = (StatusCode, axum::Json<Value>);

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        ...
    }
}
````

这里不再单纯依赖宏，而是自己写完整提取流程。

它先拆 request：

````rust
let (mut parts, body) = req.into_parts();
````

然后读取匹配到的路径：

````rust
let path = parts
    .extract::<MatchedPath>()
    .await
    .map(|path| path.as_str().to_owned())
    .ok();
````

为什么要先提取 `MatchedPath`？

因为后面 `Json` 会消费请求 body。  
需要在 body 被消费前，先从 request parts 里拿到额外信息。

再把 request 拼回去：

````rust
let req = Request::from_parts(parts, body);
````

最后运行真正的 `axum::Json<T>`：

````rust
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
````

这个方案的特点：

- 优点：最灵活，可以把 `MatchedPath` 等额外上下文放进错误响应。
- 缺点：样板代码最多，也最容易写错。

### 关键顺序：先读 parts，再读 body

手写 `FromRequest` 时最容易踩的坑是**顺序**。看源码里的这两段：

````rust
// 第 1 步：从 parts 里读 MatchedPath（不消费 body）
let (mut parts, body) = req.into_parts();
let path = parts.extract::<MatchedPath>().await...;

// 第 2 步：把 parts 和 body 拼回去，再交给 axum::Json<T>（消费 body）
let req = Request::from_parts(parts, body);
axum::Json::<T>::from_request(req, state).await
````

为什么必须按这个顺序？

这和第 10 章讲的"body 只能消费一次"是一回事。`axum::Json<T>` 在提取时会**消费整个请求 body**。一旦它执行完，`req` 就没了，你再也拿不到 `parts`。

所以规则是：

```text
不消费 body 的提取（parts 类：Method、Uri、Headers、MatchedPath...）先做。
消费 body 的提取（Json、Form、Bytes...）最后做，做完就结束。
```

如果反过来，先调 `axum::Json::from_request`，body 就被吃掉了，后面想读 `MatchedPath` 虽然技术上还能从 parts 读（因为 parts 没被动），但代码会乱、容易写错。保持"parts 先、body 后"的顺序最安全。

这也解释了为什么 axum 把 extractor 分成两类（第 10 章讲过）：

| 类型 | 能拿什么 | 能否多次提取 |
| --- | --- | --- |
| `FromRequestParts` | method、uri、headers、extensions | 可以，parts 能反复读 |
| `FromRequest` | 上面那些 + 整个 body（会消费） | 一次性的，body 只能消费一次 |

## 三种方式怎么选

可以先用这张表判断：

| 方式 | 适合场景 | 代价 |
| --- | --- | --- |
| `WithRejection` | 快速把已有 extractor 的 rejection 换成自己的错误 | handler 类型比较长 |
| derive `FromRequest` | 想要更干净的 handler 参数 | 依赖 macro，每种包装器都要定义类型 |
| 手写 `FromRequest` | 需要读取 request parts、路径、header 等上下文 | 代码最多，复杂度最高 |

新手建议顺序：

```text
先理解 WithRejection
再看 derive FromRequest
最后再看手写 FromRequest
```

## 函数职责速查

- `main`：注册三个对比接口并启动服务。
- `with_rejection::handler`：用 `WithRejection<Json<Value>, ApiError>` 接收 JSON。
- `with_rejection::ApiError`：把 `JsonRejection` 转成自定义 JSON 响应。
- `derive_from_request::Json<T>`：用 derive 宏包装 `axum::Json<T>`。
- `derive_from_request::ApiError`：保存状态码和错误文本。
- `custom_extractor::Json<T>`：手写 `FromRequest`，失败时把路径也放进错误响应。

## 带中文注释的手写版

这一章有多个文件，手写时建议按这个顺序：

```text
main.rs
with_rejection.rs
derive_from_request.rs
custom_extractor.rs
```

### `main.rs`

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-customize-extractor-error
//! ```

// 声明三个模块，分别演示三种自定义 rejection 的方式。
mod custom_extractor;
mod derive_from_request;
mod with_rejection;

// 引入 POST 路由函数和 Router。
use axum::{routing::post, Router};
// tracing_subscriber 用来初始化日志。
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 初始化日志系统。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=trace", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 注册三个接口，分别对应三种实现方式。
    let app = Router::new()
        .route("/with-rejection", post(with_rejection::handler))
        .route("/custom-extractor", post(custom_extractor::handler))
        .route("/derive-from-request", post(derive_from_request::handler));

    // 绑定本机 3000 端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 输出监听地址。
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动 HTTP 服务。
    axum::serve(listener, app).await;
}
````

### `with_rejection.rs` 核心版

````rust
use axum::{extract::rejection::JsonRejection, response::IntoResponse, Json};
use axum_extra::extract::WithRejection;
use serde_json::{json, Value};
use thiserror::Error;

// 用 WithRejection 包装 Json<Value>，失败时转成 ApiError。
pub async fn handler(
    WithRejection(Json(value), _): WithRejection<Json<Value>, ApiError>,
) -> impl IntoResponse {
    Json(dbg!(value))
}

// 自定义错误类型。
#[derive(Debug, Error)]
pub enum ApiError {
    // #[from] 自动生成 From<JsonRejection> for ApiError。
    #[error(transparent)]
    JsonExtractorRejection(#[from] JsonRejection),
}

// 让 ApiError 可以变成 HTTP 响应。
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

### `derive_from_request.rs` 核心版

````rust
use axum::{
    extract::rejection::JsonRejection, extract::FromRequest, http::StatusCode,
    response::IntoResponse,
};
use serde::Serialize;
use serde_json::{json, Value};

// handler 使用的是本模块自定义的 Json<T>。
pub async fn handler(Json(value): Json<Value>) -> impl IntoResponse {
    Json(dbg!(value))
}

// 用 derive 宏生成 FromRequest，内部通过 axum::Json 提取。
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(ApiError))]
pub struct Json<T>(T);

// 成功时，让自定义 Json<T> 也能作为响应返回。
impl<T: Serialize> IntoResponse for Json<T> {
    fn into_response(self) -> axum::response::Response {
        let Self(value) = self;
        axum::Json(value).into_response()
    }
}

// 自定义错误类型。
#[derive(Debug)]
pub struct ApiError {
    status: StatusCode,
    message: String,
}

// 把原始 JsonRejection 转成自己的 ApiError。
impl From<JsonRejection> for ApiError {
    fn from(rejection: JsonRejection) -> Self {
        Self {
            status: rejection.status(),
            message: rejection.body_text(),
        }
    }
}

// 把 ApiError 转成统一 JSON 响应。
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

### `custom_extractor.rs` 核心版

````rust
use axum::{
    extract::{rejection::JsonRejection, FromRequest, MatchedPath, Request},
    http::StatusCode,
    response::IntoResponse,
    RequestPartsExt,
};
use serde_json::{json, Value};

// handler 使用手写的 Json<T> extractor。
pub async fn handler(Json(value): Json<Value>) -> impl IntoResponse {
    Json(dbg!(value));
}

// 自定义 Json extractor。
pub struct Json<T>(pub T);

impl<S, T> FromRequest<S> for Json<T>
where
    // 内部复用 axum::Json<T>，并要求它的 rejection 是 JsonRejection。
    axum::Json<T>: FromRequest<S, Rejection = JsonRejection>,
    S: Send + Sync,
{
    // 失败时直接返回状态码和 JSON body。
    type Rejection = (StatusCode, axum::Json<Value>);

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        // 拆开 request，先读取 parts 里的 MatchedPath。
        let (mut parts, body) = req.into_parts();

        let path = parts
            .extract::<MatchedPath>()
            .await
            .map(|path| path.as_str().to_owned())
            .ok();

        // 重新组装 request，再交给 axum::Json<T> 解析 body。
        let req = Request::from_parts(parts, body);

        match axum::Json::<T>::from_request(req, state).await {
            // 成功时取出 axum::Json<T> 里的 T。
            Ok(value) => Ok(Self(value.0)),
            // 失败时构造自己的 JSON 错误响应。
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

## 运行和验证

运行前先确认 `examples/customize-extractor-error/Cargo.toml` 里的本地依赖能找到项目根目录下的 `axum/` 和 `axum-extra/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-customize-extractor-error
````

发送正确 JSON：

````bash
curl -i -X POST http://127.0.0.1:3000/with-rejection \
  -H 'content-type: application/json' \
  -d '{"hello":"world"}'
````

三个接口都可以用同样方式测试：

````bash
curl -i -X POST http://127.0.0.1:3000/derive-from-request \
  -H 'content-type: application/json' \
  -d '{"hello":"world"}'

curl -i -X POST http://127.0.0.1:3000/custom-extractor \
  -H 'content-type: application/json' \
  -d '{"hello":"world"}'
````

发送错误 JSON：

````bash
curl -i -X POST http://127.0.0.1:3000/with-rejection \
  -H 'content-type: application/json' \
  -d '{"hello":'
````

预期响应是 JSON 错误，其中 `origin` 会显示当前方案来源：

````json
{
  "message": "...",
  "origin": "with_rejection"
}
````

常见卡点：

- 这章自定义的是 extractor 失败时的 rejection，不是 handler 内部业务错误。
- `JsonRejection` 里已经包含状态码和错误文本，可以复用。
- 手写 `FromRequest` 时，如果要读取 `MatchedPath`，要在 body 被 `Json` 消费前读取。
- `custom_extractor::handler` 里的 `Json` 是本模块自定义类型，不是 `axum::Json`。

## 手写任务

完成本章代码后，试着做三个小改动：

1. 把三个方案返回的 JSON 都加上 `"code": "INVALID_JSON"`。
2. 给 `custom_extractor` 的错误响应再加一个 `"method"` 字段，尝试从 request parts 里读取 HTTP 方法。
3. 分别给三个接口发送错误 JSON，对比 `origin` 字段是否符合预期。

## 本章真正要记住什么

- extractor 失败时产生 rejection，handler 不会被调用。
- 可以把 Axum 默认 rejection 转成项目自己的错误响应。
- `WithRejection` 最简单，derive `FromRequest` 更整洁，手写 `FromRequest` 最灵活。
- `IntoResponse` 是错误类型接入 HTTP 响应的关键。
- 真实 API 通常要统一错误响应格式，方便前端处理。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/customize-extractor-error/Cargo.toml`
- `examples/customize-extractor-error/README.md`
- `examples/customize-extractor-error/src/main.rs`
- `examples/customize-extractor-error/src/with_rejection.rs`
- `examples/customize-extractor-error/src/derive_from_request.rs`
- `examples/customize-extractor-error/src/custom_extractor.rs`
