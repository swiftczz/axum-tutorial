# 第 28 章 tokio-redis:内存存储与无类型的值

## 本章你将做出什么

一个连上 Redis 的服务,两条路由都读同一个 key:

```
GET  /   用 State 拿连接池,临时借一条连接
POST /   用自定义提取器,进 handler 前借好连接
```

结构和[第 24 章](24-sqlx-postgres.md)、[第 25 章](25-tokio-postgres.md)完全一致,重合部分不重复讲。

本章的新内容是 **Redis 这类存储的特点**:值是无类型的字节,怎么和 Rust 的类型系统对接;以及它和[第 22 章](22-key-value-store.md)那个自己写的 `HashMap` 相比多了什么。

学完本章你将掌握:

- Redis 相对进程内 `HashMap` 的实际价值
- `AsyncCommands` trait:命令即方法
- **无类型的值如何靠 `FromRedisValue` / `ToRedisArgs` 和 Rust 类型对接**
- 为什么 `conn.set::<&str, &str, ()>(...)` 要写三个 turbofish 参数

## 前置知识

### Redis 相对进程内 HashMap 多了什么

[第 22 章](22-key-value-store.md)用 `Arc<RwLock<HashMap<String, Bytes>>>` 做了一个 KV 存储,功能上和 Redis 的 `GET`/`SET` 一样。那为什么还要引入一个外部服务?

**跨进程共享。** 这是最根本的理由。部署两个实例时,进程内的 `HashMap` 各存各的——用户请求打到不同实例会看到不同数据。Redis 是独立进程,所有实例共享同一份。

**重启不丢。** Redis 支持 RDB 快照和 AOF 日志两种持久化。虽然它是内存数据库,但数据能活过进程重启。

**过期时间。** `SET key value EX 3600` 一小时后自动删除。[第 22 章](22-key-value-store.md#代价与边界)提过那个内存无限增长的问题,Redis 用 TTL 加淘汰策略(LRU/LFU)解决了它。

**原子操作。** `INCR`、`SETNX`、`LPUSH` 这些命令是原子的。[第 23 章](23-todos.md#代价与边界)那个读-改-写竞态,用 `INCR` 一条命令就没有了。

**丰富的数据结构。** 不只是字符串:List(队列)、Set(去重)、Hash(对象)、Sorted Set(排行榜)、Stream(消息流)。很多需求用对了数据结构就是一条命令的事。

**代价**是多了一次网络往返(本地约 0.1ms,跨机房可能几毫秒),以及一个需要运维的组件。

**什么时候不需要 Redis:**单实例部署、数据可以丢、并发不高——那 `HashMap` 更简单也更快。**别为了用而用。**

### AsyncCommands:命令即方法

`redis` crate 把 Redis 的每条命令映射成一个 trait 方法:

```rust
use redis::AsyncCommands;

conn.set("key", "value").await?;
let v: String = conn.get("key").await?;
conn.expire("key", 60).await?;
conn.incr("counter", 1).await?;
```

`AsyncCommands` 是个**扩展 trait**,必须显式导入才能用这些方法——和[第 09 章](09-consume-body-in-extractor-or-middleware.md#collect-与-bodyext)的 `BodyExt`、[第 21 章](21-testing.md#oneshot-与-readycall)的 `ServiceExt` 是同一个套路。

**忘了 `use redis::AsyncCommands` 会报"方法不存在"**,这是新手最常撞的墙。

也可以手动构造命令,用于 trait 没覆盖的场景:

```rust
redis::cmd("SET").arg("key").arg("value").query_async(&mut conn).await?;
```

### 无类型的值怎么和 Rust 对接

**这是本章的核心话题。**

Redis 的值本质上是**字节串**,没有类型信息。存 `42` 和存 `"42"` 在 Redis 里是同一个东西。这和 PostgreSQL 完全不同——那边每列都有明确的 SQL 类型([第 25 章](25-tokio-postgres.md#手动从-row-取值)讲过 `int4` 对应 `i32`)。

`redis` crate 用两个 trait 搭起类型桥梁:

**`ToRedisArgs`** —— Rust 值 → Redis 字节。`String`、`&str`、各种整数、`Vec<u8>`、以及它们的切片都实现了它。

**`FromRedisValue`** —— Redis 字节 → Rust 值。`String`、整数、`bool`、`Vec<T>`、`HashMap<K, V>`、`Option<T>` 都实现了它。

所以取值时**类型由左边的标注决定**:

```rust
let s: String = conn.get("counter").await?;   // 得到 "42"
let n: i64 = conn.get("counter").await?;      // 得到 42
```

同一个 key,标注不同类型就得到不同结果——`redis` crate 会尝试解析。**解析不了才报错**,比如把 `"abc"` 读成 `i64`。

**`Option<T>` 用来处理 key 不存在:**

```rust
let v: Option<String> = conn.get("maybe-missing").await?;
```

不写 `Option` 而 key 不存在时,`String` 的解析会失败(Redis 返回 nil)。**这是很常见的运行时错误来源** —— 本章「代价与边界」会展开。

**存复杂结构要自己序列化。** Redis 不认识 struct,通常的做法是 `serde_json::to_string` 存进去、读出来再 `from_str`。也有 `redis` 的 `json` feature 支持 RedisJSON 模块,但那需要 Redis 装扩展。

### 那三个 turbofish 参数

示例里有这么一行:

```rust
conn.set::<&str, &str, ()>("foo", "bar").await.unwrap();
```

三个类型参数看起来很吓人,拆开就清楚了。`set` 的签名大致是:

```rust
async fn set<K: ToRedisArgs, V: ToRedisArgs, RV: FromRedisValue>(
    &mut self, key: K, value: V
) -> RedisResult<RV>
```

- **`K = &str`** —— key 的类型
- **`V = &str`** —— value 的类型
- **`RV = ()`** —— **返回值**的类型

前两个能从参数推断出来,问题在第三个:`SET` 命令的返回值(Redis 回 `OK`)可以解析成 `String`、`bool`、或者干脆忽略成 `()`。**编译器不知道你要哪个**,所以必须显式指定。

更常见的写法是用类型标注让编译器推断:

```rust
let _: () = conn.set("foo", "bar").await.unwrap();
```

两种等价。示例用 turbofish 是为了把三个参数摆出来给你看。

**这是 `redis` crate 使用中最容易困惑的地方** —— 每个"不关心返回值"的命令都要处理这件事。

### 连接池

结构和[第 25 章](25-tokio-postgres.md#驱动与连接池的分离)一样,`bb8` 加一个 manager。区别是这次 manager 就是 `redis::Client` 本身:

```rust
let client = redis::Client::open("redis://localhost").unwrap();
let pool = bb8::Pool::builder().build(client).await.unwrap();
```

`redis` crate 的 `bb8` feature 让 `Client` 直接实现了 `bb8::ManageConnection`,所以不需要额外的适配 crate([第 25 章](25-tokio-postgres.md)那边需要 `bb8-postgres`)。

**注意 `Client::open` 只是解析 URL,不建立连接。** 真正的连接在 `pool.get()` 时才建。所以 URL 写错了要到第一次用池子才报错——这就是示例在 `main` 里做一次 ping 的原因。

## 项目搭建

### 启动 Redis

```bash
docker run --name axum-tutorial-redis -p 6379:6379 -d redis:7
```

确认能连上:

```bash
docker exec -it axum-tutorial-redis redis-cli ping
```

```
PONG
```

用完清理:

```bash
docker stop axum-tutorial-redis && docker rm axum-tutorial-redis
```

### 创建项目

```bash
cargo new tokio-redis
cd tokio-redis
```

```bash
cargo add axum
cargo add tokio --features full
cargo add bb8
cargo add redis --no-default-features --features tokio-comp,bb8
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`redis` 的 `--no-default-features` 关掉了同步客户端(默认会带上),`tokio-comp` 启用 tokio 异步支持,`bb8` 提供连接池集成。

## 分步构建

### 步骤 1:连上 Redis 并做启动检查

**要做什么**:建立连接池,启动时 ping 一下确认可用。

```rust
// src/main.rs
use axum::{extract::State, http::StatusCode, routing::get, Router};
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

    let app = Router::new()
        .route("/", get(using_connection_pool_extractor))
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

/// Utility function for mapping any error into a `500 Internal Server Error`
/// response.
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
```

**启动检查那个块**做了三件事:写一个值、读回来、断言相等。它比单纯的 `PING` 验证得更彻底——**确认读写链路真的通**,而不只是网络可达。

用块表达式框住是为了让 `conn` 及时归还连接池,[第 22 章](22-key-value-store.md#持锁跨-await能让服务卡死的陷阱)讲过这个习惯。

`.unwrap()` 让连不上时启动失败,[第 24 章](24-sqlx-postgres.md#步骤-1连上数据库用-state-拿池子)讲过为什么这是对的。

**`conn.set::<&str, &str, ()>("foo", "bar")`** 就是前置知识讲的三个 turbofish 参数。

**`let result: String = conn.get("foo")`** —— 类型标注决定了怎么解析返回值。

**注意 `"foo"` 这个 key 是硬编码的**,两条路由都只读它。这个示例演示的是**连接池和提取器的写法**,不是 Redis 的数据操作——和[第 24](24-sqlx-postgres.md)、[25 章](25-tokio-postgres.md)那两个只跑 `select 1+1` 的示例一样。

> **更好的写法**:`bb8::Pool::builder()` 又一次没有任何配置。[第 24 章](24-sqlx-postgres.md#为什么需要连接池)讲过 `max_size` 和获取超时为什么必须设。另外连接 URL `"redis://localhost"` 硬编码在源码里,应该像[第 24 章](24-sqlx-postgres.md#步骤-1连上数据库用-state-拿池子)那样从 `REDIS_URL` 环境变量读——本章示例连那一步都省了。

**跑起来看**:

```bash
cargo run
```

```
DEBUG tokio_redis: connecting to redis
DEBUG tokio_redis: successfully connected to redis and pinged it
DEBUG tokio_redis: listening on 127.0.0.1:3000
```

```bash
curl http://127.0.0.1:3000/
```

```
bar
```

**这个 `bar` 是启动检查写进去的。** 从 redis-cli 确认:

```bash
docker exec -it axum-tutorial-redis redis-cli get foo
```

```
"bar"
```

**验证跨进程共享。** 用 redis-cli 改掉这个值:

```bash
docker exec -it axum-tutorial-redis redis-cli set foo "changed from cli"
curl http://127.0.0.1:3000/
```

```
changed from cli
```

**服务读到了外部进程写的数据** —— 这是[第 22 章](22-key-value-store.md)那个进程内 `HashMap` 做不到的。

**验证类型解析。** 存一个数字:

```bash
docker exec -it axum-tutorial-redis redis-cli set foo 42
curl http://127.0.0.1:3000/
```

```
42
```

`String` 类型能读出来。但如果把 handler 里的类型标注临时改成 `i64`:

```rust
    let result: i64 = conn.get("foo").await.map_err(internal_error)?;
    Ok(result.to_string())
```

也能工作——因为 `"42"` 能解析成 `i64`。**再把值改回文本试试:**

```bash
docker exec -it axum-tutorial-redis redis-cli set foo "bar"
curl -s http://127.0.0.1:3000/
```

```
Response was of incompatible type - TypeError: "Could not convert from string." (response was string-data('"bar"'))
```

**同一份数据、同一个 key,类型标注不同结果完全不同。** 这就是前置知识讲的"值无类型"的直接后果——**类型正确性完全靠你自己保证,Redis 帮不了你**。

改回 `String`,并把值设回 `bar`:

```bash
docker exec -it axum-tutorial-redis redis-cli set foo bar
```

**验证 key 不存在的行为。** 删掉它:

```bash
docker exec -it axum-tutorial-redis redis-cli del foo
curl -s http://127.0.0.1:3000/
```

```
Response was of incompatible type - TypeError: "Response type not string compatible." (response was nil)
```

**`500` 错误,而不是 `404`。** 因为 `String` 无法从 nil 解析。正确做法是用 `Option<String>`,练习题 1 会做。

恢复:

```bash
docker exec -it axum-tutorial-redis redis-cli set foo bar
```

### 步骤 2:自定义连接提取器

**要做什么**:和[第 25 章](25-tokio-postgres.md#步骤-2自定义连接提取器)一样的提取器。

```rust
// src/main.rs
use axum::{
    extract::{FromRef, FromRequestParts, State},        // 改动
    http::{request::Parts, StatusCode},                 // 改动
    routing::get,
    Router,
};
use redis::AsyncCommands;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // ...(连接池和启动检查不变)

    // 改动:挂上 POST
    let app = Router::new()
        .route(
            "/",
            get(using_connection_pool_extractor).post(using_connection_extractor),
        )
        .with_state(pool);

    // ...
}

// ...(ConnectionPool、using_connection_pool_extractor 不变)

// 新增:we can also write a custom extractor that grabs a connection from the pool
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

// 新增
async fn using_connection_extractor(
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<String, (StatusCode, String)> {
    let result: String = conn.get("foo").await.map_err(internal_error)?;

    Ok(result)
}

// ...(internal_error 不变)
```

**这段和[第 25 章](25-tokio-postgres.md#步骤-2自定义连接提取器)几乎一字不差**,只是把 `PostgresConnectionManager<NoTls>` 换成了 `redis::Client`。`FromRef` 的原理见[第 24 章](24-sqlx-postgres.md#fromref提取器只依赖状态的一部分),`get_owned()` 的理由见[第 25 章](25-tokio-postgres.md#get_owned-与生命周期)。

**`DatabaseConnection(mut conn)` 带 `mut`** —— `AsyncCommands` 的方法要求 `&mut self`,因为它们要往连接上写命令。这和[第 25 章](25-tokio-postgres.md#步骤-2自定义连接提取器)的 `tokio-postgres` 不同(那边是 `&self`)。

**跑起来看**:

```bash
cargo run
```

```bash
curl http://127.0.0.1:3000/
curl -X POST http://127.0.0.1:3000/
```

```
bar
bar
```

两条路由行为一致。[第 24 章](24-sqlx-postgres.md#两种取连接的方式)对比过两种取连接方式,结论不变:**默认用方式一。**

不过对 Redis 来说,方式二(先借连接)的场景比数据库多一些——**Redis 的事务(`MULTI`/`EXEC`)、`WATCH` 乐观锁、以及 pipeline 都要求多个命令在同一条连接上**。这时提取器方式反而更自然。

## 完整代码

`src/main.rs`:

```rust
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
```

## 代价与边界

**这个示例几乎没演示 Redis 的能力。** 一个硬编码的 key、只有 `GET`,没有过期、没有原子操作、没有任何数据结构。想真正学 Redis 得看它自己的文档——本章的价值在于**连接池和提取器的写法**,那部分和[第 24](24-sqlx-postgres.md)、[25 章](25-tokio-postgres.md)是通用的。

**key 不存在会变成 500。** 步骤 1 验证过。这是 Redis 应用里最常见的错误来源:`String` 无法从 nil 解析。**永远用 `Option<T>` 接收可能不存在的 key**,然后自己决定返回 `404` 还是默认值。

**类型解析错误只在运行时暴露。** 同一个 key 存了什么类型的数据,只能靠约定和文档保证。这比[第 25 章](25-tokio-postgres.md#手动从-row-取值)的 `try_get` 更危险——那至少还有数据库的列类型兜底。**缓解办法是给每类数据封装读写函数**,把类型集中在一处。

**没有过期时间。** 示例写进去的 key 永不过期。缓存场景必须设 TTL,否则内存只增不减——这和[第 22 章](22-key-value-store.md#代价与边界)那个问题一样,只是从进程内存挪到了 Redis 内存。

**连接池未配置、URL 硬编码。** 步骤 1 的「更好的写法」已经指出。

**没有考虑 Redis 不可用的降级。** 示例里 Redis 挂了就返回 500。真实的缓存场景通常应该**降级到数据源**(缓存拿不到就查数据库),而不是让整个请求失败。这个判断取决于 Redis 在你的架构里是"缓存"还是"数据源"。

**单点 Redis。** 生产环境通常用主从或 Cluster。`redis` crate 有 `cluster-async` feature 支持,但连接方式和本章不同。

## 动手练习

**1. 用 `Option<String>` 正确处理 key 不存在。**

提示:handler 返回类型改成 `Result<String, (StatusCode, String)>`,取值写成 `let result: Option<String> = conn.get("foo").await...`,然后 `result.ok_or((StatusCode::NOT_FOUND, "key not found".into()))`。验证:`redis-cli del foo` 之后请求应该得到 `404` 而不是 `500`。**这是本章最该改的一处。**

**2. 做一个真正的 KV 接口,带过期时间。**

提示:改成 `GET /{key}` 和 `POST /{key}`,用 `Path<String>` 取 key。写入时用 `conn.set_ex(key, value, 60)` 设 60 秒过期。验证:写入后立刻读能拿到,`redis-cli ttl <key>` 能看到剩余秒数。这一步做完,[第 22 章](22-key-value-store.md)那个内存 KV 就有了跨进程、可过期的版本。

**3. 用 `INCR` 做一个计数器,体会原子操作。**

提示:`GET /count` 里用 `let n: i64 = conn.incr("counter", 1).await?`。然后并发打 100 次请求,确认最终值正好是 100:

```bash
for i in $(seq 100); do curl -s -o /dev/null http://127.0.0.1:3000/count & done; wait
docker exec -it axum-tutorial-redis redis-cli get counter
```

对比[第 23 章](23-todos.md#代价与边界)那个读-改-写竞态——**如果用"先 GET 再 SET"实现,并发下结果一定小于 100**。练习题可以两种都写一遍对比。

## 小结与下一章预告

Redis 相对进程内 `HashMap` 多了五样东西:跨进程共享、持久化、过期时间、原子操作、丰富的数据结构。代价是一次网络往返和一个要运维的组件——**单实例、数据可丢、并发不高的场景别为了用而用**。

`AsyncCommands` 把每条 Redis 命令映射成方法,是扩展 trait,**必须 `use redis::AsyncCommands`** 才能调用。

本章最重要的是**无类型值的对接问题**:Redis 的值就是字节,类型完全由你取值时的标注决定。同一个 key 标注 `String` 和 `i64` 结果不同,标注错了运行时才失败。**key 不存在时必须用 `Option<T>` 接收**,否则会变成 500 而不是 404——步骤 1 验证过这个陷阱。

那三个 turbofish 参数 `set::<K, V, RV>` 里,前两个能推断,第三个是**返回值类型**——因为 `SET` 的回复既能解析成 `String` 也能忽略成 `()`,编译器不知道你要哪个。

连接池部分和[第 25 章](25-tokio-postgres.md)完全同构,`redis::Client` 直接实现了 `bb8::ManageConnection`,不需要额外的适配 crate。

[第 29 章 mongodb](29-mongodb.md) 是「状态与数据库」阶段的最后一章,也是这批示例里**唯一一个完整 CRUD**。MongoDB 是文档数据库——没有固定 schema,文档就是 JSON 结构。你会看到它和 Rust 类型系统的对接方式(靠 serde),`_id` 这个特殊字段怎么处理,以及一个 axum 层面的细节:**为什么它把 `Collection<T>` 直接当 `State`,而不是像前面那样传连接池**。
