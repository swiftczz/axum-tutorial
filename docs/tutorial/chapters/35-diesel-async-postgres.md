# 35. diesel-async-postgres

对应示例：`examples/diesel-async-postgres`

本章目标：使用 `diesel-async` 连接 PostgreSQL，理解它和上一章同步 Diesel 的差异，并掌握 async 查询、连接池和自定义 extractor 的写法。

上一章使用 Diesel 同步查询，所以需要 `deadpool-diesel` 的 `interact`。  
这一章使用 `diesel-async`，查询本身是异步的，可以直接 `.await`。

## 这个小项目在做什么

应用仍然提供两个接口：

```text
GET  /user/list   -> 查询所有用户
POST /user/create -> 创建用户
```

数据库表仍然是：

```text
users(id, name, hair_color)
```

启动时会运行 migration，创建 `users` 表。

请求主线是：

```text
程序启动
-> 读取 DATABASE_URL
-> 创建 AsyncDieselConnectionManager
-> 创建 bb8 异步连接池
-> 运行 migration
-> Router 保存 pool
-> create_user 通过 State<Pool> 获取连接
-> list_users 通过 DatabaseConnection extractor 获取连接
-> Diesel async 查询 .await
-> 返回 JSON
```

## 和上一章最大的区别

同步 Diesel 版：

````rust
conn.interact(|conn| {
    diesel_query.get_result(conn)
})
.await
````

Diesel Async 版：

````rust
diesel_query
    .get_result(&mut conn)
    .await
````

也就是说：

```text
同步 Diesel：阻塞查询放进 interact
Diesel Async：查询本身可以 await
```

其他概念基本一样：

```text
migration
schema
User
NewUser
连接池
State
Json
错误处理
```

## 文件和依赖

这个 example 有四个主要文件：

1. `examples/diesel-async-postgres/Cargo.toml`：声明 Axum、Diesel、diesel-async、migration、serde。
2. `examples/diesel-async-postgres/src/main.rs`：实现 schema、模型、异步连接池、migration、路由。
3. `examples/diesel-async-postgres/migrations/.../up.sql`：创建 `users` 表。
4. `examples/diesel-async-postgres/migrations/.../down.sql`：删除 `users` 表。

关键依赖：

- `diesel`：提供 schema、derive 和查询 DSL。
- `diesel-async`：提供异步 PostgreSQL 连接、异步查询和 bb8 集成。
- `diesel_migrations`：嵌入 migration。
- `axum`：提供 Router、State、Json、FromRequestParts。
- `serde`：请求和响应 JSON。
- `tokio`：异步运行时。

`Cargo.toml` 关键配置：

````toml
diesel = "~2.3"
diesel-async = { version = "0.9", features = ["postgres", "bb8", "migrations"] }
diesel_migrations = "~2.3"
````

`diesel-async` 启用了：

- `postgres`：PostgreSQL 支持。
- `bb8`：bb8 连接池支持。
- `migrations`：异步 migration 支持。

## 第一步：migration 和 schema 仍然一样

`up.sql`：

````sql
CREATE TABLE "users"(
    "id" SERIAL PRIMARY KEY,
    "name" TEXT NOT NULL,
    "hair_color" TEXT
);
````

`table!`：

````rust
table! {
    users (id) {
        id -> Integer,
        name -> Text,
        hair_color -> Nullable<Text>,
    }
}
````

这里和上一章几乎一样。

这说明：

```text
diesel-async 改变的是连接和执行方式
不是表结构定义方式
```

## 第二步：定义 User 和 NewUser

源码：

````rust
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
````

`User` 用于数据库查询结果和响应 JSON。  
`NewUser` 用于请求 JSON 和插入数据库。

和上一章一样：

```text
id 由数据库生成，所以 NewUser 没有 id
hair_color 允许为空，所以用 Option<String>
```

## 第三步：定义异步连接池类型

源码：

````rust
type Pool = bb8::Pool<AsyncPgConnection>;
````

这里的 `bb8` 来自：

````rust
diesel_async::pooled_connection::bb8
````

连接类型是：

````rust
AsyncPgConnection
````

所以本章连接池可以理解成：

```text
bb8 管理的一组 AsyncPgConnection
```

后面 handler 里会用：

````rust
State(pool): State<Pool>
````

拿到它。

## 第四步：创建 AsyncDieselConnectionManager

源码：

````rust
let db_url = std::env::var("DATABASE_URL").unwrap();

let config = AsyncDieselConnectionManager::<diesel_async::AsyncPgConnection>::new(db_url);
let pool = bb8::Pool::builder().build(config).await.unwrap();
````

流程是：

1. 从 `DATABASE_URL` 读取连接字符串。
2. 创建 `AsyncDieselConnectionManager`。
3. 用 `bb8::Pool::builder()` 创建连接池。

和 tokio-postgres 章节类似：

```text
manager 负责创建连接
pool 负责复用连接
```

只是这里的连接是 Diesel Async 的 `AsyncPgConnection`。

## 第五步：运行 migration

源码：

````rust
let mut harness = AsyncMigrationHarness::new(pool.get_owned().await.unwrap());
harness.run_pending_migrations(MIGRATIONS).unwrap();
````

这段从连接池拿一个 owned connection，然后创建 migration harness。  
接着执行未运行过的 migration。

`MIGRATIONS` 来自：

````rust
pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!();
````

没有传路径时，默认使用 crate 下的 `migrations` 目录。

## 第六步：注册路由

源码：

````rust
let app = Router::new()
    .route("/user/list", get(list_users))
    .route("/user/create", post(create_user))
    .with_state(pool);
````

两个接口：

```text
GET  /user/list
POST /user/create
```

连接池作为应用 state。  
后面 `create_user` 直接从 State 获取 pool，`list_users` 演示自定义 extractor 获取连接。

## 第七步：create_user 直接异步插入

源码：

````rust
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
````

和上一章相比，最大的变化是没有：

````rust
conn.interact(...)
````

这里直接：

````rust
.get_result(&mut conn)
.await
````

因为 `diesel_async::RunQueryDsl` 提供了异步执行能力。

handler 逻辑是：

1. 从 state 拿 pool。
2. 从 pool 借连接。
3. insert 用户。
4. 返回插入后的用户 JSON。

## 第八步：自定义 DatabaseConnection extractor

源码：

````rust
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
````

这和第 33 章 tokio-postgres 的 extractor 很像。

它的作用是：

```text
handler 需要 DatabaseConnection
-> Axum 从 state 里找到 Pool
-> pool.get_owned() 借一个 owned connection
-> 包成 DatabaseConnection
```

`FromRequestParts` 仍然合适，因为数据库连接 extractor 不需要读取请求体。

## 第九步：list_users 使用自定义连接

源码：

````rust
async fn list_users(
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<Json<Vec<User>>, (StatusCode, String)> {
    let res = User::query()
        .load(&mut conn)
        .await
        .map_err(internal_error)?;
    Ok(Json(res))
}
````

handler 里没有 `State(pool)`。  
它直接拿到：

````rust
DatabaseConnection(mut conn)
````

查询时：

````rust
User::query().load(&mut conn).await
````

返回：

````rust
Json<Vec<User>>
````

这展示了 Diesel Async 查询列表的基本写法。

## 第十步：RunQueryDsl 的作用

源码导入：

````rust
use diesel_async::{
    pooled_connection::{bb8, AsyncDieselConnectionManager},
    AsyncMigrationHarness, AsyncPgConnection, RunQueryDsl,
};
````

`RunQueryDsl` 很关键。  
它让 Diesel 查询拥有异步方法：

````rust
.get_result(&mut conn).await
.load(&mut conn).await
````

如果忘记导入它，编译器可能会提示找不到对应方法。

## 函数职责速查

- `main`：初始化日志，读取 `DATABASE_URL`，创建异步连接池，运行 migration，注册路由并启动服务。
- `create_user`：从 State 获取连接池，异步插入用户，返回插入后的用户。
- `DatabaseConnection::from_request_parts`：从 state 中取 pool，并借出 owned async connection。
- `list_users`：通过自定义 extractor 获取连接，异步查询用户列表。
- `internal_error`：把内部错误转成 HTTP 500。
- `User` / `NewUser`：分别表示响应模型和创建请求模型。

## 带中文注释的手写版

````rust
//! Diesel Async + PostgreSQL 示例。
//!
//! ```sh
//! export DATABASE_URL=postgres://localhost/your_db
//! cargo run -p example-diesel-async-postgres
//! ```

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

// 嵌入 migrations 目录。
pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!();

// 定义 users 表结构。
table! {
    users (id) {
        id -> Integer,
        name -> Text,
        hair_color -> Nullable<Text>,
    }
}

// 查询结果和响应 JSON 模型。
#[derive(serde::Serialize, HasQuery)]
struct User {
    id: i32,
    name: String,
    hair_color: Option<String>,
}

// 创建用户请求模型。
#[derive(serde::Deserialize, Insertable)]
#[diesel(table_name = users)]
struct NewUser {
    name: String,
    hair_color: Option<String>,
}

// 异步 Diesel 连接池类型。
type Pool = bb8::Pool<AsyncPgConnection>;

#[tokio::main]
async fn main() {
    // 初始化日志。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 读取数据库连接地址。
    let db_url = std::env::var("DATABASE_URL").unwrap();

    // 创建异步 Diesel 连接管理器和 bb8 连接池。
    let config = AsyncDieselConnectionManager::<diesel_async::AsyncPgConnection>::new(db_url);
    let pool = bb8::Pool::builder().build(config).await.unwrap();

    // 启动时运行还没执行过的 migration。
    let mut harness = AsyncMigrationHarness::new(pool.get_owned().await.unwrap());
    harness.run_pending_migrations(MIGRATIONS).unwrap();

    // 注册路由，并把 pool 放进 state。
    let app = Router::new()
        .route("/user/list", get(list_users))
        .route("/user/create", post(create_user))
        .with_state(pool);

    // 启动服务。
    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {addr}");
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

// 创建用户：直接从 State 中拿连接池。
async fn create_user(
    State(pool): State<Pool>,
    Json(new_user): Json<NewUser>,
) -> Result<Json<User>, (StatusCode, String)> {
    // 从异步连接池借连接。
    let mut conn = pool.get().await.map_err(internal_error)?;

    // 异步执行 Diesel insert。
    let res = diesel::insert_into(users::table)
        .values(new_user)
        .returning(User::as_returning())
        .get_result(&mut conn)
        .await
        .map_err(internal_error)?;

    Ok(Json(res))
}

// 自定义 extractor：从 pool 中借出 owned async connection。
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

// 查询用户列表：使用自定义 DatabaseConnection extractor。
async fn list_users(
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<Json<Vec<User>>, (StatusCode, String)> {
    let res = User::query()
        .load(&mut conn)
        .await
        .map_err(internal_error)?;

    Ok(Json(res))
}

// 把内部错误转成 HTTP 500。
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
````

迁移文件：

````sql
CREATE TABLE "users"(
    "id" SERIAL PRIMARY KEY,
    "name" TEXT NOT NULL,
    "hair_color" TEXT
);
````

## 运行和验证

先准备 PostgreSQL，并设置连接地址：

````bash
export DATABASE_URL='postgres://localhost/your_db'
````

运行：

````bash
cargo run -p example-diesel-async-postgres
````

创建用户：

````bash
curl -X POST http://127.0.0.1:3000/user/create \
  -H 'content-type: application/json' \
  -d '{"name":"Alice","hair_color":"black"}'
````

查询用户列表：

````bash
curl http://127.0.0.1:3000/user/list
````

如果启动失败，优先检查：

```text
DATABASE_URL 是否设置
PostgreSQL 是否启动
数据库是否存在
当前用户是否有建表权限
```

## 常见卡点

### 1. 为什么这一章不用 interact？

因为 `diesel-async` 提供异步查询方法。  
查询可以直接：

````rust
.await
````

上一章同步 Diesel 才需要 `interact`。

### 2. 为什么还需要 diesel crate？

`diesel-async` 负责异步执行。  
表结构、derive、查询 DSL 仍然来自 Diesel。

### 3. 为什么要导入 RunQueryDsl？

异步的 `.get_result(...).await` 和 `.load(...).await` 来自 `RunQueryDsl`。  
忘记导入时，编译器可能找不到这些方法。

### 4. create_user 和 list_users 为什么获取连接方式不同？

这是为了演示两种写法：

```text
create_user -> 直接 State<Pool>
list_users  -> 自定义 DatabaseConnection extractor
```

真实项目可以选择一种风格统一使用。

### 5. Diesel Async 是否一定比同步 Diesel 更好？

不一定。  
它更适合全异步栈，但同步 Diesel + `interact` 也可以工作。

选择时要考虑团队熟悉度、生态、项目已有代码和部署环境。

## 手写任务

建议按下面顺序自己敲一遍：

1. 复用上一章的 migration 和 `table!`。
2. 定义 `User` 和 `NewUser`。
3. 定义 `type Pool = bb8::Pool<AsyncPgConnection>`。
4. 用 `AsyncDieselConnectionManager` 创建连接池。
5. 运行 pending migrations。
6. 写 `create_user`，直接用 `State<Pool>`。
7. 实现 `DatabaseConnection` extractor。
8. 写 `list_users`，使用自定义 extractor。

加深练习：

1. 把 `list_users` 改成也使用 `State<Pool>`，对比代码。
2. 新增 `/user/{id}`，查询单个用户。
3. 给错误响应改成统一 JSON 格式。

## 本章真正要记住什么

Diesel Async 版和同步 Diesel 版的核心业务结构一样：

```text
migration
schema
model
pool
handler
Json response
```

关键差异是执行查询的方式：

```text
同步 Diesel：conn.interact(|conn| query.get_result(conn)).await
Diesel Async：query.get_result(&mut conn).await
```

理解这个差异后，你就能看懂两种 Diesel 接入 Axum 的常见写法。

## 源码对照

本章手写版对应源码：

- `examples/diesel-async-postgres/src/main.rs`
- `examples/diesel-async-postgres/migrations/2023-03-14-180127_add_users/up.sql`
- `examples/diesel-async-postgres/migrations/2023-03-14-180127_add_users/down.sql`
- `examples/diesel-async-postgres/Cargo.toml`
