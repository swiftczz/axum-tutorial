# 37. mongodb

对应示例：`examples/mongodb`

前几章连 PostgreSQL 和 Redis,这章连 MongoDB。用 MongoDB 官方 Rust driver 实现最小 CRUD 接口,理解 database/collection/document、`_id` 映射、`State<Collection<T>>`。MongoDB 是文档型数据库,适合灵活 schema 的文档数据。



相比前面章节新引入：**MongoDB `Client` 自带连接池、`Collection<T>`、`doc!` 宏、`#[serde(rename = "_id")]`**。

## Cargo.toml

````toml
[package]
name = "example-mongodb"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
mongodb = { version = "3.3.0", default-features = false, features = ["bson-3", "compat-3-3-0", "rustls-tls"] }
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
tower-http = { version = "0.6.1", features = ["add-extension", "trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 关键概念

> **新面孔：MongoDB `Client` 自带连接池**
>
> 不需要 bb8。`Collection<Member>` 告诉 driver 文档和 Rust 结构体互转。`doc!{"_id": id}` 构造 BSON 查询条件。


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
    let db_connection_str = std::env::var("DATABASE_URL").unwrap_or_else(|_| {
        "mongodb://admin:password@127.0.0.1:27017/?authSource=admin".to_string()
    });

    let client = Client::with_uri_str(db_connection_str).await.unwrap();

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
        .route("/create", post(create_member))
        .route("/read/{id}", get(read_member))
        .route("/update", put(update_member))
        .route("/delete/{id}", delete(delete_member))
        .layer(TraceLayer::new_for_http())
        .with_state(collection)
}

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

fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}

#[derive(Debug, Deserialize, Serialize)]
struct Member {
    #[serde(rename = "_id")]
    id: u32,
    name: String,
    active: bool,
}
````

## 运行

先启动 MongoDB,默认地址 `mongodb://admin:password@127.0.0.1:27017/?authSource=admin`(也可用 `DATABASE_URL` 覆盖):

````bash
cd examples
cargo run -p example-mongodb
````

CRUD:

````bash
# 创建
curl -X POST http://127.0.0.1:3000/create \
  -H 'content-type: application/json' \
  -d '{"_id":1,"name":"Alice","active":true}'

# 读取
curl http://127.0.0.1:3000/read/1

# 更新(替换整个 document)
curl -X PUT http://127.0.0.1:3000/update \
  -H 'content-type: application/json' \
  -d '{"_id":1,"name":"Alice Updated","active":false}'

# 删除
curl -X DELETE http://127.0.0.1:3000/delete/1
````

## 解读

### MongoDB 三个层级

粗略类比 PostgreSQL:

```text
PostgreSQL database → MongoDB database
PostgreSQL table    → MongoDB collection
PostgreSQL row      → MongoDB document
```

本章用 database `axum-mongo`、collection `members`、document `Member`。document 本质是 BSON,Rust 里通过 serde 和结构体互相转换。

### `Client` 自带连接池(本章关键差异)

MongoDB driver 的 `Client` 内部基于 tokio 实现了**自带连接池**——`Client` 持有可配置的连接池(默认 `max_pool_size` = 100,可通过 `ClientOptions::max_pool_size(...)` 调整),所有操作自动从内部池借连接。**不需要**像 Redis/PostgreSQL 那样自己套 bb8。

和前面几章对比:

| 方案 | 连接池在哪 |
| --- | --- |
| SQLx(32) | driver 自带 `PgPool` |
| tokio-postgres + bb8(33) | driver 不带,**你用 bb8 自己建** |
| Diesel + deadpool(34) | driver 不带,**你用 deadpool 自己建** |
| Redis + bb8(36) | driver 不带,**你用 bb8 自己建**(或 `MultiplexedConnection`) |
| MongoDB(37) | driver 自带,`Client` 就是池 |

`Client` 可廉价 clone(内部是 `Arc`),放进 axum state 用 `State<Client>` 提取。本章共享单位是 `Collection<Member>`(从 Client 选出来),`Client` 在 main 持有。

### `_id` 映射(最容易踩坑)

````rust
#[derive(Debug, Deserialize, Serialize)]
struct Member {
    #[serde(rename = "_id")]
    id: u32,
    name: String,
    active: bool,
}
````

MongoDB 默认主键字段叫 `_id`,但 Rust 里写 `id` 更自然。`#[serde(rename = "_id")]` 让 Rust 字段名 `id` 序列化到 MongoDB/JSON 时变成 `_id`。请求 JSON 用 `{"_id":1,...}`,Rust 里用 `member.id`。

### 选择 collection

````rust
let collection: Collection<Member> = client.database("axum-mongo").collection("members");
````

`Collection<Member>` 告诉 driver:从这个 collection 读写的 document 和 Rust `Member` 互相转换。然后 `.with_state(collection)` 放进 axum state,handler 用 `State(db): State<Collection<Member>>` 提取。

### CRUD 四个操作

**创建 `insert_one`:** 返回 `InsertOneResult`(含 inserted id)。

**查询 `find_one`:** 用 `doc! { "_id": id }` 做过滤条件,返回 `Option<Member>`(可能找不到 → `None` → JSON `null`)。`doc!` 宏构造 BSON document。

**替换 `replace_one`:** `replace_one(filter, replacement)` 是**替换整个 document**(不是局部更新某字段)。局部更新用 `$set` 操作。

**删除 `delete_one`:** 返回 `DeleteResult`(含删除条数)。

## 常见问题

**字段为什么叫 `_id`?** MongoDB 默认主键是 `_id`,Rust 用 `id` 更顺手,`#[serde(rename = "_id")]` 做映射。

**MongoDB 需要 migration 吗?** example 没有 migration——文档数据库通常不要求先建表结构。但真实项目仍需管理数据结构变化,只是方式和 SQL migration 不同。

**`replace_one` 是局部更新吗?** 不是,是替换整个 document。只改某字段用 `$set`。

**`read_member` 为什么返回 `Option<Member>`?** 按 `_id` 查可能找不到,找不到返回 `None`(JSON `null`)。

**Client 需要连接池吗?** 不需要,mongo-rust-driver 自带连接池,`Client` 就是池,可廉价 clone 放 state。

## 手写任务

按下面顺序敲:

1. 定义 `Member`,给 `id` 加 `#[serde(rename = "_id")]`。
2. 创建 MongoDB `Client`。
3. `run_command(doc! { "ping": 1 })` 验证连接。
4. `app(client)` 里拿 `Collection<Member>`。
5. `.with_state(collection)` 放进 Router。
6. 写 `create_member` 调 `insert_one`。
7. 写 `read_member` 调 `find_one`。
8. 写 `update_member` 和 `delete_member`。

加深练习:

1. 新增 `/list` 查询所有 member。
2. `replace_one` 改成只更新 `active` 字段(用 `$set`)。
3. 找不到 member 时返回 404 而不是 JSON `null`。

## 小结

- MongoDB 接入 axum 核心模型:`Client` 连接 → `database()` 选库 → `collection()` 选集合并指定文档类型 → `State<Collection<Member>>` 共享 → handler 调 CRUD 方法 → serde 负责 struct ↔ BSON/JSON。
- **MongoDB `Client` 自带连接池**,不需自己套 bb8,可廉价 clone 放 state;和 SQLx 一样是"driver 自带池",和 tokio-postgres/Redis(需 bb8)形成对比。
- `_id` 是最容易踩坑的点:`#[serde(rename = "_id")]` 让 Rust 字段 `id` 映射到 MongoDB `_id`。
- `doc!` 宏构造 BSON document(过滤条件等);`find_one` 返回 `Option`,`replace_one` 是整文档替换不是局部更新。
- MongoDB 不强制先建表,适合灵活 schema 文档数据。

## 源码对照

- `examples/mongodb/Cargo.toml`
- `examples/mongodb/src/main.rs`
