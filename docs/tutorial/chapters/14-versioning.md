# 14. versioning

对应示例：`examples/versioning`

把自定义 extractor 用在一个真实后端场景上:API 版本管理。路径里的 `v1`/`v2`/`v3` 被自定义 extractor 转成 Rust enum `Version`,handler 不用自己 match 字符串。



相比前面章节新引入：**`FromRequestParts`（只读 parts）、`RequestPartsExt::extract`、`Path<HashMap>`**。

## Cargo.toml

````toml
[package]
name = "example-versioning"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

[dev-dependencies]
http-body-util = "0.1.0"
tower = { version = "0.5.2", features = ["util"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 关键概念

> **新面孔：`FromRequestParts`**
>
> 版本 extractor 只读 parts（路径参数），不读 body，用 `FromRequestParts`。`Path<HashMap<String, String>>` 提取所有路径参数。


## 完整代码

````rust
use axum::{
    extract::{FromRequestParts, Path},
    http::{request::Parts, StatusCode},
    response::{Html, IntoResponse, Response},
    routing::get,
    RequestPartsExt, Router,
};
use std::collections::HashMap;
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

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app()).await;
}

fn app() -> Router {
    Router::new().route("/{version}/foo", get(handler))
}

async fn handler(version: Version) -> Html<String> {
    Html(format!("received request with version {version:?}"))
}

#[derive(Debug)]
enum Version {
    V1,
    V2,
    V3,
}

impl<S> FromRequestParts<S> for Version
where
    S: Send + Sync,
{
    type Rejection = Response;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let params: Path<HashMap<String, String>> =
            parts.extract().await.map_err(IntoResponse::into_response)?;

        let version = params
            .get("version")
            .ok_or_else(|| (StatusCode::NOT_FOUND, "version param missing").into_response())?;

        match version.as_str() {
            "v1" => Ok(Version::V1),
            "v2" => Ok(Version::V2),
            "v3" => Ok(Version::V3),
            _ => Err((StatusCode::NOT_FOUND, "unknown version").into_response()),
        }
    }
}
````

## 运行

````bash
cd examples
cargo run -p example-versioning
````

合法版本:

````bash
curl -i http://127.0.0.1:3000/v1/foo
# 预期: received request with version V1

curl -i http://127.0.0.1:3000/v2/foo
# 预期: received request with version V2
````

未知版本:

````bash
curl -i http://127.0.0.1:3000/v4/foo
# 预期 404: unknown version
````

运行测试:

````bash
cargo test -p example-versioning
````

## 解读

### 为什么需要版本

真实 API 会演进。接口字段、校验规则、业务语义变了,就开新版本:`GET /v1/users` → `GET /v2/users`。老客户端访问 v1,新客户端访问 v2。版本可放在路径(`/v1/foo`)、Header(`api-version: v1`)或 Query(`/foo?version=v1`)。本章演示路径版本。

### handler 直接接收 enum

````rust
async fn handler(version: Version) -> Html<String> {
    Html(format!("received request with version {version:?}"))
}
````

参数不是 `Path<String>` 而是 `Version`——版本解析被封装进 extractor,handler 不用关心字符串,也不用自己 match。用 enum 比裸字符串更清楚:编译器能帮你检查 match 是否覆盖所有版本。

### 自定义 `Version` extractor

````rust
impl<S> FromRequestParts<S> for Version
where
    S: Send + Sync,
{
    type Rejection = Response;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let params: Path<HashMap<String, String>> =
            parts.extract().await.map_err(IntoResponse::into_response)?;

        let version = params
            .get("version")
            .ok_or_else(|| (StatusCode::NOT_FOUND, "version param missing").into_response())?;

        match version.as_str() {
            "v1" => Ok(Version::V1),
            ...
        }
    }
}
````

版本来自路径参数,不读 body,所以实现 `FromRequestParts`。用 `RequestPartsExt::extract()` 从 parts 里运行 `Path<HashMap<String, String>>` extractor,把所有路径参数提取成 HashMap,再 match 字符串转 enum。

| 路径值 | 结果 |
| --- | --- |
| `v1` / `v2` / `v3` | `Version::V1` / `V2` / `V3` |
| 其他 | 404 `unknown version` |

### 404 还是 400?

未知版本返回 404(表示"这个版本的接口不存在")还是 400(表示"客户端请求格式有问题"),是设计选择:

- **404**(example 选择):把 `/v4/foo` 理解成"请求一个不存在的资源",REST API 惯例倾向这个。
- **400**:把版本号理解成"客户端发了一个不支持的版本值",更像输入错误。

选哪个都行,但整个 API 要保持一致。

### 更地道的写法

example 用 `Path<HashMap>` + 手写 extractor 绕了一层。如果不需要定制错误响应,可以让 `Version` 直接实现 `Deserialize`,用 `Path<Version>` 一步到位:

````rust
#[derive(Debug, Deserialize)]
#[serde(rename_all = "lowercase")]
enum Version { V1, V2, V3 }

async fn handler(Path(version): Path<Version>) -> Html<String> { ... }
````

`#[serde(rename_all = "lowercase")]` 把 `V1` 映射成 `"v1"`。两种写法取舍:

| 写法 | 优点 | 缺点 |
| --- | --- | --- |
| `Path<Version>`(serde) | 代码少,地道 | 错误响应是 axum 默认 rejection,不易定制 |
| `Path<HashMap>` + 手写 extractor | 完全控制错误响应和状态码 | 代码多 |

要统一错误格式(像第 11 章那种自定义 JSON)走手写路线;不需要定制就用 serde。

## 手写任务

跑通后做三个小改动:

1. 新增 `Version::V4`,让 `/v4/foo` 返回成功。
2. 把未知版本错误从纯文本改成 JSON,例如 `{"error":"unknown version"}`。
3. 新增路由 `/{version}/bar`,复用同一个 `Version` extractor。

## 小结

- API 版本可放在路径里,自定义 extractor 把字符串转换成业务 enum。
- 版本来自路径参数不读 body,用 `FromRequestParts`。
- handler 接收 enum 比裸字符串清晰,编译器能帮你检查 match 覆盖。
- `Path<HashMap<String, String>>` 适合动态读取多个路径参数。
- 不需要定制错误响应时,`Path<Version>`(serde 自动)比手写 extractor 更简洁。

## 源码对照

- `examples/versioning/Cargo.toml`
- `examples/versioning/src/main.rs`
