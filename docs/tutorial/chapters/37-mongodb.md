# 37. mongodb

对应示例：`examples/mongodb`

本章目标：使用 MongoDB 官方 Rust driver 实现最小 CRUD 接口，理解 database、collection、document、`_id` 映射和 Axum `State<Collection<T>>`。

前面几章连接 PostgreSQL 和 Redis。  
这一章连接 MongoDB。

## 这个小项目在做什么

应用管理一种数据：

```text
Member
```

字段：

```text
_id
name
active
```

提供四个接口：

```text
POST   /create     -> 创建 member
GET    /read/{id}  -> 按 id 查询 member
PUT    /update     -> 按请求体里的 id 替换 member
DELETE /delete/{id}-> 按 id 删除 member
```

请求主线是：

```text
程序启动
-> 读取 DATABASE_URL
-> 创建 MongoDB Client
-> ping 数据库
-> 选择 axum-mongo 数据库里的 members 集合
-> 把 Collection<Member> 放进 Router state
-> handler 从 State 取 collection
-> 调用 insert_one/find_one/replace_one/delete_one
-> 返回 JSON
```

## 先理解 MongoDB 的三个词

MongoDB 里最常见的三个层级是：

```text
database
collection
document
```

可以粗略类比：

```text
PostgreSQL database -> MongoDB database
PostgreSQL table    -> MongoDB collection
PostgreSQL row      -> MongoDB document
```

本章使用：

```text
database:   axum-mongo
collection: members
document:   Member
```

MongoDB document 本质上是 BSON 数据。  
在 Rust 里通过 `serde` 把它和结构体互相转换。

## 文件和依赖

这个 example 有两个主要文件：

1. `examples/mongodb/Cargo.toml`：声明 Axum、mongodb、serde、Tokio、tower-http tracing。
2. `examples/mongodb/src/main.rs`：连接 MongoDB，定义 `Member`，实现 CRUD handler。

关键依赖：

- `mongodb`：MongoDB 官方 Rust driver。
- `serde`：把 Rust struct 和 BSON/JSON 互相转换。
- `axum`：提供 Router、Path、State、Json。
- `tower-http`：提供 `TraceLayer`。
- `tokio`：异步运行时。

MongoDB 依赖配置：

````toml
mongodb = { version = "3.3.0", default-features = false, features = ["bson-3", "compat-3-3-0", "rustls-tls"] }
````

## 第一步：读取连接字符串并创建 Client

源码：

````rust
let db_connection_str = std::env::var("DATABASE_URL").unwrap_or_else(|_| {
    "mongodb://admin:password@127.0.0.1:27017/?authSource=admin".to_string()
});
let client = Client::with_uri_str(db_connection_str).await.unwrap();
````

和前面数据库章节一样，连接地址优先从环境变量读取：

```text
DATABASE_URL
```

如果没有设置，就使用默认值：

```text
mongodb://admin:password@127.0.0.1:27017/?authSource=admin
```

`Client::with_uri_str(...).await` 会创建 MongoDB client。

MongoDB 的 `Client` 内部会管理连接，不需要你自己再额外包一层连接池。

## 第二步：启动时 ping 数据库

源码：

````rust
client
    .database("axum-mongo")
    .run_command(doc! { "ping": 1 })
    .await
    .unwrap();
println!("Pinged your database. Successfully connected to MongoDB!");
````

这里执行 MongoDB 的 `ping` 命令。  
作用是启动时确认数据库真的可用。

`doc!` 宏用来构造 BSON document：

````rust
doc! { "ping": 1 }
````

如果 MongoDB 没启动、用户名密码错、地址不通，程序会在这里失败。

## 第三步：定义 Member 类型

源码：

````rust
#[derive(Debug, Deserialize, Serialize)]
struct Member {
    #[serde(rename = "_id")]
    id: u32,
    name: String,
    active: bool,
}
````

MongoDB 默认主键字段叫：

```text
_id
```

但 Rust 结构体里写字段名 `id` 更自然。

所以用：

````rust
#[serde(rename = "_id")]
id: u32,
````

意思是：

```text
Rust 字段名叫 id
序列化到 MongoDB/JSON 时字段名叫 _id
```

`Deserialize` 让请求 JSON 可以变成 `Member`。  
`Serialize` 让 `Member` 可以变成响应 JSON。

## 第四步：创建 app 并选择 collection

源码：

````rust
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
````

这里选择：

```text
database("axum-mongo")
collection("members")
```

并指定集合里的文档类型是：

````rust
Collection<Member>
````

这很重要。  
它告诉 MongoDB driver：

```text
从这个 collection 读写的 document 和 Rust Member 互相转换
```

然后通过：

````rust
.with_state(collection)
````

把 collection 放进 Axum state。

## 第五步：创建 member

源码：

````rust
async fn create_member(
    State(db): State<Collection<Member>>,
    Json(input): Json<Member>,
) -> Result<Json<InsertOneResult>, (StatusCode, String)> {
    let result = db.insert_one(input).await.map_err(internal_error)?;

    Ok(Json(result))
}
````

handler 参数：

```text
State(db)   -> MongoDB collection
Json(input) -> 请求体里的 Member
```

核心操作：

````rust
db.insert_one(input).await
````

返回类型是：

````rust
InsertOneResult
````

里面包含插入结果，例如 inserted id。

请求示例：

````json
{"_id":1,"name":"Alice","active":true}
````

## 第六步：按 id 查询 member

源码：

````rust
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
````

路径：

```text
/read/{id}
```

`Path(id): Path<u32>` 从 URL 中提取 id。

查询条件：

````rust
doc! { "_id": id }
````

返回类型是：

````rust
Option<Member>
````

因为按 id 查询可能找不到：

```text
Some(member) -> 找到了
None         -> 没找到
```

## 第七步：替换更新 member

源码：

````rust
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
````

这里用的是：

````rust
replace_one(filter, replacement)
````

它不是局部更新某个字段，而是用新的 document 替换旧 document。

过滤条件：

````rust
doc! { "_id": input.id }
````

替换内容：

````rust
input
````

返回类型：

````rust
UpdateResult
````

里面会包含匹配了几条、修改了几条等信息。

## 第八步：按 id 删除 member

源码：

````rust
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
````

路径：

```text
/delete/{id}
```

过滤条件：

````rust
doc! { "_id": id }
````

返回：

````rust
DeleteResult
````

里面会告诉你删除了几条记录。

## 第九步：错误转换和 TraceLayer

错误转换：

````rust
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
````

和前面章节一样，把内部错误转成 HTTP 500。

路由上还加了：

````rust
.layer(TraceLayer::new_for_http())
````

用来记录 HTTP 请求日志。  
数据库接口加 trace 很有用，因为你能看到每个 CRUD 请求的状态码和耗时。

## 函数职责速查

- `main`：读取连接字符串，创建 MongoDB client，ping 数据库，初始化日志，启动服务。
- `app`：选择 database 和 collection，注册 CRUD 路由，把 collection 放进 state。
- `create_member`：插入一个 member document。
- `read_member`：按 `_id` 查询一个 member。
- `update_member`：按 `_id` 替换一个 member。
- `delete_member`：按 `_id` 删除一个 member。
- `internal_error`：把 MongoDB 错误转成 HTTP 500。
- `Member`：MongoDB document 和 HTTP JSON 的数据模型。

## 带中文注释的手写版

````rust
//! MongoDB + Axum CRUD 示例。
//!
//! ```not_rust
//! cargo run -p example-mongodb
//! ```

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
    // 读取 MongoDB 连接地址。
    let db_connection_str = std::env::var("DATABASE_URL").unwrap_or_else(|_| {
        "mongodb://admin:password@127.0.0.1:27017/?authSource=admin".to_string()
    });

    // 创建 MongoDB client。
    let client = Client::with_uri_str(db_connection_str).await.unwrap();

    // 启动时 ping 数据库，确认连接可用。
    client
        .database("axum-mongo")
        .run_command(doc! { "ping": 1 })
        .await
        .unwrap();
    println!("Pinged your database. Successfully connected to MongoDB!");

    // 初始化日志。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 启动服务。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("Listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app(client)).await.unwrap();
}

// 创建 Router，并把 members collection 放进 state。
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

// 创建 member。
async fn create_member(
    State(db): State<Collection<Member>>,
    Json(input): Json<Member>,
) -> Result<Json<InsertOneResult>, (StatusCode, String)> {
    let result = db.insert_one(input).await.map_err(internal_error)?;

    Ok(Json(result))
}

// 按 id 查询 member。
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

// 替换更新 member。
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

// 按 id 删除 member。
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

// 把内部错误转成 HTTP 500。
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}

// MongoDB document 模型。
#[derive(Debug, Deserialize, Serialize)]
struct Member {
    // Rust 字段叫 id，MongoDB/JSON 字段叫 _id。
    #[serde(rename = "_id")]
    id: u32,
    name: String,
    active: bool,
}
````

## 运行和验证

先启动 MongoDB。  
默认连接地址是：

```text
mongodb://admin:password@127.0.0.1:27017/?authSource=admin
```

也可以通过环境变量指定：

````bash
export DATABASE_URL='mongodb://admin:password@127.0.0.1:27017/?authSource=admin'
````

运行：

````bash
cargo run -p example-mongodb
````

创建 member：

````bash
curl -X POST http://127.0.0.1:3000/create \
  -H 'content-type: application/json' \
  -d '{"_id":1,"name":"Alice","active":true}'
````

读取 member：

````bash
curl http://127.0.0.1:3000/read/1
````

更新 member：

````bash
curl -X PUT http://127.0.0.1:3000/update \
  -H 'content-type: application/json' \
  -d '{"_id":1,"name":"Alice Updated","active":false}'
````

删除 member：

````bash
curl -X DELETE http://127.0.0.1:3000/delete/1
````

## 常见卡点

### 1. 为什么字段叫 _id？

MongoDB 默认主键字段是 `_id`。  
Rust 里用 `id` 更顺手，所以通过 `#[serde(rename = "_id")]` 做映射。

### 2. MongoDB 需要 migration 吗？

这个 example 没有 migration。  
MongoDB 是文档数据库，通常不要求像关系型数据库那样先创建表结构。

但真实项目仍然需要管理数据结构变化，只是方式和 SQL migration 不一样。

### 3. replace_one 是局部更新吗？

不是。  
`replace_one` 是替换整个 document。

如果只想改某个字段，通常会用 MongoDB 的 `$set` 更新操作。

### 4. read_member 为什么返回 Option<Member>？

因为按 `_id` 查询可能找不到。  
找不到时返回 `None`，序列化成 JSON 会是：

```json
null
```

### 5. Client 需要连接池吗？

MongoDB driver 的 `Client` 内部管理连接。  
一般不需要像 Redis 或 PostgreSQL 那样自己再套 bb8。

**为什么 Mongo 不需要外部连接池？** 因为 mongo-rust-driver 内部就基于 tokio 实现了**自带连接池**。你创建的 `Client` 持有一个可配置的连接池（默认 `max_pool_size` = 100，可通过 `ClientOptions::max_pool_size(...)` 调整），所有 `collection.find(...)`、`collection.insert_one(...)` 操作都自动从内部池借连接。

这和前面几章形成对比：

| 方案 | 连接池在哪 |
| --- | --- |
| SQLx（32 章） | driver 自带 `PgPool` |
| tokio-postgres + bb8（33 章） | driver 不带，**你用 bb8 自己建** |
| Diesel + deadpool（34 章） | driver 不带，**你用 deadpool 自己建** |
| Redis + bb8（36 章） | driver 不带，**你用 bb8 自己建**（或用 `MultiplexedConnection`） |
| MongoDB（37 章） | driver 自带，`Client` 就是池 |

所以 Mongo 的 handler 里你看到的共享单位是 `Client` 或 `Collection`，而不是 pool。`Client` 可以被廉价 clone（内部是 `Arc`），放进 Axum state 用 `State<Client>` 提取即可。

## 手写任务

建议按下面顺序自己敲一遍：

1. 定义 `Member`，给 `id` 加 `#[serde(rename = "_id")]`。
2. 创建 MongoDB `Client`。
3. 用 `run_command(doc! { "ping": 1 })` 验证连接。
4. 在 `app(client)` 中拿到 `Collection<Member>`。
5. 用 `.with_state(collection)` 把集合放进 Router。
6. 写 `create_member`，调用 `insert_one`。
7. 写 `read_member`，调用 `find_one`。
8. 写 `update_member` 和 `delete_member`。

加深练习：

1. 新增 `/list`，查询所有 member。
2. 把 `replace_one` 改成只更新 `active` 字段。
3. 找不到 member 时返回 404，而不是 JSON `null`。

## 本章真正要记住什么

MongoDB 接入 Axum 的核心模型是：

```text
Client 连接 MongoDB
database 选择数据库
collection 选择集合和文档类型
State<Collection<Member>> 共享集合
handler 调用 CRUD 方法
serde 负责 Rust struct 和 BSON/JSON 转换
```

最容易踩坑的是 `_id`：

````rust
#[serde(rename = "_id")]
id: u32
````

理解这个映射后，MongoDB 的基本 CRUD 就比较直观。

## 源码对照

本章手写版对应源码：

- `examples/mongodb/src/main.rs`
- `examples/mongodb/Cargo.toml`
