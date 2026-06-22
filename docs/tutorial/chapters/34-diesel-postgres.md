# 34. diesel-postgres

对应示例：`examples/diesel-postgres`

前两章用异步 PostgreSQL 客户端,这章用 Diesel。Diesel 的查询 API 是**同步阻塞风格**,所以用 `deadpool-diesel` 把它接入 axum 异步服务——核心是通过 `conn.interact(...)` 把同步查询扔到阻塞线程池执行。用 Diesel + PostgreSQL 实现用户创建与列表查询,理解 migration、schema、model、连接池、`interact` 的关系。



相比前面章节新引入：**`embed_migrations!`、`table!`、`deadpool-diesel`、`conn.interact`（本质是 `spawn_blocking`）**。

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

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## migrations/.../up.sql

````sql
CREATE TABLE "users"(
    "id" SERIAL PRIMARY KEY,
    "name" TEXT NOT NULL,
    "hair_color" TEXT
);
````

`down.sql` 是 `DROP TABLE "users";`,用于回滚。

## 关键概念

> **新面孔：`deadpool-diesel` + `conn.interact`**
>
> Diesel 是同步 driver，`interact` 把闭包扔到阻塞线程池（本质 `spawn_blocking`），不卡 async runtime。返回两层 Result。


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

pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!("migrations/");

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

    let manager = deadpool_diesel::postgres::Manager::new(db_url, deadpool_diesel::Runtime::Tokio1);
    let pool = deadpool_diesel::postgres::Pool::builder(manager)
        .build()
        .unwrap();

    {
        let conn = pool.get().await.unwrap();
        conn.interact(|conn| conn.run_pending_migrations(MIGRATIONS).map(|_| ()))
            .await
            .unwrap()
            .unwrap();
    }

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

fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
````

## 运行

设置数据库连接(必须有):

````bash
export DATABASE_URL='postgres://postgres:password@localhost/mydb'
cd examples
cargo run -p example-diesel-postgres
````

创建用户:

````bash
curl -X POST http://127.0.0.1:3000/user/create \
  -H 'content-type: application/json' \
  -d '{"name":"Alice","hair_color":"black"}'
# 预期: {"id":1,"name":"Alice","hair_color":"black"}
````

查询列表:

````bash
curl http://127.0.0.1:3000/user/list
# 预期: [{"id":1,"name":"Alice","hair_color":"black"}]
````

## 解读

### Diesel vs SQLx

```text
SQLx:  写 SQL 字符串,async API 执行
Diesel: Rust 类型描述表结构,Rust DSL 构造查询,编译器参与检查更多类型关系
```

创建用户不写 `insert into users ...`,而是写 Rust DSL:

````rust
diesel::insert_into(users::table)
    .values(new_user)
    .returning(User::as_returning())
    .get_result(conn)
````

### `interact` 的本质:包装 `spawn_blocking`(本章核心)

Diesel 是**同步 driver**——查询函数(如 `User::query().load(conn)`)会**阻塞当前线程**直到数据库返回。在 `async fn` 里直接调用会卡住整个 Tokio worker 线程,这个 worker 上其他所有请求都被阻塞。Tokio 默认只有几个 worker 线程(通常等于 CPU 核心数),一个被卡住就意味着并发能力下降。

Rust 异步生态的标准解法是 **`tokio::task::spawn_blocking`**:把阻塞任务扔到专门的**阻塞线程池**(和异步 worker 线程池分开),让异步线程继续处理其他请求。

`conn.interact(|conn| ...)` 内部做的就是这件事:

```text
conn.interact(closure)
  → 把 closure 扔到阻塞线程池(类似 spawn_blocking)
  → 在阻塞线程里从 pool 取连接、执行 closure(Diesel 同步查询在这里跑)
  → 查询完成后,结果通过 channel 传回 async 线程
  → .interact(...).await 拿到结果
```

所以 Diesel 同步查询实际上是在**另一个线程**跑的,不卡 async runtime。这也是 `interact` 返回的 Future 能安全 `.await` 的原因——它等的是"阻塞线程把活干完"的通知,不是自己阻塞。

理解了这个本质,你就明白 diesel-async(第 35 章)的价值:查询本身变异步,不需要 `interact` 包装,代码更直接,少一次线程间切换。

> 新手记住:在 `async fn` 里调用任何会阻塞的同步代码(文件 I/O、CPU 密集计算、同步 DB driver),都要用 `spawn_blocking` 包一层。`interact` 是 deadpool-diesel 帮你自动做的封装。

### migration 嵌入二进制

````rust
pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!("migrations/");
````

`embed_migrations!` 把 `migrations/` 目录嵌入应用二进制(路径相对 `CARGO_MANIFEST_DIR`)。启动时直接运行 migration,不需在运行目录找 SQL 文件。`run_pending_migrations` 只执行还没运行过的 migration,不会重复建表。

### schema + model

`table!` 告诉 Diesel 表结构(主键 + 列类型),通常通过 `diesel-cli` 生成 `schema.rs`,本 example 集中写在 main.rs。类型对应:`Integer → i32`、`Text → String`、`Nullable<Text> → Option<String>`。

两个 model 分工:

- `User`(查询结果 + 响应):`Serialize` 转 JSON,`HasQuery` 提供 `User::query()`/`User::as_returning()`。
- `NewUser`(创建请求):`Deserialize` 从请求 JSON 解析,`Insertable` + `#[diesel(table_name = users)]` 让 Diesel 知道能插入 users 表。没有 `id` 字段(数据库自动生成)。

### 创建用户 + 两层 map_err

````rust
let res = conn
    .interact(|conn| {
        diesel::insert_into(users::table).values(new_user).returning(User::as_returning()).get_result(conn)
    })
    .await
    .map_err(internal_error)?      // 第一层:interact 任务本身失败
    .map_err(internal_error)?;     // 第二层:闭包里的 Diesel 查询失败
````

`interact` 有两层可能失败,所以要处理两层 `Result`。`returning(User::as_returning())` 让数据库返回插入后的 User(含生成的 id),很实用。

### 查询用户列表

````rust
let res = conn.interact(|conn| User::query().load(conn)).await...?;
Ok(Json(res))   // Json<Vec<User>>
````

`User::query().load(conn)` 查询用户并加载成 `Vec<User>`。

### 必须设置 DATABASE_URL

````rust
let db_url = std::env::var("DATABASE_URL").unwrap();
````

没有默认值,不设环境变量直接 panic。数据库示例里常见,不同人本地配置差异大。

## 常见问题

**为什么必须设 DATABASE_URL?** 源码 `unwrap()` 没默认值,不设会 panic。

**为什么 Diesel 查询放 `interact` 里?** Diesel 同步查询阻塞线程,axum 在 Tokio async runtime 不能直接在 async handler 长时间阻塞。`interact` 把阻塞查询扔到阻塞线程池(类似 `spawn_blocking`)。

**创建用户为什么有两个 `map_err`?** `interact` 外层可能失败(任务本身),闭包里 Diesel 查询也可能失败,两层 `Result`。

**migration 每次重复建表?** 不会,`run_pending_migrations` 只执行未运行过的。

**`hair_color` 为什么是 `Option<String>`?** 数据库字段 `TEXT` 允许为空,Rust 用 `Option<String>` 表示可能为空。

## 手写任务

按下面顺序敲:

1. 写 migration 创建 `users` 表。
2. 写 `table!` schema。
3. 定义 `User` 和 `NewUser`。
4. 读 `DATABASE_URL`。
5. 创建 `deadpool_diesel::postgres::Pool`。
6. 启动时运行 `run_pending_migrations`。
7. 写 `/user/create` 插入用户并返回。
8. 写 `/user/list` 查询所有用户。

加深练习:

1. 给 `users` 加 `created_at` 字段,写新 migration。
2. 新增 `/user/{id}` 查询单个用户。
3. 错误响应改成统一 JSON 格式。

## 小结

- Diesel 版 axum 数据库接口核心模型:migration 定义结构 → `table!` schema → model 表示请求/响应 → deadpool-diesel 管连接池 → `interact` 执行阻塞查询 → handler 返回 Json。
- Diesel 查询是同步阻塞风格,在 async handler 里要通过 `conn.interact(...)` 执行,本质是包装 `spawn_blocking` 把查询扔到阻塞线程池。
- `interact` 返回两层 `Result`(任务本身 + 闭包查询),要两个 `map_err`。
- `embed_migrations!` 把 migration 嵌入二进制,启动时 `run_pending_migrations` 自动建表。
- Diesel DSL 用 Rust 类型构造查询(`insert_into(...).values(...).returning(...)`),编译器参与检查;`returning` 让数据库返回插入后的行(含自动生成的 id)。
- 新项目可看第 35 章 diesel-async,查询本身异步不需 `interact` 包装,代码更直接。

## 源码对照

- `examples/diesel-postgres/Cargo.toml`
- `examples/diesel-postgres/src/main.rs`
- `examples/diesel-postgres/migrations/2023-03-14-180127_add_users/up.sql`
- `examples/diesel-postgres/migrations/2023-03-14-180127_add_users/down.sql`
