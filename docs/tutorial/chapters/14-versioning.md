# 14. versioning

对应示例：`examples/versioning`

本章目标：用自定义提取器实现 API 版本解析，把路径里的 `v1`、`v2`、`v3` 转成 Rust enum。

前面几章已经学过自定义 extractor。  
这一章把它用在一个真实后端常见问题上：

```text
API 版本管理
```

## 这个小项目在做什么

这个 example 只有一个路由模板：

```text
GET /{version}/foo
```

实际访问可以是：

```text
GET /v1/foo
GET /v2/foo
GET /v3/foo
```

handler 不直接接收字符串，而是接收一个自定义 enum：

````rust
async fn handler(version: Version) -> Html<String>
````

请求主线是：

```text
GET /v1/foo
-> Version extractor 从路径参数里取 version
-> "v1" 转成 Version::V1
-> handler 返回 received request with version V1
```

如果访问未知版本：

```text
GET /v4/foo
-> Version extractor 发现不支持
-> 返回 404 unknown version
```

## 先理解为什么需要版本

真实 API 经常会演进。比如一开始接口是：

```text
GET /v1/users
```

后来返回字段、校验规则、业务语义变了，就可能增加：

```text
GET /v2/users
```

这样老客户端继续访问 `v1`，新客户端访问 `v2`。

版本可以放在很多位置：

| 方式 | 示例 |
| --- | --- |
| 路径 | `/v1/foo` |
| Header | `api-version: v1` |
| Query | `/foo?version=v1` |

本章演示的是路径版本。

## 文件和依赖

这个 example 有两个文件：

1. `examples/versioning/Cargo.toml`：声明 Axum、Tokio、tracing 和测试依赖。
2. `examples/versioning/src/main.rs`：实现 Router、`Version` enum、自定义 extractor 和测试。

关键依赖：

- `axum`：提供 `FromRequestParts`、`Path`、`RequestPartsExt`、`IntoResponse`。
- `tokio`：提供异步运行时和异步测试。
- `tracing` / `tracing-subscriber`：初始化日志。
- `tower`、`http-body-util`：测试里直接调用 Router 和读取响应体。

## 第一步：把 Router 提成 app 函数

源码：

````rust
fn app() -> Router {
    Router::new().route("/{version}/foo", get(handler))
}
````

这里的 `{version}` 是路径参数。

例如：

```text
/v1/foo -> version = "v1"
/v2/foo -> version = "v2"
```

把 Router 放进 `app()` 的好处和前面一样：

```text
main 用它启动服务
测试用它直接发请求
```

## 第二步：handler 直接接收 Version

源码：

````rust
async fn handler(version: Version) -> Html<String> {
    Html(format!("received request with version {version:?}"))
}
````

handler 参数不是：

````rust
Path<String>
````

而是：

````rust
Version
````

这说明版本解析已经被封装进 extractor。  
handler 不需要关心路径字符串，也不需要自己写 match。

## 第三步：定义版本 enum

源码：

````rust
#[derive(Debug)]
enum Version {
    V1,
    V2,
    V3,
}
````

用 enum 表示版本，比用裸字符串更清楚：

```text
"v1" -> Version::V1
"v2" -> Version::V2
"v3" -> Version::V3
```

好处是：

- handler 里不会到处散落字符串比较。
- 编译器能帮你检查 match 是否覆盖所有版本。
- 后续版本逻辑可以围绕 enum 组织。

## 第四步：实现 FromRequestParts

源码框架：

````rust
impl<S> FromRequestParts<S> for Version
where
    S: Send + Sync,
{
    type Rejection = Response;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        ...
    }
}
````

为什么是 `FromRequestParts`？

因为版本来自路径参数，不需要读取请求 body。和第 10、12 章同理：**只读 parts（method/uri/headers/路径参数）的 extractor 用 `FromRequestParts`，需要消费 body 的用 `FromRequest`**。两者区别在附录 A0 和第 10 章有详细说明，这里不再重复。

## 第五步：从路径参数里取 version

核心源码：

````rust
let params: Path<HashMap<String, String>> =
    parts.extract().await.map_err(IntoResponse::into_response)?;

let version = params
    .get("version")
    .ok_or_else(|| (StatusCode::NOT_FOUND, "version param missing").into_response())?;
````

这里用了 `RequestPartsExt::extract()`，从 parts 里运行另一个 extractor：

```text
Path<HashMap<String, String>>
```

它会把所有路径参数提取成 HashMap：

```text
{
  "version": "v1"
}
```

如果没有 `version` 这个参数，就返回：

```text
404 version param missing
```

这个情况一般表示路由和 extractor 没配对好。

## 第六步：把字符串转成 enum

源码：

````rust
match version.as_str() {
    "v1" => Ok(Version::V1),
    "v2" => Ok(Version::V2),
    "v3" => Ok(Version::V3),
    _ => Err((StatusCode::NOT_FOUND, "unknown version").into_response()),
}
````

这是本章最直观的转换：

| 路径值 | 结果 |
| --- | --- |
| `v1` | `Version::V1` |
| `v2` | `Version::V2` |
| `v3` | `Version::V3` |
| 其他 | 404 `unknown version` |

这里把未知版本当成 404。  
含义是：

```text
这个版本的接口不存在
```

### 404 还是 400？怎么选

真实项目里，未知版本到底返回 404 还是 400，是个有意义的设计选择。判断依据是**你想表达什么语义**：

- **404 Not Found**：表示"这个版本的接口根本不存在"。example 选了 404，因为它把 `/v4/foo` 理解成"请求一个不存在的资源"——v4 这条接口压根没有。对客户端来说，升级到 v4 没用，必须改回 v1/v2/v3。
- **400 Bad Request**：表示"客户端请求格式有问题"。如果你把版本号理解成"客户端发了一个不支持的版本值"，那它更像输入错误。

选哪个都行，但要在整个 API 里保持一致。REST API 的惯例更倾向 404（因为 `/v4/foo` 作为路径确实不存在），而 GraphQL 风格的 API 可能倾向 400。

### 更地道的写法：让 Version 自己实现 Deserialize

example 用 `Path<HashMap<String, String>>` 再手动 match，绕了一层。其实更地道的写法是让 `Version` 直接实现 `serde::Deserialize`（或 `FromStr`），然后用 `Path<Version>` 一步到位：

````rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
#[serde(rename_all = "lowercase")]
enum Version {
    V1,
    V2,
    V3,
}

// handler 直接用 Path<Version>
async fn handler(Path(version): Path<Version>) -> Html<String> {
    Html(format!("received request with version {version:?}"))
}
````

这样 `/v1/foo` 会自动反序列化成 `Version::V1`（`#[serde(rename_all = "lowercase")]` 把 `V1` 映射成 `"v1"`），未知版本时 serde 会返回反序列化错误（默认 400）。

那为什么 example 偏偏用 HashMap + 手写 extractor？因为**它想完全控制错误响应**：自定义 404 文案、自定义状态码、在转换过程中插入其他逻辑（比如记日志）。手写 extractor 的灵活性正在于此。

两种写法的取舍：

| 写法 | 优点 | 缺点 |
| --- | --- | --- |
| `Path<Version>`（serde 自动） | 代码少，地道 | 错误响应是 axum 默认的 rejection，不易定制 |
| `Path<HashMap>` + 手写 extractor（example 做法） | 完全控制错误响应和状态码 | 代码多 |

真实项目里，如果你不需要定制错误响应，用 `Path<Version>` 更简洁；如果你要统一错误格式（比如第 11 章那种自定义 JSON 错误），就走 example 这条手写路线。

## 第七步：理解测试

源码里有两个测试：

- `test_v1`：请求 `/v1/foo`，预期 200 和 `Version::V1`。
- `test_v4`：请求 `/v4/foo`，预期 404 和 `unknown version`。

这两个测试覆盖了：

```text
支持的版本
不支持的版本
```

## 函数职责速查

- `main`：初始化日志，调用 `app()`，绑定端口并启动服务。
- `app`：注册 `GET /{version}/foo`。
- `handler`：接收已经解析好的 `Version`，返回 HTML。
- `Version`：表示支持的 API 版本。
- `from_request_parts`：从路径参数中读取 version，并转换成 enum。
- `test_v1`：验证支持的版本。
- `test_v4`：验证未知版本返回 404。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-versioning
//! ```

// 引入 FromRequestParts、Path、请求 parts、状态码、Html、响应转换、GET 路由和 Router。
use axum::{
    extract::{FromRequestParts, Path},
    http::{request::Parts, StatusCode},
    response::{Html, IntoResponse, Response},
    routing::get,
    RequestPartsExt, Router,
};
// HashMap 用来接收路径参数。
use std::collections::HashMap;
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

    // 构造应用路由。
    let app = app();

    // 绑定本机 3000 端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 输出监听地址。
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动 HTTP 服务。
    axum::serve(listener, app).await;
}

// 构造 Router，方便 main 和测试复用。
fn app() -> Router {
    Router::new().route("/{version}/foo", get(handler))
}

// handler 直接接收自定义 Version extractor。
async fn handler(version: Version) -> Html<String> {
    Html(format!("received request with version {version:?}"))
}

// 支持的 API 版本。
#[derive(Debug)]
enum Version {
    V1,
    V2,
    V3,
}

// Version 来自路径参数，不读取 body，所以实现 FromRequestParts。
impl<S> FromRequestParts<S> for Version
where
    S: Send + Sync,
{
    // 失败时返回完整 Response。
    type Rejection = Response;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        // 从路径参数中提取所有参数到 HashMap。
        let params: Path<HashMap<String, String>> =
            parts.extract().await.map_err(IntoResponse::into_response)?;

        // 读取名为 version 的路径参数。
        let version = params
            .get("version")
            .ok_or_else(|| (StatusCode::NOT_FOUND, "version param missing").into_response())?;

        // 把字符串版本号转换成 Rust enum。
        match version.as_str() {
            "v1" => Ok(Version::V1),
            "v2" => Ok(Version::V2),
            "v3" => Ok(Version::V3),
            _ => Err((StatusCode::NOT_FOUND, "unknown version").into_response()),
        }
    }
}
````

## 运行和验证

运行前先确认 `examples/versioning/Cargo.toml` 里的 `axum = { path = "../../axum" }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-versioning
````

请求 v1：

````bash
curl -i http://127.0.0.1:3000/v1/foo
````

预期响应体：

````text
received request with version V1
````

请求 v2：

````bash
curl -i http://127.0.0.1:3000/v2/foo
````

预期响应体：

````text
received request with version V2
````

请求未知版本：

````bash
curl -i http://127.0.0.1:3000/v4/foo
````

预期状态码是 `404 Not Found`，响应体：

````text
unknown version
````

运行测试：

````bash
cd examples
cargo test -p example-versioning
````

常见卡点：

- `Version` 来自路径参数，所以实现 `FromRequestParts`。
- 路由里的 `{version}` 必须和 `params.get("version")` 对应。
- `Path<HashMap<String, String>>` 适合动态读取多个路径参数。
- 未知版本返回 404，是一种 API 设计选择，不是唯一答案。

## 手写任务

完成本章代码后，试着做三个小改动：

1. 新增 `Version::V4`，让 `/v4/foo` 返回成功。
2. 把未知版本错误从纯文本改成 JSON，例如 `{"error":"unknown version"}`。
3. 新增一个路由 `/{version}/bar`，复用同一个 `Version` extractor。

## 本章真正要记住什么

- API 版本可以放在路径里，例如 `/v1/foo`。
- 自定义 extractor 可以把字符串路径参数转换成业务 enum。
- 不读取 body 的 extractor 使用 `FromRequestParts`。
- handler 接收 `Version`，比接收裸字符串更清晰。
- 版本不存在时应该返回明确错误。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/versioning/Cargo.toml`
- `examples/versioning/src/main.rs`
