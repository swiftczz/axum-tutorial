# 第 23 章 todos:一个完整的 REST CRUD

## 本章你将做出什么

一个待办事项 API,四个方法各司其职:

```
GET    /todos          列出(支持 offset/limit 分页)
POST   /todos          创建
PATCH  /todos/{id}     部分更新
DELETE /todos/{id}     删除
```

这是本教程第一个**完整的 REST 应用**。前面章节的路由都是为了演示某个机制,这一章的路由是为了完成一件真实的事——而且要用符合 REST 惯例的方式完成。

学完本章你将掌握:

- REST 的资源建模:URL 表示资源、方法表示操作
- `PATCH` 与 `PUT` 的区别,以及**部分更新怎么用类型表达**
- `Query<T>` 提取器与分页参数的处理
- 四个操作各自该返回什么状态码
- 一个真实存在的并发缺陷:读-改-写竞态

## 前置知识

### REST 的核心约定

REST 不是协议也不是标准,是一套关于"怎么设计 HTTP API"的惯例。核心只有两条:

**URL 标识资源,不描述动作。**

```
✓ /todos           待办事项集合
✓ /todos/{id}      单个待办事项
✗ /getTodos        动作跑进了 URL
✗ /todos/delete    同上
```

**方法表达动作。** 同一个 URL 配不同方法就是不同操作——这也是[第 02 章](02-routes-and-handlers-close-together.md#router-与-methodrouter-的分工)讲的 `MethodRouter` 存在的意义:

| 方法 | 语义 | 本章路由 |
|---|---|---|
| `GET` | 读取,不改变任何东西 | `GET /todos` |
| `POST` | 创建新资源 | `POST /todos` |
| `PUT` | 整体替换 | (本章不用) |
| `PATCH` | 部分更新 | `PATCH /todos/{id}` |
| `DELETE` | 删除 | `DELETE /todos/{id}` |

这套约定的价值在于**可预测**:别人看到 `DELETE /todos/42` 不用查文档就知道它干什么。

### PUT 与 PATCH:整体替换 vs 部分更新

这是 REST 里最容易搞错的一对。

**`PUT` 是整体替换。** 请求体是资源的**完整新状态**,没提到的字段应该被清空或重置:

```
PUT /todos/42
{"text": "买牛奶"}
```

严格按语义,这会把 `completed` 重置为默认值——因为你没提它,等于说"这个资源现在只有 text"。

**`PATCH` 是部分更新。** 请求体只包含**要改的字段**,没提到的保持原样:

```
PATCH /todos/42
{"completed": true}
```

只改 `completed`,`text` 不动。

**用类型表达这个区别,是本章最值得学的一点:**

```rust
#[derive(Debug, Deserialize)]
struct UpdateTodo {
    text: Option<String>,
    completed: Option<bool>,
}
```

**每个字段都是 `Option`。** `None` 表示"客户端没提这个字段",`Some(v)` 表示"改成 v"。handler 里:

```rust
if let Some(text) = input.text {
    todo.text = text;
}
```

`Option` 在这里的含义**不是"值可以为空"，而是"字段可以缺席"** —— 这两件事在 API 设计里完全不同,但用同一个类型表达,是 `PATCH` 建模的经典难点。

顺带一提,如果业务上确实需要"把某个字段设成 null",就得用 `Option<Option<T>>` 配合 serde 的 `#[serde(default, deserialize_with = ...)]` 来区分"没提"和"提了但是 null"。本章不涉及,但知道有这个坑就好。

### Query 提取器与分页

`Query<T>` 从 URL 的查询字符串反序列化。[第 12 章](12-validator.md#form-在-get-请求下读查询字符串)提过 `Form` 在 `GET` 下也读查询字符串,但表达"我要读 query"的意图时 `Query` 更直白。

```rust
#[derive(Debug, Deserialize, Default)]
pub struct Pagination {
    pub offset: Option<usize>,
    pub limit: Option<usize>,
}
```

两个字段都是 `Option`,所以 `/todos`、`/todos?limit=10`、`/todos?offset=20&limit=10` 都能解析。

**分页是列表接口的必需品**,原因很实际:数据量增长后,一次返回全部会让响应体巨大、内存暴涨、客户端渲染卡死。**加分页的成本很低,不加的代价是某天突然爆炸。**

`offset`/`limit` 是最简单的分页方案。它的缺点是数据量大时 `offset` 越大越慢(要跳过前面所有记录),而且翻页期间有新数据插入会导致重复或遗漏。生产级方案通常用游标分页(基于上一页最后一条的 ID),但那需要稳定的排序。

### 各操作该返回什么状态码

REST 对状态码也有惯例:

| 操作 | 成功 | 说明 |
|---|---|---|
| `GET` 列表 | `200` | 空列表也是 `200`,不是 `404` |
| `POST` 创建 | **`201 Created`** | 而不是 `200`,表示"创建了新资源" |
| `PATCH` 更新 | `200` + 更新后的资源 | 让客户端拿到最终状态 |
| `DELETE` 删除 | **`204 No Content`** | 删除成功没什么可返回的 |
| 资源不存在 | `404` | `PATCH`/`DELETE` 一个不存在的 id |

**`201` 和 `204` 这两个是最常被忽略的。** 用 `200` 也能工作,但丢掉了语义信息——规范的客户端会根据 `201` 知道"资源创建成功了",根据 `204` 知道"不用解析响应体"。

[第 08 章](08-parse-body-based-on-content-type.md#json收-json-与发-json)见过 `201` 的写法:`(StatusCode::CREATED, Json(todo))`。

## 项目搭建

```bash
cargo new todos
cd todos
```

```bash
cargo add axum
cargo add serde --features derive
cargo add tokio --features full
cargo add tower --features util,timeout
cargo add tower-http --features add-extension,trace
cargo add tracing
cargo add tracing-subscriber --features env-filter
cargo add uuid --features serde,v4
```

依赖和[第 16 章](16-dependency-injection.md#项目搭建)、[第 22 章](22-key-value-store.md#项目搭建)基本重合,不再解释。

## 分步构建

### 步骤 1:数据模型与列表接口

**要做什么**:定义 `Todo` 类型和共享存储,实现 `GET /todos`。

```rust
// src/main.rs
use axum::{
    extract::{Query, State},
    response::IntoResponse,
    routing::get,
    Json, Router,
};
use serde::{Deserialize, Serialize};
use std::{
    collections::HashMap,
    sync::{Arc, RwLock},
};
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
        .route("/todos", get(todos_index))
        .with_state(db);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

// The query parameters for todos index
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

type Db = Arc<RwLock<HashMap<Uuid, Todo>>>;

#[derive(Debug, Serialize, Clone)]
struct Todo {
    id: Uuid,
    text: String,
    completed: bool,
}
```

**`type Db = Arc<RwLock<HashMap<Uuid, Todo>>>`** 直接用了 `HashMap` 而没有像[第 22 章](22-key-value-store.md#步骤-1读写共享状态)那样包一层 `AppState` struct。示例简单时可以,但加字段时要改所有签名——第 22 章那个写法更耐扩展。

**`pagination: Query<Pagination>` 没有解构。** 所以下面要写 `pagination.offset`,靠 `Deref` 自动解引用到内部的 `Pagination`。这和前面章节的 `Query(pagination): Query<Pagination>` 写法等价,只是省了一次解构。

**`.skip().take()` 做分页。** `unwrap_or(0)` 和 `unwrap_or(usize::MAX)` 提供默认值——不传参数就是"从头开始、要全部"。

**`.cloned()` 是必需的。** 迭代器给的是 `&Todo` 引用,而引用不能逃出读锁的作用域。这里必须复制一份出来——[第 22 章](22-key-value-store.md#步骤-1读写共享状态)讲过同样的限制。

`Todo` 派生了 `Clone` 就是为了这里。

**`db.values()` 的顺序是不确定的。** `HashMap` 不保证遍历顺序,所以分页结果**在多次请求之间可能不一致**——这是个真实缺陷,「代价与边界」会详细说。

**跑起来看**:

```bash
cargo run
```

```bash
curl -s http://127.0.0.1:3000/todos
```

```json
[]
```

空列表,状态码 `200`(不是 `404`——集合存在,只是没有元素)。

分页参数也能解析(虽然现在没数据看不出效果):

```bash
curl -i -s 'http://127.0.0.1:3000/todos?offset=5&limit=10' | head -1
```

```
HTTP/1.1 200 OK
```

传个非法值试试:

```bash
curl -i -s 'http://127.0.0.1:3000/todos?limit=abc' | head -1
```

```
HTTP/1.1 400 Bad Request
```

`abc` 转不成 `usize`,`Query` 提取器拒绝了。[第 03 章](03-form.md#步骤-3观察提取失败时的行为)讲过这类拒绝。

### 步骤 2:创建

**要做什么**:实现 `POST /todos`,返回 `201`。

```rust
// src/main.rs
use axum::{
    extract::{Query, State},
    http::StatusCode,                       // 新增
    response::IntoResponse,
    routing::get,
    Json, Router,
};
// ...

#[tokio::main]
async fn main() {
    // ...

    // 改动:同一路径挂上 POST
    let app = Router::new()
        .route("/todos", get(todos_index).post(todos_create))
        .with_state(db);

    // ...
}

// ...(Pagination、todos_index 不变)

// 新增
#[derive(Debug, Deserialize)]
struct CreateTodo {
    text: String,
}

// 新增
async fn todos_create(State(db): State<Db>, Json(input): Json<CreateTodo>) -> impl IntoResponse {
    let todo = Todo {
        id: Uuid::new_v4(),
        text: input.text,
        completed: false,
    };

    db.write().unwrap().insert(todo.id, todo.clone());

    (StatusCode::CREATED, Json(todo))
}

// ...(Db、Todo 不变)
```

**`CreateTodo` 只有 `text` 一个字段。** `id` 由服务端生成,`completed` 默认 `false`——**客户端不该也不能指定它们**。这是[第 08 章](08-parse-body-based-on-content-type.md#json收-json-与发-json)讲的"请求类型和响应类型分开定义"的实际理由:如果直接用 `Todo` 当输入类型,客户端就能自己指定 id,那会引发一堆问题(id 冲突、覆盖别人的数据)。

**`Json(input)` 必须是最后一个参数**,因为它是 `FromRequest` 类别([第 03 章](03-form.md#提取器axum-的核心机制))。`State` 是 `FromRequestParts`,所以排在前面。

**`(StatusCode::CREATED, Json(todo))`** 返回 `201` 和创建好的资源。返回资源本身很重要——客户端需要知道服务端生成的 `id`。

**跑起来看**:

```bash
cargo run
```

```bash
curl -i -s -X POST http://127.0.0.1:3000/todos \
  -H 'Content-Type: application/json' \
  -d '{"text":"买牛奶"}'
```

```
HTTP/1.1 201 Created
content-type: application/json
content-length: 76

{"id":"3f2a1c4e-8b7d-4e1f-9a2b-6c5d4e3f2a1b","text":"买牛奶","completed":false}
```

**`201 Created`**,而且返回了完整的 todo 包括生成的 id。

再建两条,然后列出:

```bash
curl -s -X POST http://127.0.0.1:3000/todos -H 'Content-Type: application/json' -d '{"text":"写代码"}' > /dev/null
curl -s -X POST http://127.0.0.1:3000/todos -H 'Content-Type: application/json' -d '{"text":"睡觉"}' > /dev/null
curl -s http://127.0.0.1:3000/todos
```

```json
[{"id":"...","text":"写代码","completed":false},{"id":"...","text":"买牛奶","completed":false},{"id":"...","text":"睡觉","completed":false}]
```

**注意顺序不是创建顺序** —— `HashMap` 的遍历顺序不确定。多刷几次还会变。

试试分页:

```bash
curl -s 'http://127.0.0.1:3000/todos?limit=1'
```

返回一条,但**是哪一条不确定**。

### 步骤 3:部分更新

**要做什么**:实现 `PATCH /todos/{id}`,理解 `Option` 字段的作用。

```rust
// src/main.rs
use axum::{
    extract::{Path, Query, State},          // 改动:加上 Path
    http::StatusCode,
    response::IntoResponse,
    routing::{get, patch},                  // 改动:加上 patch
    Json, Router,
};
// ...

#[tokio::main]
async fn main() {
    // ...

    // 改动:加上 PATCH 路由
    let app = Router::new()
        .route("/todos", get(todos_index).post(todos_create))
        .route("/todos/{id}", patch(todos_update))
        .with_state(db);

    // ...
}

// ...(前面的不变)

// 新增
#[derive(Debug, Deserialize)]
struct UpdateTodo {
    text: Option<String>,
    completed: Option<bool>,
}

// 新增
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
```

**`UpdateTodo` 的两个字段都是 `Option`** —— 前置知识讲过这是 `PATCH` 的建模方式。`None` = 客户端没提,保持原样。

**读-改-写三步**:

```rust
let mut todo = db.read().unwrap().get(&id).cloned().ok_or(...)?;   // 读
if let Some(text) = input.text { todo.text = text; }               // 改
db.write().unwrap().insert(todo.id, todo.clone());                 // 写
```

注意**读锁和写锁是分开获取的**,中间没有持锁。这符合[第 22 章](22-key-value-store.md#持锁跨-await能让服务卡死的陷阱)"缩短临界区"的原则,但也**引入了一个竞态**——「代价与边界」会讲。

**`.ok_or(StatusCode::NOT_FOUND)?`** 把 `Option` 转成 `Result` 并用 `?` 提前返回。返回类型是 `Result<impl IntoResponse, StatusCode>`,[第 16 章](16-dependency-injection.md#步骤-2用-trait-对象注入)用过同样的形式——错误只有一种时,直接返回 `StatusCode` 最省事。

**三个提取器的顺序**:`Path`、`State` 是 `FromRequestParts`,`Json` 是 `FromRequest` 必须最后。

**跑起来看**:

```bash
cargo run
```

先创建一条,记下 id:

```bash
ID=$(curl -s -X POST http://127.0.0.1:3000/todos \
  -H 'Content-Type: application/json' -d '{"text":"买牛奶"}' \
  | grep -o '"id":"[^"]*"' | cut -d'"' -f4)
echo $ID
```

**只更新 `completed`**:

```bash
curl -s -X PATCH "http://127.0.0.1:3000/todos/$ID" \
  -H 'Content-Type: application/json' \
  -d '{"completed":true}'
```

```json
{"id":"...","text":"买牛奶","completed":true}
```

**`text` 保持原样,只有 `completed` 变了。** 这就是 `PATCH` 的语义,靠 `Option` 字段实现。

**只更新 `text`**:

```bash
curl -s -X PATCH "http://127.0.0.1:3000/todos/$ID" \
  -H 'Content-Type: application/json' \
  -d '{"text":"买酸奶"}'
```

```json
{"id":"...","text":"买酸奶","completed":true}
```

`completed` 保持 `true`。

**发一个空对象**:

```bash
curl -s -X PATCH "http://127.0.0.1:3000/todos/$ID" \
  -H 'Content-Type: application/json' -d '{}'
```

```json
{"id":"...","text":"买酸奶","completed":true}
```

什么都没改——两个字段都是 `None`。**这是合理的 `PATCH` 行为**。

更新不存在的 id:

```bash
curl -i -s -X PATCH http://127.0.0.1:3000/todos/00000000-0000-0000-0000-000000000000 \
  -H 'Content-Type: application/json' -d '{"text":"x"}' | head -1
```

```
HTTP/1.1 404 Not Found
```

**对比一下如果字段不是 `Option` 会怎样。** 临时把 `UpdateTodo` 改成:

```rust
struct UpdateTodo {
    text: String,
    completed: bool,
}
```

handler 里直接赋值。然后:

```bash
curl -i -s -X PATCH "http://127.0.0.1:3000/todos/$ID" \
  -H 'Content-Type: application/json' -d '{"completed":true}' | head -1
```

```
HTTP/1.1 422 Unprocessable Entity
```

**缺 `text` 字段直接被拒绝了** —— 因为 `String` 不是 `Option`,serde 认为它必填。这就变成了 `PUT` 的语义:必须提供完整资源。改回 `Option`。

### 步骤 4:删除与中间件

**要做什么**:补上 `DELETE`,加上超时中间件,抵达官方示例的最终形态。

```rust
// src/main.rs
use axum::{
    error_handling::HandleErrorLayer,        // 新增
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
    time::Duration,                          // 新增
};
use tower::{BoxError, ServiceBuilder};       // 新增
use tower_http::trace::TraceLayer;           // 新增
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};
use uuid::Uuid;

#[tokio::main]
async fn main() {
    // ...(日志初始化和 db 不变)

    // Compose the routes
    let app = Router::new()
        .route("/todos", get(todos_index).post(todos_create))
        // 改动:加上 DELETE
        .route("/todos/{id}", patch(todos_update).delete(todos_delete))
        // 新增:Add middleware to all routes
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

    // ...(listener 和 serve 不变)
}

// ...(前面的 handler 不变)

// 新增
async fn todos_delete(Path(id): Path<Uuid>, State(db): State<Db>) -> impl IntoResponse {
    if db.write().unwrap().remove(&id).is_some() {
        StatusCode::NO_CONTENT
    } else {
        StatusCode::NOT_FOUND
    }
}

// ...(Db、Todo 不变)
```

**`todos_delete` 用 `remove()` 的返回值判断存在性。** `HashMap::remove` 返回 `Option<V>`:`Some` 表示删掉了,`None` 表示本来就没有。**一次操作同时完成"删除"和"判断存在"**,不需要先 `contains_key` 再 `remove`——那样既多一次查找,又有竞态。

**`204 No Content` vs `404`** 是前置知识讲的惯例。

**`HandleErrorLayer` 的闭包这次返回 `Result`。** 和[第 22 章](22-key-value-store.md#步骤-5全局中间件与错误处理)那个返回 `impl IntoResponse` 的写法不同:`Ok(...)` 表示"错误已处理,返回这个响应",`Err(...)` 表示"我处理不了,继续往上抛"。两种签名都支持。

**`.into_inner()`** 把 `ServiceBuilder` 转成 `Layer`。较新版本的 axum 里通常不需要这一行(`ServiceBuilder` 本身就实现了 `Layer`),但官方示例保留着,照抄无害。

**跑起来看**:

```bash
cargo run
```

完整走一遍 CRUD:

```bash
# 创建
ID=$(curl -s -X POST http://127.0.0.1:3000/todos \
  -H 'Content-Type: application/json' -d '{"text":"买牛奶"}' \
  | grep -o '"id":"[^"]*"' | cut -d'"' -f4)

# 读取
curl -s http://127.0.0.1:3000/todos

# 更新
curl -s -X PATCH "http://127.0.0.1:3000/todos/$ID" \
  -H 'Content-Type: application/json' -d '{"completed":true}'

# 删除
curl -i -s -X DELETE "http://127.0.0.1:3000/todos/$ID" | head -1

# 确认删掉了
curl -s http://127.0.0.1:3000/todos
```

```
[{"id":"...","text":"买牛奶","completed":false}]
{"id":"...","text":"买牛奶","completed":true}
HTTP/1.1 204 No Content
[]
```

**`204 No Content`,而且没有响应体** —— 符合规范。

重复删除:

```bash
curl -i -s -X DELETE "http://127.0.0.1:3000/todos/$ID" | head -1
```

```
HTTP/1.1 404 Not Found
```

**这里有个可以讨论的设计选择**:重复删除返回 `404` 还是 `204`?

- 返回 `404`(本例)——"你要删的东西不存在",信息更准确
- 返回 `204` ——**幂等**:调用一次和调用多次结果相同,客户端重试更安全

REST 规范要求 `DELETE` 是幂等的,但"幂等"指的是**服务端状态**相同(删完就是没有),不要求响应相同。两种做法都常见,团队内统一即可。

## 完整代码

`src/main.rs`:

```rust
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

    // Compose the routes
    let app = Router::new()
        .route("/todos", get(todos_index).post(todos_create))
        .route("/todos/{id}", patch(todos_update).delete(todos_delete))
        // Add middleware to all routes
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
    axum::serve(listener, app).await.unwrap();
}

// The query parameters for todos index
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
```

## 代价与边界

**读-改-写竞态。** 这是本章最值得注意的问题。`todos_update` 分三步:

```rust
let mut todo = db.read()...get(&id).cloned()...;   // 1. 读(释放读锁)
todo.text = text;                                  // 2. 改
db.write().unwrap().insert(todo.id, todo);         // 3. 写(获取写锁)
```

**步骤 1 和 3 之间锁是释放的。** 如果两个请求并发更新同一条 todo:

```
请求 A:读到 {text: "买牛奶", completed: false}
请求 B:读到 {text: "买牛奶", completed: false}
请求 A:写入 {text: "买酸奶", completed: false}
请求 B:写入 {text: "买牛奶", completed: true}   ← A 的修改丢了
```

**这叫丢失更新(lost update)。** A 改的 `text` 被 B 覆盖了,因为 B 是基于旧数据算出来的。

正确做法是**整个读-改-写在同一个写锁里完成**:

```rust
let mut db = db.write().unwrap();
let todo = db.get_mut(&id).ok_or(StatusCode::NOT_FOUND)?;
if let Some(text) = input.text { todo.text = text; }
if let Some(completed) = input.completed { todo.completed = completed; }
Ok(Json(todo.clone()))
```

数据库场景下对应的是事务,或者用乐观锁(带版本号更新)。**这是个真实缺陷,不是理论问题** —— 并发更新同一条记录在真实应用里很常见。练习题 2 会让你修。

**分页顺序不确定。** `HashMap::values()` 不保证顺序,而且 Rust 的 `HashMap` 默认用随机种子,**同一个进程内多次遍历顺序也可能不同**。这意味着:

- `?offset=0&limit=10` 和 `?offset=10&limit=10` 可能返回重复的项,也可能漏掉某些项
- 客户端翻页会看到混乱的结果

分页**必须建立在稳定排序之上**。修法是用 `BTreeMap`(按 key 排序),或者取出来后显式排序。练习题 3 会做。

**offset 分页在大数据量下会变慢。** `skip(n)` 要真的迭代 n 次。数据量大时应该用游标分页。

**数据在内存里,重启丢失。** 和[第 22 章](22-key-value-store.md#代价与边界)一样。持久化从[第 24 章](24-sqlx-postgres.md)开始。

**没有校验。** `text` 可以是空字符串、可以是 10MB 的文本。[第 12 章](12-validator.md)讲的 `validator` 正是用来补这个的。

**`GET /todos/{id}` 缺失。** 官方示例只有列表没有单个查询,而 REST 惯例里 `GET /todos/{id}` 是标配。练习题 1 会补。

## 动手练习

**1. 补上 `GET /todos/{id}`。**

提示:`.route("/todos/{id}", get(todos_show).patch(todos_update).delete(todos_delete))`。handler 逻辑和 `todos_update` 的读取部分一样,找不到返回 `404`。这是 REST 的标配接口,官方示例漏了。

**2. 修掉读-改-写竞态。**

提示:改用一个写锁包住整个操作,`db.get_mut(&id)` 拿到可变引用直接改。注意 `get_mut` 返回的引用借用了 `db`,所以最后 `Json(todo.clone())` 要在锁作用域内完成,或者先克隆再释放锁。改完对比两版代码,想清楚为什么原来那版有问题。

**3. 让分页顺序稳定。**

提示:两种改法。简单的是把 `HashMap` 换成 `BTreeMap`(按 `Uuid` 排序,稳定但顺序无业务含义);更实际的是给 `Todo` 加一个 `created_at` 字段,取出后按它排序。做完连续请求 `?limit=2` 和 `?offset=2&limit=2`,确认没有重复和遗漏。

**4. 给 `text` 加校验。**

提示:用[第 12 章](12-validator.md)的 `validator`,给 `CreateTodo` 加 `#[validate(length(min = 1, max = 200))]`。你需要一个 `ValidatedJson` 提取器——参照第 12 章的 `ValidatedForm` 写一个。

## 小结与下一章预告

REST 的核心是两条约定:**URL 标识资源、方法表达动作**。四个操作各有惯例状态码,其中 `201 Created` 和 `204 No Content` 最常被忽略——用 `200` 也能跑,但丢掉了语义。

`PATCH` 与 `PUT` 的区别在于部分更新还是整体替换,而**部分更新靠 `Option` 字段建模**:`None` 表示"客户端没提这个字段"。本章验证过,把字段从 `Option<String>` 改成 `String` 会让缺字段的请求直接被拒——语义就从 `PATCH` 变成了 `PUT`。

分页是列表接口的必需品,加它的成本很低、不加的代价是某天突然爆炸。但**分页必须建立在稳定排序之上**,本章示例用 `HashMap` 遍历,顺序不确定,翻页会重复和遗漏。

本章还留了一个真实缺陷:`todos_update` 的读-改-写分三步,中间释放了锁,并发更新同一条记录会**丢失更新**。正确做法是整个操作在一个写锁里完成——数据库场景下对应的就是事务。

到这里,你已经能写出一个结构完整的 REST 服务了。但它有个根本问题:**所有数据都在内存里,重启就没了**。[第 24 章 sqlx-postgres](24-sqlx-postgres.md) 接上真正的数据库:连接池怎么建、怎么通过 `State` 共享、SQL 怎么写、以及一个 axum 特有的话题——**如何让"从连接池取连接"这件事变成一个提取器**。
