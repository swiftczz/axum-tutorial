# 36. tokio-redis

对应示例：`examples/tokio-redis`

前面几章数据库都是 PG/MongoDB 这类持久存储。这章用 **Redis**——内存键值存储，速度极快（μs 级），常用作缓存、session、排行榜、pub/sub。

结构和 ch33 tokio-postgres 几乎一样：bb8 连接池 + State，外加自定义 extractor 写法。区别只是 client 换成 `redis::Client`，操作换成 `get`/`set`。

分 3 步：先建连接池 + ping，再用 `State<Pool>` 写 GET handler，最后用自定义 extractor 重写。

相比前面章节新引入：**`redis` crate、`bb8::Pool<redis::Client>`（redis 内置 bb8 支持，不用 `bb8-redis`）、`AsyncCommands` trait 的 `get`/`set`**。

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
redis = { version = "0.32", features = ["tokio-comp"] }
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

`redis` 启用 `tokio-comp`（tokio runtime 兼容）。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：bb8 连接池 + ping 验证

Redis 的 `redis::Client` 直接实现 bb8 的 manager 接口，所以建池比 PG 那章还简单——不用单独的 manager crate。这步建好池，启动时 `set`/`get` 一下验证连接。

````rust
use axum::{
    routing::get,
    Router,
};
use redis::AsyncCommands;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

type ConnectionPool = bb8::Pool<redis::Client>;

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

    // 启动时 ping 数据库
    {
        let mut conn = pool.get().await.unwrap();
        conn.set::<&str, &str, ()>("foo", "bar").await.unwrap();
        let result: String = conn.get("foo").await.unwrap();
        assert_eq!(result, "bar");
    }
    tracing::debug!("successfully connected to redis and pinged it");

    let app = Router::new().route("/", get(|| async { "ok" })).with_state(pool);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
````

验证（需要本地 Redis）：

````bash
# 用 docker 启 redis
docker run -d --name redis -p 6379:6379 redis:7

cd examples
cargo run -p example-tokio-redis
````

看到 `successfully connected to redis and pinged it` 就说明连上了。

> **新面孔：`redis::Client`**
>
> Redis 客户端，`Client::open("redis://localhost")` 建立连接（URL 格式 `redis://[:password@]host[:port][/database]`）。
>
> Redis 的 `Client` 直接实现 bb8 的 manager 接口，所以 `bb8::Pool::builder().build(client).await` 直接能用——**不需要单独的 `bb8-redis` crate**（Redis crate 内置 bb8 支持）。

> **新面孔：`AsyncCommands` trait**
>
> 提供 `get`/`set`/`del`/`incr` 等命令方法的 trait。`use redis::AsyncCommands;` 后才能调 `conn.set(...)`/`conn.get(...)`。
>
> `conn.set::<K, V, RV>("foo", "bar").await`：三个泛型参数分别是 key 类型、value 类型、返回类型。`set` 的返回类型 `()` 表示不需要返回值（写命令典型）。
>
> `conn.get("foo").await`：返回类型靠左侧 `let result: String = ...` 类型推导，所以不用 turbofish。

---

## 第二步：用 `State<Pool>` 写 GET handler

加一个 GET handler，从 Redis 读取 `foo` 的值返回。

````rust
use axum::{
    extract::State,
    http::StatusCode,
};

async fn using_connection_pool_extractor(
    State(pool): State<ConnectionPool>,
) -> Result<String, (StatusCode, String)> {
    let mut conn = pool.get().await.map_err(internal_error)?;
    let result: String = conn.get("foo").await.map_err(internal_error)?;
    Ok(result)
}

fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}

# #[tokio::main]
# async fn main() {
#     // ...
    let app = Router::new()
        .route("/", get(using_connection_pool_extractor))
        .with_state(pool);
#     // ...
# }
````

验证：

````bash
curl http://127.0.0.1:3000/
# 返回 "bar"（启动时 set 的值）
````

这步和 ch33 完全同构，只是连接类型从 `PostgresConnectionManager` 换成 `redis::Client`，查询从 `query_one(sql, &[])` 换成 `conn.get(key)`。

---

## 第三步：自定义 `DatabaseConnection` extractor 重写

和 ch33 一样，可以把"借连接"封装成自定义 extractor，handler 参数更干净。

````rust
use axum::{
    extract::{FromRef, FromRequestParts},
    http::request::Parts,
};

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

验证：

````bash
curl -X POST http://127.0.0.1:3000/
# 返回 "bar"
````

模式与 ch33 一模一样：`FromRequestParts` + `FromRef<S>` + `pool.get_owned()`。详细解释见第 33 章。

---

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
        // ping the database before starting
        let mut conn = pool.get().await.unwrap();
        conn.set::<&str, &str, ()>("foo", "bar").await.unwrap();
        let result: String = conn.get("foo").await.unwrap();
        assert_eq!(result, "bar");
    }
    tracing::debug!("successfully connected to redis and pinged it");

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

type ConnectionPool = bb8::Pool<redis::Client>;

async fn using_connection_pool_extractor(
    State(pool): State<ConnectionPool>,
) -> Result<String, (StatusCode, String)> {
    let mut conn = pool.get().await.map_err(internal_error)?;
    let result: String = conn.get("foo").await.map_err(internal_error)?;
    Ok(result)
}

// we can also write a custom extractor that grabs a connection from the pool
// which setup is appropriate depends on your application
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
# 启动 Redis（docker）
docker run -d --name redis -p 6379:6379 redis:7

cd examples
cargo run -p example-tokio-redis
````

两种 handler：

````bash
curl http://127.0.0.1:3000/        # GET 走 pool extractor，返回 "bar"
curl -X POST http://127.0.0.1:3000/ # POST 走 DatabaseConnection extractor，返回 "bar"
````

## 解读

### 和 ch33 的对比

```text
ch33 tokio-postgres:
  Pool<PostgresConnectionManager<NoTls>> + conn.query_one(sql, &[])

ch36 tokio-redis:
  Pool<redis::Client> + conn.get(key) / conn.set(key, value)
```

骨架完全一样，只换 client 类型和命令 API。说明 axum 的"State + Pool + 自定义 extractor"模式是通用的——所有存储后端都这套结构。

### Redis 用途（不只是缓存）

虽然 Redis 常被叫"缓存"，但它能做更多：

- **缓存**：`get`/`set` 配合 TTL（`set_ex(key, value, seconds)`）
- **Session**：把 session ID → user 数据存 Redis，多实例共享
- **排行榜**：`ZADD`/`ZREVRANGE` 有序集合
- **Pub/Sub**：`SUBSCRIBE`/`PUBLISH` 跨进程消息
- **限流**：`INCR` + EXPIRE 实现"每分钟 N 次"限制

## 常见问题

**为什么不用 `bb8-redis` crate？** Redis crate 本身已实现 bb8 manager 接口（`redis::Client: ManageConnection`），所以 `bb8::Pool::builder().build(client)` 直接能用，不需要额外 crate。

**`AsyncCommands` trait 怎么知道有哪些方法？** 导入 `use redis::AsyncCommands;` 后 IDE 自动补全。常见方法：`get`/`set`/`del`/`incr`/`exists`/`expire`/`set_ex`/`hset`/`hget`。

**为什么 Redis 是 `mut conn` 而 PG 不是？** redis crate 的命令方法要求 `&mut self`（内部有缓冲区），所以 `let mut conn = ...`。PG 的 `query_one` 是 `&self`。

## 手写任务

1. 加 `POST /set/:key/:value` 用 `conn.set(key, value)` 写入。
2. 加 `incr` handler：`conn.incr::<_, _, i64>("counter")` 实现 counter。
3. 加 `set_ex` 带过期：`conn.set_ex("temp", "value", 60)` 60 秒过期。

## 小结

这章用 3 步讲了 Redis 集成，结构和 ch33 tokio-postgres 同构：

1. **连接池 + ping**：`bb8::Pool::builder().build(redis::Client::open(...))` + `set`/`get` 验证。
2. **State<Pool> handler**：`pool.get().await` 借连接，`conn.get(key)` 读 Redis。
3. **自定义 DatabaseConnection extractor**：和 ch33 同构。

核心：**axum 的 "State + Pool + 自定义 extractor" 模式对所有存储后端通用**。Redis 用作缓存、session、排行榜等场景都很合适。

## 源码对照

- `examples/tokio-redis/Cargo.toml`
- `examples/tokio-redis/src/main.rs`
