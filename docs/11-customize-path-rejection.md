# 第 11 章 customize-path-rejection:区分失败原因,分别给出状态码

## 本章你将做出什么

一个路由 `GET /users/{user_id}/teams/{team_id}`,两个参数都是 `u32`。当路径参数解析失败时,返回结构化 JSON,而且**根据失败的具体原因给出不同的状态码和定位信息**:

```bash
curl http://127.0.0.1:3000/users/abc/teams/1
```

```json
{"message":"Invalid URL: Cannot parse `user_id` with value `abc` to a `u32`","location":"user_id"}
```

`location` 字段告诉客户端是**哪个参数**出的问题——这是[第 10 章](10-customize-extractor-error.md)那种笼统转换做不到的。

学完本章你将掌握:

- `PathRejection` 和 `ErrorKind` 两层错误结构,以及如何逐个匹配
- `#[non_exhaustive]` 枚举为什么必须写 `_ =>` 分支
- **一个关键判断:什么错误该返回 `4xx`,什么该返回 `5xx`**
- `FromRequestParts` 的自定义实现([第 10 章](10-customize-extractor-error.md)是 `FromRequest`)

## 前置知识

### 路径参数会怎么失败

[第 04 章](04-templates.md#动态路径与路径参数)用过 `Path<String>`,那时没失败过——任何路径段都是合法的 `String`。换成 `Path<u32>` 或结构体就不一样了。

失败分两大类,这个分类是本章的骨架:

**第一类:客户端的错。** 请求 `/users/abc/teams/1`,但 `user_id` 声明为 `u32`,`abc` 转不过去。这是调用方传错了参数,该返回 `4xx`,并且要告诉对方**哪个参数**错了。

**第二类:程序员的错。** 路由写的是 `/users/{user_id}`(一个参数),提取器却是 `Path<(u32, u32)>`(要两个)。这跟客户端发什么请求毫无关系,**任何请求都会失败**。返回 `4xx` 是在撒谎——它暗示"你改改请求就能成功",而实际上客户端做什么都没用。这类必须返回 `500`。

**区分这两类是本章最有价值的部分。** 很多项目把提取失败一律映射成 `400`,结果线上出现配置错误时,监控看到的是"客户端请求异常率升高",排查方向从一开始就是错的。

### PathRejection:两层结构

`axum::extract::Path` 失败时产生 `PathRejection`,它是个枚举:

| 变体 | 含义 | 该谁负责 |
|---|---|---|
| `FailedToDeserializePathParams` | 参数拿到了,但反序列化失败 | 通常是客户端 |
| `MissingPathParams` | 路由里根本没有路径参数 | **程序员** |

`FailedToDeserializePathParams` 内部还有一层。调用 `.into_kind()` 得到 `ErrorKind`,才知道具体是什么问题:

| `ErrorKind` 变体 | 含义 | 该谁负责 |
|---|---|---|
| `WrongNumberOfParameters` | 路由参数个数与目标类型不匹配 | **程序员** |
| `ParseErrorAtKey { key, .. }` | 具名参数 `key` 转换失败 | 客户端 |
| `ParseErrorAtIndex { index, .. }` | 元组第 `index` 个参数转换失败 | 客户端 |
| `ParseError { .. }` | 单参数转换失败(无键名) | 客户端 |
| `InvalidUtf8InPathParam { key }` | 参数 `key` 的百分号编码解不出合法 UTF-8 | 客户端 |
| `UnsupportedType { .. }` | 目标类型 `Path` 不支持(如嵌套 map) | **程序员** |
| `Message(String)` | serde 报的其它错误 | 视情况 |

带 `key` 或 `index` 的那几个变体是本章 `location` 字段的来源——**它们精确指出了是哪个参数出问题**,这个信息在默认的纯文本拒绝里也有,但客户端得靠解析字符串才能拿到。

### #[non_exhaustive] 与兜底分支

`PathRejection` 和 `ErrorKind` 都标了 `#[non_exhaustive]`。这个属性的含义是:**crate 作者保留将来添加新变体的权利,而这不算破坏性变更。**

后果是你在别的 crate 里 `match` 它时,编译器强制要求一个 `_ =>` 兜底分支——哪怕你已经把当前所有变体都列全了。因为下个版本可能多出一个变体,没有兜底的话你的代码就编译不过了。

这是个合理的设计:错误枚举天生容易增长,`non_exhaustive` 让库能在不发布大版本的前提下细化错误分类。代价是使用方必须处理"未知错误"这种情况。

本章代码里有两个 `_ =>` 分支,都是这个原因。它们的处理方式值得注意:**统一返回 `500`,并把错误原文放进消息里**。理由是——遇到了当前代码不认识的错误类型,你无法判断它该算客户端错还是服务端错,保守起见按服务端错处理,同时保留原文以便排查。

### FromRequestParts 的自定义实现

[第 10 章](10-customize-extractor-error.md#步骤-2做法一手写-fromrequest)实现的是 `FromRequest`(消耗 body),本章实现 `FromRequestParts`(只读元信息)。差别在[第 08 章](08-parse-body-based-on-content-type.md#fromrequest-与-fromrequestparts-的实现细节)讲过:

```rust
async fn from_request_parts(
    parts: &mut Parts,
    state: &S,
) -> Result<Self, Self::Rejection>
```

拿到的是 `&mut Parts` 而非 `Request`。这带来一个实际好处:**这个自定义 `Path` 可以和 `Json`、`Form` 等 body 提取器同时出现在 handler 参数里,而且不必放最后**。如果实现成 `FromRequest`,它就会霸占 body,handler 里再也放不下别的 body 提取器了。

**规则:能用 `FromRequestParts` 就不要用 `FromRequest`。** 只有真正需要读 body 时才选后者。

## 项目搭建

```bash
cargo new customize-path-rejection
cd customize-path-rejection
```

```bash
cargo add axum
cargo add serde --features derive
cargo add tokio --features full
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

本章不需要 `serde_json` —— 用 `axum::Json` 包一个实现了 `Serialize` 的结构体就够了,不用手拼 JSON 值。

## 分步构建

### 步骤 1:用内置 Path,记下默认失败长什么样

**要做什么**:先用 `axum::extract::Path` 跑通正常路径,再制造几种失败作为对比基准。

```rust
// src/main.rs
use axum::{extract::Path, response::IntoResponse, routing::get, Router};
use serde::{Deserialize, Serialize};
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

    // build our application with a route
    let app = Router::new().route("/users/{user_id}/teams/{team_id}", get(handler));

    // run it
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
```

`Path<Params>` 用结构体接收多个路径参数:**字段名必须和路由里的 `{}` 名字对应**。`{user_id}` → `user_id` 字段,`{team_id}` → `team_id` 字段。顺序无所谓,靠名字匹配。

`Params` 同时派生 `Deserialize`(用于从路径解析)和 `Serialize`(用于返回 JSON)。

**跑起来看**:

```bash
cargo run
```

正常请求:

```bash
curl -i http://127.0.0.1:3000/users/1/teams/2
```

```
HTTP/1.1 200 OK
content-type: application/json
content-length: 27

{"user_id":1,"team_id":2}
```

现在制造失败。**类型转换失败**:

```bash
curl -i http://127.0.0.1:3000/users/abc/teams/2
```

```
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8

Invalid URL: Cannot parse `user_id` with value `abc` to a `u32`
```

状态码对,信息也够清楚——**但它是纯文本,而且"哪个参数错了"这个信息埋在句子里**,客户端要拿到它得写正则。这就是要改的地方。

**数值溢出**(`u32` 最大约 42 亿):

```bash
curl -i http://127.0.0.1:3000/users/99999999999/teams/2
```

```
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8

Invalid URL: Cannot parse `user_id` with value `99999999999` to a `u32`
```

同一类错误。

**注意有些"失败"根本到不了提取器**:

```bash
curl -i http://127.0.0.1:3000/users/1
```

```
HTTP/1.1 404 Not Found
```

路径段数量不对,`Router` 那一层就没匹配上,直接 `404`,提取器压根没被调用。**路由匹配和参数解析是两个阶段**,这一点容易混淆。

### 步骤 2:自己的 Path,处理反序列化失败

**要做什么**:定义 `Path<T>` 包装内置提取器,匹配 `ErrorKind` 的各个变体,产出带 `location` 的 JSON。

```rust
// src/main.rs
use axum::{
    extract::{path::ErrorKind, rejection::PathRejection, FromRequestParts},   // 改动
    http::{request::Parts, StatusCode},                                        // 新增
    response::IntoResponse,
    routing::get,
    Router,
};
use serde::{de::DeserializeOwned, Deserialize, Serialize};                     // 改动:加 DeserializeOwned
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// ...(main 不变)

// ...(handler 和 Params 不变)

// 新增:We define our own `Path` extractor that customizes the error from `axum::extract::Path`
struct Path<T>(T);

impl<S, T> FromRequestParts<S> for Path<T>
where
    // these trait bounds are copied from `impl FromRequest for axum::extract::path::Path`
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
                                // this error is caused by the programmer using an unsupported type
                                // (such as nested maps) so respond with `500` instead
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

// 新增:错误响应的结构
#[derive(Serialize)]
struct PathError {
    message: String,
    location: Option<String>,
}
```

**注意 `use` 的变化:`extract::Path` 被移除了。** 现在 `Path` 这个名字指向你自己定义的类型,`handler` 的签名一个字都不用改就自动用上了新的提取器。内置的那个改用全路径 `axum::extract::Path` 调用。

这个"同名遮蔽"的手法很实用:**替换实现时调用方无需改动**。[第 10 章](10-customize-extractor-error.md)做法一也是同一套路。

**trait bound 直接抄了内置实现的。** 注释写得明白:`these trait bounds are copied from impl FromRequest for axum::extract::path::Path`。既然是委托给内置提取器,约束当然要一致——自己另写一套只会导致某些本该能用的类型用不了。

**`inner.into_kind()`** 是从 `FailedToDeserializePathParams` 挖出 `ErrorKind` 的方法。注意它消耗 `inner`,所以下面 `match &kind` 用的是引用。

**`kind.to_string()`** 复用了 `ErrorKind` 的 `Display` 实现——错误描述不用自己写,axum 已经写好了。这和[第 10 章](10-customize-extractor-error.md#小结与下一章预告)那条规则一致:**定制只是换包装,不重写内容**。

**`let mut status = StatusCode::BAD_REQUEST;`** 先给个默认值,只在 `UnsupportedType` 分支里改成 `500`。这是本章的核心判断,下一步展开讲。

**跑起来看**:

```bash
cargo run
```

```bash
curl -i http://127.0.0.1:3000/users/abc/teams/2
```

```
HTTP/1.1 400 Bad Request
content-type: application/json
content-length: 92

{"message":"Invalid URL: Cannot parse `user_id` with value `abc` to a `u32`","location":"user_id"}
```

**`location` 字段直接给出了出错的参数名**,客户端不用解析字符串。这来自 `ErrorKind::ParseErrorAtKey { key, .. }` 分支里的 `key.clone()`。

换第二个参数出错:

```bash
curl -s http://127.0.0.1:3000/users/1/teams/xyz
```

```json
{"message":"Invalid URL: Cannot parse `team_id` with value `xyz` to a `u32`","location":"team_id"}
```

`location` 跟着变了。正常路径不受影响:

```bash
curl -s http://127.0.0.1:3000/users/1/teams/2
```

```json
{"user_id":1,"team_id":2}
```

### 步骤 3:400 还是 500,这是本章的关键判断

**要做什么**:亲手制造一个"程序员的错",验证它确实返回 `500` 而不是 `400`。

前置知识讲了两类失败的区别,现在把它做实。**临时**把 handler 的提取器类型改成元组,但路由不动:

```rust
// 临时改动:路由有两个参数,这里却只要一个
async fn handler(Path(user_id): Path<(u32,)>) -> impl IntoResponse {
    axum::Json(user_id)
}
```

`Params` 结构体暂时用不上,编译器会警告,忽略即可。

**跑起来看**:

```bash
cargo run
```

发一个**完全合法**的请求:

```bash
curl -i http://127.0.0.1:3000/users/1/teams/2
```

```
HTTP/1.1 500 Internal Server Error
content-type: application/json
content-length: 96

{"message":"Wrong number of path arguments for `Path`. Expected 1 but got 2","location":null}
```

**`500`,而且请求本身完全没问题。** 客户端换任何路径都无法成功——这正是"程序员的错"的特征。

等等,这个分支在代码里写的是:

```rust
ErrorKind::WrongNumberOfParameters { .. } => PathError {
    message: kind.to_string(),
    location: None,
},
```

没有改 `status`,所以是默认的 `400`。

> **更好的写法**:`WrongNumberOfParameters` 应该返回 `500` 而不是 `400`。它的含义是"路由声明了 N 个参数,但提取器的目标类型需要 M 个"——这是**代码写错了**,客户端换什么请求都修不好。官方示例只给 `UnsupportedType` 提升了状态码,漏了这一个;两者其实是同一类问题(都在上面那张表的"程序员"一列)。
>
> 改法很简单,和 `UnsupportedType` 那一支一样加一行:
>
> ```rust
> ErrorKind::WrongNumberOfParameters { .. } => {
>     status = StatusCode::INTERNAL_SERVER_ERROR;
>     PathError {
>         message: kind.to_string(),
>         location: None,
>     }
> }
> ```
>
> 练习题 1 会让你动手改。为什么这个数字值得较真,见下面"400 与 500 的划分为什么重要"。

这类不一致正是本章想让你警觉的地方。练习题 1 会让你把它改对。

把 handler 改回 `Path<Params>`。

**再看那两个 `_ =>` 兜底分支。** 前置知识说过 `#[non_exhaustive]` 强制要求它们。它们的处理方式一致:

```rust
_ => (
    StatusCode::INTERNAL_SERVER_ERROR,
    PathError {
        message: format!("Unhandled path rejection: {rejection}"),
        location: None,
    },
),
```

**返回 `500` 并保留错误原文。** 这个选择的逻辑是:代码遇到了它不认识的错误类型,没有依据判断该算谁的错。此时:

- 返回 `4xx` 是在断言"客户端有错",但你并不知道
- 返回 `5xx` 是在说"服务端出了预期外的情况",这个陈述总是成立的——毕竟你的代码确实没覆盖到这种情况

配合保留原文,线上真出现新变体时,日志里能看到 `Unhandled path rejection: ...`,一眼就知道该去更新这段匹配了。

**这是处理 `non_exhaustive` 枚举的通用模式:兜底分支保守处理 + 保留原始信息以便发现。**

### 400 与 500 的划分为什么重要

这个判断影响的不只是一个数字。

**监控和告警的分流。** 绝大多数团队的告警规则是按 `5xx` 比例配的,`4xx` 通常只做统计不告警——因为客户端传错参数是常态,不该半夜叫醒工程师。如果把配置错误映射成 `400`,**部署了一个路由和提取器不匹配的版本,监控上一片绿,只有用户在抱怨**。

**排查方向的第一步。** 看到 `4xx` 升高,工程师会去查"最近哪个客户端改了调用方式";看到 `5xx` 升高,才会去查"我们刚发布了什么"。状态码错了,排查方向从一开始就偏了。

**客户端的重试策略。** 规范的客户端对 `4xx` 不重试(重试也没用),对 `5xx` 会退避重试。把服务端错误标成 `4xx`,客户端就失去了自愈机会。

判断标准可以简化成一句话:**这个错误,客户端改变请求内容有可能修好吗?** 能就是 `4xx`,不能就是 `5xx`。

按这个标准回看本章的表:`ParseErrorAtKey`(换个合法参数就行)、`InvalidUtf8InPathParam`(换个正确编码)是 `4xx`;`WrongNumberOfParameters`、`UnsupportedType`(客户端做什么都没用)是 `5xx`。

[第 14 章](14-error-handling.md)会把这个判断扩展到整个应用的错误处理体系。

## 完整代码

`src/main.rs`:

```rust
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

    // build our application with a route
    let app = Router::new().route("/users/{user_id}/teams/{team_id}", get(handler));

    // run it
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

// We define our own `Path` extractor that customizes the error from `axum::extract::Path`
struct Path<T>(T);

impl<S, T> FromRequestParts<S> for Path<T>
where
    // these trait bounds are copied from `impl FromRequest for axum::extract::path::Path`
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
                                // this error is caused by the programmer using an unsupported type
                                // (such as nested maps) so respond with `500` instead
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
```

## 动手练习

**1. 把 `WrongNumberOfParameters` 改成返回 `500`。**

提示:照着 `UnsupportedType` 分支的写法,在分支里加一行 `status = StatusCode::INTERNAL_SERVER_ERROR;`。改完用步骤 3 的方法(把 handler 临时改成 `Path<(u32,)>`)验证。想一想:如果这个服务已经上线,这个改动会让监控图表发生什么变化?

**2. 在错误响应里加上 `expected_type` 字段。**

提示:`ErrorKind::ParseErrorAtKey` 有个 `expected_type` 字段(用 `..` 忽略掉了),把它取出来放进 `PathError`。这样客户端能明确知道"这个参数应该是 `u32`",而不用从消息里猜。注意 `PathError` 要加一个 `Option<String>` 字段,并考虑没有这个信息的分支该填什么。

**3. 验证 `FromRequestParts` 带来的自由度。**

提示:给路由加一个 `POST` 版本,handler 同时接收自定义的 `Path<Params>` 和 `axum::Json<serde_json::Value>`。因为 `Path` 实现的是 `FromRequestParts`,它可以放在 `Json` **前面**。试着把两者顺序对调,看编译器报什么错——这能让你直观理解[第 08 章](08-parse-body-based-on-content-type.md#fromrequest-与-fromrequestparts-的实现细节)讲的"body 提取器必须放最后"。

## 小结与下一章预告

`PathRejection` 是两层结构:外层区分"反序列化失败"和"路由里没有参数",内层的 `ErrorKind` 才说清具体原因。带 `key` / `index` 的变体给出了出错参数的位置,这是构造 `location` 字段的依据。

两个枚举都是 `#[non_exhaustive]`,必须写 `_ =>` 兜底。兜底的正确姿势是**返回 `500` 并保留错误原文**——遇到不认识的错误类型时,没有依据断言"客户端有错",而"服务端遇到预期外情况"这个陈述总是成立的。

本章最重要的不是代码,是那个判断标准:**客户端改变请求内容有可能修好吗?** 能就 `4xx`,不能就 `5xx`。它决定了告警会不会响、排查方向对不对、客户端该不该重试。参数个数不匹配、类型不受支持这类配置错误,必须是 `5xx`。

`Path` 保证了参数的**类型**正确,但类型正确不等于业务合法——`u32` 类型的 `user_id` 可以是 0,`String` 类型的邮箱可以是"随便乱写"。[第 12 章 validator](12-validator.md) 引入 `validator` crate,在反序列化成功**之后**再加一层业务规则校验,并用 `thiserror` 把两类错误合并进同一个错误类型。你还会看到 axum 的 `Form` 提取器一个容易忽略的行为:**遇到 `GET` 请求时,它读的是查询字符串而不是 body**。
