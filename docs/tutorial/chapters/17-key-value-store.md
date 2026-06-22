# 17. key-value-store

对应示例：`examples/key-value-store`

本章目标：手写内存 KV 服务，综合理解状态、原始 `Bytes` 请求体、按路由加 middleware、admin routes 和统一错误处理。

这一章像是前面 Todo API 的另一个方向：  
Todo API 处理结构化 JSON，KV 服务处理任意字节。

## 这个小项目在做什么

这个 example 实现一个简单内存 key/value store：

| 方法 | 路径 | 作用 |
| --- | --- | --- |
| `GET` | `/{key}` | 读取 key 对应的 bytes |
| `POST` | `/{key}` | 把请求 body 存到 key |
| `GET` | `/keys` | 列出所有 key |
| `DELETE` | `/admin/keys` | 删除所有 key，需要 Bearer token |
| `DELETE` | `/admin/key/{key}` | 删除指定 key，需要 Bearer token |

请求主线示例：

```text
POST /hello
body: world
-> kv_set 读取 Path(key) 和 Bytes body
-> 写入 HashMap

GET /hello
-> kv_get 读取 HashMap
-> 返回 Bytes
```

这个 example 的重点是：同一个 Router 里可以针对不同路由和不同 handler 添加不同 middleware。

## 先理解 KV 服务

KV 是 key/value 的缩写：

```text
key   -> value
"foo" -> b"hello"
"img" -> 图片 bytes
```

和 Todo API 不同，KV 的 value 不要求是 JSON。  
它可以是任意字节，所以 handler 使用：

```rust
Bytes
```

而不是：

```rust
Json<T>
```

## 文件和依赖

这个 example 有两个文件：

1. `examples/key-value-store/Cargo.toml`：声明 Axum、Tower、tower-http、Tokio、tracing。
2. `examples/key-value-store/src/main.rs`：实现内存状态、KV handler、admin routes 和 middleware。

关键依赖：

- `axum`：提供 `Bytes`、`Path`、`State`、`Router`、`DefaultBodyLimit`。
- `tower`：提供 `ServiceBuilder`、timeout、load shed、concurrency limit。
- `tower-http`：提供压缩、请求体限制、trace、Bearer auth。
- `tokio`、`tracing`、`tracing-subscriber`：运行时和日志。

## 第一步：定义共享状态

源码：

````rust
type SharedState = Arc<RwLock<AppState>>;

#[derive(Default)]
struct AppState {
    db: HashMap<String, Bytes>,
}
````

状态结构是：

```text
HashMap<String, Bytes>
```

也就是：

```text
key 是字符串
value 是原始字节
```

外面包 `Arc<RwLock<...>>` 的原因和 Todo 章一样：

- `Arc`：多个 handler 共享同一份状态。
- `RwLock`：并发读写 HashMap 时做同步保护。

## 第二步：创建 Router 并共享 state

源码：

````rust
let shared_state = SharedState::default();

let app = Router::new()
    ...
    .with_state(Arc::clone(&shared_state));
````

`shared_state` 会被主路由和部分 service 共享。

这里出现了两次 `Arc::clone(&shared_state)`。  
`Arc::clone` 不是复制整个 HashMap，而是增加共享引用计数，让多个地方指向同一份数据。

## 第三步：实现读取 key

源码：

````rust
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
````

这个 handler 做两件事：

- `Path(key)` 从 URL 里提取 key。
- `State(state)` 访问共享 HashMap。

如果找到 value，就返回 `Bytes`。  
如果找不到，就返回 404。

`Bytes` 可以直接作为响应体返回。

## 第四步：实现写入 key

源码：

````rust
async fn kv_set(Path(key): Path<String>, State(state): State<SharedState>, bytes: Bytes) {
    state.write().unwrap().db.insert(key, bytes);
}
````

这里的第三个参数是：

```rust
bytes: Bytes
```

它会把整个请求 body 读成原始字节。

请求例子：

```text
POST /hello
body: world
```

结果：

```text
db["hello"] = b"world"
```

这个 handler 没有显式返回值，成功时就是空响应。

## 第五步：给不同 handler 加不同 layer

核心路由：

````rust
.route(
    "/{key}",
    get(kv_get.layer(CompressionLayer::new()))
        .post_service(
            kv_set
                .layer((
                    DefaultBodyLimit::disable(),
                    RequestBodyLimitLayer::new(1024 * 5_000),
                ))
                .with_state(Arc::clone(&shared_state)),
        ),
)
````

这里有两个特别点。

第一，`GET /{key}` 加了压缩：

```text
kv_get.layer(CompressionLayer::new())
```

读取大 value 时，响应可以被压缩。

第二，`POST /{key}` 加了请求体大小限制：

```text
DefaultBodyLimit::disable()
RequestBodyLimitLayer::new(1024 * 5_000)
```

因为 value 可能比默认 body limit 大，所以先关闭默认限制，再设置约 5MB 上限。

为什么这里用 `post_service`？

因为 `kv_set.layer(...).with_state(...)` 已经变成一个 service。  
`post_service` 可以把这个 service 挂到 POST 方法上。

## 第六步：列出所有 key

源码：

````rust
async fn list_keys(State(state): State<SharedState>) -> String {
    let db = &state.read().unwrap().db;

    db.keys()
        .map(|key| key.to_string())
        .collect::<Vec<String>>()
        .join("\n")
}
````

这个 handler 返回纯文本：

```text
key1
key2
key3
```

注意：`HashMap` 的 key 顺序不稳定，所以不要依赖返回顺序。

## 第七步：admin routes

源码：

````rust
fn admin_routes() -> Router<SharedState> {
    async fn delete_all_keys(State(state): State<SharedState>) {
        state.write().unwrap().db.clear();
    }

    async fn remove_key(Path(key): Path<String>, State(state): State<SharedState>) {
        state.write().unwrap().db.remove(&key);
    }

    Router::new()
        .route("/keys", delete(delete_all_keys))
        .route("/key/{key}", delete(remove_key))
        .layer(ValidateRequestHeaderLayer::bearer("secret-token"))
}
````

这组路由会被挂到 `/admin` 下：

````rust
.nest("/admin", admin_routes())
````

所以最终路径是：

```text
DELETE /admin/keys
DELETE /admin/key/{key}
```

admin 路由加了 Bearer token 校验：

```text
Authorization: Bearer secret-token
```

没有这个 header，就不能访问 admin 接口。

## 第八步：全局 middleware

源码：

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

这组 middleware 加在所有路由上：

- `TraceLayer`：打印 HTTP 请求日志。
- `timeout`：请求超过 10 秒就超时。
- `concurrency_limit(1024)`：最多同时处理 1024 个请求。
- `load_shed()`：服务压力过大时快速拒绝请求。
- `HandleErrorLayer`：把 middleware 错误转成 HTTP 响应。

## 第九步：统一处理 middleware 错误

源码：

````rust
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

这段把 Tower middleware 的错误分成三类：

| 错误 | 状态码 |
| --- | --- |
| 超时 | `408 Request Timeout` |
| 过载 | `503 Service Unavailable` |
| 未知内部错误 | `500 Internal Server Error` |

`Cow<'_, str>` 可以在静态字符串和动态字符串之间灵活选择。  
新手第一遍只要知道它最后会变成响应 body。

## 函数职责速查

- `main`：初始化日志，创建共享状态，注册路由、admin routes 和 middleware。
- `SharedState`：共享内存状态类型。
- `AppState`：保存真正的 HashMap。
- `kv_get`：读取 key，返回 bytes 或 404。
- `kv_set`：读取原始 request body bytes，写入 key。
- `list_keys`：列出所有 key。
- `admin_routes`：返回受 Bearer auth 保护的管理路由。
- `handle_error`：把 timeout、过载等 middleware 错误转成响应。

## 带中文注释的手写版

````rust
//! 简单内存 key/value store，演示 Axum 多种能力组合。
//!
//! Run with:
//!
//! ```not_rust
//! cargo run -p example-key-value-store
//! ```

// 引入 Bytes、错误处理 layer、extractor、Handler、状态码、响应转换、路由函数和 Router。
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
// Cow 用于返回静态或动态错误文本，HashMap 存储 KV，Arc/RwLock 共享状态。
use std::{
    borrow::Cow,
    collections::HashMap,
    sync::{Arc, RwLock},
    time::Duration,
};
// Tower 提供错误类型和 ServiceBuilder。
use tower::{BoxError, ServiceBuilder};
// tower-http 提供压缩、请求体限制、trace 和 Bearer auth。
use tower_http::{
    compression::CompressionLayer, limit::RequestBodyLimitLayer, trace::TraceLayer,
    validate_request::ValidateRequestHeaderLayer,
};
// tracing_subscriber 用来初始化日志。
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 初始化日志系统。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 创建共享状态。
    let shared_state = SharedState::default();

    // 组合 KV 路由、admin 路由和全局 middleware。
    let app = Router::new()
        .route(
            "/{key}",
            // GET /{key} 读取 value，并对响应做压缩。
            get(kv_get.layer(CompressionLayer::new()))
                // POST /{key} 写入 value，并设置 body limit。
                .post_service(
                    kv_set
                        .layer((
                            DefaultBodyLimit::disable(),
                            RequestBodyLimitLayer::new(1024 * 5_000 /* ~5mb */),
                        ))
                        .with_state(Arc::clone(&shared_state)),
                ),
        )
        // GET /keys 列出所有 key。
        .route("/keys", get(list_keys))
        // 把 admin 路由挂到 /admin 下。
        .nest("/admin", admin_routes())
        // 给所有路由添加 middleware。
        .layer(
            ServiceBuilder::new()
                .layer(HandleErrorLayer::new(handle_error))
                .load_shed()
                .concurrency_limit(1024)
                .timeout(Duration::from_secs(10))
                .layer(TraceLayer::new_for_http()),
        )
        // 给主 Router 设置共享 state。
        .with_state(Arc::clone(&shared_state));

    // 绑定本机 3000 端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 输出监听地址。
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动 HTTP 服务。
    axum::serve(listener, app).await;
}

// 共享状态类型。
type SharedState = Arc<RwLock<AppState>>;

// 应用状态：真正的数据存在 db 里。
#[derive(Default)]
struct AppState {
    db: HashMap<String, Bytes>,
}

// GET /{key}：读取 key 对应的 value。
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

// POST /{key}：把整个请求 body 作为 Bytes 存入 key。
async fn kv_set(Path(key): Path<String>, State(state): State<SharedState>, bytes: Bytes) {
    state.write().unwrap().db.insert(key, bytes);
}

// GET /keys：列出所有 key。
async fn list_keys(State(state): State<SharedState>) -> String {
    let db = &state.read().unwrap().db;

    db.keys()
        .map(|key| key.to_string())
        .collect::<Vec<String>>()
        .join("\n")
}

// admin routes：删除 key，需要 Bearer token。
#[allow(deprecated)]
fn admin_routes() -> Router<SharedState> {
    // DELETE /admin/keys：删除所有 key。
    async fn delete_all_keys(State(state): State<SharedState>) {
        state.write().unwrap().db.clear();
    }

    // DELETE /admin/key/{key}：删除指定 key。
    async fn remove_key(Path(key): Path<String>, State(state): State<SharedState>) {
        state.write().unwrap().db.remove(&key);
    }

    #[allow(deprecated)]
    Router::new()
        .route("/keys", delete(delete_all_keys))
        .route("/key/{key}", delete(remove_key))
        // admin 路由需要 Authorization: Bearer secret-token。
        .layer(ValidateRequestHeaderLayer::bearer("secret-token"))
}

// 统一处理 Tower middleware 错误。
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

## 运行和验证

运行前先确认 `examples/key-value-store/Cargo.toml` 里的 `axum = { path = "../../axum" }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-key-value-store
````

写入 key：

````bash
curl -i -X POST http://127.0.0.1:3000/hello \
  --data-binary 'world'
````

读取 key：

````bash
curl -i http://127.0.0.1:3000/hello
````

预期响应体：

````text
world
````

列出 keys：

````bash
curl -i http://127.0.0.1:3000/keys
````

删除指定 key：

````bash
curl -i -X DELETE http://127.0.0.1:3000/admin/key/hello \
  -H 'authorization: Bearer secret-token'
````

删除全部 key：

````bash
curl -i -X DELETE http://127.0.0.1:3000/admin/keys \
  -H 'authorization: Bearer secret-token'
````

常见卡点：

- admin 接口没有 Bearer token 会被拒绝。
- value 是原始 bytes，不需要是 JSON。
- `POST /{key}` 有约 5MB 的 body limit。
- `GET /keys` 返回顺序不稳定，因为底层是 HashMap。
- 这是内存服务，重启后数据会丢失。

## 手写任务

完成本章代码后，试着做四个小改动：

1. 把 body limit 从约 5MB 改成 1KB，尝试写入更大的 body。
2. 把 admin token 从 `secret-token` 改成自己的字符串。
3. 给 `GET /keys` 返回值末尾加一个换行。
4. 新增 `GET /admin/stats`，返回当前 key 数量。

## 本章真正要记住什么

- `Bytes` 适合接收和返回原始字节数据。
- `State<SharedState>` 可以让所有 handler 共享内存状态。
- 可以给单个 handler 加 layer，也可以给整个 Router 加 layer。
- admin routes 可以用 `.nest("/admin", ...)` 组织。
- Bearer auth、body limit、compression、timeout、trace 都可以通过 Tower/Tower HTTP layer 组合。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/key-value-store/Cargo.toml`
- `examples/key-value-store/src/main.rs`
