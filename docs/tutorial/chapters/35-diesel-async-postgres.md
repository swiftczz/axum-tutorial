# 35. diesel-async-postgres

对应示例：`examples/diesel-async-postgres`

上一章用 Diesel 同步查询需 `deadpool-diesel` 的 `interact`,这章用 `diesel-async`——查询本身是异步的,可以直接 `.await`,不需要 `interact` 包装。理解它和同步 Diesel 的差异,掌握 async 查询、连接池、自定义 extractor。

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

`diesel-async` 启用 `postgres`(PG 支持)、`bb8`(连接池)、`migrations`(异步 migration)。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

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

    let config = AsyncDieselConnectionManager::<diesel_async::AsyncPgConnection>::new(db_url);
    let pool = bb8::Pool::builder().build(config).await.unwrap();

    let mut harness = AsyncMigrationHarness::new(pool.get_owned().await.unwrap());
    harness.run_pending_migrations(MIGRATIONS).unwrap();

    let app = Router::new()
        .route("/user/list", get(list_users))
        .route("/user/create", post(create_user))
        .with_state(pool);

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

struct DatabaseConnection(
    bb8::PooledConnection<'static, AsyncDieselConnectionManager<AsyncPgConnection>>,
)

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

创建用户:

````bash
curl -X POST http://127.0.0.1:3000/user/create \
  -H 'content-type: application/json' \
  -d '{"name":"Alice","hair_color":"black"}'
````

查询列表:

````bash
curl http://127.0.0.1:3000/user/list
````

启动失败检查:DATABASE_URL 是否设置、PostgreSQL 是否启动、数据库是否存在、用户是否有建表权限。

## 解读

### 和上一章最大的区别

```text
同步 Diesel:   conn.interact(|conn| diesel_query.get_result(conn)).await
                (interact 把同步查询扔阻塞线程池,等通知)
Diesel Async:  diesel_query.get_result(&mut conn).await
                (查询本身异步,直接 await)
```

其他概念基本一样(migration/schema/User/NewUser/连接池/State/Json/错误处理)。**diesel-async 改变的是连接和执行方式,不是表结构定义方式**。

### 异步连接池

````rust
type Pool = bb8::Pool<AsyncPgConnection>;

let config = AsyncDieselConnectionManager::<AsyncPgConnection>::new(db_url);
let pool = bb8::Pool::builder().build(config).await.unwrap();
````

这里的 `bb8` 来自 `diesel_async::pooled_connection::bb8`,连接类型是 `AsyncPgConnection`。和 tokio-postgres 章节类似:manager 负责创建连接,pool 负责复用,只是连接是 Diesel Async 的 `AsyncPgConnection`。

### `create_user` 直接异步插入

````rust
async fn create_user(
    State(pool): State<Pool>,
    Json(new_user): Json<NewUser>,
) -> Result<Json<User>, (StatusCode, String)> {
    let mut conn = pool.get().await.map_err(internal_error)?;

    let res = diesel::insert_into(users::table)
        .values(new_user)
        .returning(User::as_returning())
        .get_result(&mut conn)     // 直接 .await,没有 interact!
        .await
        .map_err(internal_error)?;

    Ok(Json(res))
}
````

最大变化是**没有 `conn.interact(...)`**,直接 `.get_result(&mut conn).await`。因为 `diesel_async::RunQueryDsl` 提供异步执行能力。注意只有一层 `map_err`(没有 interact 那层)。

**为什么还需要 `diesel` crate?** `diesel-async` 负责异步执行,表结构/derive/查询 DSL 仍然来自 Diesel。

**为什么导入 `RunQueryDsl`?** 异步的 `.get_result(...).await` 和 `.load(...).await` 来自这个 trait,忘记导入编译器找不到方法。

### 自定义 `DatabaseConnection` extractor

````rust
struct DatabaseConnection(
    bb8::PooledConnection<'static, AsyncDieselConnectionManager<AsyncPgConnection>>,
)

impl<S> FromRequestParts<S> for DatabaseConnection
where S: Send + Sync, Pool: FromRef<S>,
{
    async fn from_request_parts(_parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let pool = Pool::from_ref(state);
        let conn = pool.get_owned().await.map_err(internal_error)?;
        Ok(Self(conn))
    }
}
````

模式和第 33 章 tokio-postgres 几乎一样:从 state 取 pool → `pool.get_owned()` 借 owned connection → 包装。`FromRequestParts` 合适因为不读 body。

### `list_users` 用自定义 extractor

````rust
async fn list_users(
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<Json<Vec<User>>, (StatusCode, String)> {
    let res = users::table.select(User::as_select()).load(&mut conn).await.map_err(internal_error)?;
    Ok(Json(res))
}
````

`create_user` 用 `State<Pool>` 演示一种写法,`list_users` 用自定义 extractor 演示另一种——真实项目选一种风格统一即可。

### migration 和 schema 同上章

`up.sql`、`table!`、`User`、`NewUser` 都和第 34 章一样。migration 用 `AsyncMigrationHarness`:

````rust
let mut harness = AsyncMigrationHarness::new(pool.get_owned().await.unwrap());
harness.run_pending_migrations(MIGRATIONS).unwrap();
````

`embed_migrations!()` 不传路径时默认用 crate 下 `migrations` 目录。

## 常见问题

**为什么不用 interact?** diesel-async 提供异步查询方法,可直接 `.await`;上章同步 Diesel 才需要 interact。

**为什么需要 `diesel` crate?** diesel-async 只负责异步执行,表结构/derive/查询 DSL 仍来自 Diesel。

**为什么导入 `RunQueryDsl`?** 异步 `.get_result(...).await`/`.load(...).await` 来自它,忘导入找不到方法。

**两种获取连接方式?** `create_user` 直接 `State<Pool>`,`list_users` 自定义 `DatabaseConnection` extractor——演示两种写法,真实项目统一一种。

**Diesel Async 一定比同步好?** 不一定,它更适合全异步栈,选择时考虑团队熟悉度、生态、已有代码、部署环境。

## 手写任务

按下面顺序敲:

1. 复用上章 migration 和 `table!`。
2. 定义 `User` 和 `NewUser`。
3. 定义 `type Pool = bb8::Pool<AsyncPgConnection>`。
4. `AsyncDieselConnectionManager` 创建连接池。
5. 运行 pending migrations。
6. 写 `create_user` 直接用 `State<Pool>`。
7. 实现 `DatabaseConnection` extractor。
8. 写 `list_users` 用自定义 extractor。

加深练习:

1. `list_users` 改成也用 `State<Pool>`,对比代码。
2. 新增 `/user/{id}` 查询单个用户。
3. 错误响应改成统一 JSON 格式。

## 小结

- Diesel Async 和同步 Diesel 业务结构一样(migration/schema/model/pool/handler/Json response),关键差异是查询执行方式:
  ```text
  同步 Diesel:   conn.interact(|conn| query.get_result(conn)).await
  Diesel Async:  query.get_result(&mut conn).await
  ```
- diesel-async 改变连接和执行方式,不改变表结构定义;`RunQueryDsl` 提供异步方法,必须导入。
- 不需要 `interact` 包装、少一层 `map_err`、查询直接 `.await`,代码更直接,也少一次线程间切换。
- 异步连接池用 bb8 + `AsyncPgConnection`,manager 是 `AsyncDieselConnectionManager`。
- 自定义 extractor 模式同 tokio-postgres 章;真实项目两种获取连接写法选一种统一。

## 源码对照

- `examples/diesel-async-postgres/Cargo.toml`
- `examples/diesel-async-postgres/src/main.rs`
- `examples/diesel-async-postgres/migrations/2023-03-14-180127_add_users/up.sql`
- `examples/diesel-async-postgres/migrations/2023-03-14-180127_add_users/down.sql`
