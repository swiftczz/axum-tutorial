# 36. tokio-redis

对应示例：`examples/tokio-redis`

前几章连 PostgreSQL,这章连 Redis。用异步 Redis 客户端连接 Redis,理解 `redis::Client`、`bb8` 连接池、`AsyncCommands`,以及 axum handler 读取 Redis 数据。Redis 常用作内存型 key-value 系统(缓存、登录会话、限流计数、排行榜、短期状态)。



相比前面章节新引入：**`redis::Client` + bb8 Pool、`AsyncCommands`（`get`/`set`）、Redis 单线程与 `MultiplexedConnection`**。

## Cargo.toml

````toml
[package]
name = "example-tokio-redis"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
bb8 = "0.9"
redis = { version = "1", default-features = false, features = ["tokio-comp", "bb8"] }
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

- `tokio-comp`:Tokio 异步支持。
- `bb8`:让 `redis::Client` 配合 bb8 连接池。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 完整代码

````rust
use axum::{
    extract::{FromRef, FromRequestParts, State},
    http::{request::Parts, StatusCode},
    routing::get,
    Router,
};
use redis::AsyncCommands;
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

    tracing::debug!("connecting to redis");
    let client = redis::Client::open("redis://localhost").unwrap();
    let pool = bb8::Pool::builder().build(client).await.unwrap();

    {
        let mut conn = pool.get().await.unwrap();
        conn.set::<&str, &str, ()>("foo", "bar").await.unwrap();
        let result: String = conn.get("foo").await.unwrap();
        assert_eq!(result, "bar");
    }
    tracing::debug!("successfully connected to redis and pinged it");

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

type ConnectionPool = bb8::Pool<redis::Client>;

async fn using_connection_pool_extractor(
    State(pool): State<ConnectionPool>,
) -> Result<String, (StatusCode, String)> {
    let mut conn = pool.get().await.map_err(internal_error)?;
    let result: String = conn.get("foo").await.map_err(internal_error)?;
    Ok(result)
}

struct DatabaseConnection(bb8::PooledConnection<'static, redis::Client>);

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
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<String, (StatusCode, String)> {
    let result: String = conn.get("foo").await.map_err(internal_error)?;
    Ok(result)
}

fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
````

## 运行

先启动 Redis,默认地址 `redis://localhost`:

````bash
cd examples
cargo run -p example-tokio-redis
````

验证:

````bash
curl http://127.0.0.1:3000/         # GET
curl -X POST http://127.0.0.1:3000/  # POST
````

两个都返回 `bar`。启动失败检查:Redis 是否启动、6379 端口是否可达、是否需要密码、地址是否要改成 `redis://:password@localhost`。

## 解读

### Redis vs PostgreSQL

| | PostgreSQL | Redis |
| --- | --- | --- |
| 类型 | 关系型数据库 | 内存型 key-value 系统 |
| 适合 | 结构化数据(用户/订单/文章表)、复杂查询、事务 | 缓存、登录会话、限流计数、排行榜、短期状态 |

本章只演示最基础的 key-value:`SET foo bar` / `GET foo`。理解成"通过 key 找 value 的高速存储"。

### Redis 连接池 vs 单连接多路复用

Redis 必须用连接池吗?不一定。Redis 是**单线程**处理命令的,一条连接吞吐其实很大。两种主流用法:

1. **连接池(本章用法)**:bb8 维护多条连接,每次请求借一条。适合命令执行时间较长、想提高并发度。
2. **`MultiplexedConnection`(单连接多路复用)**:只开一条底层 TCP 连接,通过 futures 分流让多个任务**共享这一条连接**并发执行。连接数少更省资源,很多生产 Redis 代码用这种。

````rust
// MultiplexedConnection 大致用法
let conn = client.get_multiplexed_async_connection().await?;
let conn_clone = conn.clone();   // 共享底层连接,多任务并发用
conn.set("a", 1).await?;
````

本章用 bb8 池是 example 选择。命令都很轻量时,`MultiplexedConnection` 通常更简单。理解"Redis 单线程,连接不是瓶颈"这个本质,选哪个都心里有数。

**注意**:`redis::Client` 本身**不持有连接**,只是"连接工厂"几乎零成本,真正持有连接的是 Pool 或 MultiplexedConnection。

### 启动前 ping Redis

````rust
{
    let mut conn = pool.get().await.unwrap();
    conn.set::<&str, &str, ()>("foo", "bar").await.unwrap();
    let result: String = conn.get("foo").await.unwrap();
    assert_eq!(result, "bar");
}
````

启动前检查:借连接 → `SET foo bar` → `GET foo` → 确认 `bar`。Redis 没启动或连接失败,启动阶段就失败,比服务起来后才发现更清晰。外层花括号让 conn 尽快离开作用域还回池。

### `AsyncCommands` 提供 get/set

````rust
use redis::AsyncCommands;   // 必须导入!
````

`set`/`get` 等异步方法来自 `AsyncCommands` trait。忘导入编译器报 `no method named get/set`。

`set::<&str, &str, ()>("foo", "bar")` 的类型参数:key 是 `&str`、value 是 `&str`、返回值忽略为 `()`。

### 两种获取连接写法

**GET 用 `State<ConnectionPool>`(最直接):**

````rust
async fn using_connection_pool_extractor(
    State(pool): State<ConnectionPool>,
) -> Result<String, (StatusCode, String)> {
    let mut conn = pool.get().await.map_err(internal_error)?;
    let result: String = conn.get("foo").await.map_err(internal_error)?;
    Ok(result)
}
````

注意 `conn` 是 `mut`,因为 Redis 命令需要可变连接引用。

**POST 用自定义 `DatabaseConnection` extractor:**

````rust
struct DatabaseConnection(bb8::PooledConnection<'static, redis::Client>);

impl<S> FromRequestParts<S> for DatabaseConnection
where ConnectionPool: FromRef<S>, S: Send + Sync,
{
    async fn from_request_parts(_parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let pool = ConnectionPool::from_ref(state);
        let conn = pool.get_owned().await.map_err(internal_error)?;
        Ok(Self(conn))
    }
}
````

虽然叫 `DatabaseConnection`,这里包的是 Redis 连接(Redis 也常被称为一种数据库)。模式同前几章:从 state 取 pool → `get_owned()` 借 owned connection → 包装。`FromRequestParts` 合适因为不读 body。

## 常见问题

**Redis 需要建表吗?** 不需要,本章只是 key-value,不像 PostgreSQL 先建表。

**为什么导入 `AsyncCommands`?** `get`/`set` 来自这个 trait,忘导入找不到方法。

**为什么 Redis 也用连接池?** 高并发服务每请求新建连接有成本,连接池复用减少开销;但 Redis 单线程,`MultiplexedConnection` 也是常见选择。

**Redis 数据永久保存吗?** 取决于配置。Redis 常作内存数据库,持久化策略/过期时间/重启行为按项目配置。

**Redis 能替代 PostgreSQL 吗?** 通常不能。Redis 适合缓存和快速 key-value,PostgreSQL 适合长期结构化数据和复杂查询。

## 手写任务

按下面顺序敲:

1. 启动 Redis。
2. `redis::Client::open("redis://localhost")` 创建 client。
3. `bb8::Pool::builder().build(client).await` 创建连接池。
4. 启动时用 `set foo bar` 和 `get foo` 验证连接。
5. pool 放进 `.with_state(pool)`。
6. 写 GET handler 用 `State<ConnectionPool>` 读 `foo`。
7. 实现 `DatabaseConnection` extractor。
8. 写 POST handler 通过 extractor 读 `foo`。

加深练习:

1. 新增 `POST /cache/{key}/{value}` 写入任意 key。
2. 新增 `GET /cache/{key}` 读取任意 key。
3. 给 key 设置过期时间,理解缓存失效。

## 小结

- Redis 接入 axum 核心模型和数据库连接池很像:启动创建 client 和 pool → state 保存 pool → handler 借连接 → `AsyncCommands` 执行命令 → 结果返回客户端。
- Redis 是单线程,一条连接吞吐很大;连接池(bb8)和 `MultiplexedConnection`(单连接多路复用)都常见,命令轻量时后者更省资源。
- `redis::Client` 不持有连接只是连接工厂,真正持有连接的是 Pool/MultiplexedConnection。
- `AsyncCommands` trait 提供 `get`/`set` 等异步方法,必须导入。
- 两种获取连接写法:`State<ConnectionPool>`(直接)和自定义 `DatabaseConnection` extractor(可复用);`conn` 要 `mut`(Redis 命令需可变连接引用)。

## 源码对照

- `examples/tokio-redis/Cargo.toml`
- `examples/tokio-redis/src/main.rs`
