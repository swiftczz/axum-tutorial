# 16. todos

对应示例：`examples/todos`

这是前 16 章里第一个真正像"业务后端"的例子。手写一个完整 REST Todo API,综合使用 `State`、`Query`、`Path`、`Json`、共享内存状态和 middleware。数据存在内存 `HashMap`,不接数据库。

## Cargo.toml

````toml
[package]
name = "example-todos"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
tower = { version = "0.5.2", features = ["util", "timeout"] }
tower-http = { version = "0.6.1", features = ["add-extension", "trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
uuid = { version = "1.0", features = ["serde", "v4"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

````rust
use axum::{
    error_handling::HandleErrorLayer,
    extract::{Path, Query, State},
    http::StatusCode,
    response::IntoResponse,
    routing::{get, patch},
    Json, Router,
};
use serde::{Deserialize, Serialize};
use std::{
    collections::HashMap,
    sync::{Arc, RwLock},
    time::Duration,
};
use tower::{BoxError, ServiceBuilder};
use tower_http::trace::TraceLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};
use uuid::Uuid;

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

    let db = Db::default();

    let app = Router::new()
        .route("/todos", get(todos_index).post(todos_create))
        .route("/todos/{id}", patch(todos_update).delete(todos_delete))
        .layer(
            ServiceBuilder::new()
                .layer(HandleErrorLayer::new(|error: BoxError| async move {
                    if error.is::<tower::timeout::error::Elapsed>() {
                        Ok(StatusCode::REQUEST_TIMEOUT)
                    } else {
                        Err((
                            StatusCode::INTERNAL_SERVER_ERROR,
                            format!("Unhandled internal error: {error}"),
                        ))
                    }
                }))
                .timeout(Duration::from_secs(10))
                .layer(TraceLayer::new_for_http())
                .into_inner(),
        )
        .with_state(db);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app).await;
}

#[derive(Debug, Deserialize, Default)]
pub struct Pagination {
    pub offset: Option<usize>,
    pub limit: Option<usize>,
}

async fn todos_index(pagination: Query<Pagination>, State(db): State<Db>) -> impl IntoResponse {
    let todos = db.read().unwrap();

    let todos = todos
        .values()
        .skip(pagination.offset.unwrap_or(0))
        .take(pagination.limit.unwrap_or(usize::MAX))
        .cloned()
        .collect::<Vec<_>>();

    Json(todos)
}

#[derive(Debug, Deserialize)]
struct CreateTodo {
    text: String,
}

async fn todos_create(State(db): State<Db>, Json(input): Json<CreateTodo>) -> impl IntoResponse {
    let todo = Todo {
        id: Uuid::new_v4(),
        text: input.text,
        completed: false,
    };

    db.write().unwrap().insert(todo.id, todo.clone());

    (StatusCode::CREATED, Json(todo))
}

#[derive(Debug, Deserialize)]
struct UpdateTodo {
    text: Option<String>,
    completed: Option<bool>,
}

async fn todos_update(
    Path(id): Path<Uuid>,
    State(db): State<Db>,
    Json(input): Json<UpdateTodo>,
) -> Result<impl IntoResponse, StatusCode> {
    let mut todo = db
        .read()
        .unwrap()
        .get(&id)
        .cloned()
        .ok_or(StatusCode::NOT_FOUND)?;

    if let Some(text) = input.text {
        todo.text = text;
    }

    if let Some(completed) = input.completed {
        todo.completed = completed;
    }

    db.write().unwrap().insert(todo.id, todo.clone());

    Ok(Json(todo))
}

async fn todos_delete(Path(id): Path<Uuid>, State(db): State<Db>) -> impl IntoResponse {
    if db.write().unwrap().remove(&id).is_some() {
        StatusCode::NO_CONTENT
    } else {
        StatusCode::NOT_FOUND
    }
}

type Db = Arc<RwLock<HashMap<Uuid, Todo>>>;

#[derive(Debug, Serialize, Clone)]
struct Todo {
    id: Uuid,
    text: String,
    completed: bool,
}
````

## 运行

````bash
cd examples
cargo run -p example-todos
````

创建 Todo:

````bash
curl -i -X POST http://127.0.0.1:3000/todos \
  -H 'content-type: application/json' \
  -d '{"text":"learn axum"}'
# 预期 201 Created,响应体里有 id
````

查询列表:

````bash
curl -i 'http://127.0.0.1:3000/todos?offset=0&limit=10'
````

更新(把 `<id>` 换成创建时返回的):

````bash
curl -i -X PATCH http://127.0.0.1:3000/todos/<id> \
  -H 'content-type: application/json' \
  -d '{"completed":true}'
````

删除:

````bash
curl -i -X DELETE http://127.0.0.1:3000/todos/<id>
# 预期 204 No Content
````

## 解读

### REST API 的四个动作

| 方法 | 路径 | handler | 作用 |
| --- | --- | --- | --- |
| `GET` | `/todos` | `todos_index` | 返回 Todo 列表 |
| `POST` | `/todos` | `todos_create` | 创建 Todo |
| `PATCH` | `/todos/{id}` | `todos_update` | 部分更新 |
| `DELETE` | `/todos/{id}` | `todos_delete` | 删除 |

REST 用 HTTP 方法表达动作、用路径表达资源:`/todos` 是集合,`/todos/{id}` 是具体某个。同一路径可挂多个方法。

### `Db = Arc<RwLock<HashMap<Uuid, Todo>>>`

````rust
type Db = Arc<RwLock<HashMap<Uuid, Todo>>>;
````

三层包装,每层解决一个问题:

- `HashMap<Uuid, Todo>`:用 id 快速找到 Todo。
- `RwLock<...>`:允许并发多读、写时独占。多个请求可能同时访问 HashMap,Rust 不允许无同步的并发可变访问。
- `Arc<...>`:让多个 handler 共享同一份数据。

**为什么 `Arc` 而不是 `Rc`?** axum 在多个**线程**上并发处理请求,`Rc` 不是线程安全的。Rust 用两个 marker trait 约束:

```text
Send:这个类型的值可以安全地从一个线程转移到另一个线程
Sync:这个类型的值可以安全地被多个线程同时引用
```

`Arc<T>` 要求 `T: Send + Sync`,`RwLock<T>` 也要求 `T: Send + Sync`。后面你会反复看到泛型约束 `S: Send + Sync`,就是保证"能跨线程用"。axum 的 handler 和 state 都必须满足。

`Arc::clone` 只是把引用计数 +1,不复制数据,很廉价。普通 `&` 引用做不到共享,因为它的生命周期和单个请求绑定。

这只是教学示例,真实项目通常换成数据库连接池。

### `with_state(db)` 共享状态

````rust
let db = Db::default();
let app = Router::new()
    ...
    .with_state(db);
````

`Db::default()` 创建 `Arc::new(RwLock::new(HashMap::new()))`。`with_state(db)` 把它交给 Router,handler 用 `State(db): State<Db>` 取出(解构语法见第 15 章)。

### 三个 extractor 组合

`todos_update` 同时用三个 extractor:

````rust
async fn todos_update(
    Path(id): Path<Uuid>,            // URL 路径参数
    State(db): State<Db>,            // 共享状态
    Json(input): Json<UpdateTodo>,   // 请求 body
) -> Result<impl IntoResponse, StatusCode> {
````

axum 按顺序提取。找不到 Todo 时 `ok_or(StatusCode::NOT_FOUND)?` 提前返回 404。

### PATCH 用 `Option` 字段

````rust
struct UpdateTodo {
    text: Option<String>,
    completed: Option<bool>,
}
````

PATCH 表示部分更新,客户端可只传一个字段(`{"completed":true}`)。`Option` 让字段可选。

### 状态码约定

- 创建: `201 Created`
- 删除成功: `204 No Content`(成功但无响应体)
- 找不到资源: `404 Not Found`

### middleware 三件套

````rust
.layer(
    ServiceBuilder::new()
        .layer(HandleErrorLayer::new(...))
        .timeout(Duration::from_secs(10))
        .layer(TraceLayer::new_for_http())
        .into_inner(),
)
````

- `timeout(10s)`:请求超过 10 秒就超时。
- `HandleErrorLayer`:把 middleware 层的错误(如超时)转成 HTTP 响应。超时发生在 Tower middleware 层,不是 handler 返回值,必须用这个 layer 转换。
- `TraceLayer`:记录 HTTP 请求日志。

## 手写任务

跑通后做四个小改动:

1. 给 `Todo` 加 `created_at` 字段,创建时写入当前时间。
2. 给 `GET /todos` 加 `completed` query 参数,只返回已完成或未完成 Todo。
3. 尝试更新不存在的 id,确认返回 404。
4. 把 `DELETE` 改成返回被删除的 Todo,而不是只返回 204。

## 小结

- REST API 用 HTTP 方法表达动作、用路径表达资源,同一路径可挂多个方法。
- `State<T>` 用来把共享状态传给 handler。
- `Arc<RwLock<HashMap<...>>>` 做教学用内存状态;`Arc` 解决共享所有权,`RwLock` 解决并发读写,跨线程要求 `Send + Sync`。
- `Query<T>` 解析 query string,`Path<T>` 解析路径参数,`Json<T>` 解析 body,可组合使用。
- 创建 201、删除成功 204、找不到 404。
- Tower middleware 可统一加 timeout、trace 和错误处理,`HandleErrorLayer` 把 middleware 错误转成 HTTP 响应。

## 源码对照

- `examples/todos/Cargo.toml`
- `examples/todos/src/main.rs`
