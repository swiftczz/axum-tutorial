# 34. diesel-postgres

对应示例：`examples/diesel-postgres`

> 5 个数据库方案的对比见[第 32 章"先理解：5 个数据库方案怎么选"](./32-sqlx-postgres.md#先理解5-个数据库方案怎么选)。本章是同步 Diesel + deadpool-diesel 的写法。

前两章用异步 PostgreSQL 客户端，这章用 Diesel。Diesel 的查询 API 是**同步阻塞风格**，所以用 `deadpool-diesel` 把它接入 axum 异步服务——核心是通过 `conn.interact(...)` 把同步查询扔到阻塞线程池执行。用 Diesel + PostgreSQL 实现用户创建与列表查询。

分 4 步：先建 schema/model + 连接池，再加 migration，再加 `create_user`（理解 `interact`），最后加 `list_users`。

相比前面章节新引入：**`embed_migrations!`、`table!`、`deadpool-diesel`、`conn.interact`（本质是 `spawn_blocking`）、Diesel 的 `HasQuery`/`Insertable` derive**。

## Cargo.toml

````toml
[package]
name = "example-diesel-postgres"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["macros"] }
deadpool-diesel = { version = "0.6.1", features = ["postgres"] }
diesel = { version = "2", features = ["postgres"] }
diesel_migrations = "2"
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

本章相比前面章节新增三个 Diesel 相关依赖：`diesel`（同步 ORM，启用 `postgres` feature）+ `deadpool-diesel`（把同步 Diesel 接入 async 的连接池，提供 `conn.interact(...)` 把同步查询扔进阻塞线程池）+ `diesel_migrations`（编译期嵌入 SQL migration）。对比第 35 章的 `diesel-async`（异步 Diesel，直接 `.await` 不需 `interact`）。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：schema + model + `deadpool-diesel` 连接池

先搭最小可运行版本：定义 `users` 表、`User` model，创建 deadpool-diesel 连接池，跑一个 health check 路由。这步把数据库基础设施搭好。

````rust
use axum::{routing::get, Router};
use diesel::prelude::*;
use std::net::SocketAddr;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 正常情况下这部分由 diesel 自动生成的 schema.rs 提供
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

    // 建连接池
    let manager = deadpool_diesel::postgres::Manager::new(db_url, deadpool_diesel::Runtime::Tokio1);
    let pool = deadpool_diesel::postgres::Pool::builder(manager)
        .build()
        .unwrap();

    let app = Router::new()
        .route("/health", get(|| async { "ok" }))
        .with_state(pool);

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {addr}");
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
````

验证（需要本地 PostgreSQL，并设好 `DATABASE_URL`）：

````bash
export DATABASE_URL='postgres://localhost/your_db'
cd examples
cargo run -p example-diesel-postgres
````

````bash
curl http://127.0.0.1:3000/health
````

返回 `ok` 说明池建好了。

> **新面孔：`table!` 宏**
>
> Diesel 用 `table!` 宏定义表结构（通常由 `diesel print-schema` 自动生成到 `schema.rs`）。宏生成 `users::table`、`users::id`、`users::name` 等常量，是 Diesel 查询 DSL 的基础。
>
> 教程为了单文件清晰，手写 `table!`；真实项目用 `diesel print-schema` 从数据库自动生成。

> **新面孔：`deadpool-diesel` 连接池**
>
> Diesel 的连接是**同步阻塞**的（`PgConnection` 不能在 async 上下文直接用）。`deadpool-diesel` 提供异步连接池：
> - `Manager::new(url, Runtime::Tokio1)` 创建 manager
> - `Pool::builder(manager).build()` 建池
> - `pool.get().await` 借一个异步 `deadpool_diesel::Connection` wrapper
>
> 连接是 `deadpool_diesel` 包装过的，不是裸 `PgConnection`——下一步会看到 `conn.interact(...)` 用法。

> **新面孔：`HasQuery` / `Insertable` derive**
>
> - `User` derive `HasQuery`：能用 `User::query()` 链式查询（Diesel 2.x 的新 API）
> - `User` derive `serde::Serialize`：作为 `Json` 响应
> - `NewUser` derive `Insertable` + `#[diesel(table_name = users)]`：可插入 `users` 表
>
> 两套 model 是常见模式：`User` 是查询出来的完整行，`NewUser` 是插入时的输入（无 id，因为 id 由数据库生成）。

---

## 第二步：跑 migration

数据库表结构要用 migration 建。这步在启动时自动跑 pending migrations——Diesel 的 migration 是同步的，所以要在 `conn.interact` 里跑。

````rust
use diesel_migrations::{embed_migrations, EmbeddedMigrations, MigrationHarness}; // 新增

// 把 migrations/ 目录嵌入二进制
pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!("migrations/");

# #[tokio::main]
# async fn main() {
#     // ... 建池 ...
#     let pool = deadpool_diesel::postgres::Pool::builder(manager).build().unwrap();

    // 启动时跑 pending migrations
    {
        let conn = pool.get().await.unwrap();
        conn.interact(|conn| conn.run_pending_migrations(MIGRATIONS).map(|_| ()))
            .await
            .unwrap()
            .unwrap();
    }

#     let app = Router::new().route("/health", get(|| async { "ok" })).with_state(pool);
#     // ...
# }
````

`migrations/2023-03-14-180127_add_users/up.sql`：

````sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    hair_color TEXT
)
````

`down.sql`：

````sql
DROP TABLE users
````

> **新面孔：`conn.interact`（阻塞查询的解法）**
>
> Diesel 的 `run_pending_migrations` 是**同步阻塞**的——直接调会阻塞 tokio runtime。`conn.interact(|conn| ...)` 把同步闭包扔到专门的阻塞线程池执行（本质是 `spawn_blocking`），返回 future 让你 `.await`。
>
> 这是 deadpool-diesel 的核心：**把同步 Diesel API 包成 async 友好**。后面所有 Diesel 查询都要走 `interact`。

> **新面孔：`embed_migrations!("migrations/")`**
>
> 编译时把 SQL 文件嵌入二进制（路径相对 `CARGO_MANIFEST_DIR`）。部署时不需要带 migrations 目录，启动自动跑 pending migrations。

---

## 第三步：`create_user`——用 `interact` 跑插入

加 `POST /user/create`。Diesel 查询通过 `conn.interact(|conn| query.get_result(conn))` 扔到阻塞线程池。

````rust
use axum::{
    extract::State,
    http::StatusCode,
    response::Json,
    routing::{get, post},
};

#[derive(serde::Deserialize, Insertable)]
#[diesel(table_name = users)]
struct NewUser {
    name: String,
    hair_color: Option<String>,
}

async fn create_user(
    State(pool): State<deadpool_diesel::postgres::Pool>,
    Json(new_user): Json<NewUser>,
) -> Result<Json<User>, (StatusCode, String)> {
    let conn = pool.get().await.map_err(internal_error)?;
    let res = conn
        .interact(|conn| {
            diesel::insert_into(users::table)
                .values(new_user)
                .returning(User::as_returning())
                .get_result(conn)
        })
        .await
        .map_err(internal_error)?
        .map_err(internal_error)?;
    Ok(Json(res))
}

/// 把任何错误映射成 `500 Internal Server Error` 响应
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}

# #[tokio::main]
# async fn main() {
#     // ...
    let app = Router::new()
        .route("/health", get(|| async { "ok" }))
        .route("/user/create", post(create_user))
        .with_state(pool);
#     // ...
# }
````

验证：

````bash
curl -X POST http://127.0.0.1:3000/user/create \
  -H 'content-type: application/json' \
  -d '{"name":"Alice","hair_color":"black"}'
````

返回创建好的用户 JSON（包含数据库生成的 `id`）。

> **新面孔：Diesel 查询 DSL（`insert_into` / `values` / `returning`）**
>
> Diesel 的查询是类型安全的链式 DSL：
> - `insert_into(users::table)`：插入到 users 表
> - `.values(new_user)`：插入 NewUser（必须 derive `Insertable`）
> - `.returning(User::as_returning())`：返回插入的完整 User 行
> - `.get_result(conn)`：在同步 conn 上执行
>
> 因为 Diesel 是同步的，整条链路在 `interact` 的闭包里跑。`.get_result(conn)` 返回 `Result<User, DieselError>`——这就是为什么外面有两层 `map_err`（一层 interact、一层 query）。

> **新面孔：两层错误来源**
>
> `conn.interact(...).await?` 有两层错误：
> 1. 第一层 `map_err`：interact 本身失败（pool 故障、panic 等）
> 2. 第二层 `map_err`：闭包返回的 Diesel 查询错误（约束冲突、SQL 错误等）
>
> 这是同步 API 包 async 的代价——下一章 diesel-async 直接 `.await` 查询，只有一层。

---

## 第四步：`list_users`——查询列表

加 `GET /user/list`。同样是 `interact` 模式，只是查询换成 `User::query().load(conn)`。

````rust
async fn list_users(
    State(pool): State<deadpool_diesel::postgres::Pool>,
) -> Result<Json<Vec<User>>, (StatusCode, String)> {
    let conn = pool.get().await.map_err(internal_error)?;
    let res = conn
        .interact(|conn| User::query().load(conn))
        .await
        .map_err(internal_error)?
        .map_err(internal_error)?;
    Ok(Json(res))
}

# #[tokio::main]
# async fn main() {
#     // ...
    let app = Router::new()
        .route("/health", get(|| async { "ok" }))
        .route("/user/list", get(list_users))
        .route("/user/create", post(create_user))
        .with_state(pool);
#     // ...
# }
````

验证：

````bash
curl http://127.0.0.1:3000/user/list
````

返回所有用户列表。

> **新面孔：`User::query().load(conn)`**
>
> `HasQuery` derive 提供的链式查询入口。`User::query()` 等价于 `users::table.select(User::as_select())`，`.load(conn)` 加载所有行成 `Vec<User>`。
>
> 可在中间加过滤：`User::query().filter(users::name.eq("Alice")).load(conn)`。

---

## 完整代码

````rust
use axum::{
    extract::State,
    http::StatusCode,
    response::Json,
    routing::{get, post},
    Router,
};
use diesel::prelude::*;
use diesel_migrations::{embed_migrations, EmbeddedMigrations, MigrationHarness};
use std::net::SocketAddr;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// this embeds the migrations into the application binary
// the migration path is relative to the `CARGO_MANIFEST_DIR`
pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!("migrations/");

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
    let manager = deadpool_diesel::postgres::Manager::new(db_url, deadpool_diesel::Runtime::Tokio1);
    let pool = deadpool_diesel::postgres::Pool::builder(manager)
        .build()
        .unwrap();

    // run the migrations on server startup
    {
        let conn = pool.get().await.unwrap();
        conn.interact(|conn| conn.run_pending_migrations(MIGRATIONS).map(|_| ()))
            .await
            .unwrap()
            .unwrap();
    }

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
    State(pool): State<deadpool_diesel::postgres::Pool>,
    Json(new_user): Json<NewUser>,
) -> Result<Json<User>, (StatusCode, String)> {
    let conn = pool.get().await.map_err(internal_error)?;
    let res = conn
        .interact(|conn| {
            diesel::insert_into(users::table)
                .values(new_user)
                .returning(User::as_returning())
                .get_result(conn)
        })
        .await
        .map_err(internal_error)?
        .map_err(internal_error)?;
    Ok(Json(res))
}

async fn list_users(
    State(pool): State<deadpool_diesel::postgres::Pool>,
) -> Result<Json<Vec<User>>, (StatusCode, String)> {
    let conn = pool.get().await.map_err(internal_error)?;
    let res = conn
        .interact(|conn| User::query().load(conn))
        .await
        .map_err(internal_error)?
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
````

## 运行

````bash
export DATABASE_URL='postgres://localhost/your_db'
cd examples
cargo run -p example-diesel-postgres
````

创建用户：

````bash
curl -X POST http://127.0.0.1:3000/user/create \
  -H 'content-type: application/json' \
  -d '{"name":"Alice","hair_color":"black"}'
````

查询列表：

````bash
curl http://127.0.0.1:3000/user/list
````

启动失败检查：`DATABASE_URL` 是否设置、PostgreSQL 是否启动、数据库是否存在、用户是否有建表权限。

## 常见问题

**为什么必须用 `conn.interact`？** Diesel 是同步阻塞的，直接调会阻塞 tokio runtime。`interact` 把同步闭包扔到阻塞线程池（本质 `spawn_blocking`）。

**为什么有两层 `map_err`？** 第一层是 interact 本身失败（pool/panic），第二层是闭包返回的 Diesel 查询错误。下一章 diesel-async 直接 `.await`，只有一层。

**`table!` 是手写的吗？** 教程手写是为单文件清晰；真实项目用 `diesel print-schema` 从数据库自动生成 `schema.rs`。

**为什么 `User` 和 `NewUser` 分开？** `User` 是查询出来的完整行（含 id），`NewUser` 是插入时的输入（无 id，由数据库生成）。这是常见模式。

## 手写任务

按下面顺序敲：

1. 写 `table!` 定义 `users` 表。
2. 定义 `User`（derive `Serialize` + `HasQuery`）和 `NewUser`（derive `Deserialize` + `Insertable`）。
3. 用 `deadpool_diesel::postgres::Manager` + `Pool::builder` 建池。
4. 写 migration（`up.sql` / `down.sql`），用 `embed_migrations!` 嵌入。
5. 启动时 `conn.interact(|conn| conn.run_pending_migrations(...))` 跑 migration。
6. 写 `create_user`：`conn.interact(|conn| insert_into...get_result(conn))`。
7. 写 `list_users`：`conn.interact(|conn| User::query().load(conn))`。
8. 给每个错误路径加 `internal_error`。

加深练习：

1. 加 `DELETE /user/{id}`：`User::query().filter(users::id.eq(id)).delete(conn)`。
2. `list_users` 加分页：`.limit(20).offset(0)`。
3. 错误响应改成统一 JSON 格式而不是 `(StatusCode, String)`。

## 小结

这章用 4 步把 Diesel + deadpool-diesel 搭起来：

1. **schema + pool**：`table!` + `User`/`NewUser` + `deadpool_diesel::postgres::Pool`。
2. **migration**：`embed_migrations!` + `conn.interact(|conn| run_pending_migrations)`。
3. **`create_user`**：`insert_into...get_result(conn)` 包在 `interact` 里，两层 `map_err`。
4. **`list_users`**：`User::query().load(conn)`，同样的 `interact` 模式。

核心：**Diesel 是同步的，所有查询必须走 `conn.interact(|conn| ...)`**——本质是 `spawn_blocking`。代价是两层错误来源。下一章 diesel-async 改成直接 `.await`。

## 源码对照

- `examples/diesel-postgres/Cargo.toml`
- `examples/diesel-postgres/src/main.rs`
- `examples/diesel-postgres/migrations/2023-03-14-180127_add_users/up.sql`
- `examples/diesel-postgres/migrations/2023-03-14-180127_add_users/down.sql`
