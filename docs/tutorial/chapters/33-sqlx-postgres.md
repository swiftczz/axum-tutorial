# 33. sqlx-postgres

对应示例：`examples/sqlx-postgres`

进入数据库章节(32-37)。用 SQLx 连接 PostgreSQL,理解连接池 `PgPool`、axum `State`、自定义数据库连接 extractor。核心心智:Web 请求 → handler → 从连接池拿数据库连接 → 执行 SQL → 把结果变成 HTTP 响应。

## 先理解:5 个数据库方案怎么选

32-37 章讲 5 种数据库接入方式,动手前先建立全局印象:

| 方案 | 章节 | 异步/同步 | 连接池 | 编译期 SQL 校验 | ORM 程度 |
| --- | --- | --- | --- | --- | --- |
| **SQLx** | 32 | 原生异步 | 自带 `PgPool` | 有(`query!` 宏,需连数据库编译) | 半 ORM(写 SQL,自动映射) |
| **tokio-postgres + bb8** | 33 | 原生异步 | bb8 通用池 | 无 | 最底层(裸 SQL + 手动取列) |
| **Diesel(同步)+ deadpool** | 34 | 同步(`interact` 包阻塞线程) | deadpool-diesel | 有(`table!` + 类型系统) | 重 ORM(DSL 写查询) |
| **Diesel Async + bb8** | 35 | 原生异步 | bb8 | 有 | 重 ORM |
| **Redis** | 36 | 原生异步 | bb8 或 `MultiplexedConnection` | 无 | 无(KV 命令式) |
| **MongoDB** | 37 | 原生异步 | `Client` 自带内部池 | 无 | 文档型(BSON) |

**怎么选:**

- 想省心、写 SQL 自动映射 → **SQLx**(32 章),编译期校验 SQL 是杀手锏。
- 想完全控制、不要 ORM/宏 → **tokio-postgres**(33 章),最轻量但手动取列管池。
- 想强类型 DSL、不怕重 → **Diesel**(34 同步/35 异步),类型安全最强但学习曲线陡。
- 同步还是异步?新项目优先 35 章 diesel-async;必须用同步 API 时用 34 章。
- 缓存/计数器/排行榜 → **Redis**(36 章)。
- 文档型数据/灵活 schema → **MongoDB**(37 章)。

**关键概念:同步 driver 在异步程序里怎么办?** Diesel(34 章)是同步 driver,查询会阻塞当前线程,直接在 `async fn` 里调用会卡住整个 Tokio worker。所以用 `pool.interact(|conn| ...)` 把同步查询扔到专门的阻塞线程池(底层 `tokio::task::spawn_blocking`)。这是同步 driver 接入异步 runtime 的标准做法。而 SQLx(32)/tokio-postgres(33)/diesel-async(35)原生异步,可直接 `.await`。

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

- `runtime-tokio`:SQLx 用 Tokio 异步运行时。
- `postgres`:启用 PostgreSQL 支持。
- `any`:启用通用数据库抽象,本例代码实际没用(直接用 `PgPool`),example 沿用配置,你自己用 Postgres 可以不加。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

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

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .acquire_timeout(Duration::from_secs(3))
        .connect(&db_connection_str)
        .await
        .expect("can't connect to database");

    let app = Router::new()
        .route(
            "/",
            get(using_connection_pool_extractor).post(using_connection_extractor),
        )
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

fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
````

## 运行

需要先有 PostgreSQL 服务。设置连接地址(或用默认 `postgres://postgres:password@localhost`):

````bash
export DATABASE_URL='postgres://postgres:password@localhost'
cd examples
cargo run -p example-sqlx-postgres
````

验证:

````bash
curl http://127.0.0.1:3000/         # GET
curl -X POST http://127.0.0.1:3000/  # POST
````

两个都返回 `hello world from pg`。连接失败检查:PostgreSQL 是否启动、DATABASE_URL 是否正确、用户名密码、端口可达。

## 解读

### 为什么需要连接池

不能每来一个请求都重新建数据库连接(TCP 连接 + 认证 + 初始化成本高,容易压垮数据库)。连接池的做法:程序启动时创建一组连接,请求来了借一个,用完还回去。`PgPool` 就是 PostgreSQL 连接池。

### 创建 PgPool

````rust
let pool = PgPoolOptions::new()
    .max_connections(5)                          // 最多 5 个连接
    .acquire_timeout(Duration::from_secs(3))     // 借连接最多等 3 秒
    .connect(&db_connection_str).await
    .expect("can't connect to database");
````

数据库连不上启动就失败(`.expect`)——服务没数据库无法正常工作。`DATABASE_URL` 放环境变量,代码不用写死不同环境的地址。

### 写法一:`State<PgPool>`(最直接)

````rust
async fn using_connection_pool_extractor(
    State(pool): State<PgPool>,
) -> Result<String, (StatusCode, String)> {
    sqlx::query_scalar("select 'hello world from pg'")
        .fetch_one(&pool).await
        .map_err(internal_error)
}
````

`query_scalar` 查询单值,`fetch_one(&pool)` 从池借连接执行 SQL 拿一行,用完连接回池。

### 写法二:自定义 `DatabaseConnection` extractor

````rust
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
        .fetch_one(&mut *conn).await
        .map_err(internal_error)
}
````

handler 参数里需要 `DatabaseConnection` 时,axum 自动从 state 找到 `PgPool`,acquire 一个连接包装好。handler 更聚焦业务,连接获取逻辑可复用,以后能扩展事务/鉴权。

**为什么用 `FromRequestParts` 不是 `FromRequest`?** 数据库连接 extractor 只需 state + parts,不需读 body。读 body 的 extractor 会消费请求体,数据库连接不该碰 body。

**`FromRef<S>` 是什么?** "可以从应用 state S 中取出 PgPool"。本章 state 本身就是 `PgPool`,所以 `PgPool::from_ref(state)` 拿到。好处是 state 变复杂(如 `AppState { db: PgPool, config: AppConfig }`)时仍能扩展。

### 两种写法怎么选

| 写法 | 优点 | 适合 |
| --- | --- | --- |
| `State<PgPool>` | 简单直观 | 大多数普通查询 |
| 自定义 extractor | handler 聚焦业务、连接获取逻辑复用、可扩展事务/鉴权 | 多 handler 重复连接获取逻辑 |

初学先掌握 `State<PgPool>`,等重复逻辑出现再抽象。

### SQLx 的 `query!` 宏——编译期 SQL 校验

本章用 `query_scalar("...")` 是**运行期**执行(SQL 字符串运行时才发给 Postgres,列名错/类型不匹配运行时才报错)。但 SQLx 真正的招牌是 **`query!` 宏**——**编译期**校验 SQL:

````rust
let row: (String,) = sqlx::query_as("select name from users where id = $1")
    .bind(user_id)
    .fetch_one(&pool).await?;
````

编译期真的连数据库执行 EXPLAIN 检查:SQL 语法对不对、表名列名存不存在、返回列类型能不能映射到 Rust 类型。列名写错(如 `select nam`)编译就失败,根本到不了运行时。这是 SQLx 相对 tokio-postgres(33 章)最大的优势。

用 `query!` 的前提:编译时能连上数据库(通过 `DATABASE_URL`),或用 `cargo sqlx prepare` 生成离线缓存(`.sqlx/` 目录)让 CI/无数据库环境也能编译。本例没用是因为 SQL 是常量 `select 'hello world from pg'` 没列名可校验;真实项目查询带表名列名时**优先用 `query!`/`query_as!`**,把 SQL 错误消灭在编译期。

### 错误转换

````rust
fn internal_error<E>(err: E) -> (StatusCode, String)
where E: std::error::Error {
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
````

任意数据库错误转 500 + 错误文本。真实项目通常不把数据库错误原文返回用户。

## 常见问题

**没建表也能查询?** SQL 是 `select 'hello world from pg'`,让 Postgres 返回字符串常量,不需要任何表。

**PgPool 是真实连接?** 不是,是连接池管理多个连接。handler 把 pool 传给 SQLx,SQLx 从池借连接执行查询。

**`max_connections` 设多少?** 取决于数据库能力、服务实例数、请求量、查询耗时。连接太少请求排队,太多压垮数据库。

**能不能每请求新建 PgPool?** 不要,连接池启动时创建一次放进 state 复用。

## 手写任务

按下面顺序敲:

1. 读 `DATABASE_URL`。
2. `PgPoolOptions::new()` 创建连接池。
3. `pool` 放进 `.with_state(pool)`。
4. 写 GET handler 用 `State(pool): State<PgPool>`。
5. handler 里执行 `sqlx::query_scalar(...)`。
6. 写 `internal_error` 把 SQLx 错误转 500。
7. 再写 `DatabaseConnection` extractor。
8. POST handler 验证自定义 extractor。

加深练习:

1. SQL 改成 `select now()` 返回当前数据库时间。
2. 新建 `users` 表,查询一行用户。
3. state 改成 `AppState { db: PgPool }`,练习 `FromRef`。

## 小结

- 数据库章节心智模型:启动创建连接池 → Router state 保存池 → handler 从 state/extractor 获得数据库能力 → SQLx 执行 SQL → 错误转 HTTP 响应。
- `PgPool` 是连接池不是单个连接;启动时创建一次放进 state 复用。
- 初学先掌握 `State<PgPool>`,重复逻辑多了再写自定义 `FromRequestParts` extractor。
- 自定义 extractor 用 `FromRequestParts`(只读 state 不读 body),`FromRef<S>` 让从复杂 state 取 PgPool。
- **SQLx 招牌是 `query!` 宏——编译期校验 SQL**,真实项目优先用 `query!`/`query_as!` 把错误消灭在编译期。

## 源码对照

- `examples/sqlx-postgres/Cargo.toml`
- `examples/sqlx-postgres/src/main.rs`
