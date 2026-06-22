# 14. versioning

对应示例：`examples/versioning`

API 演进时常见需求：**同一个接口同时支持多个版本**（`/v1/users`、`/v2/users`）。这章用自定义 extractor 把"解析版本"封装成 `Version` 类型，handler 直接声明 `version: Version` 就拿到枚举值，避免到处写 string 匹配。

分 2 步：先写 `Version` extractor（从 URL 路径解析 v1/v2/v3 枚举），再加错误处理（未知版本返回 404）。

相比前面章节新引入：**自定义 `FromRequestParts` extractor 解析路径参数、`Path<HashMap<String, String>>`、枚举作为 extractor 返回值**。

## Cargo.toml

````toml
[package]
name = "example-versioning"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["macros"] }
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：`Version` extractor——从路径解析枚举

核心是写一个 `Version` 枚举（V1/V2/V3），实现 `FromRequestParts` 从 URL 路径参数 `{version}` 解析出来。handler 写 `version: Version` 自动提取。

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
    // rejection 是 Response 类型——可以构造任意状态码/消息的响应
    type Rejection = Response;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        // 从路径参数提取 version
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

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(/* tracing 初始化 */)
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app = Router::new().route("/{version}/foo", get(handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler(version: Version) -> Html<String> {
    Html(format!("received request with version {version:?}"))
}
````

验证：

````bash
cd examples
cargo run -p example-versioning

curl http://127.0.0.1:3000/v1/foo    # received request with version V1
curl http://127.0.0.1:3000/v2/foo    # received request with version V2
curl http://127.0.0.1:3000/v3/foo    # received request with version V3
curl http://127.0.0.1:3000/v4/foo    # unknown version（404）
````

> **新面孔：枚举作为 Extractor**
>
> 前面章节 extractor 都是 struct（`User`、`Json<T>`）。这里 `Version` 是枚举——任何类型只要实现 `FromRequestParts`（或 `FromRequest`）就能当 extractor。
>
> 好处：handler 写 `version: Version` 拿到的是**类型安全**的枚举值，编译期保证不会写错版本字符串。

> **新面孔：`Path<HashMap<String, String>>`**
>
> 第 16 章 todos 用过 `Path<u32>`（单参数按位置）。这章用 `Path<HashMap<String, String>>`——按**名字**提取所有路径参数。
>
> 路由 `/{version}/foo` 定义了名为 `version` 的参数。`params.get("version")` 按名字取值。这种方式适合参数多或不确定顺序的场景。

> **新面孔：`type Rejection = Response`**
>
> 第 11 章 `ServerError` rejection 是固定类型。这里 rejection 直接是 `Response`——最灵活的 rejection 类型，可以构造任意响应。
>
> `(StatusCode::NOT_FOUND, "unknown version").into_response()` 用元组 `IntoResponse` 构造 404 响应。`.into_response()` 把任何 `IntoResponse` 类型转成 `Response`。

> **新面孔：`RequestPartsExt::extract`**
>
> 在 `FromRequestParts` 的实现里，`parts.extract::<T>().await` 跑其他 parts 类 extractor。这里在 `Version` 的 `from_request_parts` 里又跑了 `Path<HashMap>` extractor——避免手写 URL 解析。
>
> `extract` 是 `RequestPartsExt` trait 提供的方法，必须 `use axum::RequestPartsExt;` 才能用。

---

## 第二步：理解错误处理链

```text
请求 GET /v4/foo
  → axum 路由匹配 /{version}/foo，version="v4"
  → handler 进入前，提取 Version extractor
  → Version::from_request_parts 解析 "v4"
  → match "v4" 走 _ 分支 → Err((StatusCode::NOT_FOUND, "unknown version").into_response())
  → handler 不执行，直接返回这个 Response 给客户端
```

axum 看到 extractor 返回 `Err(rejection)`，直接把 rejection 当响应返回，**handler 根本不会被调用**。所以这章 handler 永远拿不到无效版本——校验在进入 handler 前完成。

---

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

    // build our application with some routes
    let app = app();

    // run it
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
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

#[cfg(test)]
mod tests;
````

## 运行

````bash
cd examples
cargo run -p example-versioning

curl http://127.0.0.1:3000/v1/foo   # received request with version V1
curl http://127.0.0.1:3000/v4/foo   # unknown version (404)
````

## 解读

### API 版本化的几种方案

| 方案 | URL | 优点 | 缺点 |
| --- | --- | --- | --- |
| **路径版本（这章）** | `/v1/users` | 直观、缓存友好 | 改动 URL |
| Header 版本 | `/users` + `Accept: application/vnd.x.v1+json` | URL 干净 | 客户端要会发 header |
| Query 版本 | `/users?version=1` | URL 简单 | 缓存不友好 |
| Host 版本 | `v1.api.example.com/users` | 完全隔离 | DNS/证书管理麻烦 |

这章用路径版本——最常见、最直观。

### 自定义 extractor 模式

这章 `Version` extractor 模式可复用——任何"从请求某部分解析出枚举"的场景都能这么写：

- `Locale` extractor：从 `Accept-Language` 解析 en/zh/ja
- `SortOrder` extractor：从 query 参数解析 asc/desc
- `Pagination` extractor：从 query 解析 page/page_size

核心是 `FromRequestParts` + 内部用已有 extractor（`Path`/`Query`）解析。

## 常见问题

**为什么 rejection 用 `Response` 而不是自定义错误类型？** `Response` 是最灵活的——可以任意构造。这章错误简单（"unknown version"），不需要复杂错误类型。如果错误复杂（多种错误码、JSON body），用自定义错误类型 + `IntoResponse` 更好（参考 ch19 error-handling）。

**`Path<HashMap>` 和 `Path<Struct>` 区别？** HashMap 灵活（参数顺序无所谓）但没类型检查；Struct 类型安全但要 derive Deserialize。这章参数只有一个，HashMap 简单。

**怎么处理废弃版本？** 加 `Version::Deprecated` 变体，匹配时返回 410 Gone + 提示升级。

## 手写任务

1. 加 `Version::V4`，行为和 V3 不同（返回不同响应）。
2. 加 `Deprecated` 标记：V1 返回 `Deprecation` header 警告。
3. 写 `Locale` extractor 从 `Accept-Language` 解析。
4. 把 rejection 改成 JSON 格式：`{"error": "unknown version", "version": "v4"}`。

## 小结

这章用 1 个 extractor 讲了 API 版本化：

1. **`Version` 枚举**：V1/V2/V3 + `FromRequestParts` 实现，从 URL 路径解析。
2. **错误处理**：未知版本 rejection 直接返回 404 响应，handler 不被调用。

核心模式：**枚举 + `FromRequestParts` + 内部用 `Path`/`Query` 解析**，把"字符串匹配"封装成类型安全的 extractor。这套模式适用于 locale、排序、分页等任意"从请求解析枚举"的场景。

## 源码对照

- `examples/versioning/Cargo.toml`
- `examples/versioning/src/main.rs`
- `examples/versioning/tests/tests.rs`
