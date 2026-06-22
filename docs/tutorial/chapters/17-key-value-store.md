# 17. key-value-store

对应示例：`examples/key-value-store`

上一章的 Todo API 处理结构化 JSON，这章处理**任意字节**——一个内存 key/value store。更重要的是，这章演示 **同一个 Router 里给不同路由加不同 middleware**、admin routes、Bearer auth、compression 等能力的组合。

我们分 5 步从零搭出来。

## Cargo.toml

````toml
[package]
name = "example-key-value-store"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }
tower = { version = "0.5.2", features = ["util", "timeout", "load-shed", "limit"] }
tower-http = { version = "0.6.1", features = [
    "add-extension",
    "auth",
    "compression-full",
    "limit",
    "trace",
] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：最小 KV 服务（GET/POST）

Todo API 的 value 是结构体（`Todo`），KV 的 value 是**任意字节**（文本、图片 bytes 都行），所以 handler 用 `Bytes` 而不是 `Json<T>`。

````rust
use axum::{
    body::Bytes,
    extract::{Path, State},
    http::StatusCode,
    routing::get,
    Router,
};
use std::{
    borrow::Cow,
    collections::HashMap,
    sync::{Arc, RwLock},
};

type SharedState = Arc<RwLock<AppState>>;

#[derive(Default)]
struct AppState {
    db: HashMap<String, Bytes>,
}

#[tokio::main]
async fn main() {
    let shared_state = SharedState::default();

    let app = Router::new()
        .route("/{key}", get(kv_get).post(kv_set))
        .with_state(shared_state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await;
}

// GET /{key}: 读取 value
async fn kv_get(
    Path(key): Path<String>,
    State(state): State<SharedState>,
) -> Result<Bytes, StatusCode> {
    let db = &state.read().unwrap().db;
    if let Some(value) = db.get(&key) {
        Ok(value.clone())
    } else {
        Err(StatusCode::NOT_FOUND)
    }
}

// POST /{key}: 把整个请求 body 存到 key
async fn kv_set(
    Path(key): Path<String>,
    State(state): State<SharedState>,
    bytes: Bytes,
) {
    state.write().unwrap().db.insert(key, bytes);
}
````

> **新面孔：`Bytes`**
>
> `Bytes` 是 axum 提供的"原始字节"类型。和 `Json<T>` 不同，它不解析内容格式——请求 body 是什么字节就存什么字节。响应时直接返回这些字节。
>
> `kv_set` 的第三个参数 `bytes: Bytes` 不需要包装类型（不像 `Json(input): Json<T>`），因为 `Bytes` 本身实现了 extractor trait，axum 直接把请求 body 读成 `Bytes` 绑定到参数。

跑起来验证：

````bash
curl -X POST http://127.0.0.1:3000/hello --data-binary 'world'
curl http://127.0.0.1:3000/hello
# 预期: world
````

---

## 第二步：加列表查询和按路由加不同 layer

Todo API 的 middleware 加在**整个 Router** 外层，所有路由共用。这章演示更精细的做法：**同一个路径的不同方法可以加不同 layer**。

先加 `GET /keys` 列出所有 key：

````rust
async fn list_keys(State(state): State<SharedState>) -> String {
    let db = &state.read().unwrap().db;
    db.keys()
        .map(|key| key.to_string())
        .collect::<Vec<String>>()
        .join("\n")
}
````

现在关键部分——给 GET 响应加压缩，给 POST 请求加 body limit：

````rust
use axum::extract::DefaultBodyLimit;
use axum::handler::Handler;
use tower_http::{
    compression::CompressionLayer,
    limit::RequestBodyLimitLayer,
};

let app = Router::new()
    .route(
        "/{key}",
        get(kv_get.layer(CompressionLayer::new()))   // GET 响应可压缩
            .post_service(
                kv_set
                    .layer((
                        DefaultBodyLimit::disable(),        // 关闭默认 body limit
                        RequestBodyLimitLayer::new(1024 * 5_000),  // 设 5MB 上限
                    ))
                    .with_state(Arc::clone(&shared_state)),
            ),
    )
    .route("/keys", get(list_keys))
    .with_state(Arc::clone(&shared_state));
````

> **新面孔：`get(...).post_service(...)`**
>
> 同一路径的不同方法加不同 layer。GET 加 `CompressionLayer`（响应大 value 时可压缩），POST 加 body limit（value 可能比默认限制大）。
>
> 为什么用 `post_service` 而不是 `post`？因为 `kv_set.layer(...).with_state(...)` 已经变成一个 service（不再是简单 handler）。`post_service` 用来挂 service。

---

## 第三步：加 admin 路由和 Bearer auth

删除操作需要权限保护。我们用 `.nest("/admin", ...)` 挂一组受保护的路由，用 `ValidateRequestHeaderLayer::bearer(...)` 做 Bearer token 认证。

````rust
use tower_http::validate_request::ValidateRequestHeaderLayer;

#[allow(deprecated)]
fn admin_routes() -> Router<SharedState> {
    async fn delete_all_keys(State(state): State<SharedState>) {
        state.write().unwrap().db.clear();
    }

    async fn remove_key(Path(key): Path<String>, State(state): State<SharedState>) {
        state.write().unwrap().db.remove(&key);
    }

    #[allow(deprecated)]
    Router::new()
        .route("/keys", delete(delete_all_keys))
        .route("/key/{key}", delete(remove_key))
        .layer(ValidateRequestHeaderLayer::bearer("secret-token"))
}

// main 里:
let app = Router::new()
    .route(...)
    .route("/keys", get(list_keys))
    .nest("/admin", admin_routes())
    .with_state(Arc::clone(&shared_state));
````

> **新面孔：`nest` + `ValidateRequestHeaderLayer`**
>
> `.nest("/admin", admin_routes())` 把独立 Router 挂到 `/admin` 前缀下。最终路径是 `DELETE /admin/keys` 和 `DELETE /admin/key/{key}`。
>
> `ValidateRequestHeaderLayer::bearer("secret-token")` 要求请求带 `Authorization: Bearer secret-token`，否则被拒绝。没有这个 header 就不能访问 admin 接口。

验证：

````bash
# 删除需要 token
curl -X DELETE http://127.0.0.1:3000/admin/key/hello \
  -H 'authorization: Bearer secret-token'

# 没 token 会被拒绝
curl -i -X DELETE http://127.0.0.1:3000/admin/key/hello
````

---

## 第四步：加全局中间件

在所有路由外层加 timeout、并发限制、过载保护、trace 日志：

````rust
use axum::error_handling::HandleErrorLayer;
use std::{borrow::Cow, time::Duration};
use tower::{BoxError, ServiceBuilder};

async fn handle_error(error: BoxError) -> impl IntoResponse {
    if error.is::<tower::timeout::error::Elapsed>() {
        return (StatusCode::REQUEST_TIMEOUT, Cow::from("request timed out"));
    }
    if error.is::<tower::load_shed::error::Overloaded>() {
        return (StatusCode::SERVICE_UNAVAILABLE, Cow::from("service is overloaded, try again later"));
    }
    (StatusCode::INTERNAL_SERVER_ERROR, Cow::from(format!("Unhandled internal error: {error}")))
}

let app = Router::new()
    .route(...)
    .nest("/admin", admin_routes())
    .layer(
        ServiceBuilder::new()
            .layer(HandleErrorLayer::new(handle_error))
            .load_shed()                           // 过载时快速拒绝
            .concurrency_limit(1024)               // 最多 1024 并发
            .timeout(Duration::from_secs(10))      // 10 秒超时
            .layer(TraceLayer::new_for_http()),    // HTTP 请求日志
    )
    .with_state(Arc::clone(&shared_state));
````

> **新面孔：`load_shed` / `concurrency_limit` / `Cow`**
>
> 和 ch16 的 `timeout` + `HandleErrorLayer` 一样，但这里多了两个中间件：
>
> - `concurrency_limit(1024)`：最多同时处理 1024 个请求。超了的排队或拒绝。
> - `load_shed()`：服务压力过大时**快速拒绝**而不是排队等待（返回 `Overloaded` 错误）。
>
> `HandleErrorLayer::new(handle_error)` 把这些中间件产生的错误转成 HTTP 响应：超时 408、过载 503、未知 500。
>
> `Cow<'_, str>` 可以在静态字符串和动态字符串之间灵活选择，最终变成响应 body。

---

## 完整代码

````rust
use axum::{
    body::Bytes,
    error_handling::HandleErrorLayer,
    extract::{DefaultBodyLimit, Path, State},
    handler::Handler,
    http::StatusCode,
    response::IntoResponse,
    routing::{delete, get},
    Router,
};
use std::{
    borrow::Cow,
    collections::HashMap,
    sync::{Arc, RwLock},
    time::Duration,
};
use tower::{BoxError, ServiceBuilder};
use tower_http::{
    compression::CompressionLayer, limit::RequestBodyLimitLayer, trace::TraceLayer,
    validate_request::ValidateRequestHeaderLayer,
};
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

    let shared_state = SharedState::default();

    let app = Router::new()
        .route(
            "/{key}",
            get(kv_get.layer(CompressionLayer::new()))
                .post_service(
                    kv_set
                        .layer((
                            DefaultBodyLimit::disable(),
                            RequestBodyLimitLayer::new(1024 * 5_000 /* ~5mb */),
                        ))
                        .with_state(Arc::clone(&shared_state)),
                ),
        )
        .route("/keys", get(list_keys))
        .nest("/admin", admin_routes())
        .layer(
            ServiceBuilder::new()
                .layer(HandleErrorLayer::new(handle_error))
                .load_shed()
                .concurrency_limit(1024)
                .timeout(Duration::from_secs(10))
                .layer(TraceLayer::new_for_http()),
        )
        .with_state(Arc::clone(&shared_state));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app).await;
}

type SharedState = Arc<RwLock<AppState>>;

#[derive(Default)]
struct AppState {
    db: HashMap<String, Bytes>,
}

async fn kv_get(
    Path(key): Path<String>,
    State(state): State<SharedState>,
) -> Result<Bytes, StatusCode> {
    let db = &state.read().unwrap().db;

    if let Some(value) = db.get(&key) {
        Ok(value.clone())
    } else {
        Err(StatusCode::NOT_FOUND)
    }
}

async fn kv_set(Path(key): Path<String>, State(state): State<SharedState>, bytes: Bytes) {
    state.write().unwrap().db.insert(key, bytes);
}

async fn list_keys(State(state): State<SharedState>) -> String {
    let db = &state.read().unwrap().db;

    db.keys()
        .map(|key| key.to_string())
        .collect::<Vec<String>>()
        .join("\n")
}

#[allow(deprecated)]
fn admin_routes() -> Router<SharedState> {
    async fn delete_all_keys(State(state): State<SharedState>) {
        state.write().unwrap().db.clear();
    }

    async fn remove_key(Path(key): Path<String>, State(state): State<SharedState>) {
        state.write().unwrap().db.remove(&key);
    }

    #[allow(deprecated)]
    Router::new()
        .route("/keys", delete(delete_all_keys))
        .route("/key/{key}", delete(remove_key))
        .layer(ValidateRequestHeaderLayer::bearer("secret-token"))
}

async fn handle_error(error: BoxError) -> impl IntoResponse {
    if error.is::<tower::timeout::error::Elapsed>() {
        return (StatusCode::REQUEST_TIMEOUT, Cow::from("request timed out"));
    }

    if error.is::<tower::load_shed::error::Overloaded>() {
        return (
            StatusCode::SERVICE_UNAVAILABLE,
            Cow::from("service is overloaded, try again later"),
        );
    }

    (
        StatusCode::INTERNAL_SERVER_ERROR,
        Cow::from(format!("Unhandled internal error: {error}")),
    )
}
````

## 运行

````bash
cd examples
cargo run -p example-key-value-store
````

写入 key：

````bash
curl -X POST http://127.0.0.1:3000/hello --data-binary 'world'
````

读取 key：

````bash
curl http://127.0.0.1:3000/hello
# 预期: world
````

列出 keys：

````bash
curl http://127.0.0.1:3000/keys
````

删除（需要 Bearer token）：

````bash
curl -X DELETE http://127.0.0.1:3000/admin/key/hello \
  -H 'authorization: Bearer secret-token'
````

## 手写任务

跑通后做四个小改动：

1. 把 body limit 从约 5MB 改成 1KB，尝试写入更大的 body。
2. 把 admin token 从 `secret-token` 改成自己的字符串。
3. 给 `GET /keys` 返回值末尾加换行。
4. 新增 `GET /admin/stats`，返回当前 key 数量。

## 小结

这章从零搭了一个 KV 服务，5 步增量构建：

1. **最小 KV**：`Bytes` 接收任意字节，`HashMap<String, Bytes>` 存储。
2. **按路由加 layer**：`get(...).post_service(...)` 让同一路径不同方法加不同 middleware（GET 压缩、POST body limit）。
3. **admin 路由**：`.nest("/admin", ...)` + `ValidateRequestHeaderLayer::bearer(...)` 做 Bearer auth。
4. **全局中间件**：`load_shed` + `concurrency_limit` + `timeout` + `TraceLayer`，`HandleErrorLayer` 把中间件错误转 HTTP 响应。

和 ch16 Todo API 的区别：Todo 用 `Json<T>` 处理结构化数据，KV 用 `Bytes` 处理任意字节；Todo 的 middleware 全局统一，KV 按路由精细控制。

## 源码对照

- `examples/key-value-store/Cargo.toml`
- `examples/key-value-store/src/main.rs`
