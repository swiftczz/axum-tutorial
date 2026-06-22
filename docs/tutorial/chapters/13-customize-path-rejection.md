# 13. customize-path-rejection

对应示例：`examples/customize-path-rejection`

上一章自定义 `Json<T>` 解析失败的错误,这一章自定义 `Path<T>` 解析失败的错误。把 axum 默认的 `PathRejection` 转成更适合 API 的 JSON 错误,告诉客户端具体哪个路径参数错了。

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

## src/main.rs

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

    axum::serve(listener, app).await;
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

合法路径:

````bash
curl -i http://127.0.0.1:3000/users/1/teams/2
# 预期: {"user_id":1,"team_id":2}
````

非法 `user_id`:

````bash
curl -i http://127.0.0.1:3000/users/abc/teams/2
````

预期 `400 Bad Request`,响应:

````json
{
  "message": "...",
  "location": "user_id"
}
````

非法 `team_id`:

````bash
curl -i http://127.0.0.1:3000/users/1/teams/abc
# 预期 location 指向 team_id
````

## 解读

### 路径参数解析

路由 `/users/{user_id}/teams/{team_id}` 定义两个路径变量。URL 里的内容一开始都是字符串,如果 struct 要求 `u32`,axum 就要做转换:

```text
"1"   -> 1u32     成功
"abc" -> ?        转换失败 → PathRejection
```

字段名要和路由参数名对应:`{user_id}` ↔ `user_id: u32`,`{team_id}` ↔ `team_id: u32`。

### 自定义 `Path<T>` 包装器

````rust
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
            Err(rejection) => { ... }
        }
    }
}
````

没有重写路径解析逻辑,只是复用 `axum::extract::Path<T>`,定制失败时的响应。handler 里的 `Path` 是这个自定义类型,不是 `axum::extract::Path`。

### 为什么用 `FromRequestParts` 而不是 `FromRequest`

路径参数来自请求的 parts,不需要读 body。对照表:

| 要提取什么 | trait |
| --- | --- |
| path、query、header、method 等不读 body 的 | `FromRequestParts` |
| JSON、Form、Bytes 等消费 body 的 | `FromRequest` |

上一章读 JSON body 用 `FromRequest`,这章读路径参数用 `FromRequestParts`。

### `PathError` 带 `location`

````rust
#[derive(Serialize)]
struct PathError {
    message: String,
    location: Option<String>,
}
````

`location` 标识错误发生在哪个参数上,例如 `user_id`。这让前端更容易定位哪个参数错了。最常见的 `ErrorKind::ParseErrorAtKey { key, .. }` 表示"某个具名路径参数解析失败",把 `key` 塞进 `location`。

### 区分 400 和 500

| 错误来源 | 状态码 |
| --- | --- |
| 客户端传了无法解析的路径参数 | 400 |
| 服务端代码用了不支持的参数类型(`UnsupportedType`) | 500 |

用户输入错是 400,程序员写法错(比如路径参数类型用错)是 500。这个区分很重要。

## 手写任务

跑通后做三个小改动:

1. 把 `team_id` 类型从 `u32` 改成 `String`,观察 `/teams/abc` 是否还能成功。
2. 给 `PathError` 加 `code: String` 字段,例如 `INVALID_PATH_PARAM`。
3. 增加路由 `/users/{user_id}`,复用同一个自定义 `Path<T>`。

## 小结

- `Path<T>` 把 URL 路径参数解析成 Rust 类型,失败产生 `PathRejection`。
- 自定义路径 extractor 可以复用 `axum::extract::Path<T>`,只改错误响应。
- 不读 body 的 extractor 实现 `FromRequestParts`;读 body 的实现 `FromRequest`。
- 错误响应里带 `location` 能帮客户端定位哪个参数错了。
- 用户输入错通常 400,服务端类型用错通常 500。

## 源码对照

- `examples/customize-path-rejection/Cargo.toml`
- `examples/customize-path-rejection/src/main.rs`
