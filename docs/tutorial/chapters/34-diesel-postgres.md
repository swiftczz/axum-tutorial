# 34. diesel-postgres

对应示例：`examples/diesel-postgres`

本章目标：使用 Diesel 和 PostgreSQL 实现用户创建与列表查询，理解 migration、schema、model、连接池和 `interact` 的关系。

前两章使用的是异步 PostgreSQL 客户端。  
这一章使用 Diesel。Diesel 的查询 API 是同步阻塞风格，所以示例用 `deadpool-diesel` 把它接入 Axum 的异步服务。

## 这个小项目在做什么

应用提供两个接口：

```text
GET  /user/list   -> 查询所有用户
POST /user/create -> 创建用户
```

用户表结构：

```text
id
name
hair_color
```

启动时会自动运行 migration，创建 `users` 表。

请求主线是：

```text
程序启动
-> 读取 DATABASE_URL
-> 创建 deadpool-diesel 连接池
-> 运行 embedded migrations
-> Router 保存 pool
-> handler 从 State 取 pool
-> pool.get() 借连接
-> conn.interact(...) 执行 Diesel 查询
-> 返回 Json<User> 或 Json<Vec<User>>
```

## 先理解 Diesel 和 SQLx 的区别

SQLx 更接近：

```text
写 SQL 字符串
用 async API 执行
```

Diesel 更接近：

```text
用 Rust 类型描述表结构
用 Rust DSL 构造查询
编译器参与检查更多类型关系
```

例如本章创建用户不是直接写：

````sql
insert into users ...
````

而是写：

````rust
diesel::insert_into(users::table)
    .values(new_user)
    .returning(User::as_returning())
    .get_result(conn)
````

这就是 Diesel 的风格。

## 文件和依赖

这个 example 有四个主要文件：

1. `examples/diesel-postgres/Cargo.toml`：声明 Axum、Diesel、deadpool-diesel、migration、serde。
2. `examples/diesel-postgres/src/main.rs`：实现 schema、模型、连接池、migration 和路由。
3. `examples/diesel-postgres/migrations/.../up.sql`：创建 `users` 表。
4. `examples/diesel-postgres/migrations/.../down.sql`：删除 `users` 表。

关键依赖：

- `diesel`：ORM 和查询 DSL。
- `diesel_migrations`：嵌入并运行 migration。
- `deadpool-diesel`：为阻塞式 Diesel 连接提供连接池和 `interact`。
- `axum`：提供 Router、State、Json。
- `serde`：反序列化请求 JSON，序列化响应 JSON。
- `tokio`：异步运行时。

`Cargo.toml` 关键配置：

````toml
axum = { path = "../../axum", features = ["macros"] }
deadpool-diesel = { version = "0.6.1", features = ["postgres"] }
diesel = { version = "2", features = ["postgres"] }
diesel_migrations = "2"
serde = { version = "1.0", features = ["derive"] }
````

## 第一步：migration 创建 users 表

`up.sql`：

````sql
CREATE TABLE "users"(
    "id" SERIAL PRIMARY KEY,
    "name" TEXT NOT NULL,
    "hair_color" TEXT
);
````

这表示创建一张表：

- `id`：自增主键。
- `name`：不能为空的文本。
- `hair_color`：可以为空的文本。

`down.sql`：

````sql
DROP TABLE "users";
````

`up.sql` 用来升级数据库结构。  
`down.sql` 用来回滚这次升级。

## 第二步：把 migration 嵌入二进制

源码：

````rust
pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!("migrations/");
````

`embed_migrations!` 会把 `migrations/` 目录嵌入应用二进制。

路径相对：

```text
CARGO_MANIFEST_DIR
```

也就是这个 example 的 crate 根目录。

这样程序启动时可以直接运行 migration，不需要额外在运行目录里找 SQL 文件。

## 第三步：定义 schema

源码：

````rust
table! {
    users (id) {
        id -> Integer,
        name -> Text,
        hair_color -> Nullable<Text>,
    }
}
````

这段告诉 Diesel：

```text
数据库里有一张 users 表
主键是 id
每一列是什么类型
```

通常 Diesel 项目会通过命令生成 `schema.rs`。  
这个 example 为了把代码集中在一个文件里，直接把 schema 写在 `main.rs`。

数据库类型和 Rust 类型大概对应：

```text
Integer        -> i32
Text           -> String
Nullable<Text> -> Option<String>
```

## 第四步：定义返回给客户端的 User

源码：

````rust
#[derive(serde::Serialize, HasQuery)]
struct User {
    id: i32,
    name: String,
    hair_color: Option<String>,
}
````

`User` 表示数据库里查出来的一行用户。

`serde::Serialize` 让它可以变成 JSON 响应：

````rust
Json<User>
Json<Vec<User>>
````

`HasQuery` 是 Diesel 的查询辅助 derive。  
源码后面会用：

````rust
User::query()
User::as_returning()
````

来加载用户和返回插入后的用户。

## 第五步：定义创建用户的 NewUser

源码：

````rust
#[derive(serde::Deserialize, Insertable)]
#[diesel(table_name = users)]
struct NewUser {
    name: String,
    hair_color: Option<String>,
}
````

`NewUser` 表示客户端提交的创建用户请求。

它没有 `id` 字段，因为 `id` 由数据库自动生成。

`serde::Deserialize` 让 Axum 可以从请求 JSON 解析它：

````rust
Json(new_user): Json<NewUser>
````

`Insertable` 让 Diesel 知道它可以插入到 `users` 表：

````rust
#[diesel(table_name = users)]
````

## 第六步：读取 DATABASE_URL

源码：

````rust
let db_url = std::env::var("DATABASE_URL").unwrap();
````

这个 example 没有默认数据库地址。  
必须设置：

```text
DATABASE_URL
```

例如：

```text
postgres://postgres:password@localhost/mydb
```

如果没有设置，程序会直接 panic。  
这在数据库示例里很常见，因为不同人本地数据库配置差异很大。

## 第七步：创建 deadpool-diesel 连接池

源码：

````rust
let manager = deadpool_diesel::postgres::Manager::new(db_url, deadpool_diesel::Runtime::Tokio1);
let pool = deadpool_diesel::postgres::Pool::builder(manager)
    .build()
    .unwrap();
````

`Manager` 负责创建 PostgreSQL 连接。  
`Pool::builder(manager).build()` 创建连接池。

这里指定：

````rust
deadpool_diesel::Runtime::Tokio1
````

表示连接池运行在 Tokio 运行时中。

因为 Diesel 查询是阻塞式的，所以后面真正执行查询时会用：

````rust
conn.interact(...)
````

避免直接阻塞 async runtime。

### `interact` 的本质：包装 `spawn_blocking`

这里要讲清楚 `interact` 到底做了什么，否则你会觉得它是个魔法。

Diesel 是**同步 driver**——它的查询函数（如 `User::query().load(conn)`）会**阻塞当前线程**直到数据库返回。如果在 `async fn` 里直接调用，会卡住整个 Tokio worker 线程，导致这个 worker 上其他所有请求都被阻塞。Tokio 默认只有几个 worker 线程（通常等于 CPU 核心数），一个被卡住就意味着并发能力下降。

Rust 异步生态解决这个问题的标准做法是 **`tokio::task::spawn_blocking`**：把阻塞任务扔到一个专门的**阻塞线程池**（和异步 worker 线程池分开），让异步线程继续处理其他请求。

`conn.interact(|conn| ...)` 内部做的就是这件事：

```text
conn.interact(closure)
  ↓
把 closure 扔到阻塞线程池（类似 spawn_blocking）
  ↓
在阻塞线程里从 pool 取连接、执行 closure（Diesel 同步查询在这里跑）
  ↓
查询完成后，结果通过 channel 传回 async 线程
  ↓
.interact(...).await 拿到结果
```

所以 Diesel 同步查询实际上是在**另一个线程**上跑的，不会卡住 async runtime。这也是为什么 `interact` 返回的 `Future` 可以安全 `.await`——它等的是"阻塞线程把活干完"的通知，而不是自己阻塞。

理解了这个本质，你就能明白 diesel-async（第 35 章）的价值：它让查询本身变成异步的，不需要 `interact` 这层包装，代码更直接，也少了一次线程间切换。

> 新手记住：在 `async fn` 里调用任何会阻塞的同步代码（文件 I/O、CPU 密集计算、同步 DB driver），都要用 `spawn_blocking` 包一层。`interact` 是 deadpool-diesel 帮你自动做的封装。

## 第八步：启动时运行 migration

源码：

````rust
{
    let conn = pool.get().await.unwrap();
    conn.interact(|conn| conn.run_pending_migrations(MIGRATIONS).map(|_| ()))
        .await
        .unwrap()
        .unwrap();
}
````

这段在服务启动时运行未执行过的 migration。

流程是：

1. `pool.get().await` 从连接池借一个连接。
2. `conn.interact(...)` 在连接上执行阻塞的 Diesel 操作。
3. `run_pending_migrations(MIGRATIONS)` 执行还没运行过的 migration。

外层花括号让 `conn` 尽快离开作用域，还回连接池。

## 第九步：注册路由

源码：

````rust
let app = Router::new()
    .route("/user/list", get(list_users))
    .route("/user/create", post(create_user))
    .with_state(pool);
````

这里有两个接口：

```text
GET  /user/list
POST /user/create
```

并把连接池放进 Axum state。  
后面的 handler 都通过：

````rust
State(pool): State<deadpool_diesel::postgres::Pool>
````

拿到连接池。

## 第十步：创建用户

源码：

````rust
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
````

handler 参数有两个 extractor：

```text
State(pool)      -> 数据库连接池
Json(new_user)   -> 请求体 JSON
```

核心 Diesel 操作：

````rust
diesel::insert_into(users::table)
    .values(new_user)
    .returning(User::as_returning())
    .get_result(conn)
````

意思是：

```text
插入一行 users
插入的数据来自 new_user
让数据库返回插入后的 User
执行并得到结果
```

返回插入后的用户很实用，因为数据库会生成 `id`。

## 第十一步：为什么有两个 map_err

创建用户里有：

````rust
.await
.map_err(internal_error)?
.map_err(internal_error)?;
````

这是因为 `interact` 有两层可能失败：

1. `interact` 任务本身失败。
2. 闭包里的 Diesel 查询失败。

所以需要处理两层 `Result`。

可以理解成：

```text
await interact 是否成功
再看 Diesel insert 是否成功
```

## 第十二步：查询用户列表

源码：

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
````

这段返回：

````rust
Json<Vec<User>>
````

核心查询：

````rust
User::query().load(conn)
````

表示查询用户，并加载成 `Vec<User>`。

如果表里有三条用户记录，响应就是 JSON 数组。

## 第十三步：错误转换

源码：

````rust
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
````

这里统一处理：

- 连接池获取失败。
- `interact` 执行失败。
- Diesel 查询失败。

真实项目里，不建议把数据库错误原文直接返回给用户。  
更常见做法是：

```text
详细错误写日志
HTTP 响应只返回通用错误
```

## 函数职责速查

- `main`：初始化日志，读取 `DATABASE_URL`，创建连接池，运行 migration，注册路由并启动服务。
- `create_user`：解析请求 JSON，插入用户，返回插入后的用户。
- `list_users`：查询所有用户，返回用户数组。
- `internal_error`：把内部错误转成 HTTP 500。
- `MIGRATIONS`：把 migration SQL 嵌入应用。
- `User`：数据库查询结果和响应 JSON 模型。
- `NewUser`：创建用户请求模型。

## 带中文注释的手写版

````rust
//! Diesel + PostgreSQL 示例。
//!
//! ```not_rust
//! cargo run -p example-diesel-postgres
//! ```

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

// 把 migrations/ 目录嵌入应用二进制。
pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!("migrations/");

// 定义 users 表结构。
// 正常 Diesel 项目里，这通常来自生成的 schema.rs。
table! {
    users (id) {
        id -> Integer,
        name -> Text,
        hair_color -> Nullable<Text>,
    }
}

// 查询出来的用户，也是返回给客户端的 JSON 模型。
#[derive(serde::Serialize, HasQuery)]
struct User {
    id: i32,
    name: String,
    hair_color: Option<String>,
}

// 创建用户时客户端提交的数据。
// 不需要 id，因为 id 由数据库自动生成。
#[derive(serde::Deserialize, Insertable)]
#[diesel(table_name = users)]
struct NewUser {
    name: String,
    hair_color: Option<String>,
}

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

    // 读取数据库连接地址。这个 example 要求必须设置 DATABASE_URL。
    let db_url = std::env::var("DATABASE_URL").unwrap();

    // 创建 deadpool-diesel 连接管理器和连接池。
    let manager = deadpool_diesel::postgres::Manager::new(db_url, deadpool_diesel::Runtime::Tokio1);
    let pool = deadpool_diesel::postgres::Pool::builder(manager)
        .build()
        .unwrap();

    // 服务启动时运行还没执行过的 migration。
    {
        let conn = pool.get().await.unwrap();
        conn.interact(|conn| conn.run_pending_migrations(MIGRATIONS).map(|_| ()))
            .await
            .unwrap()
            .unwrap();
    }

    // 注册路由，并把连接池放进 state。
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

// 创建用户。
async fn create_user(
    State(pool): State<deadpool_diesel::postgres::Pool>,
    Json(new_user): Json<NewUser>,
) -> Result<Json<User>, (StatusCode, String)> {
    // 从连接池借连接。
    let conn = pool.get().await.map_err(internal_error)?;

    // interact 中执行阻塞式 Diesel 查询。
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

// 查询用户列表。
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

先准备 PostgreSQL，并设置数据库连接地址：

````bash
export DATABASE_URL='postgres://postgres:password@localhost/mydb'
````

运行：

````bash
cargo run -p example-diesel-postgres
````

创建用户：

````bash
curl -X POST http://127.0.0.1:3000/user/create \
  -H 'content-type: application/json' \
  -d '{"name":"Alice","hair_color":"black"}'
````

预期返回类似：

````json
{"id":1,"name":"Alice","hair_color":"black"}
````

查询用户列表：

````bash
curl http://127.0.0.1:3000/user/list
````

预期返回 JSON 数组：

````json
[{"id":1,"name":"Alice","hair_color":"black"}]
````

## 常见卡点

### 1. 为什么必须设置 DATABASE_URL？

源码直接调用了：

````rust
std::env::var("DATABASE_URL").unwrap()
````

没有默认值。  
所以不设置环境变量程序会 panic。

### 2. 为什么 Diesel 查询要放在 interact 里？

Diesel 的同步查询会阻塞线程。  
Axum 运行在 Tokio 异步运行时里，不能直接在 async handler 中长时间阻塞。

`deadpool-diesel` 的 `interact` 用来安全执行这些阻塞数据库操作。

### 3. 为什么创建用户有两个 map_err？

因为 `interact` 外层可能失败，闭包里的 Diesel 查询也可能失败。  
所以要处理两层 `Result`。

### 4. migration 会每次重复创建表吗？

不会。  
`run_pending_migrations` 只执行还没有运行过的 migration。

### 5. hair_color 为什么是 Option<String>？

数据库字段是：

````sql
hair_color TEXT
````

它允许为空。  
Rust 里用 `Option<String>` 表示可能为空。

## 手写任务

建议按下面顺序自己敲一遍：

1. 写 migration，创建 `users` 表。
2. 写 `table!` schema。
3. 定义 `User` 和 `NewUser`。
4. 读取 `DATABASE_URL`。
5. 创建 `deadpool_diesel::postgres::Pool`。
6. 启动时运行 `run_pending_migrations`。
7. 写 `/user/create`，插入用户并返回。
8. 写 `/user/list`，查询所有用户。

加深练习：

1. 给 `users` 增加 `created_at` 字段，并写新的 migration。
2. 新增 `/user/{id}` 查询单个用户。
3. 把错误响应改成统一 JSON 格式。

## 本章真正要记住什么

Diesel 版 Axum 数据库接口的核心模型是：

```text
migration 定义数据库结构
schema/table! 告诉 Diesel 表结构
model 表示请求和响应数据
deadpool-diesel 管理连接池
interact 执行阻塞式 Diesel 查询
handler 返回 Json
```

和 SQLx 最大的不同是：

```text
Diesel 查询是同步阻塞风格
在 Axum async handler 里要通过 interact 执行
```

理解这点，后面再看 Diesel Async 版本会更清楚。

## 源码对照

本章手写版对应源码：

- `examples/diesel-postgres/src/main.rs`
- `examples/diesel-postgres/migrations/2023-03-14-180127_add_users/up.sql`
- `examples/diesel-postgres/migrations/2023-03-14-180127_add_users/down.sql`
- `examples/diesel-postgres/Cargo.toml`
