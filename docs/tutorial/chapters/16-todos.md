# 16. todos

对应示例：`examples/todos`

本章目标：手写一个完整 REST Todo API，综合使用 `State`、`Query`、`Path`、`Json`、共享内存状态和 middleware。

这是前 16 章里第一个真正像“业务后端”的例子。  
它不再只是演示一个 extractor，而是把多个后端概念组合成一个小型 Todo 服务。

## 这个小项目在做什么

这个 Todo API 支持四个操作：

| 方法 | 路径 | 作用 |
| --- | --- | --- |
| `GET` | `/todos` | 返回 Todo 列表 |
| `POST` | `/todos` | 创建 Todo |
| `PATCH` | `/todos/{id}` | 更新指定 Todo |
| `DELETE` | `/todos/{id}` | 删除指定 Todo |

数据没有保存到数据库，而是保存在内存里的 `HashMap`：

```text
Uuid -> Todo
```

请求主线示例：

```text
POST /todos
-> Json<CreateTodo> 解析请求 body
-> 生成 Todo 和 UUID
-> 写入 Db
-> 返回 201 Created + JSON Todo
```

```text
PATCH /todos/{id}
-> Path<Uuid> 解析路径 id
-> Json<UpdateTodo> 解析更新字段
-> 从 Db 找到 Todo
-> 修改 text/completed
-> 返回更新后的 Todo
```

## 先理解 REST Todo API

REST API 常用 HTTP 方法表达动作：

| 动作 | 常用方法 | 示例 |
| --- | --- | --- |
| 查询列表 | `GET` | `GET /todos` |
| 创建资源 | `POST` | `POST /todos` |
| 部分更新 | `PATCH` | `PATCH /todos/{id}` |
| 删除资源 | `DELETE` | `DELETE /todos/{id}` |

路径表达资源：

```text
/todos       Todo 集合
/todos/{id} 具体某一个 Todo
```

这章的目标是把“HTTP 方法 + 路径 + 请求体 + 状态码”串起来，形成一个完整 API。

## 文件和依赖

这个 example 有两个文件：

1. `examples/todos/Cargo.toml`：声明 Axum、Serde、Tower、tower-http、uuid 等依赖。
2. `examples/todos/src/main.rs`：实现路由、状态、CRUD handler 和 middleware。

关键依赖：

- `axum`：提供 `Router`、`State`、`Query`、`Path`、`Json`、`IntoResponse`。
- `serde`：解析请求 JSON/query，并序列化响应 JSON。
- `uuid`：生成和解析 Todo ID。
- `tower`：提供 `ServiceBuilder`、timeout 和错误处理。
- `tower-http`：提供 HTTP trace 日志。
- `tokio`、`tracing`、`tracing-subscriber`：运行时和日志。

## 第一步：定义内存数据库类型

源码：

````rust
type Db = Arc<RwLock<HashMap<Uuid, Todo>>>;
````

逐个看：

- `HashMap<Uuid, Todo>`：用 Todo 的 id 快速找到 Todo。
- `RwLock<...>`：允许多个读，写入时独占。
- `Arc<...>`：让多个 handler 共享同一个 Db。

为什么需要 `Arc`？

Axum 会并发处理多个请求。  
多个请求都要访问同一份内存数据，所以需要共享所有权。

为什么需要 `RwLock`？

因为多个请求可能同时读写 HashMap。  
Rust 不允许没有同步保护的并发可变访问。

这个 Db 只是教学示例。真实项目通常会换成数据库连接池。

## 第二步：定义 Todo 响应类型

源码：

````rust
#[derive(Debug, Serialize, Clone)]
struct Todo {
    id: Uuid,
    text: String,
    completed: bool,
}
````

字段含义：

- `id`：唯一标识。
- `text`：Todo 内容。
- `completed`：是否完成。

为什么要 `Serialize`？

因为 handler 要返回 `Json(todo)` 或 `Json(Vec<Todo>)`。

为什么要 `Clone`？

因为从 `HashMap` 里取出 Todo 后，要构造响应。示例里用 `cloned()` 或 `todo.clone()` 简化所有权处理。

## 第三步：创建 Db 并放入 State

源码：

````rust
let db = Db::default();

let app = Router::new()
    ...
    .with_state(db);
````

`Db::default()` 会创建：

```text
Arc::new(RwLock::new(HashMap::new()))
```

`with_state(db)` 把这个共享状态交给 Router。

handler 里通过：

````rust
State(db): State<Db>
````

取出同一份 Db。

这就是 Axum 里共享应用状态的基本方式。

## 第四步：注册 REST 路由

源码：

````rust
let app = Router::new()
    .route("/todos", get(todos_index).post(todos_create))
    .route("/todos/{id}", patch(todos_update).delete(todos_delete))
    ...
````

拆开看：

```text
GET /todos          -> todos_index
POST /todos         -> todos_create
PATCH /todos/{id}   -> todos_update
DELETE /todos/{id}  -> todos_delete
```

同一个路径可以挂多个 HTTP 方法。  
这和前面的 `GET /`、`POST /` 表单章节是一回事，只是这里变成了 REST API。

## 第五步：查询 Todo 列表

分页参数：

````rust
#[derive(Debug, Deserialize, Default)]
pub struct Pagination {
    pub offset: Option<usize>,
    pub limit: Option<usize>,
}
````

handler：

````rust
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
````

请求例子：

```text
GET /todos?offset=0&limit=10
```

`Query<Pagination>` 会解析 query string。  
`State(db)` 会拿到共享 Db。

逻辑是：

```text
读锁打开 HashMap
-> 取所有 Todo
-> skip offset
-> take limit
-> clone 成 Vec<Todo>
-> 返回 Json(Vec<Todo>)
```

## 第六步：创建 Todo

输入类型：

````rust
#[derive(Debug, Deserialize)]
struct CreateTodo {
    text: String,
}
````

handler：

````rust
async fn todos_create(State(db): State<Db>, Json(input): Json<CreateTodo>) -> impl IntoResponse {
    let todo = Todo {
        id: Uuid::new_v4(),
        text: input.text,
        completed: false,
    };

    db.write().unwrap().insert(todo.id, todo.clone());

    (StatusCode::CREATED, Json(todo))
}
````

请求 body：

````json
{"text":"learn axum"}
````

创建时：

- `Uuid::new_v4()` 生成新 ID。
- `completed` 默认是 `false`。
- `db.write().unwrap()` 获取写锁。
- `insert(todo.id, todo.clone())` 存入 HashMap。
- 返回 `201 Created` 和新 Todo。

## 第七步：更新 Todo

输入类型：

````rust
#[derive(Debug, Deserialize)]
struct UpdateTodo {
    text: Option<String>,
    completed: Option<bool>,
}
````

为什么字段都是 `Option`？

因为 PATCH 表示部分更新。  
客户端可以只传一个字段：

````json
{"completed":true}
````

handler：

````rust
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
````

这里同时用了三个 extractor：

- `Path(id): Path<Uuid>`：从 URL 里解析 Todo ID。
- `State(db): State<Db>`：拿到共享内存数据库。
- `Json(input): Json<UpdateTodo>`：解析更新字段。

如果找不到 Todo：

```rust
ok_or(StatusCode::NOT_FOUND)?
```

会提前返回 404。

## 第八步：删除 Todo

源码：

````rust
async fn todos_delete(Path(id): Path<Uuid>, State(db): State<Db>) -> impl IntoResponse {
    if db.write().unwrap().remove(&id).is_some() {
        StatusCode::NO_CONTENT
    } else {
        StatusCode::NOT_FOUND
    }
}
````

删除逻辑很直接：

```text
从路径里解析 id
-> 获取写锁
-> remove(id)
-> 如果删到了，返回 204 No Content
-> 如果没找到，返回 404 Not Found
```

`204 No Content` 表示成功，但没有响应体。

## 第九步：添加 middleware

源码：

````rust
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
````

这段给所有路由加 middleware：

- `timeout(Duration::from_secs(10))`：请求超过 10 秒就超时。
- `HandleErrorLayer`：把 middleware 产生的错误转成 HTTP 响应。
- `TraceLayer::new_for_http()`：记录 HTTP 请求日志。

为什么 timeout 需要 `HandleErrorLayer`？

因为超时发生在 Tower middleware 层，不是普通 handler 返回值。  
要把这种错误转换成 HTTP 响应，必须加错误处理 layer。

## 函数职责速查

- `main`：初始化日志，创建 Db，注册路由、middleware 和 state，启动服务。
- `Pagination`：描述列表查询分页参数。
- `todos_index`：读取 Db，按 offset/limit 返回 Todo 列表。
- `CreateTodo`：创建 Todo 的请求 body。
- `todos_create`：生成 UUID，创建 Todo，写入 Db。
- `UpdateTodo`：更新 Todo 的请求 body，字段都是可选的。
- `todos_update`：按 id 找 Todo，按输入字段部分更新。
- `todos_delete`：按 id 删除 Todo。
- `Db`：内存数据库类型别名。
- `Todo`：API 返回的 Todo 数据结构。

## 带中文注释的手写版

````rust
//! 提供一个管理 Todos 的 RESTful Web 服务。
//!
//! API:
//!
//! - GET /todos：返回 Todo JSON 列表。
//! - POST /todos：创建新 Todo。
//! - PATCH /todos/{id}：更新指定 Todo。
//! - DELETE /todos/{id}：删除指定 Todo。
//!
//! Run with
//!
//! ```not_rust
//! cargo run -p example-todos
//! ```

// 引入错误处理 layer、extractor、状态码、响应转换、路由函数、Json 和 Router。
use axum::{
    error_handling::HandleErrorLayer,
    extract::{Path, Query, State},
    http::StatusCode,
    response::IntoResponse,
    routing::{get, patch},
    Json, Router,
};
// serde 用于请求反序列化和响应序列化。
use serde::{Deserialize, Serialize};
// HashMap 存储 Todo，Arc/RwLock 提供共享和并发访问，Duration 表示超时时间。
use std::{
    collections::HashMap,
    sync::{Arc, RwLock},
    time::Duration,
};
// Tower 提供错误类型和 ServiceBuilder。
use tower::{BoxError, ServiceBuilder};
// TraceLayer 用于 HTTP 请求日志。
use tower_http::trace::TraceLayer;
// tracing_subscriber 用来初始化日志。
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};
// UUID 用作 Todo id。
use uuid::Uuid;

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

    // 创建内存数据库。
    let db = Db::default();

    // 注册 REST 路由、middleware，并把 db 放入 state。
    let app = Router::new()
        .route("/todos", get(todos_index).post(todos_create))
        .route("/todos/{id}", patch(todos_update).delete(todos_delete))
        .layer(
            ServiceBuilder::new()
                // 把 timeout 等 middleware 错误转换成 HTTP 响应。
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
                // 所有请求最多处理 10 秒。
                .timeout(Duration::from_secs(10))
                // 输出 HTTP trace 日志。
                .layer(TraceLayer::new_for_http())
                .into_inner(),
        )
        // 让所有 handler 都能通过 State<Db> 访问 db。
        .with_state(db);

    // 绑定本机 3000 端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 输出监听地址。
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动 HTTP 服务。
    axum::serve(listener, app).await;
}

// GET /todos 的 query 参数。
#[derive(Debug, Deserialize, Default)]
pub struct Pagination {
    pub offset: Option<usize>,
    pub limit: Option<usize>,
}

// GET /todos：返回 Todo 列表。
async fn todos_index(pagination: Query<Pagination>, State(db): State<Db>) -> impl IntoResponse {
    // 获取读锁。
    let todos = db.read().unwrap();

    // 按 offset/limit 做简单分页。
    let todos = todos
        .values()
        .skip(pagination.offset.unwrap_or(0))
        .take(pagination.limit.unwrap_or(usize::MAX))
        .cloned()
        .collect::<Vec<_>>();

    // 返回 JSON 数组。
    Json(todos)
}

// POST /todos 的请求体。
#[derive(Debug, Deserialize)]
struct CreateTodo {
    text: String,
}

// POST /todos：创建 Todo。
async fn todos_create(State(db): State<Db>, Json(input): Json<CreateTodo>) -> impl IntoResponse {
    // 创建新 Todo。
    let todo = Todo {
        id: Uuid::new_v4(),
        text: input.text,
        completed: false,
    };

    // 写入内存数据库。
    db.write().unwrap().insert(todo.id, todo.clone());

    // 返回 201 和新 Todo。
    (StatusCode::CREATED, Json(todo))
}

// PATCH /todos/{id} 的请求体。字段都是 Option，表示部分更新。
#[derive(Debug, Deserialize)]
struct UpdateTodo {
    text: Option<String>,
    completed: Option<bool>,
}

// PATCH /todos/{id}：更新 Todo。
async fn todos_update(
    // 从路径解析 Todo id。
    Path(id): Path<Uuid>,
    // 访问共享 Db。
    State(db): State<Db>,
    // 解析 JSON 更新字段。
    Json(input): Json<UpdateTodo>,
) -> Result<impl IntoResponse, StatusCode> {
    // 先读取现有 Todo；找不到就返回 404。
    let mut todo = db
        .read()
        .unwrap()
        .get(&id)
        .cloned()
        .ok_or(StatusCode::NOT_FOUND)?;

    // 如果传了 text，就更新 text。
    if let Some(text) = input.text {
        todo.text = text;
    }

    // 如果传了 completed，就更新 completed。
    if let Some(completed) = input.completed {
        todo.completed = completed;
    }

    // 写回数据库。
    db.write().unwrap().insert(todo.id, todo.clone());

    // 返回更新后的 Todo。
    Ok(Json(todo))
}

// DELETE /todos/{id}：删除 Todo。
async fn todos_delete(Path(id): Path<Uuid>, State(db): State<Db>) -> impl IntoResponse {
    // 删除成功返回 204，找不到返回 404。
    if db.write().unwrap().remove(&id).is_some() {
        StatusCode::NO_CONTENT
    } else {
        StatusCode::NOT_FOUND
    }
}

// 内存数据库：多个请求共享一个 HashMap。
type Db = Arc<RwLock<HashMap<Uuid, Todo>>>;

// Todo 响应结构体。
#[derive(Debug, Serialize, Clone)]
struct Todo {
    id: Uuid,
    text: String,
    completed: bool,
}
````

## 运行和验证

运行前先确认 `examples/todos/Cargo.toml` 里的 `axum = { path = "../../axum" }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-todos
````

创建 Todo：

````bash
curl -i -X POST http://127.0.0.1:3000/todos \
  -H 'content-type: application/json' \
  -d '{"text":"learn axum"}'
````

预期状态码：

````text
HTTP/1.1 201 Created
````

响应体里会有一个 `id`，后面更新和删除要用它。

查询列表：

````bash
curl -i 'http://127.0.0.1:3000/todos?offset=0&limit=10'
````

更新 Todo，把 `<id>` 换成创建时返回的 id：

````bash
curl -i -X PATCH http://127.0.0.1:3000/todos/<id> \
  -H 'content-type: application/json' \
  -d '{"completed":true}'
````

删除 Todo：

````bash
curl -i -X DELETE http://127.0.0.1:3000/todos/<id>
````

预期状态码：

````text
HTTP/1.1 204 No Content
````

常见卡点：

- 这是内存数据库，服务重启后数据会丢失。
- `Path<Uuid>` 要求 URL 里的 id 是合法 UUID。
- `PATCH` 的字段是可选的，适合部分更新。
- `RwLock` 在 async 程序里长期持有要谨慎；这个 example 的锁范围很短，所以用于教学可以接受。
- `HashMap` 遍历顺序不稳定，所以列表返回顺序不要依赖。

## 手写任务

完成本章代码后，试着做四个小改动：

1. 给 `Todo` 增加 `created_at` 字段，创建时写入当前时间或一个固定字符串。
2. 给 `GET /todos` 增加 `completed` query 参数，只返回已完成或未完成 Todo。
3. 尝试更新不存在的 id，确认返回 404。
4. 把 `DELETE` 改成返回被删除的 Todo，而不是只返回 204。

## 本章真正要记住什么

- REST API 用 HTTP 方法表达动作，用路径表达资源。
- `State<T>` 用来把共享状态传给 handler。
- `Query<T>` 解析 query string，`Path<T>` 解析路径参数，`Json<T>` 解析 body。
- `Arc<RwLock<HashMap<...>>>` 可以做教学用内存状态。
- 创建返回 201，删除成功常用 204，找不到资源返回 404。
- Tower middleware 可以统一加 timeout、trace 和错误处理。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/todos/Cargo.toml`
- `examples/todos/src/main.rs`
