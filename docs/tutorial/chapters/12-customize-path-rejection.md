# 12. customize-path-rejection

对应示例：`examples/customize-path-rejection`

上一章自定义 `Json<T>` 解析失败的错误，这一章自定义 `Path<T>` 解析失败的错误。把 axum 默认的 `PathRejection` 转成更适合 API 的 JSON 错误，告诉客户端**具体哪个路径参数错了**。

相比上一章新引入：`FromRequestParts`（只读 parts 不读 body 的 extractor）、`ErrorKind`（路径参数错误分类）、`DeserializeOwned`。

分 3 步搭。

## Cargo.toml

````toml
[package]
name = "example-customize-path-rejection"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：最小路由和 Params 结构体

先写一个带路径参数的路由：`GET /users/{user_id}/teams/{team_id}`。

````rust
use axum::{routing::get, Router};
use serde::{Deserialize, Serialize};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/users/{user_id}/teams/{team_id}", get(handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await.unwrap();
    axum::serve(listener, app).await;
}

#[derive(Debug, Deserialize, Serialize)]
struct Params {
    user_id: u32,
    team_id: u32,
}

async fn handler(Path(params): Path<Params>) -> impl IntoResponse {
    axum::Json(params)
}
````

用 axum 默认的 `Path<T>`，访问 `/users/1/teams/2` 返回 `{"user_id":1,"team_id":2}`。但访问 `/users/abc/teams/2` 时，`abc` 无法解析成 `u32`，默认返回一个不太友好的错误。

> **新面孔：路径参数解析**
>
> URL 里的内容一开始都是字符串。路由 `{user_id}` 匹配到 `"1"` 后，如果 struct 要求 `u32`，axum 做 `"1" → 1u32` 转换。转换失败产生 `PathRejection`。
>
> 字段名要和路由参数名对应：`{user_id}` ↔ `user_id: u32`。

---

## 第二步：自定义 Path extractor + PathError

现在自定义 `Path<T>` extractor，失败时返回带 `location` 字段的 JSON 错误（告诉客户端哪个参数错了）。

````rust
use axum::{
    extract::{path::ErrorKind, rejection::PathRejection, FromRequestParts},
    http::{request::Parts, StatusCode},
    response::IntoResponse,
};

// 自定义 Path 包装器
struct Path<T>(T);

impl<S, T> FromRequestParts<S> for Path<T>
where
    T: DeserializeOwned + Send,
    S: Send + Sync,
{
    type Rejection = (StatusCode, axum::Json<PathError>);

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        match axum::extract::Path::<T>::from_request_parts(parts, state).await {
            Ok(value) => Ok(Self(value.0)),
            Err(rejection) => { /* 转换成 PathError */ }
        }
    }
}

#[derive(Serialize)]
struct PathError {
    message: String,
    location: Option<String>,  // 告诉客户端哪个参数错了
}
````

> **新面孔：`FromRequestParts`**
>
> 和上一章的 `FromRequest` 不同，`FromRequestParts` **只读 parts**（method、uri、headers、路径参数），**不消费 body**。路径参数来自 URL，不需要读 body，所以用 `FromRequestParts`。
>
> 复用 `axum::extract::Path::<T>::from_request_parts` 做实际解析，只定制失败时的响应。

---

## 第三步：ErrorKind 分类——区分 400 和 500

`PathRejection` 里的错误分多种类型（`ErrorKind`），不同类型应该返回不同状态码：

````rust
PathRejection::FailedToDeserializePathParams(inner) => {
    let mut status = StatusCode::BAD_REQUEST;
    let kind = inner.into_kind();
    let body = match &kind {
        // 某个具名参数解析失败，如 user_id
        ErrorKind::ParseErrorAtKey { key, .. } => PathError {
            message: kind.to_string(),
            location: Some(key.clone()),
        },
        // 程序员用了不支持的类型 → 服务端错误
        ErrorKind::UnsupportedType { .. } => {
            status = StatusCode::INTERNAL_SERVER_ERROR;
            PathError { message: kind.to_string(), location: None }
        }
        // 其他错误类型...
    };
    (status, axum::Json(body))
}
````

> **新面孔：`ErrorKind`**
>
> `ErrorKind` 把路径参数错误分类：`ParseErrorAtKey`（某个参数解析失败，能拿到 key 名）、`ParseErrorAtIndex`（按位置失败）、`InvalidUtf8`（非 UTF8）、`UnsupportedType`（程序员用了不支持的类型）等。
>
> `ParseErrorAtKey { key }` 最常见——`key` 就是出错的参数名（如 `"user_id"`），把它放进 `location` 字段让客户端知道改哪个参数。
>
> `UnsupportedType` 是程序员写法错（不是用户输入错），返回 500。

---

## 完整代码

````rust
use axum::{
    extract::{path::ErrorKind, rejection::PathRejection, FromRequestParts},
    http::{request::Parts, StatusCode},
    response::IntoResponse,
    routing::get,
    Router,
};
use serde::{de::DeserializeOwned, Deserialize, Serialize};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app = Router::new().route("/users/{user_id}/teams/{team_id}", get(handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler(Path(params): Path<Params>) -> impl IntoResponse {
    axum::Json(params)
}

#[derive(Debug, Deserialize, Serialize)]
struct Params {
    user_id: u32,
    team_id: u32,
}

struct Path<T>(T);

impl<S, T> FromRequestParts<S> for Path<T>
where
    T: DeserializeOwned + Send,
    S: Send + Sync,
{
    type Rejection = (StatusCode, axum::Json<PathError>);

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        match axum::extract::Path::<T>::from_request_parts(parts, state).await {
            Ok(value) => Ok(Self(value.0)),
            Err(rejection) => {
                let (status, body) = match rejection {
                    PathRejection::FailedToDeserializePathParams(inner) => {
                        let mut status = StatusCode::BAD_REQUEST;
                        let kind = inner.into_kind();
                        let body = match &kind {
                            ErrorKind::WrongNumberOfParameters { .. } => PathError {
                                message: kind.to_string(),
                                location: None,
                            },
                            ErrorKind::ParseErrorAtKey { key, .. } => PathError {
                                message: kind.to_string(),
                                location: Some(key.clone()),
                            },
                            ErrorKind::ParseErrorAtIndex { index, .. } => PathError {
                                message: kind.to_string(),
                                location: Some(index.to_string()),
                            },
                            ErrorKind::ParseError { .. } => PathError {
                                message: kind.to_string(),
                                location: None,
                            },
                            ErrorKind::InvalidUtf8InPathParam { key } => PathError {
                                message: kind.to_string(),
                                location: Some(key.clone()),
                            },
                            ErrorKind::UnsupportedType { .. } => {
                                status = StatusCode::INTERNAL_SERVER_ERROR;
                                PathError {
                                    message: kind.to_string(),
                                    location: None,
                                }
                            }
                            ErrorKind::Message(msg) => PathError {
                                message: msg.clone(),
                                location: None,
                            },
                            _ => PathError {
                                message: format!("Unhandled deserialization error: {kind}"),
                                location: None,
                            },
                        };
                        (status, body)
                    }
                    PathRejection::MissingPathParams(error) => (
                        StatusCode::INTERNAL_SERVER_ERROR,
                        PathError {
                            message: error.to_string(),
                            location: None,
                        },
                    ),
                    _ => (
                        StatusCode::INTERNAL_SERVER_ERROR,
                        PathError {
                            message: format!("Unhandled path rejection: {rejection}"),
                            location: None,
                        },
                    ),
                };

                Err((status, axum::Json(body)))
            }
        }
    }
}

#[derive(Serialize)]
struct PathError {
    message: String,
    location: Option<String>,
}
````

## 运行

````bash
cd examples
cargo run -p example-customize-path-rejection
````

合法路径：

````bash
curl http://127.0.0.1:3000/users/1/teams/2
# {"user_id":1,"team_id":2}
````

非法 `user_id`：

````bash
curl -i http://127.0.0.1:3000/users/abc/teams/2
# 400 {"message":"...","location":"user_id"}
````

## 手写任务

1. 把 `team_id` 类型从 `u32` 改成 `String`，观察 `/teams/abc` 是否成功。
2. 给 `PathError` 加 `code` 字段。
3. 新增路由 `/users/{user_id}`，复用同一个 `Path<T>`。

## 小结

这章分 3 步自定义了路径参数错误：

1. **最小路由**：`{user_id}` 路径参数 + `Params` 结构体，字段名对应。
2. **自定义 Path extractor**：`FromRequestParts`（只读 parts 不读 body），复用 `axum::extract::Path`，定制失败响应。
3. **ErrorKind 分类**：`ParseErrorAtKey` 提取 key 名放进 `location`；`UnsupportedType` 返回 500。

和上一章（ch11）的区别：ch11 自定义 `Json<T>` 用 `FromRequest`（消费 body），这章自定义 `Path<T>` 用 `FromRequestParts`（只读 parts）。

## 源码对照

- `examples/customize-path-rejection/Cargo.toml`
- `examples/customize-path-rejection/src/main.rs`
