# 32. sqlx-postgres

对应示例：`examples/sqlx-postgres`

进入数据库章节（32-37）。本章用 SQLx 连接 PostgreSQL，理解连接池 `PgPool`、axum `State`、自定义数据库连接 extractor。核心心智：Web 请求 → handler → 从连接池拿数据库连接 → 执行 SQL → 把结果变成 HTTP 响应。

本章分 4 步搭：先建连接池 → 用 `State<PgPool>` 直接查 → 写自定义 extractor → 用自定义 extractor 处理 POST。

## 先理解：5 个数据库方案怎么选

32-37 章讲 5 种数据库接入方式，动手前先建立全局印象：

| 方案 | 章节 | 异步/同步 | 连接池 | 编译期 SQL 校验 | ORM 程度 |
| --- | --- | --- | --- | --- | --- |
| **SQLx** | 32 | 原生异步 | 自带 `PgPool` | 有（`query!` 宏，需连数据库编译） | 半 ORM（写 SQL，自动映射） |
| **tokio-postgres + bb8** | 33 | 原生异步 | bb8 通用池 | 无 | 最底层（裸 SQL + 手动取列） |
| **Diesel（同步）+ deadpool** | 34 | 同步（`interact` 包阻塞线程） | deadpool-diesel | 有（`table!` + 类型系统） | 重 ORM（DSL 写查询） |
| **Diesel Async + bb8** | 35 | 原生异步 | bb8 | 有 | 重 ORM |
| **Redis** | 36 | 原生异步 | bb8 或 `MultiplexedConnection` | 无 | 无（KV 命令式） |
| **MongoDB** | 37 | 原生异步 | `Client` 自带内部池 | 无 | 文档型（BSON） |

**怎么选：**

- 想省心、写 SQL 自动映射 → **SQLx**（32 章），编译期校验 SQL 是杀手锏。
- 想完全控制、不要 ORM/宏 → **tokio-postgres**（33 章），最轻量但手动取列管池。
- 想强类型 DSL、不怕重 → **Diesel**（34 同步/35 异步），类型安全最强但学习曲线陡。
- 同步还是异步？新项目优先 35 章 diesel-async；必须用同步 API 时用 34 章。
- 缓存/计数器/排行榜 → **Redis**（36 章）。
- 文档型数据/灵活 schema → **MongoDB**（37 章）。

**关键概念：同步 driver 在异步程序里怎么办？** Diesel（34 章）是同步 driver，查询会阻塞当前线程，直接在 `async fn` 里调用会卡住整个 Tokio worker。所以用 `pool.interact(|conn| ...)` 把同步查询扔到专门的阻塞线程池（底层 `tokio::task::spawn_blocking`）。这是同步 driver 接入异步 runtime 的标准做法。而 SQLx（32）/tokio-postgres（33）/diesel-async（35）原生异步，可直接 `.await`。

## Cargo.toml

````toml
[package]
name = "example-sqlx-postgres"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

sqlx = { version = "0.9", features = ["runtime-tokio", "any", "postgres"] }
````

- `runtime-tokio`：SQLx 用 Tokio 异步运行时。
- `postgres`：启用 PostgreSQL 支持。
- `any`：启用通用数据库抽象，本例代码实际没用（直接用 `PgPool`），example 沿用配置，你自己用 Postgres 可以不加。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：创建 PgPool 连接池

这步解决一个最基本的问题：Web 应用要和数据库通信，必须先在启动时建立一个到 PostgreSQL 的**连接池**，后面所有 handler 复用它。

为什么需要连接池？不能每来一个请求都重新建数据库连接（TCP 连接 + 认证 + 初始化成本高，高并发下会压垮数据库）。连接池的做法：启动时创建一组连接，请求来了借一个、用完还回去。`PgPool` 就是 PostgreSQL 的连接池。

````rust
use axum::{routing::get, Router};
use sqlx::postgres::{PgPool, PgPoolOptions};
use tokio::net::TcpListener;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

use std::time::Duration;

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let db_connection_str = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://postgres:password@localhost".to_string());

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .acquire_timeout(Duration::from_secs(3))
        .connect(&db_connection_str)
        .await
        .expect("can't connect to database");

    let app = Router::new()
        .route("/", get(|| async { "pool ready" }))
        .with_state(pool);

    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
````

逐行看池的创建：

````rust
let pool = PgPoolOptions::new()
    .max_connections(5)                        // 池里最多 5 个连接
    .acquire_timeout(Duration::from_secs(3))   // 借不到连接最多等 3 秒
    .connect(&db_connection_str).await         // 真正连数据库
    .expect("can't connect to database");      // 连不上启动就失败
````

数据库连不上启动就失败（`.expect`）——服务没数据库无法正常工作。`DATABASE_URL` 从环境变量读，代码不用写死不同环境的地址（默认 `postgres://postgres:password@localhost`）。

> **新面孔：`PgPool` / `PgPoolOptions`**
>
> `PgPoolOptions` 是连接池的构造器（builder），链式配置参数后 `.connect().await` 拿到一个 `PgPool`。
>
> `PgPool` 是**连接池**，不是单个连接。它内部管理一组真实的 PostgreSQL 连接。handler 把 `&pool` 传给 SQLx 的查询方法，SQLx 会自动从池里借一个连接执行 SQL、用完归还。你不用手动管借还。

> **新面孔：`max_connections` / `acquire_timeout`**
>
> `max_connections`：池最多持有多少个连接。设太多会压垮数据库（Postgres 默认最多 ~100 个连接），设太少高并发时请求排队等连接。
>
> `acquire_timeout`：所有连接都被借出时，新请求最多等多久。超时就报错（而不是无限阻塞），让客户端拿到明确错误。

这步代码能编译通过。运行需要本机有 PostgreSQL：

````bash
export DATABASE_URL='postgres://postgres:password@localhost'
cd examples
cargo run -p example-sqlx-postgres
````

启动后 `curl http://127.0.0.1:3000/` 返回 `pool ready`（handler 还没碰数据库，只是确认 pool 建好了、router 起来了）。

---

## 第二步：State\<PgPool\> 直接查询（GET handler）

池建好了，下一步把它用起来：写一个 GET handler，从 state 拿出 `PgPool`，执行一条 SQL，把结果变成 HTTP 响应。

axum 的 `State<T>` 提取器在第 16/17/18 章都用过（从 Router state 里取一个值）。这里 state 类型是 `PgPool`，所以用 `State<PgPool>` 把池取出来。

````rust
use axum::{
    extract::State,
    http::StatusCode,
    routing::get,
    Router,
};
use sqlx::postgres::{PgPool, PgPoolOptions};
use tokio::net::TcpListener;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

use std::time::Duration;

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let db_connection_str = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://postgres:password@localhost".to_string());

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .acquire_timeout(Duration::from_secs(3))
        .connect(&db_connection_str)
        .await
        .expect("can't connect to database");

    let app = Router::new()
        .route("/", get(using_connection_pool_extractor))
        .with_state(pool);

    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn using_connection_pool_extractor(
    State(pool): State<PgPool>,
) -> Result<String, (StatusCode, String)> {
    sqlx::query_scalar("select 'hello world from pg'")
        .fetch_one(&pool)
        .await
        .map_err(internal_error)
}

/// 把任意错误转成 `500 Internal Server Error` 响应。
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
````

handler 干了三件事：参数里 `State(pool): State<PgPool>` 从 state 取出池；`query_scalar` 执行一条 SQL；`fetch_one(&pool)` 从池借连接、拿一行、归还。错误用 `internal_error` 统一转成 500。

> **新面孔：`query_scalar` / `fetch_one`**
>
> `query_scalar(sql)` 构造一个查询，预期结果只有**一列**（scalar 单值），自动把那一列映射成 Rust 类型（这里推断成 `String`）。和它对照的是 `query_as::<RowStruct>`（多列映射成结构体）。
>
> `.fetch_one(&pool)` 是执行器：从池借一个连接、执行 SQL、取回**一行**、归还连接。兄弟方法还有 `fetch_all`（所有行）、`fetch_optional`（可能无行）、`fetch`（返回流）。注意传的是 `&pool`（池的引用），不是单个连接。

> **SQLx 真正的招牌是 `query!` 宏——编译期校验 SQL**
>
> 本章用 `query_scalar("...")` 是**运行期**执行（SQL 字符串运行时才发给 Postgres，列名错/类型不匹配运行时才报错）。但 SQLx 真正的招牌是 **`query!` 宏**——**编译期**校验 SQL：
>
> ````rust
> let row: (String,) = sqlx::query_as("select name from users where id = $1")
>     .bind(user_id)
>     .fetch_one(&pool).await?;
> ````
>
> 编译期真的连数据库执行检查：SQL 语法对不对、表名列名存不存在、返回列类型能不能映射到 Rust 类型。列名写错（如 `select nam`）编译就失败，根本到不了运行时。这是 SQLx 相对 tokio-postgres（33 章）最大的优势。
>
> 用 `query!` 的前提：编译时能连上数据库（通过 `DATABASE_URL`），或用 `cargo sqlx prepare` 生成离线缓存（`.sqlx/` 目录）让 CI/无数据库环境也能编译。本例没用是因为 SQL 是常量 `select 'hello world from pg'` 没列名可校验；真实项目查询带表名列名时**优先用 `query!`/`query_as!`**。

> **新面孔：`internal_error` 模式**
>
> 数据库查询返回 `sqlx::Error`，handler 返回 `Result<String, (StatusCode, String)>`（axum 把 `(StatusCode, String)` 当响应）。`internal_error<E: std::error::Error>` 是个小工具函数，把任意错误转成 `(StatusCode::INTERNAL_SERVER_ERROR, err.to_string())`，`.map_err(internal_error)` 一行搞定。真实项目通常不把数据库错误原文返回用户，会换成自定义错误类型（见第 19/20 章）。

验证 GET：

````bash
curl http://127.0.0.1:3000/
# hello world from pg
````

连不上数据库检查：PostgreSQL 是否启动、`DATABASE_URL` 是否正确、用户名密码、端口可达。

---

## 第三步：自定义 DatabaseConnection extractor

`State<PgPool>` 够用，但当多个 handler 都要"拿连接、查、还连接"，每个 handler 里重复 `&pool` 就有点啰嗦；而且有时你需要**直接拿到一个连接对象**（比如要在同一个连接上开事务）。这步写一个自定义 extractor，把"从 state 取 pool → 借一个连接"这套逻辑封装起来。

在第二步的 `main.rs` 末尾加上下面这段（注意补 `use` 里的 `FromRef`、`FromRequestParts`、`request::Parts`）：

````rust
use axum::{
    extract::{FromRef, FromRequestParts, State},
    http::{request::Parts, StatusCode},
    routing::get,
    Router,
};
// ... 其余 use 不变

// 我们也可以写一个自定义 extractor，从池里抓一个连接出来。
// 哪种写法合适，取决于你的应用。
struct DatabaseConnection(sqlx::pool::PoolConnection<sqlx::Postgres>);

impl<S> FromRequestParts<S> for DatabaseConnection
where
    PgPool: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = (StatusCode, String);

    async fn from_request_parts(_parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let pool = PgPool::from_ref(state);
        let conn = pool.acquire().await.map_err(internal_error)?;
        Ok(Self(conn))
    }
}
````

这段定义了类型 `DatabaseConnection` 和它的 extractor 实现，但**还没有任何 handler 用它**（所以现在编译器会报 `dead_code` 警告——下一步 POST handler 就用上了）。

> **新面孔：`FromRequestParts`**
>
> `FromRequestParts<S>` 是 axum 的 extractor trait：实现它之后，你的类型就能直接出现在 handler 参数里被自动提取。和第 2/6 章用的 `Json<T>` 一样是 extractor——但 `Json` 是 axum 内置的，`DatabaseConnection` 是你**自己写的**。
>
> 注意是 `FromRequestParts` 不是 `FromRequest`：它只读请求的 `Parts`（method、headers、URI 等），**不读 body**。读 body 的 extractor（如 `Json`）会消费请求体；数据库连接不该碰 body，所以用 `Parts` 版本。`from_request_parts` 里做的事：从 state 拿 pool → `pool.acquire().await` 借一个连接 → 包进 `Self`。

> **新面孔：`FromRef`**
>
> `PgPool: FromRef<S>` 这个 trait bound 的意思是"可以从应用 state `S` 里取出 `PgPool`"。`PgPool::from_ref(state)` 就是取出动作。
>
> 本章 state 本身就是 `PgPool`，`PgPool` 实现了 `FromRef<PgPool>`（自己取自己），所以能直接用。好处是以后 state 变复杂（如 `AppState { db: PgPool, config: AppConfig }`），只要给 `AppState` 实现好 `FromRef`，这段 extractor 代码**一行都不用改**。第 18 章的依赖注入也是这套机制。

> **新面孔：`PoolConnection<sqlx::Postgres>`**
>
> `pool.acquire().await` 借出来的是一个真实的**连接对象** `PoolConnection`（不是池）。它实现了 `DerefMut` 到底层连接，所以可以 `&mut *conn` 当普通连接用。`PoolConnection` drop 时会自动把连接归还池（不用手动还）。

---

## 第四步：POST handler 用自定义 extractor

最后把自定义 extractor 用起来：写一个 POST handler，参数直接是 `DatabaseConnection`，和第二步的 `State<PgPool>` 写法放一起，对比两种风格。

继续在 `main.rs` 里加 POST handler，并把路由从只挂 GET 改成 GET + POST：

````rust
async fn using_connection_extractor(
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<String, (StatusCode, String)> {
    sqlx::query_scalar("select 'hello world from pg'")
        .fetch_one(&mut *conn)
        .await
        .map_err(internal_error)
}
````

路由改成：

````rust
let app = Router::new()
    .route(
        "/",
        get(using_connection_pool_extractor).post(using_connection_extractor),
    )
    .with_state(pool);
````

这里 handler 直接拿到一个**连接对象** `conn`（而不是池），所以 `fetch_one(&mut *conn)` 传的是连接的借用。两种写法做的是同一件事（查 `select 'hello world from pg'`），但获取连接的方式不同。

验证两种方法：

````bash
curl http://127.0.0.1:3000/          # GET，走 State<PgPool>
# hello world from pg
curl -X POST http://127.0.0.1:3000/   # POST，走 DatabaseConnection extractor
# hello world from pg
````

**两种写法怎么选：**

| 写法 | 优点 | 适合 |
| --- | --- | --- |
| `State<PgPool>` | 简单直观，少一层抽象 | 大多数普通查询 |
| 自定义 extractor | handler 聚焦业务、连接获取逻辑复用、可扩展（事务/鉴权） | 多 handler 重复连接获取逻辑 |

初学先掌握 `State<PgPool>`，等重复逻辑出现再抽象成 extractor。

---

## 完整代码

````rust
use axum::{
    extract::{FromRef, FromRequestParts, State},
    http::{request::Parts, StatusCode},
    routing::get,
    Router,
};
use sqlx::postgres::{PgPool, PgPoolOptions};
use tokio::net::TcpListener;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

use std::time::Duration;

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let db_connection_str = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://postgres:password@localhost".to_string());

    // set up connection pool
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .acquire_timeout(Duration::from_secs(3))
        .connect(&db_connection_str)
        .await
        .expect("can't connect to database");

    // build our application with some routes
    let app = Router::new()
        .route(
            "/",
            get(using_connection_pool_extractor).post(using_connection_extractor),
        )
        .with_state(pool);

    // run it with hyper
    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

// we can extract the connection pool with `State`
async fn using_connection_pool_extractor(
    State(pool): State<PgPool>,
) -> Result<String, (StatusCode, String)> {
    sqlx::query_scalar("select 'hello world from pg'")
        .fetch_one(&pool)
        .await
        .map_err(internal_error)
}

// we can also write a custom extractor that grabs a connection from the pool
// which setup is appropriate depends on your application
struct DatabaseConnection(sqlx::pool::PoolConnection<sqlx::Postgres>);

impl<S> FromRequestParts<S> for DatabaseConnection
where
    PgPool: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = (StatusCode, String);

    async fn from_request_parts(_parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let pool = PgPool::from_ref(state);

        let conn = pool.acquire().await.map_err(internal_error)?;

        Ok(Self(conn))
    }
}

async fn using_connection_extractor(
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<String, (StatusCode, String)> {
    sqlx::query_scalar("select 'hello world from pg'")
        .fetch_one(&mut *conn)
        .await
        .map_err(internal_error)
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

需要先有 PostgreSQL 服务。设置连接地址（或用默认 `postgres://postgres:password@localhost`）：

````bash
export DATABASE_URL='postgres://postgres:password@localhost'
cd examples
cargo run -p example-sqlx-postgres
````

验证：

````bash
curl http://127.0.0.1:3000/         # GET
curl -X POST http://127.0.0.1:3000/  # POST
````

两个都返回 `hello world from pg`。连接失败检查：PostgreSQL 是否启动、`DATABASE_URL` 是否正确、用户名密码、端口可达。

**没建表也能查询？** 能。SQL 是 `select 'hello world from pg'`，让 Postgres 返回字符串常量，不需要任何表。

**能不能每请求新建 PgPool？** 不要。连接池启动时创建一次放进 state 复用（本章就是这么做的）。

## 手写任务

按下面顺序敲，确认每步理解：

1. 第一步：读 `DATABASE_URL`，`PgPoolOptions::new()` 建池，`.with_state(pool)` 挂到 Router。
2. 第二步：写 GET handler `using_connection_pool_extractor`，用 `State(pool): State<PgPool>`，`query_scalar` + `fetch_one`，配 `internal_error`。
3. 第三步：定义 `DatabaseConnection` 结构体 + 实现 `FromRequestParts`（用 `PgPool::from_ref` + `pool.acquire`）。
4. 第四步：写 POST handler 用自定义 extractor，路由 `.post(...)` 挂上去。

加深练习：

1. SQL 改成 `select now()` 返回当前数据库时间。
2. 新建 `users` 表，查询一行用户（体会为什么真实项目该用 `query!`/`query_as!`）。
3. 把 state 改成 `AppState { db: PgPool }`，给 `AppState` 实现 `FromRef`，观察 `DatabaseConnection` extractor 是否不用改。

## 小结

这章分 4 步搭了一个连 PostgreSQL 的 axum 应用：

1. **建连接池**：`PgPoolOptions` 配 `max_connections`/`acquire_timeout`，`.connect().await` 拿到 `PgPool`（池不是单个连接，启动时建一次放进 state 复用）。
2. **`State<PgPool>` 直接查**：用第 16/17/18 章的 `State<T>` 取出池，`query_scalar` + `fetch_one(&pool)` 执行 SQL，`internal_error` 把错误转 500。
3. **自定义 extractor**：实现 `FromRequestParts`（只读 state 不读 body，不像 `Json` 消费 body），用 `FromRef<S>` 从 state 取 pool，`pool.acquire()` 借一个 `PoolConnection`。state 变复杂也不用改。
4. **POST handler 用 extractor**：和 `State<PgPool>` 写法对比——简单查询用前者，重复逻辑/要事务时用后者。

贯穿全章的关键认知：

- 数据库心智模型：**启动建连接池 → Router state 保存池 → handler 从 state/extractor 获得数据库能力 → SQLx 执行 SQL → 错误转 HTTP 响应**。
- **SQLx 的招牌是 `query!` 宏——编译期校验 SQL**，真实项目优先用 `query!`/`query_as!` 把错误消灭在编译期；本章用 `query_scalar` 是因为 SQL 是无列名常量。
- 初学先掌握 `State<PgPool>`，重复逻辑多了再写自定义 `FromRequestParts` extractor。

## 源码对照

- `examples/sqlx-postgres/Cargo.toml`
- `examples/sqlx-postgres/src/main.rs`
