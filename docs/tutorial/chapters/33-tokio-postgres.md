# 33. tokio-postgres

对应示例：`examples/tokio-postgres`

这章是第一个数据库章节，用最底层的 `tokio-postgres`——纯异步 PostgreSQL 客户端，没有 ORM、没有 DSL，直接写 SQL 字符串。理解它就理解了所有 PG 客户端的骨架：**连接池 + 查询 + 错误处理**。

本章演示**两种**获取连接的写法：直接 `State<Pool>` 和自定义 `DatabaseConnection` extractor。两种都合法，看团队偏好。

分 3 步：先建连接池和最简 handler，再用自定义 extractor 重写，最后对比两种写法。

相比前面章节新引入：**`tokio-postgres` crate、`bb8` 连接池 + `bb8-postgres`、`Pool<PostgresConnectionManager>` 作为 State、自定义 `FromRequestParts` 取连接**。

## Cargo.toml

````toml
[package]
name = "example-tokio-postgres"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
bb8 = "0.9"
bb8-postgres = "0.9"
tokio = { version = "1.0", features = ["full"] }
tokio-postgres = "0.7"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：bb8 连接池 + 用 `State<Pool>` 跑一条 SQL

先建连接池：`bb8` 是通用连接池（不光是 Postgres），`bb8-postgres` 是它的 Postgres 适配。把 `Pool<PostgresConnectionManager<NoTls>>` 作为 State，handler 用 `State<Pool>` 取连接跑 SQL。

````rust
use axum::{
    extract::State,
    http::StatusCode,
    routing::get,
    Router,
};
use bb8::Pool;
use bb8_postgres::PostgresConnectionManager;
use tokio_postgres::NoTls;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

type ConnectionPool = Pool<PostgresConnectionManager<NoTls>>;

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 建连接池
    let manager =
        PostgresConnectionManager::new_from_stringlike("host=localhost user=postgres", NoTls)
            .unwrap();
    let pool = Pool::builder().build(manager).await.unwrap();

    let app = Router::new()
        .route("/", get(using_connection_pool_extractor))
        .with_state(pool);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

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

fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
````

验证（需要本地 PostgreSQL，用户名 `postgres`）：

````bash
cd examples
cargo run -p example-tokio-postgres
````

````bash
curl http://127.0.0.1:3000/
# 返回 "2"
````

> **新面孔：`tokio-postgres` crate**
>
> 最底层的异步 PostgreSQL 客户端——没有 ORM、没有 DSL，直接写 SQL 字符串。`conn.query_one("select 1 + 1", &[]).await` 跑 SQL 返回一行，`row.try_get(0)` 按列索引取值。
>
> 后面章节会看到 sqlx/Diesel 等高级封装，但理解 tokio-postgres 就理解了所有 PG 客户端的基础。

> **新面孔：`bb8` 连接池 + `bb8-postgres`**
>
> `bb8` 是通用异步连接池。`bb8-postgres` 是它的 Postgres 适配：
> - `PostgresConnectionManager::new_from_stringlike(conn_str, NoTls)` 创建 manager（知道怎么建连接）
> - `Pool::builder().build(manager).await.unwrap()` 建池（管理连接生命周期、复用、超时）
> - `pool.get().await` 借一个连接
>
> `NoTls` 表示不加密连接（生产用 `NoTls::with_certificate(...)` 或其他 TLS backend）。

> **新面孔：`Pool<PostgresConnectionManager<NoTls>>` 作为 State**
>
> 和 ch18 依赖注入同构：`Pool` 可 cheap clone（内部 `Arc`），塞进 `with_state(pool)`，handler 用 `State<ConnectionPool>` 取。`type ConnectionPool = Pool<PostgresConnectionManager<NoTls>>` 起个短别名避免类型太长。

> **新面孔：`query_one` / `try_get`**
>
> tokio-postgres 的核心查询 API：
> - `conn.query_one(sql, &[])`：跑 SQL 返回**一行**（`NoData` 时报错）
> - `conn.query(sql, &[])`：跑 SQL 返回**多行**（`Vec<Row>`）
> - `row.try_get(0)`：按列索引取值并转类型（`try_get` 失败返回 Err）
>
> `&[]` 是 SQL 参数（这章没参数所以空数组）。带参数用 `&[&user_id]`，对应 SQL 的 `$1`。

---

## 第二步：自定义 `DatabaseConnection` extractor

第一步用 `State<Pool>` 借连接，handler 里每次都要 `pool.get().await`。这步演示另一种写法：自定义 extractor 把"借连接"封装成 `DatabaseConnection`，handler 参数更干净。

````rust
use axum::{
    extract::{FromRef, FromRequestParts},
    http::request::Parts,
};
use bb8::PooledConnection;

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

# #[tokio::main]
# async fn main() {
#     // ...
    let app = Router::new()
        .route(
            "/",
            get(using_connection_pool_extractor).post(using_connection_extractor),
        )
        .with_state(pool);
#     // ...
# }
````

> **新面孔：自定义 `FromRequestParts` 取连接**
>
> 和 ch11 自定义 extractor 同构，只是这步是从 State 借连接而不是解析 body。
>
> 关键约束 `ConnectionPool: FromRef<S>`：保证能从 state `S` 切出 pool。`FromRef::from_ref(state)` 是 axum 提供的方法。
>
> 为什么用 `FromRequestParts` 不用 `FromRequest`？取连接不读 body，所以用更轻的 `FromRequestParts`（只拿 parts）。

> **新面孔：`pool.get_owned` vs `pool.get`**
>
> - `pool.get()` → `PooledConnection<'_, ...>`，借引用（生命周期绑在 pool 上）
> - `pool.get_owned()` → `PooledConnection<'static, ...>`，借 owned（可移动，不绑 pool）
>
> 自定义 extractor 用 `get_owned` 因为 `Self(conn)` 要把连接移进 struct——引用不行。`'static` 不是说永久拥有，归还池逻辑还在 `PooledConnection` 的 Drop impl 里。

> **新面孔：`FromRef<S>` 切片出 State**
>
> `FromRef<S>` 是 axum 的 trait："从复合 State 切出一部分"。如果 state 是单一类型（这章就是 `Pool`），`Pool: FromRef<Pool>` 自动实现（identity）。
>
> 多个 sub-state 时（如 `Pool` + `Config`）需要包成 struct 手写 `FromRef`——见 ch18 依赖注入。

---

## 第三步：对比两种写法

```text
写法一 State<Pool>:
    async fn handler(State(pool): State<Pool>) {
        let conn = pool.get().await.map_err(internal_error)?;
        // 用 conn
    }

写法二 DatabaseConnection extractor:
    async fn handler(DatabaseConnection(conn): DatabaseConnection) {
        // 直接用 conn，借连接由 extractor 自动做了
    }
```

两种等价，取舍：

| 维度 | State<Pool> | 自定义 extractor |
| --- | --- | --- |
| handler 参数 | 显式 `pool.get()` 借连接 | 自动借连接 |
| 错误处理 | 每次手写 `map_err` | extractor 集中处理 |
| 灵活性 | 想不借连接也行 | handler 必须借连接 |
| 类型复杂度 | 简单 | 要写 trait impl |

真实项目统一选一种。后面 Diesel/Diesel-async/sqlx 章节都见到这两种写法。

---

## 完整代码

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

    // set up connection pool
    let manager =
        PostgresConnectionManager::new_from_stringlike("host=localhost user=postgres", NoTls)
            .unwrap();
    let pool = Pool::builder().build(manager).await.unwrap();

    // build our application with some routes
    let app = Router::new()
        .route(
            "/",
            get(using_connection_pool_extractor).post(using_connection_extractor),
        )
        .with_state(pool);

    // run it
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

// we can also write a custom extractor that grabs a connection from the pool
// which setup is appropriate depends on your application
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
# 启动 PostgreSQL（docker）
docker run -d --name pg -p 5432:5432 \
  -e POSTGRES_HOST_AUTH_METHOD=trust \
  postgres:16

cd examples
cargo run -p example-tokio-postgres
````

两种 handler 都用 `/` 路径：

````bash
curl http://127.0.0.1:3000/        # GET 走 using_connection_pool_extractor，返回 "2"
curl -X POST http://127.0.0.1:3000/ # POST 走 using_connection_extractor，返回 "2"
````

## 解读

### bb8 连接池的作用

```text
无连接池:   每次请求新建连接 → 三次握手 + 认证 → 慢
有连接池:   启动时建 N 个连接放池里 → 请求来时借一个 → 用完归还
```

`Pool::builder().build(manager)` 在后台预热连接（默认 10 个），`pool.get().await` 借空闲连接（满了就等），Drop 时归还。

### 错误模型

`tokio-postgres` 原生 async，错误只有一层（不像 Diesel 那样两层）。`internal_error` 是个通用 helper：把任何 `std::error::Error` 转成 `(500, message)`。生产环境通常要做更细的错误映射（如唯一约束冲突 → 409 而不是 500）。

## 常见问题

**为什么用 `tokio-postgres` 不用 `sqlx`/`diesel`？** 本章演示最底层。tokio-postgres 没 ORM 没 DSL，最直接理解 PG 协议和连接池。后面 sqlx 章节（ch32）有编译期 SQL 校验，更高级。

**`NoTls` 安全吗？** 本地开发够用，生产环境用 TLS（防中间人攻击）。`tokio-postgres` 支持多种 TLS backend（rustls、native-tls、openssl）。

**`pool.get` 和 `pool.get_owned` 怎么选？** handler 里临时用 `get`；要移进 struct（如自定义 extractor）用 `get_owned`。本章自定义 extractor 用 `get_owned` 因为 `Self(conn)` 要 move。

## 手写任务

1. 加 `POST /user` 用 `conn.execute("INSERT INTO users VALUES ($1, $2)", &[&id, &name])` 插入。
2. 改成多列 `CREATE TABLE` 后用 `conn.query("SELECT id, name FROM users", &[])` 列表查询。
3. 写第三种 handler，把 SQL 包成函数 `async fn get_two(conn: &Connection) -> Result<i32, ...>`，handler 调它——演示"业务逻辑函数 + handler 薄包装"分层。

## 小结

这章用 3 步讲了 tokio-postgres 基础：

1. **连接池 + State<Pool>**：`bb8-postgres` 建池，`pool.get().await` 借连接，`conn.query_one(sql, &[])` 跑 SQL。
2. **自定义 DatabaseConnection extractor**：`FromRequestParts` 封装"从 state 取 pool → 借连接"，handler 参数更干净。
3. **对比两种写法**：`State<Pool>` 灵活，自定义 extractor 整洁，真实项目统一选一种。

核心：tokio-postgres 是最底层的异步 PG 客户端，理解了 `Pool` + `query_one` + `try_get` + 自定义 extractor 这套骨架，后面 sqlx/Diesel 都是变体。

## 源码对照

- `examples/tokio-postgres/Cargo.toml`
- `examples/tokio-postgres/src/main.rs`
