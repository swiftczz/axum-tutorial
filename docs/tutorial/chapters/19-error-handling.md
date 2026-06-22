# 19. error-handling

对应示例：`examples/error-handling`

这一章更接近真实业务项目:同一个 handler 里可能既有输入错误,也有第三方库错误,还要避免把内部错误细节暴露给客户端。本章设计一个 `AppError`,把用户输入错误和内部错误分开处理,并用 middleware 记录内部错误日志。

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

## src/main.rs

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
        .layer(from_fn(log_app_errors))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
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

合法请求(`Timestamp::now()` 每三次失败一次,所以有时成功有时 500):

````bash
curl -i -X POST http://127.0.0.1:3000/users \
  -H 'content-type: application/json' \
  -d '{"name":"Alice"}'
````

非法 JSON(用户输入错误):

````bash
curl -i -X POST http://127.0.0.1:3000/users \
  -H 'content-type: application/json' \
  -d '{"name":'
````

多发几次合法 JSON 触发内部错误,客户端看到:

````json
{"message":"Something went wrong"}
````

服务端日志记录真实错误。

## 解读

### 错误分类

| 错误类型 | 示例 | 是否暴露给客户端 |
| --- | --- | --- |
| 用户输入错误 | JSON 格式错、字段类型错 | 可以暴露,帮客户端修正 |
| 内部错误 | 数据库挂了、第三方库失败 | 不应暴露细节,只记日志 |

本章 `JsonRejection` 是用户输入错误,`time_library::Error` 是内部错误。

### handler 返回 `Result<T, AppError>`

````rust
async fn users_create(
    State(state): State<AppState>,
    AppJson(params): AppJson<UserParams>,
) -> Result<AppJson<User>, AppError> {
    ...
    let created_at = Timestamp::now()?;  // ? 把 time_library::Error 转成 AppError
    ...
}
````

`Timestamp::now()?` 可能失败。因为实现了 `From<time_library::Error> for AppError`,`?` 自动转换。这是真实项目常见模式:成功返回业务响应,失败返回统一 `AppError`。

### `AppJson<T>` 统一请求和响应

````rust
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(AppError))]
struct AppJson<T>(T);
````

内部用 `axum::Json<T>` 提取请求 body,失败时转成 `AppError`。默认 `axum::Json` 失败可能返回纯文本,真实 API 通常希望所有错误都是统一 JSON 格式。

`AppJson<T>` 也实现 `IntoResponse`,复用 `axum::Json` 序列化——所以它既能做请求 extractor,也能做响应类型。

### `AppError` 两种处理路径

````rust
let (status, message, err) = match &self {
    AppError::JsonRejection(rejection) => {
        (rejection.status(), rejection.body_text(), None)  // 用户错:返回详情,不记内部日志
    }
    AppError::TimeError(_err) => (
        StatusCode::INTERNAL_SERVER_ERROR,
        "Something went wrong".to_owned(),                  // 内部错:客户端只看通用消息
        Some(self),                                         // 真实错误放进 extension
    ),
};
````

关键设计:`response.extensions_mut().insert(Arc::new(err))` 把内部错误挂在 response 上,**客户端看不到**,但后面的 middleware 能读取。

### middleware 只负责记录

````rust
async fn log_app_errors(request: Request, next: Next) -> Response {
    let response = next.run(request).await;
    if let Some(err) = response.extensions().get::<Arc<AppError>>() {
        tracing::error!(?err, "an unexpected error occurred inside a handler");
    }
    response
}
````

职责分离:

- `IntoResponse` 只负责"错误如何返回给客户端"。
- middleware 只负责"错误如何记录到服务端日志"。

避免在 `into_response` 里产生额外副作用(比如直接 `tracing::error!`)。

### 模拟第三方库

`time_library::Timestamp::now()` 每三次调用失败一次,演示内部错误。真实项目里这里可能是数据库错误、Redis 错误、外部 API 错误。

## 手写任务

跑通后做三个小改动:

1. 给 `ErrorResponse` 加 `code` 字段,JSON 错误是 `INVALID_JSON`,内部错误是 `INTERNAL_ERROR`。
2. 把内部错误的 HTTP 状态码从 500 改成 503,观察客户端响应。
3. 在 `log_app_errors` 里把请求路径也打印出来。

## 小结

- 真实项目应有统一错误类型(如 `AppError`),区分用户输入错误和内部错误。
- `From` 实现让 `?` 把底层错误自动转成 `AppError`。
- `IntoResponse` 决定错误如何返回给客户端;用户错可暴露详情,内部错只给通用消息。
- response extension 把内部错误传给 middleware 做日志,客户端看不到。
- middleware 只负责日志,不在 `into_response` 里产生副作用。

## 源码对照

- `examples/error-handling/Cargo.toml`
- `examples/error-handling/src/main.rs`
