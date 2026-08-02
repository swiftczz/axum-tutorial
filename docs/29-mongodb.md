# 第 29 章 mongodb:文档数据库与完整 CRUD

## 本章你将做出什么

一个操作 MongoDB 的完整 CRUD 服务:

```
POST   /create      创建成员
GET    /read/{id}   按 id 查询
PUT    /update      整体替换
DELETE /delete/{id} 删除
```

**这是「状态与数据库」阶段唯一一个完整的增删改查示例** —— 前面那些要么只跑 `select 1+1`,要么只读一个硬编码的 key。

学完本章你将掌握:

- 文档数据库与关系型数据库的根本差别
- MongoDB 驱动的连接管理:**为什么 `Collection<T>` 可以直接当 `State`**
- `doc!` 宏与 BSON,以及 `_id` 这个特殊字段怎么和 Rust 字段对接
- 官方示例在 REST 设计上的几处问题

## 前置知识

### 文档数据库:没有固定 schema

关系型数据库([第 24~27 章](24-sqlx-postgres.md))的核心约束是**表结构固定**:每张表有确定的列和类型,插入不符合结构的数据会被拒绝。

MongoDB 存的是**文档**——本质上就是 JSON 对象(实际编码是 BSON,下面会讲):

```json
{ "_id": 1, "name": "张三", "active": true }
{ "_id": 2, "name": "李四", "active": false, "email": "li@example.com" }
```

**注意第二条多了 `email` 字段,而这完全合法。** 同一个集合(collection,相当于表)里的文档可以有不同的结构。

| | 关系型 | MongoDB |
|---|---|---|
| 容器 | 表(table) | 集合(collection) |
| 记录 | 行(row) | 文档(document) |
| 结构 | 固定 schema | **无固定 schema** |
| 主键 | 自定义列名 | **固定叫 `_id`** |
| 关联 | JOIN | 嵌套文档 / 手动多次查询 |
| 事务 | 天然支持 | 4.0 后支持,但需副本集 |

**"无 schema" 是双刃剑。** 好处是加字段不用改表结构、不用跑迁移([第 26 章](26-diesel-postgres.md#迁移数据库结构的版本控制)那套流程在这里不存在)。代价是**数据一致性完全靠应用层保证** —— 数据库不会拦住你写入一个 `active` 是字符串的文档。

**实际上 schema 并没有消失,只是从数据库挪到了代码里。** 本章那个 `Member` struct 就是 schema,只不过它是单方面的约定——**数据库不知道也不强制**。这意味着老数据可能不符合新代码的假设,反序列化时才炸。

MongoDB 后来加了 JSON Schema 校验功能来补这个洞,但那又回到了"定义 schema"的路上。

### BSON 与 doc! 宏

MongoDB 的实际存储格式是 **BSON**(Binary JSON),它在 JSON 基础上加了 JSON 没有的类型:`ObjectId`、`Date`、`Binary`、`Decimal128`、64 位整数等。

构造 BSON 文档用 `doc!` 宏,语法接近 JSON:

```rust
doc! { "_id": id }
doc! { "age": { "$gt": 18 } }
doc! { "ping": 1 }
```

它主要用在**查询条件**上——本章四个 handler 里有三个用它指定"操作哪条文档"。

MongoDB 的查询语言本身就是文档:`{ "age": { "$gt": 18 } }` 表示"age 大于 18"。`$gt`、`$in`、`$or` 这些是操作符。这和 SQL 的 `WHERE age > 18` 是同一件事的不同表达。

**注意 `doc!` 里的字段名是字符串,拼错不会编译失败** —— 查询条件写错只会查不到数据,不报错。这是文档数据库相对 [diesel](26-diesel-postgres.md#orm用类型系统表达查询) 那种类型安全 DSL 的明显劣势。

### _id:固定的主键名

MongoDB 的主键**必须叫 `_id`**,这是硬性规定。不指定的话数据库会自动生成一个 `ObjectId`(12 字节的唯一标识)。

但 `_id` 在 Rust 里不是个好字段名——下划线开头通常表示"未使用"。所以要用 serde 重命名:

```rust
#[derive(Debug, Deserialize, Serialize)]
struct Member {
    #[serde(rename = "_id")]
    id: u32,
    name: String,
    active: bool,
}
```

`#[serde(rename = "_id")]` 让 Rust 里叫 `id`、BSON 里叫 `_id`。**这个属性同时影响序列化和反序列化**,所以存和取都对得上。

**本章示例用 `u32` 当 `_id`,由客户端指定。** 这是可以的(MongoDB 允许任意类型的 `_id`),但不是常规做法——通常用自动生成的 `ObjectId`。「代价与边界」会讲这个选择的问题。

### 为什么 Collection<T> 能直接当 State

**这是本章在 axum 层面最值得注意的地方。**

[第 24~28 章](24-sqlx-postgres.md)的模式都是:把**连接池**放进 `State`,handler 里再从池子借连接。本章不同:

```rust
fn app(client: Client) -> Router {
    let collection: Collection<Member> = client.database("axum-mongo").collection("members");

    Router::new()
        // ...
        .with_state(collection)
}
```

直接把 `Collection<Member>` 当状态,handler 里也没有"借连接"这一步:

```rust
async fn read_member(
    State(db): State<Collection<Member>>,
    Path(id): Path<u32>,
) -> ... {
    db.find_one(doc! { "_id": id }).await
}
```

原因是 **MongoDB 驱动自己管理连接池**。`Client` 内部已经维护了一个连接池,`Database` 和 `Collection` 都是**廉价可克隆的句柄**——它们持有 `Client` 的引用计数,不占用实际连接。执行操作时驱动内部自动借还连接。

所以这里不需要 `bb8`、不需要 `deadpool`、不需要自定义 `DatabaseConnection` 提取器。**这是驱动设计的差别,不是 axum 的用法差别。**

`Collection<T>` 带类型参数,意味着**这个集合的文档会被序列化/反序列化成 `T`**。同一个集合也可以拿不同的类型去操作:`client.database("x").collection::<Document>("members")` 就是拿原始 BSON 文档。

### 四个操作对应的方法

| HTTP | MongoDB 方法 | 返回类型 |
|---|---|---|
| `POST /create` | `insert_one(doc)` | `InsertOneResult` |
| `GET /read/{id}` | `find_one(filter)` | `Option<T>` |
| `PUT /update` | `replace_one(filter, doc)` | `UpdateResult` |
| `DELETE /delete/{id}` | `delete_one(filter)` | `DeleteResult` |

**注意三个写操作返回的是"结果元数据"而不是文档本身:**

- `InsertOneResult` 含 `inserted_id`
- `UpdateResult` 含 `matched_count`、`modified_count`
- `DeleteResult` 含 `deleted_count`

**`find_one` 返回 `Option<T>`** —— 找不到就是 `None`,不是错误。这比[第 28 章](28-tokio-redis.md#无类型的值怎么和-rust-对接) Redis 那个"key 不存在会解析失败"友好得多。

**`replace_one` 是整体替换,不是部分更新。** 它用新文档完全覆盖旧文档,对应 HTTP 的 `PUT` 语义([第 23 章](23-todos.md#put-与-patch整体替换-vs-部分更新)讲过 `PUT` 和 `PATCH` 的区别)。部分更新要用 `update_one` 配合 `$set` 操作符。

## 项目搭建

### 启动 MongoDB

```bash
docker run --name axum-tutorial-mongo \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password \
  -p 27017:27017 \
  -d mongo:7
```

确认能连上:

```bash
docker exec -it axum-tutorial-mongo mongosh -u admin -p password --eval 'db.runCommand({ping: 1})'
```

用完清理:

```bash
docker stop axum-tutorial-mongo && docker rm axum-tutorial-mongo
```

### 创建项目

```bash
cargo new mongodb
cd mongodb
```

```bash
cargo add axum
cargo add mongodb --no-default-features --features bson-3,compat-3-3-0,rustls-tls
cargo add serde --features derive
cargo add tokio --features full
cargo add tower-http --features add-extension,trace
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`mongodb` 的 feature 组合是官方示例指定的:`bson-3` 用 BSON 3.x,`compat-3-3-0` 是版本兼容标记,`rustls-tls` 用纯 Rust 的 TLS 实现(避免依赖系统的 OpenSSL,和[第 27 章](27-diesel-async-postgres.md#diesel-async-改变了什么)提到的 libpq 问题同理)。

## 分步构建

### 步骤 1:连接与数据模型

**要做什么**:连上 MongoDB,定义 `Member` 类型,做启动检查。

```rust
// src/main.rs
use axum::{routing::get, Router};
use mongodb::{bson::doc, Client, Collection};
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
        .route("/", get(|| async { "not implemented yet" }))
        .layer(TraceLayer::new_for_http())
        .with_state(collection)
}

// defining Member type
#[derive(Debug, Deserialize, Serialize)]
struct Member {
    #[serde(rename = "_id")]
    id: u32,
    name: String,
    active: bool,
}
```

**连接串格式**:`mongodb://用户:密码@主机:端口/?authSource=admin`。`authSource=admin` 指明去 `admin` 库验证凭据——Docker 镜像创建的 root 用户存在那里,不写会认证失败。

**`Client::with_uri_str`** 建立客户端。和[第 28 章](28-tokio-redis.md#连接池)的 `redis::Client::open` 不同,它**会真的去连**并做初步握手。

**`run_command(doc! { "ping": 1 })`** 是启动检查,直接发一条原始命令。`ping` 是 MongoDB 的内置命令,不需要任何权限。

**`fn app(client: Client) -> Router`** 这个模式[第 03 章](03-form.md#步骤-1显示表单页面)讲过,为测试服务([第 21 章](21-testing.md#fn-app-router-的价值兑现)兑现了它的价值)。

**`client.database("axum-mongo").collection("members")`** 拿到集合句柄。前置知识讲过它是廉价可克隆的,所以能直接当 `State`。

**注意一个顺序问题**:`main` 里**先连数据库、后初始化日志**。所以连接阶段的输出用的是 `println!` 而不是 `tracing::debug!`——因为那时订阅器还没注册,tracing 宏会静默丢弃。

> **更好的写法**:日志初始化应该放在最前面。示例先连数据库再配日志,导致连接阶段的信息只能用 `println!` 输出,无法被日志系统统一收集(线上环境通常靠日志采集,`println!` 到 stdout 的内容格式不一致、没有级别和时间戳)。而且如果连接失败,`.unwrap()` 的 panic 信息也不在日志体系里。把 `tracing_subscriber` 那段挪到 `main` 开头即可。

**跑起来看**:

```bash
cargo run
```

```
Pinged your database. Successfully connected to MongoDB!
DEBUG mongodb: Listening on 127.0.0.1:3000
```

**验证连不上时的行为。** 停掉容器:

```bash
docker stop axum-tutorial-mongo
cargo run
```

```
thread 'main' panicked at src/main.rs:...:
called `Result::unwrap()` on an `Err` value: Error { kind: ServerSelectionError { ... } }
```

启动失败,符合[第 24 章](24-sqlx-postgres.md#步骤-1连上数据库用-state-拿池子)讲的"快速失败"原则。重新启动:

```bash
docker start axum-tutorial-mongo
```

### 步骤 2:创建与查询

**要做什么**:实现 `POST /create` 和 `GET /read/{id}`。

```rust
// src/main.rs
use axum::{
    extract::{Path, State},                      // 新增
    http::StatusCode,                            // 新增
    routing::{get, post},                        // 改动
    Json, Router,                                // 改动
};
use mongodb::{
    bson::doc,
    results::InsertOneResult,                    // 新增
    Client, Collection,
};
// ...(其余 use 不变)

// ...(main 不变)

fn app(client: Client) -> Router {
    let collection: Collection<Member> = client.database("axum-mongo").collection("members");

    // 改动:挂上两条路由
    Router::new()
        .route("/create", post(create_member))
        .route("/read/{id}", get(read_member))
        .layer(TraceLayer::new_for_http())
        .with_state(collection)
}

// 新增:handler to create a new member
async fn create_member(
    State(db): State<Collection<Member>>,
    Json(input): Json<Member>,
) -> Result<Json<InsertOneResult>, (StatusCode, String)> {
    let result = db.insert_one(input).await.map_err(internal_error)?;

    Ok(Json(result))
}

// 新增:handler to read an existing member
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

// 新增
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}

// ...(Member 不变)
```

**`db.insert_one(input)`** 直接传 `Member` —— 因为 `Collection<Member>` 知道该怎么序列化它。`Member` 派生的 `Serialize` 在这里起作用。

**`db.find_one(doc! { "_id": id })`** 用 BSON 文档做查询条件。返回 `Option<Member>`,反序列化靠 `Deserialize`。

**`Json(input): Json<Member>` 用 `Member` 当输入类型。** 这意味着客户端必须提供 `id`、`name`、`active` 三个字段。

> **更好的写法**:输入和输出用同一个类型,让客户端可以**自己指定主键 `id`**。[第 23 章](23-todos.md#步骤-2创建)讲过为什么要分开定义:客户端指定主键会带来 id 冲突、覆盖他人数据等问题。MongoDB 的常规做法是让 `_id` 用自动生成的 `ObjectId`,Rust 侧写成 `#[serde(rename = "_id", skip_serializing_if = "Option::is_none")] id: Option<ObjectId>`,插入时不带、返回时带上。示例这么写是为了让 `read/{id}` 能用一个好记的数字 id 演示,别照搬。

**跑起来看**:

```bash
cargo run
```

创建:

```bash
curl -s -X POST http://127.0.0.1:3000/create \
  -H 'Content-Type: application/json' \
  -d '{"_id":1,"name":"张三","active":true}'
```

```json
{"inserted_id":1}
```

**注意请求体里的字段名是 `_id` 而不是 `id`** —— `#[serde(rename = "_id")]` 同时作用于反序列化,所以 HTTP 接口暴露的也是 `_id`。这个副作用示例没说明,「代价与边界」会讲。

查询:

```bash
curl -s http://127.0.0.1:3000/read/1
```

```json
{"_id":1,"name":"张三","active":true}
```

查一个不存在的:

```bash
curl -i -s http://127.0.0.1:3000/read/999
```

```
HTTP/1.1 200 OK
content-type: application/json

null
```

**`200 OK` 加 `null`,而不是 `404`。** 因为 handler 返回的是 `Json<Option<Member>>`,`None` 序列化成 `null`。

> **更好的写法**:资源不存在应该返回 `404 Not Found` 而不是 `200` + `null`。[第 23 章](23-todos.md#各操作该返回什么状态码)讲过 REST 的状态码惯例。改法很简单:`result.map(Json).ok_or((StatusCode::NOT_FOUND, "member not found".into()))`。示例直接返回 `Option` 是图省事,但这会让客户端无法用状态码判断结果,必须解析 body 才知道有没有数据。

**验证无 schema 的特性。** 从 mongosh 直接插入一条结构不同的文档:

```bash
docker exec -it axum-tutorial-mongo mongosh -u admin -p password --authenticationDatabase admin \
  axum-mongo --eval 'db.members.insertOne({_id: 2, name: "李四", active: false, email: "li@example.com"})'
```

```bash
curl -s http://127.0.0.1:3000/read/2
```

```json
{"_id":2,"name":"李四","active":false}
```

**多出来的 `email` 字段被静默忽略了** —— serde 默认跳过未知字段。数据库接受了这条文档,Rust 侧只取自己认识的部分。这就是前置知识说的"schema 从数据库挪到了代码里"。

**再验证反过来的情况:少字段。**

```bash
docker exec -it axum-tutorial-mongo mongosh -u admin -p password --authenticationDatabase admin \
  axum-mongo --eval 'db.members.insertOne({_id: 3, name: "王五"})'
curl -i -s http://127.0.0.1:3000/read/3
```

```
HTTP/1.1 500 Internal Server Error

Error { kind: BsonDeserialization(DeserializationError { message: "missing field `active`" }) ... }
```

**这次炸了。** 缺 `active` 字段的文档无法反序列化成 `Member`。

**这是文档数据库最需要警惕的地方**:数据库不拦你写入任何结构,但代码有自己的假设。**老数据、别的服务写的数据、手工改过的数据,都可能不符合当前代码的 schema,而问题只在读取时才暴露。** 关系型数据库不会有这个问题——列不存在的话插入就被拒了。

清理测试数据:

```bash
docker exec -it axum-tutorial-mongo mongosh -u admin -p password --authenticationDatabase admin \
  axum-mongo --eval 'db.members.deleteMany({_id: {$in: [2, 3]}})'
```

### 步骤 3:更新与删除

**要做什么**:补齐 `PUT /update` 和 `DELETE /delete/{id}`,完成 CRUD。

```rust
// src/main.rs
use axum::{
    extract::{Path, State},
    http::StatusCode,
    routing::{delete, get, post, put},            // 改动
    Json, Router,
};
use mongodb::{
    bson::doc,
    results::{DeleteResult, InsertOneResult, UpdateResult},   // 改动
    Client, Collection,
};
// ...

fn app(client: Client) -> Router {
    let collection: Collection<Member> = client.database("axum-mongo").collection("members");

    // 改动:四条路由齐全
    Router::new()
        .route("/create", post(create_member))
        .route("/read/{id}", get(read_member))
        .route("/update", put(update_member))
        .route("/delete/{id}", delete(delete_member))
        .layer(TraceLayer::new_for_http())
        .with_state(collection)
}

// ...(create_member、read_member 不变)

// 新增:handler to update an existing member
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

// 新增:handler to delete an existing member
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

// ...(internal_error、Member 不变)
```

**`update_member` 的 id 从 body 里来**,不是路径参数——所以路由是 `/update` 而不是 `/update/{id}`。这偏离了 REST 惯例,「代价与边界」会讲。

**`replace_one(filter, input)`** 是整体替换,对应 `PUT` 语义。注意 `doc! { "_id": input.id }` 在 `input` 被移动进 `replace_one` 之前先读了 `input.id` —— Rust 的求值顺序保证了这能工作。

**跑起来看**:

```bash
cargo run
```

完整走一遍 CRUD:

```bash
# 创建
curl -s -X POST http://127.0.0.1:3000/create \
  -H 'Content-Type: application/json' \
  -d '{"_id":10,"name":"赵六","active":true}'

# 读取
curl -s http://127.0.0.1:3000/read/10

# 更新
curl -s -X PUT http://127.0.0.1:3000/update \
  -H 'Content-Type: application/json' \
  -d '{"_id":10,"name":"赵六","active":false}'

# 确认改了
curl -s http://127.0.0.1:3000/read/10

# 删除
curl -s -X DELETE http://127.0.0.1:3000/delete/10

# 确认没了
curl -s http://127.0.0.1:3000/read/10
```

```json
{"inserted_id":10}
{"_id":10,"name":"赵六","active":true}
{"matched_count":1,"modified_count":1,"upserted_id":null}
{"_id":10,"name":"赵六","active":false}
{"deleted_count":1}
null
```

**三个写操作返回的是元数据**,前置知识讲过。`matched_count: 1, modified_count: 1` 表示匹配到 1 条、实际改了 1 条。

**验证"操作不存在的文档"。** 删一个不存在的:

```bash
curl -s -X DELETE http://127.0.0.1:3000/delete/999
```

```json
{"deleted_count":0}
```

**`200 OK`,`deleted_count: 0`。** 更新不存在的同理:

```bash
curl -s -X PUT http://127.0.0.1:3000/update \
  -H 'Content-Type: application/json' \
  -d '{"_id":999,"name":"不存在","active":true}'
```

```json
{"matched_count":0,"modified_count":0,"upserted_id":null}
```

> **更好的写法**:操作不存在的资源应该返回 `404`,而不是 `200` 加一个 `count: 0` 的元数据。客户端必须解析 body 才能知道操作有没有生效,这违反了[第 23 章](23-todos.md#各操作该返回什么状态码)讲的 REST 惯例。改法:检查 `result.deleted_count` / `result.matched_count`,为 0 时返回 `StatusCode::NOT_FOUND`。删除成功也该返回 `204 No Content` 而不是 `200` 加元数据。

**验证重复创建。** 同一个 `_id` 插两次:

```bash
curl -s -X POST http://127.0.0.1:3000/create \
  -H 'Content-Type: application/json' -d '{"_id":20,"name":"测试","active":true}'
curl -s -X POST http://127.0.0.1:3000/create \
  -H 'Content-Type: application/json' -d '{"_id":20,"name":"测试2","active":true}'
```

```json
{"inserted_id":20}
Kind: Write error: WriteError { code: 11000, ... "E11000 duplicate key error collection: axum-mongo.members index: _id_ dup key: { _id: 20 }" ...
```

**第二次失败了,`_id` 唯一约束生效** —— 这是 MongoDB 少数几个强制约束之一。但返回的是 `500` 加一段暴露了库名和集合名的错误,应该是 `409 Conflict` 加一句简短说明。

清理:

```bash
docker exec -it axum-tutorial-mongo mongosh -u admin -p password --authenticationDatabase admin \
  axum-mongo --eval 'db.members.deleteMany({})'
```

## 完整代码

`src/main.rs`:

```rust
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
```

## 代价与边界

**这个示例在 REST 设计上有几处问题**,上面的「更好的写法」逐一指出了,汇总一下:

| 问题 | 应该是 |
|---|---|
| `read` 找不到返回 `200` + `null` | `404 Not Found` |
| `delete`/`update` 影响 0 条返回 `200` + `count: 0` | `404 Not Found` |
| `delete` 成功返回 `200` + 元数据 | `204 No Content` |
| `update` 的 id 在 body 里,路由是 `/update` | `PUT /members/{id}` |
| 路由是 `/create`、`/read/{id}` 这种动作式 URL | `POST /members`、`GET /members/{id}` |
| 输入输出共用 `Member`,客户端能指定主键 | 分开定义,`_id` 用 `ObjectId` 自动生成 |
| 重复 `_id` 返回 `500` 加原始错误 | `409 Conflict` |

**最后一行的 URL 设计问题最典型。** [第 23 章](23-todos.md#rest-的核心约定)讲过 REST 的第一条约定是"URL 标识资源,不描述动作"。`/create`、`/read/{id}` 把动作放进了 URL,而动作本该由 HTTP 方法表达。规范的设计是:

```
POST   /members        创建
GET    /members/{id}   读取
PUT    /members/{id}   替换
DELETE /members/{id}   删除
```

**无 schema 意味着读取时才发现数据不合规。** 步骤 2 验证过:缺字段的文档会在反序列化时炸成 `500`。缓解办法有三个——给字段加 `Option` 或 `#[serde(default)]`、在 MongoDB 侧配 JSON Schema 校验、或者写迁移脚本统一老数据。**但这个风险不会消失,只能管理。**

**没有索引。** 示例只用 `_id` 查询(有默认索引),但真实业务一定会按其它字段查。MongoDB 不会自动建索引,没索引的查询是全集合扫描。**这是 MongoDB 性能问题的头号来源。**

**没有分页。** `find_one` 只取一条所以没问题,但一旦加了"列出全部"的接口,必须用 `.limit()` 和 `.skip()`——[第 23 章](23-todos.md#query-提取器与分页)讲过分页为什么是必需品。

**日志初始化在连接之后。** 步骤 1 的「更好的写法」指出了。

**错误信息泄露。** `internal_error` 把驱动的原始错误返回给客户端,上面那个重复 key 的响应里就带着库名和集合名。这个问题从[第 24 章](24-sqlx-postgres.md#步骤-1连上数据库用-state-拿池子)一直存在。

**没有事务。** 和前面四章一样。MongoDB 的事务需要副本集(单节点部署默认没有),这也是它常被诟病的地方。

## 动手练习

**1. 把 REST 设计改对。**

提示:路由改成 `POST /members`、`GET /members/{id}`、`PUT /members/{id}`、`DELETE /members/{id}`,用[第 02 章](02-routes-and-handlers-close-together.md#router-与-methodrouter-的分工)的 `MethodRouter` 链式写法合并同路径的不同方法。然后按上面那张表逐条改状态码:找不到返回 `404`、删除成功返回 `204`、重复 key 返回 `409`。**这个练习把[第 23 章](23-todos.md)学的 REST 惯例应用了一遍。**

**2. 改用自动生成的 `ObjectId`。**

提示:`id` 字段改成 `#[serde(rename = "_id", skip_serializing_if = "Option::is_none")] id: Option<ObjectId>`,创建时客户端不传 id,由 MongoDB 生成。`insert_one` 的返回值 `inserted_id` 里能拿到生成的 id。查询路由的 `Path<u32>` 要改成 `Path<String>` 再解析成 `ObjectId`。**做完你会理解为什么示例图省事用了 `u32`。**

**3. 加一个列表接口,体会索引和分页。**

提示:`GET /members?active=true&limit=10`。用 `db.find(doc! { "active": active }).limit(10).await?` 得到一个游标,再用 `cursor.try_collect::<Vec<_>>().await?` 收集(需要 `futures::TryStreamExt`)。然后给 `active` 建索引:

```bash
docker exec -it axum-tutorial-mongo mongosh -u admin -p password --authenticationDatabase admin \
  axum-mongo --eval 'db.members.createIndex({active: 1})'
```

用 `db.members.find({active: true}).explain()` 对比建索引前后的执行计划。

## 小结与下一章预告

MongoDB 是文档数据库,存的是 BSON 文档,**没有固定 schema**。加字段不用迁移,但数据一致性完全靠应用层保证——schema 并没有消失,只是从数据库挪进了代码里的 struct,而**数据库不知道也不强制**。本章验证过两种后果:多余字段被静默忽略、缺失字段在反序列化时炸成 `500`。

主键固定叫 `_id`,靠 `#[serde(rename = "_id")]` 和 Rust 字段对接。查询条件用 `doc!` 宏构造,**字段名是字符串,拼错不会编译失败**。

axum 层面最值得注意的是:**`Collection<T>` 可以直接当 `State`**,不需要连接池、不需要自定义连接提取器——因为 MongoDB 驱动的 `Client` 内部已经管理连接池,`Collection` 只是廉价可克隆的句柄。这是驱动设计的差别,不是 axum 用法的差别。

这个示例是本阶段唯一的完整 CRUD,但**REST 设计有七处问题**(动作式 URL、找不到返回 `200`+`null`、删除返回元数据而非 `204`、输入输出共用类型让客户端能指定主键……),「代价与边界」列了对照表,练习题 1 让你改一遍。

---

「状态与数据库」阶段(22~29)到此结束。你现在能把数据存进内存、关系型数据库、Redis 和 MongoDB,也理解了连接池、`FromRef` 提取器、共享可变状态这些通用机制。

接下来是「认证与安全」阶段。[第 30 章 jwt](30-jwt.md) 讲最常用的无状态认证方案:JWT 的三段结构是什么、签名怎么防篡改、为什么"无状态"既是优点也是它最大的坑(**签发出去的 token 撤销不了**),以及怎么写一个从 `Authorization` 头部提取并校验身份的提取器。
