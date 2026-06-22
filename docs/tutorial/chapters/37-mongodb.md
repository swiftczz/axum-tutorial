# 37. mongodb

对应示例：`examples/mongodb`

前面几章数据库都是 PostgreSQL/Redis。这章换 **MongoDB**——文档型 NoSQL 数据库，存 JSON-like 文档（BSON）。和 SQL 数据库最大区别：无表结构约束、查询用 `doc! { ... }` 而非 SQL、Rust 端 model 直接 serde 到文档。

实现一个 members 的 CRUD（Create/Read/Update/Delete），掌握 MongoDB 的 `Client`/`Collection`、`doc!` 查询、`Collection<Member>` 作为 State 的模式。

分 3 步：先建连接和 model，再加 create/read，最后加 update/delete。

相比前面章节新引入：**`mongodb` crate、`Client::with_uri_str`、`Collection<T>` 泛型集合、`doc!` 宏（BSON 文档）、`insert_one`/`find_one`/`replace_one`/`delete_one`**。

## Cargo.toml

````toml
[package]
name = "example-mongodb"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
mongodb = "3"
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
tower-http = { version = "0.6", features = ["trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：连接 MongoDB + 定义 Member model + 路由骨架

先建连接：用 `Client::with_uri_str` 连 MongoDB，ping 验证。再定义 `Member` model（serde 到 BSON 文档），最后把 `Collection<Member>` 作为 State 注入 Router。

````rust
use axum::{
    routing::{get, post},
    Router,
};
use mongodb::{bson::doc, Client, Collection};
use serde::{Deserialize, Serialize};
use tower_http::trace::TraceLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[derive(Debug, Deserialize, Serialize)]
struct Member {
    #[serde(rename = "_id")]
    id: u32,
    name: String,
    active: bool,
}

#[tokio::main]
async fn main() {
    let db_connection_str = std::env::var("DATABASE_URL").unwrap_or_else(|_| {
        "mongodb://admin:password@127.0.0.1:27017/?authSource=admin".to_string()
    });
    let client = Client::with_uri_str(db_connection_str).await.unwrap();

    // ping 验证连接
    client
        .database("axum-mongo")
        .run_command(doc! { "ping": 1 })
        .await
        .unwrap();
    println!("Pinged your database. Successfully connected to MongoDB!");

    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("Listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app(client)).await.unwrap();
}

fn app(client: Client) -> Router {
    let collection: Collection<Member> = client.database("axum-mongo").collection("members");

    Router::new()
        .layer(TraceLayer::new_for_http())
        .with_state(collection)
}
````

验证（需要本地 MongoDB）：

````bash
# 用 docker 启 mongo
docker run -d --name mongo -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password \
  mongo:7

cd examples
cargo run -p example-mongodb
````

看到 `Pinged your database. Successfully connected to MongoDB!` 就说明连上了。这步还没 handler，下一步加。

> **新面孔：`mongodb::Client`**
>
> MongoDB 客户端，`Client::with_uri_str(uri).await` 建立连接池。`uri` 格式 `mongodb://user:pass@host:port/?authSource=db`。
>
> Client 内部维护连接池，可 clone 共享——所以 `fn app(client: Client)` 直接 move 进 Router 没问题。

> **新面孔：`doc!` 宏**
>
> `bson::doc! { ... }` 构造 BSON 文档（MongoDB 的查询格式）。`doc! { "ping": 1 }` 等价于 MongoDB shell 的 `{ ping: 1 }`。
>
> 后面查询都用 `doc!`：`doc! { "_id": id }` 表示"找 `_id` 等于 `id` 的文档"。

> **新面孔：`Collection<T>`**
>
> MongoDB 集合（类似 SQL 的表），`<T>` 参数告诉 driver 怎么序列化/反序列化文档。这里 `Collection<Member>` 让 driver 自动把 `Member` 转 BSON 存进去、把查到的文档转回 `Member`。
>
> `client.database("axum-mongo").collection("members")` 选 `axum-mongo` 数据库的 `members` 集合。集合不存在第一次写入时自动创建。

> **新面孔：`#[serde(rename = "_id")]`**
>
> MongoDB 用 `_id` 作为主键（注意下划线前缀）。Rust 字段命名规范不允许下划线开头，所以用 `serde(rename = "_id")` 把 `id` 字段映射到 BSON 的 `_id`。

---

## 第二步：`create_member` + `read_member`

加两个 handler：POST 创建一个 member，GET 按 id 查 member。

````rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    routing::{get, post},
    Json,
};
use mongodb::results::{InsertOneResult};

async fn create_member(
    State(db): State<Collection<Member>>,
    Json(input): Json<Member>,
) -> Result<Json<InsertOneResult>, (StatusCode, String)> {
    let result = db.insert_one(input).await.map_err(internal_error)?;
    Ok(Json(result))
}

async fn read_member(
    State(db): State<Collection<Member>>,
    Path(id): Path<u32>,
) -> Result<Json<Option<Member>>, (StatusCode, String)> {
    let result = db
        .find_one(doc! { "_id": id })
        .await
        .map_err(internal_error)?;
    Ok(Json(result))
}

fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}

# fn app(client: Client) -> Router {
#     let collection: Collection<Member> = client.database("axum-mongo").collection("members");
#     Router::new()
#         .route("/create", post(create_member))
#         .route("/read/{id}", get(read_member))
#         .layer(TraceLayer::new_for_http())
#         .with_state(collection)
# }
````

> **新面孔：`insert_one` / `find_one`**
>
> MongoDB 的基础操作。`Collection<Member>` 提供 typed 方法：
> - `db.insert_one(member).await` → `InsertOneResult`（含生成的 `_id`）
> - `db.find_one(doc! { "_id": id }).await` → `Option<Member>`（找不到是 None）
>
> 因为 `Collection<Member>` 是泛型的，driver 自动处理 `Member ↔ BSON` 转换——不用手动 `to_bson` / `from_bson`。错误处理只有一层 `map_err`（不像 Diesel 那样两层），因为 mongodb crate 原生 async。

> **新面孔：`Path<u32>` extractor**
>
> 第 16 章 todos 用过 `Path`。`Path(id): Path<u32>` 从 URL 路径参数提取。路由 `/read/{id}` 里的 `{id}` 被解析成 `u32`（要能 parse 成 u32，否则 400）。

验证：

````bash
curl -X POST http://127.0.0.1:3000/create \
  -H 'content-type: application/json' \
  -d '{"id":1,"name":"Alice","active":true}'
# 返回 {"inserted_id":1,...}

curl http://127.0.0.1:3000/read/1
# 返回 {"_id":1,"name":"Alice","active":true}
````

---

## 第三步：`update_member` + `delete_member`

加 PUT（整体替换）和 DELETE 两个 handler，完成 CRUD。

````rust
use mongodb::results::{DeleteResult, UpdateResult};
use axum::routing::{delete, put};

async fn update_member(
    State(db): State<Collection<Member>>,
    Json(input): Json<Member>,
) -> Result<Json<UpdateResult>, (StatusCode, String)> {
    let result = db
        .replace_one(doc! { "_id": input.id }, input)
        .await
        .map_err(internal_error)?;
    Ok(Json(result))
}

async fn delete_member(
    State(db): State<Collection<Member>>,
    Path(id): Path<u32>,
) -> Result<Json<DeleteResult>, (StatusCode, String)> {
    let result = db
        .delete_one(doc! { "_id": id })
        .await
        .map_err(internal_error)?;
    Ok(Json(result))
}

# fn app(client: Client) -> Router {
#     let collection: Collection<Member> = client.database("axum-mongo").collection("members");
#     Router::new()
#         .route("/create", post(create_member))
#         .route("/read/{id}", get(read_member))
#         .route("/update", put(update_member))
#         .route("/delete/{id}", delete(delete_member))
#         .layer(TraceLayer::new_for_http())
#         .with_state(collection)
# }
````

> **新面孔：`replace_one` vs `update_one`**
>
> 两个更新方法：
> - `replace_one(filter, new_doc)`：**整体替换**文档（除 `_id` 外全替换），这章用这个
> - `update_one(filter, doc! { "$set": { ... } })`：**部分更新**用 `$set` / `$inc` 等操作符
>
> 这里 `update_member` 接收完整 `Member` 做 `replace_one`，简单粗暴。如果想"只改 active 字段"用 `update_one(doc!{"_id":id}, doc!{"$set": {"active": false}})`。

> **新面孔：`delete_one`**
>
> `db.delete_one(doc! { "_id": id })` 删除匹配的第一个文档。返回 `DeleteResult`（含 `deleted_count`，0 或 1）。

验证：

````bash
curl -X PUT http://127.0.0.1:3000/update \
  -H 'content-type: application/json' \
  -d '{"id":1,"name":"Alice Smith","active":false}'
# 返回 {"matched_count":1,"modified_count":1,...}

curl -X DELETE http://127.0.0.1:3000/delete/1
# 返回 {"deleted_count":1,...}
````

---

## 完整代码

````rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    routing::{delete, get, post, put},
    Json, Router,
};
use mongodb::{
    bson::doc,
    results::{DeleteResult, InsertOneResult, UpdateResult},
    Client, Collection,
};
use serde::{Deserialize, Serialize};
use tower_http::trace::TraceLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // connecting to mongodb
    let db_connection_str = std::env::var("DATABASE_URL").unwrap_or_else(|_| {
        "mongodb://admin:password@127.0.0.1:27017/?authSource=admin".to_string()
    });
    let client = Client::with_uri_str(db_connection_str).await.unwrap();

    // pinging the database
    client
        .database("axum-mongo")
        .run_command(doc! { "ping": 1 })
        .await
        .unwrap();
    println!("Pinged your database. Successfully connected to MongoDB!");

    // logging middleware
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // run it
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("Listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app(client)).await.unwrap();
}

// defining routes and state
fn app(client: Client) -> Router {
    let collection: Collection<Member> = client.database("axum-mongo").collection("members");

    Router::new()
        .route("/create", post(create_member))
        .route("/read/{id}", get(read_member))
        .route("/update", put(update_member))
        .route("/delete/{id}", delete(delete_member))
        .layer(TraceLayer::new_for_http())
        .with_state(collection)
}

// handler to create a new member
async fn create_member(
    State(db): State<Collection<Member>>,
    Json(input): Json<Member>,
) -> Result<Json<InsertOneResult>, (StatusCode, String)> {
    let result = db.insert_one(input).await.map_err(internal_error)?;

    Ok(Json(result))
}

// handler to read an existing member
async fn read_member(
    State(db): State<Collection<Member>>,
    Path(id): Path<u32>,
) -> Result<Json<Option<Member>>, (StatusCode, String)> {
    let result = db
        .find_one(doc! { "_id": id })
        .await
        .map_err(internal_error)?;

    Ok(Json(result))
}

// handler to update an existing member
async fn update_member(
    State(db): State<Collection<Member>>,
    Json(input): Json<Member>,
) -> Result<Json<UpdateResult>, (StatusCode, String)> {
    let result = db
        .replace_one(doc! { "_id": input.id }, input)
        .await
        .map_err(internal_error)?;

    Ok(Json(result))
}

// handler to delete an existing member
async fn delete_member(
    State(db): State<Collection<Member>>,
    Path(id): Path<u32>,
) -> Result<Json<DeleteResult>, (StatusCode, String)> {
    let result = db
        .delete_one(doc! { "_id": id })
        .await
        .map_err(internal_error)?;

    Ok(Json(result))
}

fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}

// defining Member type
#[derive(Debug, Deserialize, Serialize)]
struct Member {
    #[serde(rename = "_id")]
    id: u32,
    name: String,
    active: bool,
}
````

## 运行

````bash
# 启动 MongoDB（docker）
docker run -d --name mongo -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password \
  mongo:7

cd examples
cargo run -p example-mongodb
````

CRUD 测试：

````bash
# 创建
curl -X POST http://127.0.0.1:3000/create \
  -H 'content-type: application/json' \
  -d '{"id":1,"name":"Alice","active":true}'

# 查询
curl http://127.0.0.1:3000/read/1

# 更新（整体替换）
curl -X PUT http://127.0.0.1:3000/update \
  -H 'content-type: application/json' \
  -d '{"id":1,"name":"Alice Smith","active":false}'

# 删除
curl -X DELETE http://127.0.0.1:3000/delete/1
````

## 解读

### MongoDB vs SQL 数据库

| 维度 | MongoDB | SQL（Postgres） |
| --- | --- | --- |
| 数据模型 | 文档（BSON/JSON-like） | 行（结构化） |
| Schema | 无约束（灵活） | 严格（migration 定义） |
| 查询 | `doc! { "_id": id }` | `SELECT * FROM ... WHERE id = $1` |
| Rust 集成 | serde 自动转 BSON/反序列化 | Diesel/sqlx 类型映射 |
| 主键 | `_id`（自动生成 ObjectId 或自定义） | `id`（SERIAL 等） |

### `Collection<Member>` 作为 State

和 ch33 的 `Pool` 作 State 同构：把 `Collection<Member>` 塞进 `with_state`，handler 用 `State<Collection<Member>>` 取。`Collection` 内部是 `Arc` 引用 client，可 cheap clone。

## 常见问题

**为什么用 `replace_one` 不用 `update_one`？** `replace_one` 整体替换（接收完整 `Member`），`update_one` 部分更新（用 `$set` 操作符）。这章 PUT 语义是整体替换所以用 `replace_one`。

**MongoDB 需要建表/migration 吗？** 不需要。MongoDB 无 schema，第一次写入时自动创建集合。结构变化也不用 migration（直接写新字段即可）。

**`_id` 必须自己指定吗？** 不指定的话 MongoDB 自动生成 `ObjectId`（24 字符 hex 字符串）。这章用 `u32` 自定义 id（业务可读），所以 `Member` 自己管 `id` 字段。

## 手写任务

1. 把 `update_member` 改成 `update_one(doc!{"_id":input.id}, doc!{"$set": {"active": input.active}})`，只更新 active 字段。
2. 加 `list_members`：`db.find(doc!{}, None).await` 返回所有，用 `StreamExt::next` 遍历。
3. 加错误处理区分"找不到"（404）和"数据库错误"（500）——`find_one` 返回 `Option`，可以判 None。

## 小结

这章用 3 步讲了 MongoDB CRUD：

1. **连接 + model**：`Client::with_uri_str` + ping 验证，`Collection<Member>` 作为 State。
2. **create/read**：`insert_one` + `find_one`，`Collection<T>` 自动 Member↔BSON 转换。
3. **update/delete**：`replace_one`（整体替换）+ `delete_one`，CRUD 完整。

核心：MongoDB 无 schema、用 `doc!` 查询、`Collection<T>` 自动 serde、错误只有一层（原生 async）。和 SQL 数据库对比：灵活 schema 换 migration 成本，文档查询换 SQL 表达力。

## 源码对照

- `examples/mongodb/Cargo.toml`
- `examples/mongodb/src/main.rs`
