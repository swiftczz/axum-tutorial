# 16. todos

对应示例：`examples/todos`

这是教程里第一个真正像"业务后端"的例子。我们用 5 步从零搭出一个完整 REST Todo API：创建、查询、更新、删除，数据存在内存里。

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

本章是第一个完整 CRUD demo，相比前面章节新增：`uuid`（生成唯一 ID，启用 `serde` 支持序列化、`v4` 用随机算法）、`tower`（axum 底层依赖的 middleware 抽象，这里手动用 `timeout` layer）和 `tower-http`（HTTP middleware 工具集，启用 `add-extension` 给请求注入扩展、`trace` 加日志）。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：定义 Todo 和内存数据库

我们先不管 HTTP，先想清楚"数据长什么样、存在哪里"。

一个 Todo 有三个字段：唯一 ID、文本内容、是否完成。ID 用 `Uuid`（通用唯一标识符），不用自增数字。

````rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use uuid::Uuid;

#[derive(Debug, Serialize, Clone)]
struct Todo {
    id: Uuid,
    text: String,
    completed: bool,
}
````

> **新面孔：`Uuid`**
>
> `Uuid` 是 128 位的通用唯一 ID，形如 `550e8400-e29b-41d4-a716-446655440000`。用它做主键，分布式系统里不会撞 ID。`uuid` crate 的 `v4` feature 提供随机生成（`Uuid::new_v4()`），`serde` feature 让它能序列化成 JSON。

接下来定义"数据库"。真实项目用 PostgreSQL 或 Redis，这里用内存 `HashMap` 做教学。但直接用 `HashMap` 有两个问题：

1. **多个请求同时访问**——需要加锁。
2. **多个 handler 共享同一份数据**——需要共享所有权。

所以套两层：`RwLock` 解决并发访问，`Arc` 解决共享所有权。

````rust
type Db = Arc<RwLock<HashMap<Uuid, Todo>>>;
````

> **新面孔：`Arc<RwLock<HashMap<K, V>>>`**
>
> 这个三层嵌套看着吓人，拆开看就三件事：
>
> - `HashMap<Uuid, Todo>`：键值表，用 Uuid 快速找到 Todo。
> - `RwLock<...>`：读写锁。多个请求可能同时读或写 HashMap，Rust 不允许无锁的并发可变访问。`RwLock` 允许多个读同时进行，写的时候独占。
> - `Arc<...>`：原子引用计数，让多个 handler **共享同一份数据**。`Arc::clone` 只增加计数（+1），不复制数据，很廉价。
>
> 为什么用 `Arc` 不用 `Rc`？因为 axum 在多个**线程**上并发处理请求，`Rc` 不是线程安全的。Rust 用 `Send + Sync` 两个 trait 约束"能跨线程用"，`Arc` 和 `RwLock` 都满足。

---

## 第二步：最小服务——返回空 Todo 列表

先搭一个能跑的最小服务：`GET /todos` 返回空数组。

````rust
use axum::{extract::State, response::IntoResponse, routing::get, Json, Router};

#[tokio::main]
async fn main() {
    let db = Db::default(); // Arc::new(RwLock::new(HashMap::new()))

    let app = Router::new()
        .route("/todos", get(todos_index))
        .with_state(db);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await;
}

async fn todos_index(State(db): State<Db>) -> impl IntoResponse {
    let todos = db.read().unwrap();
    let todos = todos.values().cloned().collect::<Vec<_>>();
    Json(todos)
}
````

跑起来验证：

````bash
cargo run -p example-todos
# 另一个终端
curl http://127.0.0.1:3000/todos
# 预期: []
````

返回空数组，因为 `Db::default()` 创建的是空 `HashMap`。

> **新面孔：`State<T>`**
>
> `State(db): State<Db>` 是 axum 的 extractor（提取器），从应用状态中取出共享数据。`.with_state(db)` 把数据放进 Router，handler 用 `State(db)` 取出来。
>
> `State(db): State<Db>` 这个写法是 Rust 的**模式匹配解构**：把 `State<Db>` 拆开，取出里面的 `Db` 绑定到变量 `db`。

---

## 第三步：加创建接口（POST）

现在加 `POST /todos`，接收 JSON body，创建新 Todo。

先定义"输入"——客户端提交的数据只有 `text`，`id` 由服务端生成，`completed` 默认 `false`。

````rust
use axum::http::StatusCode;

#[derive(Debug, Deserialize)]
struct CreateTodo {
    text: String,
}

async fn todos_create(
    State(db): State<Db>,
    Json(input): Json<CreateTodo>,
) -> impl IntoResponse {
    let todo = Todo {
        id: Uuid::new_v4(),
        text: input.text,
        completed: false,
    };

    db.write().unwrap().insert(todo.id, todo.clone());

    (StatusCode::CREATED, Json(todo))
}
````

> **新面孔：`Json<CreateTodo>`**
>
> `Json(input): Json<CreateTodo>` 是 axum 的 JSON 提取器：从请求 body 读 JSON，反序列化成 `CreateUser` 结构体，绑定到 `input`。
>
> `CreateTodo` 需要 `#[derive(Deserialize)]`（来自 serde），这样 serde 才知道怎么把 JSON `{"text":"learn axum"}` 变成 Rust 结构体。

更新路由，同一个 `/todos` 路径挂 GET 和 POST：

````rust
let app = Router::new()
    .route("/todos", get(todos_index).post(todos_create))
    .with_state(db);
````

`get(todos_index).post(todos_create)` 把两个方法绑到同一路径。现在可以验证：

````bash
curl -X POST http://127.0.0.1:3000/todos \
  -H 'content-type: application/json' \
  -d '{"text":"learn axum"}'
# 预期: {"id":"...","text":"learn axum","completed":false}

curl http://127.0.0.1:3000/todos
# 预期: [{"id":"...","text":"learn axum","completed":false}]
`````

创建返回 `201 Created`（`StatusCode::CREATED`），这是 REST 约定——创建资源用 201 而不是 200。

---

## 第四步：加更新和删除（PATCH / DELETE）

接下来加 `PATCH /todos/{id}`（部分更新）和 `DELETE /todos/{id}`（删除）。

路径里的 `{id}` 是路径参数。axum 用 `Path<Uuid>` 提取器获取它：

````rust
use axum::extract::Path;

async fn todos_update(
    Path(id): Path<Uuid>,
    State(db): State<Db>,
    Json(input): Json<UpdateTodo>,
) -> Result<impl IntoResponse, StatusCode> {
    // 先读：从 HashMap 里找到这个 todo
    let mut todo = db
        .read()
        .unwrap()
        .get(&id)
        .cloned()
        .ok_or(StatusCode::NOT_FOUND)?;

    // 只更新客户端传了的字段
    if let Some(text) = input.text {
        todo.text = text;
    }
    if let Some(completed) = input.completed {
        todo.completed = completed;
    }

    // 再写：存回 HashMap
    db.write().unwrap().insert(todo.id, todo.clone());

    Ok(Json(todo))
}
````

> **新面孔：`Path<Uuid>` 和 `Option<T>` 字段**
>
> `Path(id): Path<Uuid>` 从 URL 路径提取 `{id}` 参数，自动解析成 `Uuid`。
>
> `UpdateTodo` 的字段全是 `Option<T>`——因为 PATCH 是**部分更新**，客户端可以只传 `{"completed":true}` 而不传 `text`。`Option<String>` 表示"这个字段客户端可能传也可能不传"。
>
> `ok_or(StatusCode::NOT_FOUND)?` 是一个常用模式：如果 `get(&id)` 找不到，`ok_or` 把 `None` 变成 `Err(StatusCode::NOT_FOUND)`，`?` 提前返回 404。

删除更简单：

````rust
async fn todos_delete(Path(id): Path<Uuid>, State(db): State<Db>) -> impl IntoResponse {
    if db.write().unwrap().remove(&id).is_some() {
        StatusCode::NO_CONTENT  // 204，删除成功，没有响应体
    } else {
        StatusCode::NOT_FOUND   // 404，找不到
    }
}
````

更新路由——`/todos/{id}` 挂 PATCH 和 DELETE：

````rust
let app = Router::new()
    .route("/todos", get(todos_index).post(todos_create))
    .route("/todos/{id}", patch(todos_update).delete(todos_delete))
    .with_state(db);
````

验证：

````bash
# 更新（把 <id> 换成创建时返回的）
curl -X PATCH http://127.0.0.1:3000/todos/<id> \
  -H 'content-type: application/json' \
  -d '{"completed":true}'

# 删除
curl -i -X DELETE http://127.0.0.1:3000/todos/<id>
# 预期: 204 No Content
````

---

## 第五步：加查询分页和中间件

最后两个增强：GET /todos 支持分页查询，所有路由加超时和日志中间件。

**分页查询：**

````rust
use axum::extract::Query;

#[derive(Debug, Deserialize, Default)]
pub struct Pagination {
    pub offset: Option<usize>,
    pub limit: Option<usize>,
}

async fn todos_index(
    pagination: Query<Pagination>,
    State(db): State<Db>,
) -> impl IntoResponse {
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

> **新面孔：`Query<Pagination>`**
>
> `Query(pagination): Query<Pagination>` 从 URL query string 提取参数。访问 `/todos?offset=10&limit=5` 时，axum 自动把 `offset=10&limit=5` 反序列化成 `Pagination { offset: Some(10), limit: Some(5) }`。参数缺失时是 `None`，用 `unwrap_or(0)` 给默认值。

**中间件——超时 + 日志：**

````rust
use axum::error_handling::HandleErrorLayer;
use std::time::Duration;
use tower::{BoxError, ServiceBuilder};
use tower_http::trace::TraceLayer;

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
````

> **新面孔：`HandleErrorLayer` + `timeout`**
>
> `timeout(Duration::from_secs(10))` 让请求超过 10 秒就超时。但超时错误发生在 Tower 中间件层（不是 handler 返回值），axum 不知道怎么把它变成 HTTP 响应。`HandleErrorLayer` 负责这个转换：检查错误类型，超时返回 `408 Request Timeout`，其他错误返回 `500`。
>
> `TraceLayer` 记录每个请求的方法、路径、状态码和耗时。

---

## 完整代码

经过 5 步构建，最终完整代码就是 `examples/todos/src/main.rs`：

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

完整 CRUD 测试：

````bash
# 创建
curl -i -X POST http://127.0.0.1:3000/todos \
  -H 'content-type: application/json' \
  -d '{"text":"learn axum"}'
# 预期 201 Created

# 查询列表（带分页）
curl 'http://127.0.0.1:3000/todos?offset=0&limit=10'

# 更新（把 <id> 换成创建时返回的）
curl -i -X PATCH http://127.0.0.1:3000/todos/<id> \
  -H 'content-type: application/json' \
  -d '{"completed":true}'

# 删除
curl -i -X DELETE http://127.0.0.1:3000/todos/<id>
# 预期 204 No Content
````

## 手写任务

跑通后做四个小改动：

1. 给 `Todo` 加 `created_at` 字段，创建时写入当前时间。
2. 给 `GET /todos` 加 `completed` query 参数，只返回已完成或未完成 Todo。
3. 尝试更新不存在的 id，确认返回 404。
4. 把 `DELETE` 改成返回被删除的 Todo，而不是只返回 204。

## 小结

这章我们从零搭了一个完整 REST API，核心是 5 步增量构建：

1. **定义数据**：`Todo` 结构体 + `Db = Arc<RwLock<HashMap<Uuid, Todo>>>`（共享的、加了锁的键值表）。
2. **最小服务**：`GET /todos` 返回列表，用 `State<Db>` 提取共享数据。
3. **创建**：`POST /todos` + `Json<CreateTodo>` 提取请求体，返回 `201 Created`。
4. **更新和删除**：`PATCH/DELETE /todos/{id}` + `Path<Uuid>` 提取路径参数，`Option<T>` 支持部分更新。
5. **增强**：`Query<Pagination>` 分页 + `HandleErrorLayer`/`timeout`/`TraceLayer` 中间件。

REST 约定：GET 查询、POST 创建（201）、PATCH 部分更新、DELETE 删除（204）、找不到返回 404。

## 源码对照

- `examples/todos/Cargo.toml`
- `examples/todos/src/main.rs`
