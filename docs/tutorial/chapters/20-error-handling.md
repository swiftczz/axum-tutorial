# 20. error-handling

对应示例：`examples/error-handling`

这一章更接近真实业务项目:同一个 handler 里可能既有输入错误,也有第三方库错误,还要避免把内部错误细节暴露给客户端。本章设计一个 `AppError`,把用户输入错误和内部错误分开处理,内部错误直接在 `IntoResponse` 里记录日志。

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

        let (status, message) = match self {
            AppError::JsonRejection(rejection) => {
                // 用户输入错误,不需要日志
                (rejection.status(), rejection.body_text())
            }
            AppError::TimeError(err) => {
                // TraceLayer 已经把请求方法、URI 等放进了 span,这里只需记错误本身
                tracing::error!(%err, "error from time_library");

                // 不暴露错误细节给客户端
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "Something went wrong".to_owned(),
                )
            }
        };

        (status, AppJson(ErrorResponse { message })).into_response()
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

    #[derive(Debug)]
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

服务端日志记录真实错误(`error from time_library err=failed to get time`)。

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

`Timestamp::now()?` 可能失败。因为实现了 `From<time_library::Error> for AppError`,`?` 自动转换。

### `AppJson<T>` 统一请求和响应

````rust
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(AppError))]
struct AppJson<T>(T);
````

内部用 `axum::Json<T>` 提取请求 body,失败时转成 `AppError`。`AppJson<T>` 也实现 `IntoResponse`,复用 `axum::Json` 序列化——所以它既能做请求 extractor,也能做响应类型。

### `AppError` 的 `IntoResponse`(本章关键)

````rust
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::JsonRejection(rejection) => {
                (rejection.status(), rejection.body_text())     // 用户错:返回详情
            }
            AppError::TimeError(err) => {
                tracing::error!(%err, "error from time_library"); // 内部错:直接记日志
                (StatusCode::INTERNAL_SERVER_ERROR, "Something went wrong".to_owned())
            }
        };
        (status, AppJson(ErrorResponse { message })).into_response()
    }
}
````

两种错误的处理方式不同:

- **用户输入错误**(`JsonRejection`):返回具体错误信息(`body_text()`),帮客户端修正。不需要记日志。
- **内部错误**(`TimeError`):**直接在 `into_response` 里 `tracing::error!` 记录日志**,客户端只看通用消息 `"Something went wrong"`。

`TraceLayer` 已经把每个请求包在一个 span 里(含 method、uri、matched_path),所以 `tracing::error!` 的日志会自动带上请求上下文,不需要手动在错误日志里重复请求信息。

### 模拟第三方库

`time_library::Timestamp::now()` 每三次调用失败一次,演示内部错误。真实项目里这里可能是数据库错误、Redis 错误、外部 API 错误。

## 手写任务

跑通后做三个小改动:

1. 给 `ErrorResponse` 加 `code` 字段,JSON 错误是 `INVALID_JSON`,内部错误是 `INTERNAL_ERROR`。
2. 把内部错误的 HTTP 状态码从 500 改成 503,观察客户端响应。
3. 在 `tracing::error!` 里额外打印 `created_at` 的值。

## 小结

- 真实项目应有统一错误类型(如 `AppError`),区分用户输入错误和内部错误。
- `From` 实现让 `?` 把底层错误自动转成 `AppError`。
- `IntoResponse` 决定错误如何返回给客户端:用户错暴露详情,内部错只给通用消息。
- 内部错误直接在 `into_response` 里 `tracing::error!` 记录日志;`TraceLayer` 的 span 自动提供请求上下文。

## 源码对照

- `examples/error-handling/Cargo.toml`
- `examples/error-handling/src/main.rs`
