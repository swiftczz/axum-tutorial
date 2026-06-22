# 18. key-value-store

对应示例：`examples/key-value-store`

像 Todo API 的另一个方向:Todo 处理结构化 JSON,KV 服务处理**任意字节**。本章实现内存 key/value store,重点演示**按路由加不同 middleware**、admin routes、统一错误处理。

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

## src/main.rs

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

写入 key:

````bash
curl -i -X POST http://127.0.0.1:3000/hello --data-binary 'world'
````

读取 key:

````bash
curl -i http://127.0.0.1:3000/hello
# 预期: world
````

列出 keys:

````bash
curl -i http://127.0.0.1:3000/keys
````

删除指定 key(需要 Bearer token):

````bash
curl -i -X DELETE http://127.0.0.1:3000/admin/key/hello \
  -H 'authorization: Bearer secret-token'
````

删除全部:

````bash
curl -i -X DELETE http://127.0.0.1:3000/admin/keys \
  -H 'authorization: Bearer secret-token'
````

## 解读

### KV 用 `Bytes` 不用 `Json<T>`

KV 的 value 不要求是 JSON,可以是任意字节(文本、图片 bytes 等),所以 handler 用 `Bytes` 接收和返回:

````rust
async fn kv_set(Path(key): Path<String>, State(state): State<SharedState>, bytes: Bytes) {
    state.write().unwrap().db.insert(key, bytes);
}
````

第三个参数 `bytes: Bytes` 把整个请求 body 读成原始字节。`POST /hello` body 为 `world` 就得到 `db["hello"] = b"world"`。

### 共享状态同 Todo 章

````rust
type SharedState = Arc<RwLock<AppState>>;

#[derive(Default)]
struct AppState {
    db: HashMap<String, Bytes>,
}
````

`Arc<RwLock<...>>` 解决多 handler 共享 + 并发读写同步,原理见第 17 章。

### 按路由加不同 layer(本章重点)

````rust
.route(
    "/{key}",
    get(kv_get.layer(CompressionLayer::new()))           // GET 响应可压缩
        .post_service(
            kv_set
                .layer((
                    DefaultBodyLimit::disable(),          // 关掉默认 body limit
                    RequestBodyLimitLayer::new(1024 * 5_000),  // 设 5MB 上限
                ))
                .with_state(Arc::clone(&shared_state)),
        ),
)
````

**同一个路径的不同方法可以加不同 layer**:

- `GET /{key}` 加 `CompressionLayer`:响应可压缩,适合大 value。
- `POST /{key}` 加 body limit:value 可能比默认限制大,先关掉默认再设 5MB。

为什么用 `post_service` 而不是 `post`?因为 `kv_set.layer(...).with_state(...)` 已经变成一个 service(不再是简单 handler),`post_service` 才能挂 service。

### admin routes 用 `.nest` + Bearer auth

````rust
fn admin_routes() -> Router<SharedState> {
    ...
    Router::new()
        .route("/keys", delete(delete_all_keys))
        .route("/key/{key}", delete(remove_key))
        .layer(ValidateRequestHeaderLayer::bearer("secret-token"))
}

// main 里:
.nest("/admin", admin_routes())
````

`admin_routes()` 返回一个独立 Router,挂到 `/admin` 下,最终路径是 `DELETE /admin/keys` 和 `DELETE /admin/key/{key}`。`ValidateRequestHeaderLayer::bearer("secret-token")` 要求请求带 `Authorization: Bearer secret-token`,否则被拒绝。

### 全局 middleware 五件套

````rust
.layer(
    ServiceBuilder::new()
        .layer(HandleErrorLayer::new(handle_error))
        .load_shed()
        .concurrency_limit(1024)
        .timeout(Duration::from_secs(10))
        .layer(TraceLayer::new_for_http()),
)
````

加在所有路由上:

- `TraceLayer`:HTTP 请求日志。
- `timeout(10s)`:请求超时。
- `concurrency_limit(1024)`:最多同时处理 1024 个请求。
- `load_shed()`:压力过大时快速拒绝。
- `HandleErrorLayer`:把 middleware 错误转成 HTTP 响应。

### 统一错误处理

````rust
async fn handle_error(error: BoxError) -> impl IntoResponse {
    if error.is::<tower::timeout::error::Elapsed>() {
        return (StatusCode::REQUEST_TIMEOUT, ...);
    }
    if error.is::<tower::load_shed::error::Overloaded>() {
        return (StatusCode::SERVICE_UNAVAILABLE, ...);
    }
    (StatusCode::INTERNAL_SERVER_ERROR, ...)
}
````

把 Tower middleware 错误分三类:超时 408、过载 503、未知 500。`Cow<'_, str>` 可在静态/动态字符串间灵活选择,最终变成响应 body。

## 手写任务

跑通后做四个小改动:

1. 把 body limit 从约 5MB 改成 1KB,尝试写入更大的 body。
2. 把 admin token 从 `secret-token` 改成自己的字符串。
3. 给 `GET /keys` 返回值末尾加换行。
4. 新增 `GET /admin/stats`,返回当前 key 数量。

## 小结

- `Bytes` 适合接收和返回原始字节数据,不要求 JSON。
- 同一路径的不同方法可以加不同 layer(`get(...).post_service(...)`);`post_service` 用于挂 service 而非 handler。
- admin routes 用 `.nest("/admin", ...)` 组织,加 `ValidateRequestHeaderLayer::bearer(...)` 做 Bearer auth。
- Tower/Tower HTTP layer 可组合:compression、body limit、timeout、concurrency_limit、load_shed、trace。
- `HandleErrorLayer` 把 middleware 错误(超时/过载)转成 HTTP 响应。

## 源码对照

- `examples/key-value-store/Cargo.toml`
- `examples/key-value-store/src/main.rs`
