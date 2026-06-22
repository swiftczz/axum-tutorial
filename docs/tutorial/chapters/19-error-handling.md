# 19. error-handling

对应示例：`examples/error-handling`

本章目标：设计真实项目里的 `AppError`，把用户输入错误和内部错误分开处理，并用 middleware 记录内部错误日志。

前面几章已经讲过 extractor rejection。  
这一章更接近真实业务项目：同一个 handler 里可能既有输入错误，也有第三方库错误，还要避免把内部错误细节暴露给客户端。

## 这个小项目在做什么

这个 example 提供一个接口：

```text
POST /users
```

它做几件事：

- 用自定义 `AppJson<UserParams>` 解析请求 JSON。
- 生成自增用户 id。
- 调用模拟的第三方库 `Timestamp::now()` 生成创建时间。
- 成功时返回用户 JSON。
- JSON 输入错误时，返回格式统一的错误 JSON。
- 第三方库失败时，客户端只看到 `"Something went wrong"`，服务端日志记录真实错误。

请求主线是：

```text
POST /users
-> AppJson<UserParams> 解析 JSON
-> Timestamp::now()
-> 保存用户
-> 返回 AppJson<User>
```

错误主线是：

```text
JSON 错误
-> AppError::JsonRejection
-> 返回客户端可见的 JSON 错误
```

```text
内部库错误
-> AppError::TimeError
-> 客户端只看到通用错误
-> AppError 放进 response extension
-> log_app_errors middleware 记录详细日志
```

## 先理解错误分类

真实 API 里不要把所有错误混成一类。至少要区分：

| 错误类型 | 示例 | 是否暴露详细信息给客户端 |
| --- | --- | --- |
| 用户输入错误 | JSON 格式错、字段类型错 | 可以暴露，帮助客户端修正 |
| 内部错误 | 数据库挂了、第三方库失败 | 不应暴露细节，只记录日志 |

本章里：

- `JsonRejection` 属于用户输入错误。
- `time_library::Error` 属于内部错误。

## 文件和依赖

这个 example 有两个文件：

1. `examples/error-handling/Cargo.toml`：声明 Axum macros、Serde、tower-http、tracing。
2. `examples/error-handling/src/main.rs`：实现状态、业务 handler、`AppJson`、`AppError`、日志 middleware 和模拟第三方库。

关键依赖：

- `axum`：提供 `FromRequest` derive、`State`、`MatchedPath`、middleware、`IntoResponse`。
- `serde`：请求和响应 JSON。
- `tower-http`：HTTP trace 日志。
- `tokio`、`tracing`、`tracing-subscriber`：运行时和日志。

`Cargo.toml` 里启用了 Axum macros：

````toml
axum = { path = "../../axum", features = ["macros"] }
````

因为本章用到了：

````rust
#[derive(FromRequest)]
````

## 第一步：定义应用状态和数据结构

状态：

````rust
#[derive(Default, Clone)]
struct AppState {
    next_id: Arc<AtomicU64>,
    users: Arc<Mutex<HashMap<u64, User>>>,
}
````

它保存：

- `next_id`：自增用户 id。
- `users`：内存用户表。

请求体：

````rust
#[derive(Deserialize)]
struct UserParams {
    name: String,
}
````

响应体：

````rust
#[derive(Serialize, Clone)]
struct User {
    id: u64,
    name: String,
    created_at: Timestamp,
}
````

`created_at` 来自模拟的第三方库 `time_library`。

## 第二步：注册路由和 middleware

源码：

````rust
let app = Router::new()
    .route("/users", post(users_create))
    .layer(
        TraceLayer::new_for_http()
            .make_span_with(|req: &Request| {
                ...
            })
            .on_failure(()),
    )
    .layer(from_fn(log_app_errors))
    .with_state(state);
````

这里有两个 middleware：

- `TraceLayer`：记录请求开始和结束，并把 matched path 放进 span。
- `log_app_errors`：从 response extension 里取出 `AppError`，记录内部错误日志。

`on_failure(())` 表示禁用 TraceLayer 默认的失败日志。  
因为本章要自己精确控制哪些错误要记录。

## 第三步：业务 handler 返回 Result

源码：

````rust
async fn users_create(
    State(state): State<AppState>,
    AppJson(params): AppJson<UserParams>,
) -> Result<AppJson<User>, AppError> {
    let id = state.next_id.fetch_add(1, Ordering::SeqCst);

    let created_at = Timestamp::now()?;

    let user = User {
        id,
        name: params.name,
        created_at,
    };

    state.users.lock().unwrap().insert(id, user.clone());

    Ok(AppJson(user))
}
````

handler 返回：

```rust
Result<AppJson<User>, AppError>
```

这是一种真实项目常见模式：

```text
成功：返回业务响应
失败：返回统一 AppError
```

`Timestamp::now()?` 可能失败。  
因为后面实现了：

````rust
impl From<time_library::Error> for AppError
````

所以 `?` 可以把第三方库错误自动转成 `AppError`。

## 第四步：自定义 AppJson

源码：

````rust
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(AppError))]
struct AppJson<T>(T);
````

它的含义是：

```text
内部用 axum::Json<T> 提取请求 body
如果 Json 提取失败，转成 AppError
```

为什么要这样做？

因为默认 `axum::Json` 失败时可能返回纯文本。  
真实 API 通常希望所有错误都是统一 JSON 格式。

成功响应也复用 `axum::Json`：

````rust
impl<T> IntoResponse for AppJson<T>
where
    axum::Json<T>: IntoResponse,
{
    fn into_response(self) -> Response {
        axum::Json(self.0).into_response()
    }
}
````

所以 `AppJson<T>` 既可以做请求 extractor，也可以做响应类型。

## 第五步：定义 AppError

源码：

````rust
#[derive(Debug)]
enum AppError {
    JsonRejection(JsonRejection),
    TimeError(time_library::Error),
}
````

当前应用有两类错误：

- `JsonRejection`：请求 body JSON 不合法。
- `TimeError`：第三方时间库失败。

转换实现：

````rust
impl From<JsonRejection> for AppError {
    fn from(rejection: JsonRejection) -> Self {
        Self::JsonRejection(rejection)
    }
}

impl From<time_library::Error> for AppError {
    fn from(error: time_library::Error) -> Self {
        Self::TimeError(error)
    }
}
````

这让 `?` 可以自动把底层错误转成 `AppError`。

## 第六步：把 AppError 转成响应

核心逻辑：

````rust
let (status, message, err) = match &self {
    AppError::JsonRejection(rejection) => {
        (rejection.status(), rejection.body_text(), None)
    }
    AppError::TimeError(_err) => {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            "Something went wrong".to_owned(),
            Some(self),
        )
    }
};
````

JSON 错误：

```text
客户端输入错
-> 返回具体错误信息
-> 不记录成内部错误
```

时间库错误：

```text
服务端内部错
-> 客户端只看到 Something went wrong
-> 真实错误放进 response extension
```

响应 body 统一是：

````rust
#[derive(Serialize)]
struct ErrorResponse {
    message: String,
}
````

最终构造响应：

````rust
let mut response = (status, AppJson(ErrorResponse { message })).into_response();
if let Some(err) = err {
    response.extensions_mut().insert(Arc::new(err));
}
response
````

`response.extensions_mut()` 是本章的关键设计：  
它允许把内部错误挂在 response 上，后面的 middleware 可以读取，但客户端看不到。

## 第七步：middleware 负责记录内部错误

源码：

````rust
async fn log_app_errors(request: Request, next: Next) -> Response {
    let response = next.run(request).await;
    if let Some(err) = response.extensions().get::<Arc<AppError>>() {
        tracing::error!(?err, "an unexpected error occurred inside a handler");
    }
    response
}
````

这段 middleware 做的是：

```text
先让请求正常进入 handler
拿到 response
检查 response 里有没有 AppError extension
有的话记录 error 日志
原样返回 response
```

这样做的好处：

- `IntoResponse` 只负责“错误如何返回给客户端”。
- middleware 负责“错误如何记录到服务端日志”。
- 避免在 `into_response` 里产生额外副作用。

## 第八步：模拟第三方库错误

`time_library::Timestamp::now()` 每三次调用失败一次：

````rust
if COUNTER.fetch_add(1, Ordering::SeqCst).is_multiple_of(3) {
    Err(Error::FailedToGetTime)
} else {
    Ok(Self(1337))
}
````

这只是为了演示内部错误。  
真实项目中这里可能是数据库错误、Redis 错误、外部 API 错误等。

## 函数职责速查

- `main`：初始化日志，创建状态，注册路由和错误日志 middleware。
- `AppState`：保存自增 id 和内存用户表。
- `users_create`：创建用户，可能因为 JSON 或时间库失败而返回 `AppError`。
- `AppJson<T>`：统一 JSON extractor 和 JSON response。
- `AppError`：应用统一错误类型。
- `IntoResponse for AppError`：决定错误返回给客户端的格式。
- `log_app_errors`：记录内部错误日志。
- `time_library`：模拟会失败的第三方库。

## 带中文注释的手写版

这一章源码较长，手写时重点抓住 `AppJson`、`AppError`、`log_app_errors` 三块。

````rust
//! 演示如何把错误转换成 HTTP 响应。
//!
//! Run with
//!
//! ```not_rust
//! cargo run -p example-error-handling
//! ```

use std::{
    collections::HashMap,
    sync::{
        atomic::{AtomicU64, Ordering},
        Arc, Mutex,
    },
};

// 引入 extractor、middleware、响应转换、路由和 Router。
use axum::{
    extract::{rejection::JsonRejection, FromRequest, MatchedPath, Request, State},
    http::StatusCode,
    middleware::{from_fn, Next},
    response::{IntoResponse, Response},
    routing::post,
    Router,
};
// serde 用于 JSON 请求和响应。
use serde::{Deserialize, Serialize};
// 模拟第三方时间库。
use time_library::Timestamp;
// TraceLayer 用于请求日志。
use tower_http::trace::TraceLayer;
// tracing_subscriber 初始化日志。
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 初始化日志系统。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 创建应用状态。
    let state = AppState::default();

    // 注册路由和错误处理相关 middleware。
    let app = Router::new()
        .route("/users", post(users_create))
        .layer(
            TraceLayer::new_for_http()
                .make_span_with(|req: &Request| {
                    let method = req.method();
                    let uri = req.uri();

                    let matched_path = req
                        .extensions()
                        .get::<MatchedPath>()
                        .map(|matched_path| matched_path.as_str());

                    tracing::debug_span!("request", %method, %uri, matched_path)
                })
                .on_failure(()),
        )
        // 自己的 middleware 负责记录 AppError 里的内部错误。
        .layer(from_fn(log_app_errors))
        .with_state(state);

    // 绑定本机 3000 端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}

// 应用状态。
#[derive(Default, Clone)]
struct AppState {
    next_id: Arc<AtomicU64>,
    users: Arc<Mutex<HashMap<u64, User>>>,
}

// 创建用户请求体。
#[derive(Deserialize)]
struct UserParams {
    name: String,
}

// 用户响应体。
#[derive(Serialize, Clone)]
struct User {
    id: u64,
    name: String,
    created_at: Timestamp,
}

// POST /users：创建用户。
async fn users_create(
    State(state): State<AppState>,
    // 使用自己的 AppJson，保证 JSON 提取错误格式统一。
    AppJson(params): AppJson<UserParams>,
) -> Result<AppJson<User>, AppError> {
    let id = state.next_id.fetch_add(1, Ordering::SeqCst);

    // 第三方库可能失败，? 会把错误转成 AppError。
    let created_at = Timestamp::now()?;

    let user = User {
        id,
        name: params.name,
        created_at,
    };

    state.users.lock().unwrap().insert(id, user.clone());

    Ok(AppJson(user))
}

// 自定义 JSON extractor/response 包装器。
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(AppError))]
struct AppJson<T>(T);

// 让 AppJson<T> 也能作为 JSON 响应返回。
impl<T> IntoResponse for AppJson<T>
where
    axum::Json<T>: IntoResponse,
{
    fn into_response(self) -> Response {
        axum::Json(self.0).into_response()
    }
}

// 应用统一错误类型。
#[derive(Debug)]
enum AppError {
    JsonRejection(JsonRejection),
    TimeError(time_library::Error),
}

// 把 AppError 转成客户端响应。
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        #[derive(Serialize)]
        struct ErrorResponse {
            message: String,
        }

        let (status, message, err) = match &self {
            // 用户输入错误：返回具体错误，不记录为内部错误。
            AppError::JsonRejection(rejection) => {
                (rejection.status(), rejection.body_text(), None)
            }
            // 内部错误：客户端只看通用消息，真实错误交给 middleware 记录。
            AppError::TimeError(_err) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                "Something went wrong".to_owned(),
                Some(self),
            ),
        };

        let mut response = (status, AppJson(ErrorResponse { message })).into_response();
        if let Some(err) = err {
            // 把内部错误放进 response extension，客户端看不到。
            response.extensions_mut().insert(Arc::new(err));
        }
        response
    }
}

impl From<JsonRejection> for AppError {
    fn from(rejection: JsonRejection) -> Self {
        Self::JsonRejection(rejection)
    }
}

impl From<time_library::Error> for AppError {
    fn from(error: time_library::Error) -> Self {
        Self::TimeError(error)
    }
}

// middleware：记录 response extension 里的内部错误。
async fn log_app_errors(request: Request, next: Next) -> Response {
    let response = next.run(request).await;
    if let Some(err) = response.extensions().get::<Arc<AppError>>() {
        tracing::error!(?err, "an unexpected error occurred inside a handler");
    }
    response
}

// 模拟第三方库。
mod time_library {
    use std::sync::atomic::{AtomicU64, Ordering};

    use serde::Serialize;

    #[derive(Serialize, Clone)]
    pub struct Timestamp(u64);

    impl Timestamp {
        pub fn now() -> Result<Self, Error> {
            static COUNTER: AtomicU64 = AtomicU64::new(0);

            if COUNTER.fetch_add(1, Ordering::SeqCst).is_multiple_of(3) {
                Err(Error::FailedToGetTime)
            } else {
                Ok(Self(1337))
            }
        }
    }

    #[derive(Debug, Clone)]
    pub enum Error {
        FailedToGetTime,
    }

    impl std::fmt::Display for Error {
        fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
            write!(f, "failed to get time")
        }
    }
}
````

## 运行和验证

运行前先确认 `examples/error-handling/Cargo.toml` 里的 `axum = { path = "../../axum", features = ["macros"] }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-error-handling
````

发送合法请求：

````bash
curl -i -X POST http://127.0.0.1:3000/users \
  -H 'content-type: application/json' \
  -d '{"name":"Alice"}'
````

因为 `Timestamp::now()` 每三次会失败一次，合法请求有时会成功，有时会返回 500。

发送非法 JSON：

````bash
curl -i -X POST http://127.0.0.1:3000/users \
  -H 'content-type: application/json' \
  -d '{"name":'
````

预期返回 JSON 错误，且这是用户输入错误，不应该记录成内部错误。

多发几次合法 JSON，触发内部错误时，客户端应看到：

````json
{"message":"Something went wrong"}
````

服务端日志会记录真实错误。

常见卡点：

- `AppJson` 同时用于请求提取和响应返回。
- 用户输入错误可以返回详细信息；内部错误不要暴露细节。
- `IntoResponse` 里把内部错误放到 extension，middleware 再统一记录。
- `TraceLayer` 的 matched path 比原始 URI 更适合聚合日志。

## 手写任务

完成本章代码后，试着做三个小改动：

1. 给 `ErrorResponse` 增加 `code` 字段，例如 JSON 错误是 `INVALID_JSON`，内部错误是 `INTERNAL_ERROR`。
2. 把内部错误的 HTTP 状态码从 500 改成 503，观察客户端响应。
3. 在 `log_app_errors` 里把请求路径也打印出来。

## 本章真正要记住什么

- 真实项目应该有统一错误类型，例如 `AppError`。
- 用户输入错误和内部错误要区别处理。
- `From` 实现让 `?` 可以把底层错误自动转成 `AppError`。
- `IntoResponse` 决定错误如何返回给客户端。
- response extension 可以把内部错误传给 middleware 做日志，不暴露给客户端。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/error-handling/Cargo.toml`
- `examples/error-handling/src/main.rs`
