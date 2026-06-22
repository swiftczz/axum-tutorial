# 35. diesel-async-postgres

对应示例：`examples/diesel-async-postgres`

> 5 个数据库方案的对比见[第 32 章"先理解：5 个数据库方案怎么选"](./32-sqlx-postgres.md#先理解5-个数据库方案怎么选)。本章是异步 Diesel（diesel-async）的写法，相比 ch34 同步版查询直接 `.await`，不需要 `interact` 包装。

上一章用 Diesel 同步查询需 `deadpool-diesel` 的 `interact` 把同步查询扔进阻塞线程池。这章换 `diesel-async`——查询本身是异步的，可以直接 `.await`，不需要 `interact` 包装。

分 4 步：先建 schema/model 和连接池，再加 migration，再加 `create_user`（用 `State<Pool>`），最后加 `list_users`（用自定义 `DatabaseConnection` extractor 演示另一种获取连接的写法）。

相比前面章节新引入：**`AsyncDieselConnectionManager` + bb8、`AsyncMigrationHarness`、`HasQuery` derive、无 `interact` 直接 `.await`、自定义 `FromRequestParts` 取连接**。

## Cargo.toml

````toml
[package]
name = "example-diesel-async-postgres"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["macros"] }
diesel = "~2.3"
diesel-async = { version = "0.9", features = ["postgres", "bb8", "migrations"] }
diesel_migrations = "~2.3"
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

`diesel-async` 启用 `postgres`（PG 支持）、`bb8`（连接池）、`migrations`（异步 migration）。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：schema + model + 连接池骨架

先搭最小可运行版本：定义 `users` 表、`User` model，创建 bb8 连接池，跑一个空的 Router。这步先把数据库基础设施搭好。

````rust
use axum::{routing::get, Router};
use diesel::prelude::*;
use diesel_async::{
    pooled_connection::{bb8, AsyncDieselConnectionManager},
    AsyncPgConnection,
};
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

    // 建连接池
    let config = AsyncDieselConnectionManager::<diesel_async::AsyncPgConnection>::new(db_url);
    let pool = bb8::Pool::builder().build(config).await.unwrap();

    let app = Router::new().route("/health", get(|| async { "ok" }));

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
cargo run -p example-diesel-async-postgres
````

````bash
curl http://127.0.0.1:3000/health
````

返回 `ok` 就说明池建好了。这步还没用到 `pool`——下一步用。

> **新面孔：`AsyncDieselConnectionManager` + bb8**
>
> 和第 33 章 tokio-postgres 的 bb8 连接池同构：manager 负责创建/回收连接，pool 负责复用。
>
> 区别是连接类型变成 `AsyncPgConnection`（diesel-async 的异步 PG 连接），manager 是 `AsyncDieselConnectionManager`。`pool.get().await` 拿到的是可直接 `.await` 查询的异步连接。

> **新面孔：`HasQuery` derive**
>
> diesel-async 为 model 提供的派生宏（替代上章同步版的 `Queryable`/`SelectClause`）。derive 后能用 `User::query()` 链式构建查询。`User` 还 derive 了 `serde::Serialize`，所以能直接作为 `Json` 响应返回。

---

## 第二步：跑 migration

数据库表结构要用 migration 建。这步在启动时自动跑 pending migrations。

````rust
use diesel_async::{AsyncMigrationHarness, RunQueryDsl}; // 新增
use diesel_migrations::{embed_migrations, EmbeddedMigrations, MigrationHarness}; // 新增

pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!();

// main 里建池之后加这几行：
# async fn main() {
#     // ... 建池 ...
#     let pool = bb8::Pool::builder().build(config).await.unwrap();

    let mut harness = AsyncMigrationHarness::new(pool.get_owned().await.unwrap());
    harness.run_pending_migrations(MIGRATIONS).unwrap();

#     let app = Router::new().route("/health", get(|| async { "ok" }));
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

> **新面孔：`AsyncMigrationHarness`**
>
> 上章用同步的 `MigrationHarness::run_pending_migrations`，这章换成异步的 `AsyncMigrationHarness`。它需要从池里借一个 owned 连接（`pool.get_owned().await`）来跑 migration。
>
> `embed_migrations!()` 宏在编译时把 `migrations/` 目录嵌入二进制，部署时不需要带 SQL 文件。

---

## 第三步：`create_user`——直接 `.await` 异步插入

加 `POST /user/create`，用 `State<Pool>` 获取连接。这是和上章同步版最大的区别——查询直接 `.await`，没有 `interact` 包装。

````rust
use axum::{
    extract::State,
    http::StatusCode,
    response::Json,
    routing::{get, post},
    Router,
};

#[derive(serde::Deserialize, Insertable)]
#[diesel(table_name = users)]
struct NewUser {
    name: String,
    hair_color: Option<String>,
}

# type Pool = bb8::Pool<AsyncPgConnection>;

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

/// 把任何错误映射成 `500 Internal Server Error` 响应
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}

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

返回创建好的用户 JSON。

> **新面孔：直接 `.await` 的异步查询（`RunQueryDsl`）**
>
> 对比上一章同步 Diesel：
>
> ```text
> 同步 Diesel:  conn.interact(|conn| query.get_result(conn)).await
>                 （interact 把同步查询扔阻塞线程池，等通知）
> Diesel Async: query.get_result(&mut conn).await
>                 （查询本身异步，直接 await）
> ```
>
> `diesel_async::RunQueryDsl` 提供了 `.get_result(...).await` 和 `.load(...).await` 这些异步方法，所以**必须导入 `RunQueryDsl`**，否则编译器找不到方法。
>
> 错误处理也只有一层 `map_err`（同步版有两层：interact 的 + 查询的）。

---

## 第四步：`list_users`——自定义 `DatabaseConnection` extractor

第三步用 `State<Pool>` 获取连接。这步演示另一种写法：自定义 extractor 把"从池借连接"封装成 `DatabaseConnection`，handler 参数更清爽。

````rust
use axum::{
    extract::{FromRef, FromRequestParts},
    http::request::Parts,
};

struct DatabaseConnection(bb8::PooledConnection<'static, AsyncDieselConnectionManager<AsyncPgConnection>>);

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

> **新面孔：自定义 `FromRequestParts` 取连接**
>
> 模式和第 33 章 tokio-postgres 的 `DatabaseConnection` 几乎一样：从 state 取 pool → `pool.get_owned()` 借 owned 连接 → 包装。
>
> - `Pool: FromRef<S>` 约束保证能从 state 切出 pool（同 ch18 依赖注入）
> - `FromRequestParts` 合适因为不读 body（只是从 state 取连接）
> - `bb8::PooledConnection` 借出后自动归还池
>
> 两种写法（`State<Pool>` vs 自定义 extractor）都合法，真实项目选一种风格统一即可。

---

## 完整代码

````rust
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
````

## 运行

````bash
export DATABASE_URL='postgres://localhost/your_db'
cd examples
cargo run -p example-diesel-async-postgres
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

**为什么不用 `interact`？** diesel-async 提供异步查询方法，可直接 `.await`；上章同步 Diesel 才需要 interact。

**为什么需要 `diesel` crate？** diesel-async 只负责异步执行，表结构 / derive / 查询 DSL 仍来自 Diesel。

**为什么导入 `RunQueryDsl`？** 异步 `.get_result(...).await` / `.load(...).await` 来自它，忘导入找不到方法。

**两种获取连接方式？** `create_user` 直接 `State<Pool>`，`list_users` 自定义 `DatabaseConnection` extractor——演示两种写法，真实项目统一一种。

**Diesel Async 一定比同步好？** 不一定，它更适合全异步栈，选择时考虑团队熟悉度、生态、已有代码、部署环境。

## 手写任务

按下面顺序敲：

1. 复用上章 migration 和 `table!`。
2. 定义 `User` 和 `NewUser`。
3. 定义 `type Pool = bb8::Pool<AsyncPgConnection>`。
4. `AsyncDieselConnectionManager` 创建连接池。
5. 运行 pending migrations。
6. 写 `create_user` 直接用 `State<Pool>`。
7. 实现 `DatabaseConnection` extractor。
8. 写 `list_users` 用自定义 extractor。

加深练习：

1. `list_users` 改成也用 `State<Pool>`，对比代码。
2. 新增 `/user/{id}` 查询单个用户。
3. 错误响应改成统一 JSON 格式。

## 小结

这章用 4 步把 diesel-async 的关键组件搭起来：

1. **schema + pool**：`table!` + `User` + bb8 连接池（`AsyncDieselConnectionManager`）。
2. **migration**：`AsyncMigrationHarness` 启动时跑 pending migrations。
3. **`create_user`**：`State<Pool>` 取连接，查询直接 `.await`，没有 `interact`。
4. **`list_users`**：自定义 `DatabaseConnection` extractor，演示另一种取连接的写法。

核心区别：

```text
同步 Diesel:   conn.interact(|conn| query.get_result(conn)).await
Diesel Async:  query.get_result(&mut conn).await
```

diesel-async 改变的是连接和执行方式，不是表结构定义；`RunQueryDsl` 提供异步方法，必须导入。

## 源码对照

- `examples/diesel-async-postgres/Cargo.toml`
- `examples/diesel-async-postgres/src/main.rs`
- `examples/diesel-async-postgres/migrations/2023-03-14-180127_add_users/up.sql`
- `examples/diesel-async-postgres/migrations/2023-03-14-180127_add_users/down.sql`
