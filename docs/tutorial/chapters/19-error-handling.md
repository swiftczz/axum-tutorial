# 19. error-handling

对应示例：`examples/error-handling`

这一章更接近真实业务项目：同一个 handler 里可能既有用户输入错误，也有第三方库错误，还要避免把内部错误细节暴露给客户端。我们用 4 步从默认的 axum 错误响应开始，逐步搭出一个生产级的错误处理方案。

## Cargo.toml

````toml
[package]
name = "example-error-handling"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["macros"] }
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
tower-http = { version = "0.6.1", features = ["trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

启用 axum 的 `macros` feature 才能用 `#[derive(FromRequest)]`。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：默认错误响应长什么样

先写一个最简单的 `POST /users`，用 axum 自带的 `axum::Json` 提取请求体。发非法 JSON 看默认行为。

````rust
use std::{
    collections::HashMap,
    sync::{atomic::{AtomicU64, Ordering}, Arc, Mutex},
};
use axum::{extract::State, routing::post, Json, Router};
use serde::{Deserialize, Serialize};

#[derive(Default, Clone)]
struct AppState {
    next_id: Arc<AtomicU64>,
    users: Arc<Mutex<HashMap<u64, User>>>,
}

#[derive(Deserialize)]
struct UserParams { name: String }

#[derive(Serialize, Clone)]
struct User { id: u64, name: String }

async fn users_create(
    State(state): State<AppState>,
    Json(params): Json<UserParams>,
) -> Json<User> {
    let id = state.next_id.fetch_add(1, Ordering::SeqCst);
    let user = User { id, name: params.name };
    state.users.lock().unwrap().insert(id, user.clone());
    Json(user)
}
````

合法请求没问题。但发非法 JSON：

````bash
curl -i -X POST http://127.0.0.1:3000/users \
  -H 'content-type: application/json' -d '{"name":'
````

响应是 **纯文本**（`content-type: text/plain`），不是 JSON。前端按 JSON 解析会炸。这就是我们要解决的问题。

---

## 第二步：自定义 `AppJson<T>` 提取器

拦截 `axum::Json` 解析失败的那一刻，把错误改成 JSON 格式。定义自己的提取器 `AppJson<T>`，内部复用 `axum::Json`，替换 rejection 类型。

````rust
use axum::{
    extract::{rejection::JsonRejection, FromRequest},
    response::{IntoResponse, Response},
    http::StatusCode,
};

#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(AppError))]
struct AppJson<T>(T);

impl<T> IntoResponse for AppJson<T>
where axum::Json<T>: IntoResponse,
{
    fn into_response(self) -> Response {
        axum::Json(self.0).into_response()
    }
}

#[derive(Debug)]
enum AppError {
    JsonRejection(JsonRejection),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        #[derive(Serialize)]
        struct ErrorResponse { message: String }

        let (status, message) = match &self {
            AppError::JsonRejection(rejection) => (rejection.status(), rejection.body_text()),
        };
        (status, AppJson(ErrorResponse { message })).into_response()
    }
}

impl From<JsonRejection> for AppError {
    fn from(rejection: JsonRejection) -> Self { Self::JsonRejection(rejection) }
}
````

> **新面孔：`#[derive(FromRequest)]`**
>
> `via(axum::Json)` 内部用 `axum::Json` 做提取；`rejection(AppError)` 提取失败时返回 `AppError`。一行 derive 就让我们接管了 JSON 解析错误的格式。
>
> `AppJson<T>` 同时是请求提取器（靠 derive）和响应类型（靠手写 `IntoResponse`），两边都走统一的 JSON 格式。

handler 改用 `AppJson`，再发非法 JSON，响应变成了 `content-type: application/json` + `{"message":"..."}`。

---

## 第三步：加内部错误和 response extension

真实业务里还有内部错误（数据库挂了、第三方库失败），**不该暴露细节给客户端**。

加一个模拟的 `time_library`（每三次调用失败一次），`AppError` 加第二个变体 `TimeError`：

````rust
use std::sync::Arc;

#[derive(Debug)]
enum AppError {
    JsonRejection(JsonRejection),
    TimeError(time_library::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        #[derive(Serialize)]
        struct ErrorResponse { message: String }

        let (status, message, err) = match &self {
            AppError::JsonRejection(rejection) => {
                // 用户输入错误：暴露详情
                (rejection.status(), rejection.body_text(), None)
            }
            AppError::TimeError(_err) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                "Something went wrong".to_owned(),
                Some(self),  // 内部错误悄悄挂到 response 上
            ),
        };

        let mut response = (status, AppJson(ErrorResponse { message })).into_response();
        if let Some(err) = err {
            response.extensions_mut().insert(Arc::new(err));
        }
        response
    }
}
````

> **新面孔：response extension + `Arc<AppError>`**
>
> `response.extensions_mut().insert(...)` 往响应附带的类型键值存储里塞东西，**不会序列化进 HTTP body**，客户端完全看不到。
>
> 用户输入错误塞 `None`（不记日志），内部错误塞 `Some(self)`（要记日志）——分类就在 match 的第三个元素里。用 `Arc` 包一层因为 `AppError` 没有 `Clone`。

---

## 第四步：middleware 记录内部错误日志

加一个 middleware 从 response extension 取出 `Arc<AppError>` 记日志，做到职责分离：

````rust
use axum::{extract::{MatchedPath, Request}, middleware::{from_fn, Next}, response::Response};
use tower_http::trace::TraceLayer;

async fn log_app_errors(request: Request, next: Next) -> Response {
    let response = next.run(request).await;
    if let Some(err) = response.extensions().get::<Arc<AppError>>() {
        tracing::error!(?err, "an unexpected error occurred inside a handler");
    }
    response
}

// main 里:
let app = Router::new()
    .route("/users", post(users_create))
    .layer(from_fn(log_app_errors))   // 内层
    .layer(                            // 外层
        TraceLayer::new_for_http()
            .make_span_with(|req: &Request| {
                let method = req.method();
                let uri = req.uri();
                let matched_path = req.extensions().get::<MatchedPath>().map(|p| p.as_str());
                tracing::debug_span!("request", %method, %uri, matched_path)
            })
            .on_failure(()),
    )
    .with_state(state);
````

> **新面孔：`from_fn` + `Next`（middleware）+ 职责分离**
>
> middleware 就是异步函数：接收 `Request` 和 `next: Next`，调 `next.run(request).await` 交给后续 handler，拿到 `Response` 后还能加工。
>
> 为什么不在 `IntoResponse` 里直接 `tracing::error!`？那样会让"错误转响应"产生副作用。更好的设计：
> - `IntoResponse` 只管"错误怎么返回给客户端"。
> - middleware 只管"内部错误怎么记到日志"。
>
> 桥梁是 response extension：`IntoResponse` 挂上去，middleware 取下来。
>
> `TraceLayer` 放最外层（最后 `.layer()`），它的 `request` span 包裹住 `log_app_errors`，所以 middleware 里的 `tracing::error!` 会自动带上 method/uri 上下文。

---

## 完整代码

````rust
use std::{
    collections::HashMap,
    sync::{
        atomic::{AtomicU64, Ordering},
        Arc, Mutex,
    },
};

use axum::{
    extract::{rejection::JsonRejection, FromRequest, MatchedPath, Request, State},
    http::StatusCode,
    middleware::{from_fn, Next},
    response::{IntoResponse, Response},
    routing::post,
    Router,
};
use serde::{Deserialize, Serialize};
use time_library::Timestamp;
use tower_http::trace::TraceLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let state = AppState::default();

    let app = Router::new()
        .route("/users", post(users_create))
        .layer(from_fn(log_app_errors))
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
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

#[derive(Default, Clone)]
struct AppState {
    next_id: Arc<AtomicU64>,
    users: Arc<Mutex<HashMap<u64, User>>>,
}

#[derive(Deserialize)]
struct UserParams {
    name: String,
}

#[derive(Serialize, Clone)]
struct User {
    id: u64,
    name: String,
    created_at: Timestamp,
}

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

#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(AppError))]
struct AppJson<T>(T);

impl<T> IntoResponse for AppJson<T>
where
    axum::Json<T>: IntoResponse,
{
    fn into_response(self) -> Response {
        axum::Json(self.0).into_response()
    }
}

#[derive(Debug)]
enum AppError {
    JsonRejection(JsonRejection),
    TimeError(time_library::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        #[derive(Serialize)]
        struct ErrorResponse {
            message: String,
        }

        let (status, message, err) = match &self {
            AppError::JsonRejection(rejection) => {
                (rejection.status(), rejection.body_text(), None)
            }
            AppError::TimeError(_err) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                "Something went wrong".to_owned(),
                Some(self),
            ),
        };

        let mut response = (status, AppJson(ErrorResponse { message })).into_response();
        if let Some(err) = err {
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

async fn log_app_errors(request: Request, next: Next) -> Response {
    let response = next.run(request).await;
    if let Some(err) = response.extensions().get::<Arc<AppError>>() {
        tracing::error!(?err, "an unexpected error occurred inside a handler");
    }
    response
}

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

## 运行

````bash
cd examples
cargo run -p example-error-handling
````

合法请求（`Timestamp::now()` 每三次失败一次）：

````bash
curl -i -X POST http://127.0.0.1:3000/users \
  -H 'content-type: application/json' -d '{"name":"Alice"}'
````

非法 JSON（用户输入错误，客户端看到详情）：

````bash
curl -i -X POST http://127.0.0.1:3000/users \
  -H 'content-type: application/json' -d '{"name":'
````

多发几次合法 JSON 触发内部错误，客户端只看到 `{"message":"Something went wrong"}`，服务端日志记录真实错误。

## 手写任务

1. 给 `ErrorResponse` 加 `code` 字段。
2. 把内部错误状态码从 500 改成 503。
3. 在 `log_app_errors` 里把请求路径也打印出来。

## 小结

这章用 4 步搭出生产级错误处理：

1. **看清默认行为**：`axum::Json` 解析失败返回纯文本，不符合统一 JSON 规范。
2. **自定义 `AppJson<T>`**：`#[derive(FromRequest)]` 接管提取错误的格式。
3. **`AppError` 分类处理**：用户错暴露详情，内部错只给通用消息 + 悄悄挂到 response extension。
4. **middleware 记日志**：从 response extension 取 `Arc<AppError>` 记日志，职责分离。

核心思想：**用户输入错误可以暴露帮客户端修正，内部错误绝不暴露细节只记日志**。

## 源码对照

- `examples/error-handling/Cargo.toml`
- `examples/error-handling/src/main.rs`
