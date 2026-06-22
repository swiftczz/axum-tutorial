# 34. tokio-postgres

对应示例：`examples/tokio-postgres`

上一章用 SQLx,这章换成更底层的 `tokio-postgres`,并用 `bb8` 提供连接池。多一个角色:**连接管理器 manager**——告诉连接池如何创建 PostgreSQL 连接。理解连接管理器、连接池、查询 row、自定义 extractor。

## Cargo.toml

````toml
[package]
name = "example-tokio-postgres"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
bb8 = "0.9.0"
bb8-postgres = "0.9.0"
tokio = { version = "1.0", features = ["full"] }
tokio-postgres = "0.7.2"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

````rust
use axum::{
    extract::{FromRef, FromRequestParts, State},
    http::{request::Parts, StatusCode},
    routing::get,
    Router,
};
use bb8::{Pool, PooledConnection};
use bb8_postgres::PostgresConnectionManager;
use tokio_postgres::NoTls;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let manager =
        PostgresConnectionManager::new_from_stringlike("host=localhost user=postgres", NoTls)
            .unwrap();

    let pool = Pool::builder().build(manager).await.unwrap();

    let app = Router::new()
        .route(
            "/",
            get(using_connection_pool_extractor).post(using_connection_extractor),
        )
        .with_state(pool);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

type ConnectionPool = Pool<PostgresConnectionManager<NoTls>>;

async fn using_connection_pool_extractor(
    State(pool): State<ConnectionPool>,
) -> Result<String, (StatusCode, String)> {
    let conn = pool.get().await.map_err(internal_error)?;

    let row = conn
        .query_one("select 1 + 1", &[])
        .await
        .map_err(internal_error)?;
    let two: i32 = row.try_get(0).map_err(internal_error)?;

    Ok(two.to_string())
}

struct DatabaseConnection(PooledConnection<'static, PostgresConnectionManager<NoTls>>);

impl<S> FromRequestParts<S> for DatabaseConnection
where
    ConnectionPool: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = (StatusCode, String);

    async fn from_request_parts(_parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let pool = ConnectionPool::from_ref(state);
        let conn = pool.get_owned().await.map_err(internal_error)?;
        Ok(Self(conn))
    }
}

async fn using_connection_extractor(
    DatabaseConnection(conn): DatabaseConnection,
) -> Result<String, (StatusCode, String)> {
    let row = conn
        .query_one("select 1 + 1", &[])
        .await
        .map_err(internal_error)?;
    let two: i32 = row.try_get(0).map_err(internal_error)?;

    Ok(two.to_string())
}

fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
````

## 运行

需要先启动 PostgreSQL,确保 `host=localhost user=postgres` 可用:

````bash
cd examples
cargo run -p example-tokio-postgres
````

验证:

````bash
curl http://127.0.0.1:3000/         # GET
curl -X POST http://127.0.0.1:3000/  # POST
````

两个都返回 `2`。连接失败检查:PostgreSQL 是否启动、用户 `postgres` 是否存在、是否需要密码、连接串是否要加 `password` 或 `dbname`。

## 解读

### tokio-postgres vs SQLx

```text
SQLx:           PgPoolOptions → PgPool → query_scalar → fetch_one(自带池和查询 API)
tokio-postgres: PostgresConnectionManager → bb8 Pool → conn.query_one → row.try_get
```

```text
SQLx 提供连接池和查询 API
tokio-postgres 提供 PostgreSQL 客户端
bb8 负责连接池(通用异步连接池)
```

所以这章多一个角色:**manager**——告诉连接池如何创建/检查 PostgreSQL 连接。

### manager + Pool

````rust
let manager = PostgresConnectionManager::new_from_stringlike(
    "host=localhost user=postgres", NoTls
).unwrap();
let pool = Pool::builder().build(manager).await.unwrap();
````

`PostgresConnectionManager` 告诉 bb8 怎么创建连接、怎么检查连接可用。`NoTls` 表示不用 TLS(本地开发可以,生产看部署环境)。`Pool::builder().build(manager)` 用 manager 创建连接池,池负责维护连接、借出、归还、不可用时重建。心智模型和 SQLx `PgPool` 一样。

**为什么不用 DATABASE_URL?** 源码直接写连接串是让 example 更短,真实项目推荐从环境变量读取。

### 类型别名

````rust
type ConnectionPool = Pool<PostgresConnectionManager<NoTls>>;
````

完整类型太长,起别名后可以写 `State(pool): State<ConnectionPool>` 而不是每次写完整泛型。Rust 常见可读性优化。

### GET:借连接 + 查询 + 取列

````rust
async fn using_connection_pool_extractor(
    State(pool): State<ConnectionPool>,
) -> Result<String, (StatusCode, String)> {
    let conn = pool.get().await.map_err(internal_error)?;       // 借连接
    let row = conn.query_one("select 1 + 1", &[]).await...?;    // 执行 SQL 返回一行
    let two: i32 = row.try_get(0).map_err(internal_error)?;     // 取第 0 列
    Ok(two.to_string())
}
````

- `pool.get().await`:从池借连接。
- `query_one("select 1 + 1", &[])`:执行 SQL 要求返回一行。`&[]` 是 SQL 参数列表,本 SQL 无参数所以空数组。
- `row.try_get(0)`:从第 0 列取值并转 `i32`。列不存在或类型不匹配返回错误。

### Row 和 try_get

`tokio-postgres` 查询返回 `Row`(数据库一行数据,可能多列)。本章 SQL `select 1 + 1` 只一列,用 `row.try_get(0)`。和 SQLx 的 `query_scalar`(直接返回单值)不同,tokio-postgres 需要手动从 Row 取列。

### 自定义 extractor(同 32 章模式)

> 模式和第 33 章七八步完全一样(`FromRequestParts` + `FromRef`),理解了 32 章这里只需关注 bb8 的 `PooledConnection` 类型差异。

````rust
struct DatabaseConnection(PooledConnection<'static, PostgresConnectionManager<NoTls>>);

impl<S> FromRequestParts<S> for DatabaseConnection
where
    ConnectionPool: FromRef<S>, S: Send + Sync,
{
    async fn from_request_parts(_parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let pool = ConnectionPool::from_ref(state);
        let conn = pool.get_owned().await.map_err(internal_error)?;  // owned connection
        Ok(Self(conn))
    }
}
````

逻辑同 32 章:从 state 取 pool → 借连接 → 包装。差异是用 `pool.get_owned()`(返回 owned connection 适合放进 extractor 结构体),不是 `pool.get()`(普通借用)。

**`get` vs `get_owned`:** `get` 借普通连接适合当前函数直接用;`get_owned` 借 owned connection 适合放进自定义 extractor 结构体。

### 两种写法怎么选

| 方案 | 特点 |
| --- | --- |
| SQLx | 一体化:自带 pool、查询 API 更高级、可用 `query!` 宏做 SQL 检查 |
| tokio-postgres | 更接近 PostgreSQL 客户端:bb8 管池、手动从 Row 取列、最底层 |

初学先掌握共同模型(连接池 + State + 查询 + 错误处理),再根据项目选。

## 常见问题

**为什么不用 DATABASE_URL?** 源码直接写连接串让 example 短,真实项目推荐从环境变量读。

**bb8 是什么?** 通用异步连接池库。tokio-postgres 负责连 PostgreSQL,bb8 负责复用管理多个连接。

**`query_one` 和 `try_get`?** `query_one` 执行 SQL 返回一行,`try_get(0)` 从这行取第 0 列。

**SQL 参数为什么是 `&[]`?** 本章 SQL 无参数所以空数组;有参数就放进数组。

## 手写任务

按下面顺序敲:

1. `PostgresConnectionManager::new_from_stringlike` 创建 manager。
2. `Pool::builder().build(manager).await` 创建连接池。
3. 定义 `type ConnectionPool = ...`。
4. pool 放进 `.with_state(pool)`。
5. 写 GET handler 从 `State<ConnectionPool>` 借连接。
6. `query_one("select 1 + 1", &[])` 查询。
7. `row.try_get(0)` 取结果。
8. 实现 `DatabaseConnection` extractor。

加深练习:

1. 连接串改成从环境变量读。
2. SQL 改成带参数,如 `select $1::int + $2::int`。
3. 新建表,练习从 row 取多列字段。

## 小结

- tokio-postgres 和 SQLx 共同模型:启动创建连接池 → state 保存池 → handler/extractor 借连接 → 执行 SQL → 结果转 HTTP 响应。
- 不同点:SQLx 自带 `PgPool`,tokio-postgres 需要 bb8 + `PostgresConnectionManager`(manager 告诉 pool 如何创建连接)。
- tokio-postgres 查询返回 `Row`,需手动 `try_get(0)` 取列;SQLx 的 `query_scalar` 直接返回单值。
- 自定义 extractor 模式同 32 章,差异是用 `pool.get_owned()` 借 owned connection 放进结构体。
- 学会共同模型后切换数据库库不是背 API 而是复用思路。

## 源码对照

- `examples/tokio-postgres/Cargo.toml`
- `examples/tokio-postgres/src/main.rs`
