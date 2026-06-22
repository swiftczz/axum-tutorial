# 12. customize-path-rejection

对应示例：`examples/customize-path-rejection`

本章目标：自定义路径参数解析错误，把 Axum 默认的 `PathRejection` 转成更适合 API 的 JSON 错误响应。

上一章自定义的是 `Json<T>` 解析失败的错误。  
这一章自定义的是 `Path<T>` 解析失败的错误。

## 这个小项目在做什么

这个 example 只有一个接口：

```text
GET /users/{user_id}/teams/{team_id}
```

它期望两个路径参数都是 `u32`：

```rust
struct Params {
    user_id: u32,
    team_id: u32,
}
```

如果请求是：

```text
GET /users/1/teams/2
```

解析成功，返回：

```json
{"user_id":1,"team_id":2}
```

如果请求是：

```text
GET /users/abc/teams/2
```

`abc` 不能解析成 `u32`，默认 `Path` extractor 会失败。  
本章要把这个失败转换成更清楚的 JSON 错误：

```json
{
  "message": "...",
  "location": "user_id"
}
```

请求主线是：

```text
GET /users/{user_id}/teams/{team_id}
-> 自定义 Path<Params> 运行
-> 内部调用 axum::extract::Path<Params>
-> 成功：handler 返回 JSON 参数
-> 失败：把 PathRejection 转成 PathError JSON
```

## 先理解路径参数解析

路由里这段：

```text
/users/{user_id}/teams/{team_id}
```

表示 URL 中有两个变量：

| URL | `user_id` | `team_id` |
| --- | --- | --- |
| `/users/1/teams/2` | `1` | `2` |
| `/users/42/teams/7` | `42` | `7` |

但 URL 里的内容一开始都是字符串。  
如果 Rust 结构体要求 `u32`，Axum 就要做转换：

```text
"1" -> 1u32
"abc" -> 转换失败
```

这个转换失败，就是本章要自定义的错误。

## 文件和依赖

这个 example 有两个文件：

1. `examples/customize-path-rejection/Cargo.toml`：声明 Axum、Serde、Tokio、tracing。
2. `examples/customize-path-rejection/src/main.rs`：实现路由、自定义 `Path<T>`、错误转换和响应。

关键依赖：

- `axum`：提供 `FromRequestParts`、`PathRejection`、`ErrorKind`、`IntoResponse`、`Json`。
- `serde`：让路径参数能反序列化，让错误和成功响应能序列化。
- `tokio`：提供异步运行时。
- `tracing` / `tracing-subscriber`：初始化日志。

## 第一步：注册带路径参数的路由

源码：

````rust
let app = Router::new().route("/users/{user_id}/teams/{team_id}", get(handler));
````

这个路由包含两个路径参数：

```text
{user_id}
{team_id}
```

handler 写成：

````rust
async fn handler(Path(params): Path<Params>) -> impl IntoResponse {
    axum::Json(params)
}
````

注意：这里的 `Path` 不是 `axum::extract::Path`，而是本文件后面自定义的 `Path<T>`。

## 第二步：定义路径参数结构体

源码：

````rust
#[derive(Debug, Deserialize, Serialize)]
struct Params {
    user_id: u32,
    team_id: u32,
}
````

字段名必须和路由参数名对应：

| 路由参数 | struct 字段 |
| --- | --- |
| `{user_id}` | `user_id: u32` |
| `{team_id}` | `team_id: u32` |

`Deserialize` 用于从路径参数反序列化。  
`Serialize` 用于成功时返回 JSON。

## 第三步：定义自己的 Path 包装器

源码：

````rust
struct Path<T>(T);
````

这和前面几章的包装器模式一样：

```text
创建一个自己的 Path<T>
内部复用 axum::extract::Path<T>
失败时改成自己的错误响应
```

handler 中这句：

````rust
async fn handler(Path(params): Path<Params>) -> impl IntoResponse {
````

左边的 `Path(params)` 是模式匹配，用来把包装器里的 `Params` 拿出来。

## 第四步：实现 FromRequestParts

源码框架：

````rust
impl<S, T> FromRequestParts<S> for Path<T>
where
    T: DeserializeOwned + Send,
    S: Send + Sync,
{
    type Rejection = (StatusCode, axum::Json<PathError>);

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        ...
    }
}
````

为什么这里是 `FromRequestParts`，不是 `FromRequest`？

因为路径参数来自请求的 parts，不需要读取 body。

可以这样记：

| 要提取什么 | trait |
| --- | --- |
| path、query、header、method 等不需要 body 的信息 | `FromRequestParts` |
| JSON、Form、Bytes 等需要消费 body 的信息 | `FromRequest` |

上一章读 JSON body 用 `FromRequest`。  
这一章读路径参数，所以用 `FromRequestParts`。

## 第五步：复用 Axum 原始 Path 提取器

核心源码：

````rust
match axum::extract::Path::<T>::from_request_parts(parts, state).await {
    Ok(value) => Ok(Self(value.0)),
    Err(rejection) => {
        ...
    }
}
````

这段做的是：

```text
先让 axum::extract::Path<T> 正常解析
成功：取出 T，包装成自己的 Path<T>
失败：进入错误转换逻辑
```

也就是说，本章没有重写路径解析逻辑。  
它只是复用 Axum 的解析能力，并定制失败时的响应。

## 第六步：把 PathRejection 转成 PathError

错误类型：

````rust
#[derive(Serialize)]
struct PathError {
    message: String,
    location: Option<String>,
}
````

`message` 是错误描述。  
`location` 表示错误发生在哪个参数上，例如 `user_id`。

核心分支：

````rust
PathRejection::FailedToDeserializePathParams(inner) => {
    let mut status = StatusCode::BAD_REQUEST;

    let kind = inner.into_kind();
    let body = match &kind {
        ErrorKind::ParseErrorAtKey { key, .. } => PathError {
            message: kind.to_string(),
            location: Some(key.clone()),
        },
        ...
    };

    (status, body)
}
````

最常见的是 `ParseErrorAtKey`：

```text
某个具名路径参数解析失败
```

例如：

```text
/users/abc/teams/2
```

`abc` 不能解析成 `u32`，所以 `location` 可以设置成 `user_id`。

## 第七步：区分 400 和 500

大多数用户输入错误应该返回 400：

```text
客户端传错路径参数
-> 400 Bad Request
```

但有些错误是程序员写法导致的，例如 `UnsupportedType`：

````rust
ErrorKind::UnsupportedType { .. } => {
    status = StatusCode::INTERNAL_SERVER_ERROR;
    PathError {
        message: kind.to_string(),
        location: None,
    }
}
````

这类问题不是用户输入错，而是服务端代码用了不支持的类型，所以返回 500。

这个区分很重要：

| 错误来源 | 状态码 |
| --- | --- |
| 客户端传了无法解析的路径参数 | 400 |
| 服务端代码使用了不支持的路径参数类型 | 500 |

## 函数职责速查

- `main`：初始化日志，注册带路径参数的路由，启动服务。
- `handler`：用自定义 `Path<Params>` 提取路径参数，成功时返回 JSON。
- `Params`：描述 `user_id` 和 `team_id` 两个路径参数。
- `Path<T>`：自定义路径参数 extractor 包装器。
- `from_request_parts`：复用 `axum::extract::Path<T>`，并把失败转换成 JSON。
- `PathError`：统一的路径参数错误响应结构。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-customize-path-rejection
//! ```

// 引入路径错误类型、PathRejection、FromRequestParts、请求 parts、状态码、响应转换、GET 路由和 Router。
use axum::{
    extract::{path::ErrorKind, rejection::PathRejection, FromRequestParts},
    http::{request::Parts, StatusCode},
    response::IntoResponse,
    routing::get,
    Router,
};
// serde 用于路径参数反序列化和响应序列化。
use serde::{de::DeserializeOwned, Deserialize, Serialize};
// tracing_subscriber 用来初始化日志。
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 初始化日志系统。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 注册带两个路径参数的路由。
    let app = Router::new().route("/users/{user_id}/teams/{team_id}", get(handler));

    // 绑定本机 3000 端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 输出监听地址。
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动 HTTP 服务。
    axum::serve(listener, app).await;
}

// handler 使用自定义 Path<Params> 提取路径参数。
async fn handler(Path(params): Path<Params>) -> impl IntoResponse {
    // 成功时把路径参数作为 JSON 返回。
    axum::Json(params)
}

// 路径参数结构体，字段名和路由里的 {user_id}/{team_id} 对应。
#[derive(Debug, Deserialize, Serialize)]
struct Params {
    user_id: u32,
    team_id: u32,
}

// 自定义 Path extractor，内部包装真正解析出来的 T。
struct Path<T>(T);

// 路径参数不需要读取 body，所以实现 FromRequestParts。
impl<S, T> FromRequestParts<S> for Path<T>
where
    // 这些 trait bound 来自 axum::extract::Path 的实现要求。
    T: DeserializeOwned + Send,
    S: Send + Sync,
{
    // 失败时返回状态码和 JSON 错误 body。
    type Rejection = (StatusCode, axum::Json<PathError>);

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        // 复用 Axum 原生 Path<T> 的解析逻辑。
        match axum::extract::Path::<T>::from_request_parts(parts, state).await {
            // 成功时取出内部值并包装成自己的 Path<T>。
            Ok(value) => Ok(Self(value.0)),
            // 失败时把 PathRejection 转换成自己的 PathError。
            Err(rejection) => {
                let (status, body) = match rejection {
                    // 路径参数反序列化失败，例如 "abc" 不能转成 u32。
                    PathRejection::FailedToDeserializePathParams(inner) => {
                        let mut status = StatusCode::BAD_REQUEST;

                        let kind = inner.into_kind();
                        let body = match &kind {
                            // 参数数量不对。
                            ErrorKind::WrongNumberOfParameters { .. } => PathError {
                                message: kind.to_string(),
                                location: None,
                            },

                            // 具名参数解析失败，例如 user_id。
                            ErrorKind::ParseErrorAtKey { key, .. } => PathError {
                                message: kind.to_string(),
                                location: Some(key.clone()),
                            },

                            // 按位置解析参数时某个 index 失败。
                            ErrorKind::ParseErrorAtIndex { index, .. } => PathError {
                                message: kind.to_string(),
                                location: Some(index.to_string()),
                            },

                            // 泛化的解析错误。
                            ErrorKind::ParseError { .. } => PathError {
                                message: kind.to_string(),
                                location: None,
                            },

                            // 路径参数里有非法 UTF-8。
                            ErrorKind::InvalidUtf8InPathParam { key } => PathError {
                                message: kind.to_string(),
                                location: Some(key.clone()),
                            },

                            // 程序员用了不支持的类型，这属于服务端错误。
                            ErrorKind::UnsupportedType { .. } => {
                                status = StatusCode::INTERNAL_SERVER_ERROR;
                                PathError {
                                    message: kind.to_string(),
                                    location: None,
                                }
                            }

                            // 其他带 message 的错误。
                            ErrorKind::Message(msg) => PathError {
                                message: msg.clone(),
                                location: None,
                            },

                            // 兜底分支，避免未来新增错误类型时完全丢失信息。
                            _ => PathError {
                                message: format!("Unhandled deserialization error: {kind}"),
                                location: None,
                            },
                        };

                        (status, body)
                    }
                    // 缺少路径参数通常是路由配置或提取方式问题，按服务端错误处理。
                    PathRejection::MissingPathParams(error) => (
                        StatusCode::INTERNAL_SERVER_ERROR,
                        PathError {
                            message: error.to_string(),
                            location: None,
                        },
                    ),
                    // 其他 PathRejection 兜底处理。
                    _ => (
                        StatusCode::INTERNAL_SERVER_ERROR,
                        PathError {
                            message: format!("Unhandled path rejection: {rejection}"),
                            location: None,
                        },
                    ),
                };

                // 返回自定义错误响应。
                Err((status, axum::Json(body)))
            }
        }
    }
}

// 统一的路径参数错误响应。
#[derive(Serialize)]
struct PathError {
    message: String,
    location: Option<String>,
}
````

## 运行和验证

运行前先确认 `examples/customize-path-rejection/Cargo.toml` 里的 `axum = { path = "../../axum" }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-customize-path-rejection
````

请求合法路径：

````bash
curl -i http://127.0.0.1:3000/users/1/teams/2
````

预期响应体：

````json
{"user_id":1,"team_id":2}
````

请求非法 `user_id`：

````bash
curl -i http://127.0.0.1:3000/users/abc/teams/2
````

预期状态码是 `400 Bad Request`，响应体类似：

````json
{
  "message": "...",
  "location": "user_id"
}
````

请求非法 `team_id`：

````bash
curl -i http://127.0.0.1:3000/users/1/teams/abc
````

预期 `location` 应该指向 `team_id`。

常见卡点：

- 路径参数来自 URL，不来自 body，所以用 `FromRequestParts`。
- 路由里的 `{user_id}` 要和 `Params` 里的 `user_id` 对应。
- URL 里的参数先是字符串，再由 extractor 转成 `u32`。
- 用户输入导致的解析失败通常是 400；程序员使用不支持类型通常是 500。

## 手写任务

完成本章代码后，试着做三个小改动：

1. 把 `team_id` 类型从 `u32` 改成 `String`，观察 `/teams/abc` 是否还能成功。
2. 给 `PathError` 增加 `code: String` 字段，例如 `INVALID_PATH_PARAM`。
3. 增加一个路由 `/users/{user_id}`，复用同一个自定义 `Path<T>` 思路。

## 本章真正要记住什么

- `Path<T>` 会把 URL 路径参数解析成 Rust 类型。
- 路径参数解析失败会产生 `PathRejection`。
- 自定义路径 extractor 可以复用 `axum::extract::Path<T>`，只改错误响应。
- 不读取 body 的 extractor 应该实现 `FromRequestParts`。
- 错误响应里提供 `location` 能让客户端更容易定位哪个参数错了。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/customize-path-rejection/Cargo.toml`
- `examples/customize-path-rejection/src/main.rs`
