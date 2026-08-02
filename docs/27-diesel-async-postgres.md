# 第 27 章 diesel-async-postgres:让 ORM 原生异步

## 本章你将做出什么

和[第 26 章](26-diesel-postgres.md)功能完全一样的服务——同样两条路由、同样的 schema、同样的迁移——但换成 `diesel-async`。

**变化集中在一件事上:查询变成原生异步的。** 不再需要 `interact` 包一层,双层错误变回一层,代码明显变短。

本章以**对比**为主线,重合部分不重复讲。

学完本章你将掌握:

- `diesel-async` 消掉了什么、保留了什么
- `interact` 消失后代码结构的变化
- 异步 ORM 的连接池:`AsyncDieselConnectionManager` + `bb8`
- 三套 PostgreSQL 方案(sqlx / tokio-postgres / diesel)的完整选型对比

## 前置知识

### diesel-async 改变了什么

[第 26 章](26-diesel-postgres.md#diesel-是同步的这是个大问题)讲过 diesel 的核心矛盾:API 是阻塞的,而 axum 是异步的,所以每次查询都要用 `spawn_blocking`(封装成 `interact`)丢到阻塞线程池。

`diesel-async` 是官方维护的适配层,它**重新实现了执行层**:底层用 `tokio-postgres` 这类异步驱动收发数据,把 diesel 的查询方法改成 `async fn`。

**保留的部分**(和第 26 章完全一样):

- `table!` 宏声明的 schema
- `HasQuery`、`Insertable` 等派生宏
- 整套查询 DSL(`filter`、`eq`、`insert_into`、`returning`……)
- 迁移文件和 `embed_migrations!`
- 编译期的类型检查

**改变的部分**:

| | diesel(第 26 章) | diesel-async(本章) |
|---|---|---|
| 查询执行 | 同步阻塞 | `async fn` + `.await` |
| 是否需要 `spawn_blocking` | **需要**(`interact`) | 不需要 |
| 错误层数 | 两层 | **一层** |
| 连接池 | `deadpool-diesel` | `bb8` + `AsyncDieselConnectionManager` |
| 底层驱动 | libpq(C 库) | tokio-postgres(纯 Rust) |
| 系统依赖 | 需要 libpq | **无** |

**最后一行值得注意**:`diesel-async` 用纯 Rust 驱动,不再需要系统上装 `libpq`。这让 Docker 构建和交叉编译简单不少——[第 26 章](26-diesel-postgres.md#创建项目)提过这是 diesel 相对 sqlx 的一个额外门槛,`diesel-async` 消掉了它。

### RunQueryDsl 换了一个

这是升级时最容易踩的坑。

diesel 的查询方法(`load`、`get_result`、`execute`)来自 `RunQueryDsl` trait。`diesel-async` 定义了**自己的同名 trait**:

```rust
use diesel_async::RunQueryDsl;      // 异步版本
// 而不是
use diesel::prelude::*;             // 这里面的 RunQueryDsl 是同步版本
```

两者同名但方法签名不同(一个返回值,一个返回 future)。**同时导入会产生歧义错误**,只导入同步版本则调用 `.load(...).await` 会编译失败(同步方法返回的不是 future)。

本章示例的 `use` 语句:

```rust
use diesel::prelude::*;
use diesel_async::{
    pooled_connection::{bb8, AsyncDieselConnectionManager},
    AsyncMigrationHarness, AsyncPgConnection, RunQueryDsl,   // ← 覆盖 prelude 里的同名 trait
};
```

后导入的 `diesel_async::RunQueryDsl` 会遮蔽 prelude 里的。**遇到"方法不存在"或"没有 `.await`"的报错,先检查是不是导错了 trait。**

### 连接池:bb8 而非 deadpool

`diesel-async` 自带 `bb8` 集成([第 25 章](25-tokio-postgres.md#驱动与连接池的分离)介绍过 bb8):

```rust
let config = AsyncDieselConnectionManager::<AsyncPgConnection>::new(db_url);
let pool = bb8::Pool::builder().build(config).await.unwrap();
```

结构和[第 25 章](25-tokio-postgres.md#步骤-1建池子并查询)那个 `PostgresConnectionManager` 完全一致——都是"造 manager,再造 pool"两步。

注意这个 `bb8` 是从 `diesel_async::pooled_connection::bb8` 导入的,不是直接依赖 `bb8` crate。`diesel-async` 通过 feature 提供集成:

```toml
diesel-async = { version = "0.9", features = ["postgres", "bb8", "migrations"] }
```

它也支持 `deadpool` 和 `mobc`,换个 feature 就行。

### 异步迁移

第 26 章的迁移要包在 `interact` 里,这一章有专门的异步版本:

```rust
let mut harness = AsyncMigrationHarness::new(pool.get_owned().await.unwrap());
harness.run_pending_migrations(MIGRATIONS).unwrap();
```

`AsyncMigrationHarness` 是 `diesel-async` 的 `migrations` feature 提供的。

注意用的是 `get_owned()` 而不是 `get()` —— [第 25 章](25-tokio-postgres.md#get_owned-与生命周期)讲过原因:`harness` 要持有连接,借用活不过局部作用域。

**还有一个细节**:`embed_migrations!()` 这次**没有参数**:

```rust
pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!();
```

第 26 章写的是 `embed_migrations!("migrations/")`。不带参数时宏用默认路径 `migrations/`,效果相同。

## 项目搭建

### 数据库

沿用[第 26 章](26-diesel-postgres.md#数据库)那个 `diesel_demo` 库。为了不和上一章的数据混在一起,建议新建一个:

```bash
docker exec -it axum-tutorial-pg psql -U postgres -c 'create database diesel_async_demo;'
```

### 创建项目

```bash
cargo new diesel-async-postgres
cd diesel-async-postgres
```

```bash
cargo add axum --features macros
cargo add diesel
cargo add diesel-async --features postgres,bb8,migrations
cargo add diesel_migrations
cargo add serde --features derive
cargo add tokio --features full
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

**注意 `diesel` 这次没有 `postgres` feature** —— 驱动由 `diesel-async` 提供,`diesel` 只用来贡献 DSL 和派生宏。**也因此不需要系统装 libpq 了。**

### 迁移文件

和[第 26 章](26-diesel-postgres.md#迁移文件)完全一样:

```bash
mkdir -p migrations/2023-03-14-180127_add_users
```

`up.sql`:

```sql
-- Your SQL goes here
CREATE TABLE "users"(
    "id" SERIAL PRIMARY KEY,
    "name" TEXT NOT NULL,
    "hair_color" TEXT
);
```

`down.sql`:

```sql
-- This file should undo anything in "up.sql"
DROP TABLE "users";
```

### 环境变量

```bash
export DATABASE_URL='postgres://postgres:password@localhost/diesel_async_demo'
```

## 分步构建

### 步骤 1:异步连接池与迁移

**要做什么**:搭出骨架。schema 和模型部分和[第 26 章](26-diesel-postgres.md#步骤-1schema模型与连接池)一字不差,变化只在连接池和迁移。

```rust
// src/main.rs
use axum::{routing::get, Router};
use diesel::prelude::*;
use diesel_async::{
    pooled_connection::{bb8, AsyncDieselConnectionManager},
    AsyncMigrationHarness, AsyncPgConnection, RunQueryDsl,
};
use diesel_migrations::{embed_migrations, EmbeddedMigrations, MigrationHarness};
use std::net::SocketAddr;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!();

// normally part of your generated schema.rs file
table! {
    users (id) {
        id -> Integer,
        name -> Text,
        hair_color -> Nullable<Text>,
    }
}

#[derive(serde::Serialize, HasQuery)]
struct User {
    id: i32,
    name: String,
    hair_color: Option<String>,
}

#[derive(serde::Deserialize, Insertable)]
#[diesel(table_name = users)]
struct NewUser {
    name: String,
    hair_color: Option<String>,
}

type Pool = bb8::Pool<AsyncPgConnection>;

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let db_url = std::env::var("DATABASE_URL").unwrap();

    // set up connection pool
    let config = AsyncDieselConnectionManager::<diesel_async::AsyncPgConnection>::new(db_url);
    let pool = bb8::Pool::builder().build(config).await.unwrap();

    let mut harness = AsyncMigrationHarness::new(pool.get_owned().await.unwrap());
    harness.run_pending_migrations(MIGRATIONS).unwrap();

    let app = Router::new()
        .route("/user/list", get(|| async { "not implemented yet" }))
        .with_state(pool);

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {addr}");
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

**`type Pool = bb8::Pool<AsyncPgConnection>`** —— 起别名的理由和[第 25 章](25-tokio-postgres.md#步骤-1建池子并查询)一样,类型名太长。

**连接池和迁移这段值得和第 26 章并排看:**

```rust
// 第 26 章(同步)
let manager = deadpool_diesel::postgres::Manager::new(db_url, deadpool_diesel::Runtime::Tokio1);
let pool = deadpool_diesel::postgres::Pool::builder(manager).build().unwrap();
{
    let conn = pool.get().await.unwrap();
    conn.interact(|conn| conn.run_pending_migrations(MIGRATIONS).map(|_| ()))
        .await.unwrap().unwrap();
}

// 本章(异步)
let config = AsyncDieselConnectionManager::<AsyncPgConnection>::new(db_url);
let pool = bb8::Pool::builder().build(config).await.unwrap();
let mut harness = AsyncMigrationHarness::new(pool.get_owned().await.unwrap());
harness.run_pending_migrations(MIGRATIONS).unwrap();
```

**迁移那段从 6 行变成 2 行**,而且没有了嵌套的 `unwrap().unwrap()`。

> **更好的写法**:和[第 25 章](25-tokio-postgres.md#步骤-1建池子并查询)一样,`bb8::Pool::builder()` 没有设置 `max_size` 和 `connection_timeout`。生产环境必须配——池子满了不设超时会无限排队。另外[第 26 章](26-diesel-postgres.md#步骤-1schema模型与连接池)指出的"启动时跑迁移在多实例部署下会并发执行"的问题,这里同样存在。

**跑起来看**:

```bash
export DATABASE_URL='postgres://postgres:password@localhost/diesel_async_demo'
cargo run
```

```
listening on 127.0.0.1:3000
```

确认迁移执行了:

```bash
docker exec -it axum-tutorial-pg psql -U postgres -d diesel_async_demo -c '\dt'
```

```
 public | __diesel_schema_migrations | table | postgres
 public | users                      | table | postgres
```

**和第 26 章的结果完全一致** —— 迁移机制没变,只是执行方式从阻塞变成了异步。

### 步骤 2:查询与插入

**要做什么**:实现两条路由,看 `interact` 消失后代码变成什么样。

```rust
// src/main.rs
use axum::{
    extract::State,                          // 新增
    http::StatusCode,                        // 新增
    response::Json,                          // 新增
    routing::{get, post},                    // 改动
    Router,
};
// ...(其余 use 不变)

#[tokio::main]
async fn main() {
    // ...(连接池和迁移不变)

    // 改动:挂上真正的 handler
    let app = Router::new()
        .route("/user/list", get(list_users))
        .route("/user/create", post(create_user))
        .with_state(pool);

    // ...
}

// 新增
async fn create_user(
    State(pool): State<Pool>,
    Json(new_user): Json<NewUser>,
) -> Result<Json<User>, (StatusCode, String)> {
    let mut conn = pool.get().await.map_err(internal_error)?;

    let res = diesel::insert_into(users::table)
        .values(new_user)
        .returning(User::as_returning())
        .get_result(&mut conn)
        .await
        .map_err(internal_error)?;
    Ok(Json(res))
}

// 新增
async fn list_users(State(pool): State<Pool>) -> Result<Json<Vec<User>>, (StatusCode, String)> {
    let mut conn = pool.get().await.map_err(internal_error)?;

    let res = User::query()
        .load(&mut conn)
        .await
        .map_err(internal_error)?;
    Ok(Json(res))
}

/// Utility function for mapping any error into a `500 Internal Server Error`
/// response.
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
```

**把 `list_users` 两个版本并排看,差别一目了然:**

```rust
// 第 26 章(同步)
let conn = pool.get().await.map_err(internal_error)?;
let res = conn
    .interact(|conn| User::query().load(conn))
    .await
    .map_err(internal_error)?      // interact 本身失败
    .map_err(internal_error)?;     // 查询失败

// 本章(异步)
let mut conn = pool.get().await.map_err(internal_error)?;
let res = User::query()
    .load(&mut conn)
    .await
    .map_err(internal_error)?;     // 只有一层
```

**三处变化:**

1. **没有闭包了。** 查询直接写在 handler 里,不用包进 `interact`。
2. **只有一层 `map_err`。** [第 26 章](26-diesel-postgres.md#三层错误从哪来)那个 `Result<Result<T, E1>, E2>` 消失了——因为没有 `interact` 这层抽象,自然没有它的失败模式。
3. **`&mut conn`。** 异步连接的方法要求可变引用,所以 `let mut conn`。第 26 章的 `interact` 闭包内部替你处理了这件事。

**DSL 部分一字未改** —— `User::query()`、`insert_into(...).values(...).returning(...)` 全都一样。这正是 `diesel-async` 的设计目标:**只换执行层,不动查询层。**

**跑起来看**:

```bash
cargo run
```

```bash
curl -s -X POST http://127.0.0.1:3000/user/create \
  -H 'Content-Type: application/json' \
  -d '{"name":"张三","hair_color":"黑色"}'
```

```json
{"id":1,"name":"张三","hair_color":"黑色"}
```

```bash
curl -s -X POST http://127.0.0.1:3000/user/create \
  -H 'Content-Type: application/json' -d '{"name":"李四"}'
curl -s http://127.0.0.1:3000/user/list
```

```json
[{"id":1,"name":"张三","hair_color":"黑色"},{"id":2,"name":"李四","hair_color":null}]
```

**行为和[第 26 章](26-diesel-postgres.md#步骤-3插入数据)完全一致。** 从外部看不出任何区别——这正是重点:**换执行层不影响 API 语义,只影响运行时行为和代码形态。**

**验证编译期检查仍然有效。** 临时写一个类型不匹配的查询:

```rust
    let res = users::table
        .filter(users::id.eq("not a number"))
        .load::<User>(&mut conn)
        .await
        .map_err(internal_error)?;
```

```bash
cargo build
```

编译失败——和[第 26 章](26-diesel-postgres.md#步骤-2查询列表)一样。**类型安全没有因为异步化而打折。** 改回去。

### 步骤 3:自定义连接提取器

**要做什么**:官方示例还演示了一个 `DatabaseConnection` 提取器,补上它。

```rust
// src/main.rs
use axum::{
    extract::{FromRef, FromRequestParts, State},        // 改动
    http::{request::Parts, StatusCode},                 // 改动
    response::Json,
    routing::{get, post},
    Router,
};
// ...

// ...(create_user 不变)

// 新增:we can also write a custom extractor that grabs a connection from the pool
// which setup is appropriate depends on your application
struct DatabaseConnection(bb8::PooledConnection<'static, AsyncPgConnection>);

impl<S> FromRequestParts<S> for DatabaseConnection
where
    S: Send + Sync,
    Pool: FromRef<S>,
{
    type Rejection = (StatusCode, String);

    async fn from_request_parts(_parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let pool = Pool::from_ref(state);

        let conn = pool.get_owned().await.map_err(internal_error)?;

        Ok(Self(conn))
    }
}

// 改动:改用提取器
async fn list_users(
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<Json<Vec<User>>, (StatusCode, String)> {
    let res = User::query()
        .load(&mut conn)
        .await
        .map_err(internal_error)?;
    Ok(Json(res))
}

// ...(internal_error 不变)
```

这段和[第 25 章](25-tokio-postgres.md#步骤-2自定义连接提取器)几乎一字不差:`FromRequestParts` + `FromRef<S>` 约束 + `get_owned()`。原理见[第 24 章](24-sqlx-postgres.md#fromref提取器只依赖状态的一部分)和[第 25 章](25-tokio-postgres.md#get_owned-与生命周期)。

**注意本章两条路由用了不同的方式**:`create_user` 用 `State<Pool>`(传池子),`list_users` 用 `DatabaseConnection`(先借连接)。[第 24 章](24-sqlx-postgres.md#两种取连接的方式)详细对比过两者,结论是**默认用前者**。

**跑起来看**:

```bash
cargo run
```

```bash
curl -s http://127.0.0.1:3000/user/list
curl -s -X POST http://127.0.0.1:3000/user/create \
  -H 'Content-Type: application/json' -d '{"name":"王五"}'
```

行为不变。

## 三套 PostgreSQL 方案的完整对比

第 24 到 27 章介绍了四个示例、三套方案。这里做个总的选型参考。

| | **sqlx** | **tokio-postgres** | **diesel / diesel-async** |
|---|---|---|---|
| 范式 | SQL 优先 | SQL 优先(底层) | ORM(DSL) |
| 写查询 | SQL 字符串 | SQL 字符串 | Rust 方法链 |
| 编译期校验 | `query!` 宏(需连库) | 无 | **类型系统(不需连库)** |
| 复杂 SQL | **直接写** | **直接写** | 受 DSL 限制 |
| 迁移工具 | `sqlx migrate` | 无(自己找) | **`diesel migration`** |
| 异步 | 原生 | 原生 | 需 `diesel-async` |
| 系统依赖 | 无 | 无 | diesel 需 libpq;diesel-async 无 |
| 多数据库 | PG/MySQL/SQLite/MSSQL | 仅 PG | PG/MySQL/SQLite |
| 编译速度 | 慢 | **快** | 慢 |
| 生态成熟度 | 高 | 高 | 高 |

**选型建议:**

**多数项目选 sqlx。** SQL 优先意味着没有表达力天花板,编译期校验覆盖了最常见的错误,迁移工具齐全,而且支持多种数据库。

**选 diesel 的理由:**

- 团队更喜欢用类型系统而非字符串表达查询
- 需要**编译时不连数据库**(CI 环境受限、或者不想维护 `.sqlx` 离线缓存)
- 大量简单 CRUD,DSL 的复用性能省不少代码
- 需要在多种数据库之间切换而不改查询代码

**用 diesel 就直接上 `diesel-async`**,除非有明确理由用同步版本(比如依赖的某个 diesel 扩展还不支持异步)。同步版本的 `interact` 包装、双层错误、libpq 依赖都是净成本。

**选 tokio-postgres 的理由:**需要 `COPY`、`LISTEN/NOTIFY` 这类 PostgreSQL 特有能力,或者追求编译速度。[第 25 章](25-tokio-postgres.md#步骤-3两套方案的横向对比)详细讲过。

**一个实际提醒**:这三套都不是"选了就锁死"的决定,但迁移成本确实不低——查询代码、模型定义、错误处理都要重写。**项目早期花半天做个原型对比,比后期迁移划算得多。**

## 完整代码

`migrations/2023-03-14-180127_add_users/up.sql` 和 `down.sql` 与[第 26 章](26-diesel-postgres.md#完整代码)相同。

`src/main.rs`:

```rust
use axum::{
    extract::{FromRef, FromRequestParts, State},
    http::{request::Parts, StatusCode},
    response::Json,
    routing::{get, post},
    Router,
};
use diesel::prelude::*;
use diesel_async::{
    pooled_connection::{bb8, AsyncDieselConnectionManager},
    AsyncMigrationHarness, AsyncPgConnection, RunQueryDsl,
};
use diesel_migrations::{embed_migrations, EmbeddedMigrations, MigrationHarness};
use std::net::SocketAddr;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!();

// normally part of your generated schema.rs file
table! {
    users (id) {
        id -> Integer,
        name -> Text,
        hair_color -> Nullable<Text>,
    }
}

#[derive(serde::Serialize, HasQuery)]
struct User {
    id: i32,
    name: String,
    hair_color: Option<String>,
}

#[derive(serde::Deserialize, Insertable)]
#[diesel(table_name = users)]
struct NewUser {
    name: String,
    hair_color: Option<String>,
}

type Pool = bb8::Pool<AsyncPgConnection>;

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let db_url = std::env::var("DATABASE_URL").unwrap();

    // set up connection pool
    let config = AsyncDieselConnectionManager::<diesel_async::AsyncPgConnection>::new(db_url);
    let pool = bb8::Pool::builder().build(config).await.unwrap();

    let mut harness = AsyncMigrationHarness::new(pool.get_owned().await.unwrap());
    harness.run_pending_migrations(MIGRATIONS).unwrap();

    // build our application with some routes
    let app = Router::new()
        .route("/user/list", get(list_users))
        .route("/user/create", post(create_user))
        .with_state(pool);

    // run it with hyper
    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {addr}");
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn create_user(
    State(pool): State<Pool>,
    Json(new_user): Json<NewUser>,
) -> Result<Json<User>, (StatusCode, String)> {
    let mut conn = pool.get().await.map_err(internal_error)?;

    let res = diesel::insert_into(users::table)
        .values(new_user)
        .returning(User::as_returning())
        .get_result(&mut conn)
        .await
        .map_err(internal_error)?;
    Ok(Json(res))
}

// we can also write a custom extractor that grabs a connection from the pool
// which setup is appropriate depends on your application
struct DatabaseConnection(bb8::PooledConnection<'static, AsyncPgConnection>);

impl<S> FromRequestParts<S> for DatabaseConnection
where
    S: Send + Sync,
    Pool: FromRef<S>,
{
    type Rejection = (StatusCode, String);

    async fn from_request_parts(_parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let pool = Pool::from_ref(state);

        let conn = pool.get_owned().await.map_err(internal_error)?;

        Ok(Self(conn))
    }
}

async fn list_users(
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<Json<Vec<User>>, (StatusCode, String)> {
    let res = User::query()
        .load(&mut conn)
        .await
        .map_err(internal_error)?;
    Ok(Json(res))
}

/// Utility function for mapping any error into a `500 Internal Server Error`
/// response.
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
```

环境变量:`DATABASE_URL`(必需)

## 代价与边界

**连接池未配置。** 步骤 1 的「更好的写法」已经指出。

**启动时跑迁移的多实例问题。** 和[第 26 章](26-diesel-postgres.md#步骤-1schema模型与连接池)相同。

**错误信息泄露。** `internal_error` 把数据库错误直接返回给客户端,第 24~26 章都指出过,本章代码相同。

**`diesel-async` 比 diesel 年轻。** 生态里部分 diesel 扩展(某些类型支持、第三方 DSL 扩展)还没跟上异步版本。选之前确认你需要的功能都支持。

**`RunQueryDsl` 的导入歧义。** 前置知识讲过,这是升级时最容易踩的坑。

**没有事务示例。** 四章数据库示例都没演示事务。真实业务里"扣库存 + 建订单"这类操作必须在一个事务里,`diesel-async` 的写法是 `conn.transaction(|conn| async move { ... }.scope_boxed()).await`。这是使用数据库的必备知识,示例没覆盖。

## 动手练习

**1. 写一个事务。**

提示:加一条 `POST /user/create-two`,在**同一个事务**里插入两个用户,如果第二个失败则第一个也回滚。写法:

```rust
conn.transaction(|conn| async move {
    diesel::insert_into(users::table).values(&u1).execute(conn).await?;
    diesel::insert_into(users::table).values(&u2).execute(conn).await?;
    Ok::<_, diesel::result::Error>(())
}.scope_boxed()).await
```

需要 `use diesel_async::scoped_futures::ScopedFutureExt`。验证:让第二个插入违反约束(比如 `name` 传 null),确认第一个也没进库。**这是「代价与边界」里说的那个示例没覆盖但实际必备的知识。**

**2. 对比两章的编译产物和编译时间。**

提示:分别在第 26、27 章的项目里跑 `cargo build --release --timings`,对比总耗时。再用 `ldd target/release/diesel-postgres`(Linux)或 `otool -L`(macOS)看动态链接的库——第 26 章那个会依赖 `libpq`,本章的不会。**这个差别在做 Docker 镜像时很实际。**

**3. 把 `create_user` 也改成用 `DatabaseConnection` 提取器,然后想清楚该不该这么做。**

提示:改起来很简单。但[第 24 章](24-sqlx-postgres.md#两种取连接的方式)讲过,提取器方式会让连接在整个 handler 期间被占用。`create_user` 只有一次查询,没有理由这么做。**这个练习的重点是"能做"和"该做"的区别。**

## 小结与下一章预告

`diesel-async` 只替换执行层:schema、派生宏、查询 DSL、迁移、编译期类型检查全部保留,查询方法从同步变成 `async fn`。

带来的实际变化是三点:**不再需要 `interact` 包装**、**双层错误变回一层**、**不再依赖系统的 libpq**(底层用纯 Rust 的 tokio-postgres)。代码明显更短更直白,而且 API 语义完全不变——从外部看两章的服务行为一模一样。

升级时最容易踩的坑是 `RunQueryDsl`:`diesel_async` 有自己的同名 trait,必须导入它而不是 `diesel::prelude::*` 里那个,否则会报"方法不存在"或"不能 await"。

**用 diesel 就直接上 `diesel-async`**,同步版本的包装成本、双层错误、系统依赖都是净成本,除非有明确理由。

三套 PostgreSQL 方案的选型:**多数项目用 sqlx**(SQL 优先、无表达力天花板、编译期校验);想用类型系统表达查询、或者需要编译时不连数据库,选 diesel;需要 PostgreSQL 特有能力或追求编译速度,选 tokio-postgres。

四章数据库示例都没演示**事务**,而它是真实业务的必备品——练习题 1 补上了这一课。

关系型数据库告一段落。[第 28 章 tokio-redis](28-tokio-redis.md) 换一类存储:Redis 是内存 KV,和[第 22 章](22-key-value-store.md)那个自己写的 `HashMap` 解决同样的问题,但多了持久化、过期、原子操作和跨进程共享。你会看到连接池在这里的形态,以及一个新问题——**Redis 的值是无类型的字节,怎么和 Rust 的类型系统对接**。
