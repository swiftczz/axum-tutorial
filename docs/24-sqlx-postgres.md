# 第 24 章 sqlx-postgres:连接池与数据库提取器

## 本章你将做出什么

一个连上真实 PostgreSQL 的服务,两条路由查同一句 SQL,但**取连接的方式不同**:

```
GET  /   用 State 拿整个连接池,查询时临时借一条连接
POST /   用自定义提取器,进 handler 之前就借好一条连接
```

前面所有章节的数据都在内存里,重启即失。这一章接上真正的数据库——数据终于能活过进程。

学完本章你将掌握:

- 连接池是什么,为什么不能每次请求新建连接
- `sqlx` 的基本用法,以及它区别于 ORM 的地方
- **`FromRef`:让一个提取器只依赖状态的某一部分**
- 两种取连接方式的实际区别,以及什么时候该选哪个
- 用 Docker 起一个 PostgreSQL 做实验

## 前置知识

### 为什么需要连接池

数据库连接不是廉价资源。建立一条 PostgreSQL 连接要:TCP 三次握手 → TLS 协商(如果启用)→ 认证 → 服务端 fork 一个进程或分配一个后端。**整个过程通常是几十毫秒级**,而一次简单查询可能只要 1 毫秒。

更要命的是 PostgreSQL 的连接数上限。默认 `max_connections = 100`,而每条连接在服务端都占几 MB 内存。一个每秒处理 1000 请求的服务,如果每个请求新建连接,数据库会立刻被打垮。

**连接池**预先建立若干条连接并复用它们:

- 请求需要查询时,从池里**借**一条
- 用完**还**回去,不关闭
- 池子有上限,超出时请求排队等待

```rust
let pool = PgPoolOptions::new()
    .max_connections(5)
    .acquire_timeout(Duration::from_secs(3))
    .connect(&db_connection_str)
    .await?;
```

**`max_connections` 怎么定。** 常见的误解是"越大越好"。实际上它应该满足:

```
所有应用实例的 max_connections 之和 < 数据库的 max_connections
```

部署了 10 个实例、每个设 20,就是 200 条连接——超过默认上限,数据库会开始拒绝连接。**这是扩容时最常见的事故之一**。

**`acquire_timeout` 是必需的。** 池子满了之后请求会排队,没有超时就会无限等待——请求堆积、内存增长、最后整个服务卡死。设一个超时,让请求快速失败(`503`)比慢慢拖死好,这和[第 22 章](22-key-value-store.md#handleerrorlayer中间件也会出错) `load_shed` 的思路一致。

**`PgPool` 是廉价 `Clone` 的。** 它内部是 `Arc`,克隆只增加引用计数。所以可以直接当 `State` 用,不需要再包一层 `Arc`——这和[第 05 章](05-templates-minijinja.md#state-与-with_state)那个 `Arc<AppState>` 不同。

### sqlx:不是 ORM

Rust 的数据库库分两派:

- **ORM**(diesel、sea-orm)—— 用 Rust 的类型和方法构造查询,`users.filter(name.eq("x")).load()`
- **SQL 优先**(sqlx、tokio-postgres)—— 直接写 SQL 字符串

`sqlx` 属于后者。它的定位是"异步 SQL 工具包",不做对象关系映射,你写什么 SQL 就执行什么。

**它最出名的特性是编译期 SQL 校验**:

```rust
sqlx::query!("select name from users where id = $1", user_id)
```

带 `!` 的宏会在**编译时**连上数据库、校验 SQL 语法、检查表和列是否存在、推断返回类型。SQL 写错了编译不过。

**但本章示例用的是不带 `!` 的 `query_scalar`** —— 那是运行时版本,不做编译期校验。原因很实际:`query!` 宏要求编译时能连上数据库(或者有 `.sqlx` 离线缓存文件),这对示例来说门槛太高。

> **更好的写法**:生产项目应该用 `sqlx::query!`/`query_as!` 这些**带 `!` 的宏**,让 SQL 错误在编译期暴露——这是选择 sqlx 而非其它库的主要理由。代价是 `cargo build` 时需要数据库可达,或者提前用 `cargo sqlx prepare` 生成 `.sqlx` 离线缓存并提交进 git(CI 环境通常这么做)。示例用运行时版本是为了让你不配数据库也能编译通过,别把这当成推荐用法。

**参数占位符是 `$1`、`$2`**(PostgreSQL 风格),MySQL 是 `?`。**永远用占位符,不要拼接字符串** —— 拼接就是 SQL 注入。

### FromRef:提取器只依赖状态的一部分

这是本章最值得学的机制。

自定义提取器要访问应用状态时,面临一个设计问题:**它应该要求整个 `AppState`,还是只要求自己需要的那部分?**

要求整个 `AppState` 的问题是**耦合**:这个提取器就只能用在状态类型恰好是 `AppState` 的应用里。换个项目、或者状态结构变了,就用不了。

`FromRef` 解决这个问题:

```rust
impl<S> FromRequestParts<S> for DatabaseConnection
where
    PgPool: FromRef<S>,      // 只要求:能从 S 里取出 PgPool
    S: Send + Sync,
{
    async fn from_request_parts(_parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let pool = PgPool::from_ref(state);
        // ...
    }
}
```

约束读作:**"我不关心你的状态是什么类型,只要能从里面拿到一个 `PgPool` 就行"**。

`FromRef` 的定义很简单:

```rust
pub trait FromRef<T> {
    fn from_ref(input: &T) -> Self;
}
```

当状态就是 `PgPool` 本身时(本章的情况),axum 有一个 blanket impl 让 `T: FromRef<T>` 自动成立——所以什么都不用写。

状态是个复合结构时,用派生宏:

```rust
#[derive(Clone, FromRef)]
struct AppState {
    pool: PgPool,
    config: Arc<Config>,
}
```

派生宏为每个字段生成 `FromRef` 实现,于是 `State<PgPool>`、`State<Arc<Config>>`、`State<AppState>` 三种提取器都能用。**这让不同的 handler 各取所需,而不是都被迫接收整个状态。**

这是编写可复用提取器的标准做法,写库时尤其重要。

### 两种取连接的方式

本章两条路由演示了两种模式,区别在于**连接在什么时候被借出**。

**方式一:传池子,用时再借。**

```rust
async fn using_connection_pool_extractor(State(pool): State<PgPool>) -> ... {
    sqlx::query_scalar("select ...").fetch_one(&pool).await
}
```

`fetch_one(&pool)` 内部会临时借一条连接,查完立刻归还。**连接的占用时间 = 查询时间。**

**方式二:提取器里先借好。**

```rust
async fn using_connection_extractor(DatabaseConnection(mut conn): DatabaseConnection) -> ... {
    sqlx::query_scalar("select ...").fetch_one(&mut *conn).await
}
```

进 handler 之前就借好一条,**整个 handler 执行期间都占着它**,直到 handler 返回才归还。

**这个区别很重要。** 假设 handler 是:

```rust
async fn handler(DatabaseConnection(mut conn): DatabaseConnection) -> ... {
    let user = query("select ...").fetch_one(&mut *conn).await?;   // 1ms
    let result = call_external_api(&user).await?;                  // 500ms ← 连接被白占
    query("update ...").execute(&mut *conn).await?;                // 1ms
}
```

**那 500ms 里连接被白白占着。** 池子只有 5 条连接,5 个这样的请求就把池子占满了,第 6 个请求开始排队——而实际上数据库一直是闲的。

所以:

| | 方式一(传池子) | 方式二(先借连接) |
|---|---|---|
| 连接占用时间 | 仅查询期间 | 整个 handler |
| 多次查询是否同一连接 | 不保证 | 保证 |
| 能否用事务 | 需要显式 `begin()` | 天然可以 |
| 适合 | **大多数场景** | 需要事务或会话状态 |

**默认用方式一。** 只有确实需要"多次操作在同一条连接上"时才用方式二——最典型的就是事务(`BEGIN` 和 `COMMIT` 必须在同一连接上)。

官方注释也是这么说的:"which setup is appropriate depends on your application"。

## 项目搭建

### 启动 PostgreSQL

本章开始需要真实数据库。用 Docker 最省事:

```bash
docker run --name axum-tutorial-pg \
  -e POSTGRES_PASSWORD=password \
  -p 5432:5432 \
  -d postgres:16
```

确认起来了:

```bash
docker exec -it axum-tutorial-pg psql -U postgres -c 'select version();'
```

用完清理:

```bash
docker stop axum-tutorial-pg && docker rm axum-tutorial-pg
```

**这个容器第 25、26、27 章还会用到**,可以先留着。

### 创建项目

```bash
cargo new sqlx-postgres
cd sqlx-postgres
```

```bash
cargo add axum
cargo add tokio --features full
cargo add sqlx --features runtime-tokio,any,postgres
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`sqlx` 的三个 feature:`runtime-tokio` 指定异步运行时,`postgres` 启用 PostgreSQL 驱动,`any` 提供数据库无关的抽象层(本例其实没用到,但官方示例带着它)。

**注意 sqlx 编译较慢**,第一次 `cargo build` 可能要一两分钟。

## 分步构建

### 步骤 1:连上数据库,用 State 拿池子

**要做什么**:建立连接池,写第一条查询路由。

```rust
// src/main.rs
use axum::{extract::State, http::StatusCode, routing::get, Router};
use sqlx::postgres::{PgPool, PgPoolOptions};
use std::time::Duration;
use tokio::net::TcpListener;
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

    let db_connection_str = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://postgres:password@localhost".to_string());

    // set up connection pool
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

// we can extract the connection pool with `State`
async fn using_connection_pool_extractor(
    State(pool): State<PgPool>,
) -> Result<String, (StatusCode, String)> {
    sqlx::query_scalar("select 'hello world from pg'")
        .fetch_one(&pool)
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
```

**`std::env::var("DATABASE_URL")`** 从环境变量读连接串,读不到用默认值。**连接串里有密码,绝不能硬编码进源码**——[第 22 章](22-key-value-store.md#步骤-4管理接口与-bearer-认证)那个硬编码 token 就被批评过。这里的默认值只是为了示例能跑,生产环境应该只从环境变量读、读不到就直接退出。

**`.expect("can't connect to database")`** 在启动时失败就 panic。这是对的——**连不上数据库的服务没有存在意义**,快速失败比带病启动好,容器编排系统会看到进程退出并重启它。

**`.with_state(pool)`** 直接把 `PgPool` 当状态,没有包 `Arc`。前置知识讲过原因:它内部已经是 `Arc` 了。

**`query_scalar` 用于返回单个值的查询。** sqlx 有一族这样的方法:

- `query` — 返回原始行,自己取字段
- `query_scalar` — 返回单列单值
- `query_as` — 反序列化成 struct(要求实现 `FromRow`)

**`.fetch_one(&pool)`** 执行并要求恰好一行。相关的还有 `fetch_optional`(0 或 1 行)、`fetch_all`(全部)、`execute`(不关心返回,用于 INSERT/UPDATE)。

**`internal_error` 这个泛型辅助函数**把任意错误转成 `(StatusCode, String)`。[第 15 章](15-anyhow-error-response.md#步骤-3一个泛型-from-让-接住一切)那个泛型 `From` 是同样的思路,只是这里更直接。

> **更好的写法**:`err.to_string()` 被直接发给了客户端,这会泄露数据库的错误细节——表名、列名、约束名,甚至部分 SQL。[第 14 章](14-error-handling.md#4xx-与-5xx-决定了两件事)讲过 5xx 不该暴露内部信息。正确做法是响应固定文本、详情用 `tracing::error!` 记进日志。示例这么写是为了让你在 curl 里直接看到数据库报了什么错,省去配日志的步骤。

**跑起来看**:

确保 PostgreSQL 在跑,然后:

```bash
cargo run
```

```
listening on 127.0.0.1:3000
```

```bash
curl http://127.0.0.1:3000/
```

```
hello world from pg
```

**这句话是数据库返回的**,不是 Rust 里的字符串常量。

**验证连接失败的行为。** 停掉数据库:

```bash
docker stop axum-tutorial-pg
cargo run
```

```
thread 'main' panicked at src/main.rs:...:
can't connect to database: PoolTimedOut
```

**启动直接失败**,这是 `.expect()` 的效果。重新启动数据库:

```bash
docker start axum-tutorial-pg
```

**再验证一次运行期的查询错误。** 临时把 SQL 改成错的:

```rust
    sqlx::query_scalar("select * from no_such_table")
```

```bash
curl -i -s http://127.0.0.1:3000/ | head -1
curl -s http://127.0.0.1:3000/
```

```
HTTP/1.1 500 Internal Server Error
error returned from database: relation "no_such_table" does not exist
```

**数据库的错误信息原样返回给了客户端** —— 这正是上面「更好的写法」指出的问题:表名泄露了。如果用 `sqlx::query!` 宏,这个错误会在**编译期**就被发现。改回正确的 SQL。

### 步骤 2:自定义连接提取器

**要做什么**:写一个提取器,在进 handler 之前就从池里借好一条连接。

```rust
// src/main.rs
use axum::{
    extract::{FromRef, FromRequestParts, State},        // 改动:加上 FromRef、FromRequestParts
    http::{request::Parts, StatusCode},                 // 改动:加上 request::Parts
    routing::get,
    Router,
};
// ...(其余 use 不变)

#[tokio::main]
async fn main() {
    // ...(连接池不变)

    // 改动:同一路径挂上 POST
    let app = Router::new()
        .route(
            "/",
            get(using_connection_pool_extractor).post(using_connection_extractor),
        )
        .with_state(pool);

    // ...(listener 和 serve 不变)
}

// ...(using_connection_pool_extractor 不变)

// 新增:we can also write a custom extractor that grabs a connection from the pool
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

// 新增
async fn using_connection_extractor(
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<String, (StatusCode, String)> {
    sqlx::query_scalar("select 'hello world from pg'")
        .fetch_one(&mut *conn)
        .await
        .map_err(internal_error)
}

// ...(internal_error 不变)
```

自定义提取器的套路[第 08 章](08-parse-body-based-on-content-type.md#步骤-2实现-jsonorform)讲过,这里只看新东西。

**实现的是 `FromRequestParts` 而不是 `FromRequest`。** 因为它根本不看请求体——参数名写成 `_parts` 就是这个意思,连请求信息都不需要。这也意味着它可以和 `Json` 等 body 提取器同时使用,而且不必放最后([第 03 章](03-form.md#提取器axum-的核心机制)讲过这个分类)。

**`PgPool: FromRef<S>`** 是前置知识讲的关键约束:不要求状态**就是** `PgPool`,只要求**能从状态里取出** `PgPool`。

本章的状态就是 `PgPool` 本身,靠 blanket impl 自动满足。但这个提取器同样能用在状态是 `AppState { pool, config }` 的项目里——**只要那个 `AppState` 派生了 `FromRef`**。

**`pool.acquire()`** 从池里借一条连接,返回 `PoolConnection`。这个类型实现了 `Drop`,**离开作用域时自动归还** —— 所以不需要手动 release,忘不掉。

**`fetch_one(&mut *conn)`** 里的 `&mut *conn` 是双重解引用:`conn` 是 `PoolConnection`,`*conn` 通过 `DerefMut` 得到底层的 `PgConnection`,再取可变引用。sqlx 的执行器接口要求 `&mut PgConnection`。

写成 `&mut conn` 编译不过,因为类型对不上。**这是 sqlx 使用中常见的困惑点。**

**跑起来看**:

```bash
cargo run
```

两条路由行为一致:

```bash
curl http://127.0.0.1:3000/
curl -X POST http://127.0.0.1:3000/
```

```
hello world from pg
hello world from pg
```

**验证连接池的上限。** 临时把 `using_connection_extractor` 改成占住连接不放:

```rust
async fn using_connection_extractor(
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<String, (StatusCode, String)> {
    tokio::time::sleep(Duration::from_secs(10)).await;    // 临时:占着连接睡 10 秒
    sqlx::query_scalar("select 'hello world from pg'")
        .fetch_one(&mut *conn)
        .await
        .map_err(internal_error)
}
```

`max_connections` 是 5,所以并发 6 个请求时第 6 个应该拿不到连接:

```bash
cargo run
```

```bash
for i in 1 2 3 4 5 6; do curl -s -o /dev/null -w "%{http_code} " -X POST http://127.0.0.1:3000/ & done; wait; echo
```

```
500 500 500 500 500 500
```

等一会儿再看,前 5 个是 10 秒后成功、第 6 个是 3 秒后失败(`acquire_timeout`)。可以分开测得更清楚:

```bash
# 先并发占满 5 条
for i in 1 2 3 4 5; do curl -s -o /dev/null -X POST http://127.0.0.1:3000/ & done
sleep 1
# 第 6 个
curl -s http://127.0.0.1:3000/
```

第 6 个请求(走的是 `GET`,方式一)也会失败:

```
pool timed out while waiting for an open connection
```

**这演示了前置知识说的问题:方式二占着连接不放,会把池子耗尽,连不需要长时间操作的请求也被拖累。** 而且 `acquire_timeout` 在这里起了作用——3 秒后失败,而不是无限等待。

注意这里的 `sleep` 是在持有**数据库连接**时,不是持有锁([第 22 章](22-key-value-store.md#持锁跨-await能让服务卡死的陷阱)那个)。数据库连接是 `Send` 的,所以编译没问题——**编译器帮不了你,只能靠理解**。

改回去。

## 完整代码

`src/main.rs`:

```rust
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
```

## 代价与边界

**`max_connections(5)` 这个数字只适合示例。** 生产环境要按前置知识那条公式算:所有实例之和不能超过数据库上限。而且要考虑到 migration 工具、后台任务、运维手动连接也会占用连接数。

**没有 `min_connections`。** 默认池子是懒加载的,第一个请求才建连接,所以**冷启动后的头几个请求会慢**。生产环境通常设 `min_connections` 预热。

**没有健康检查。** 数据库重启或网络抖动后,池里可能残留失效连接。sqlx 有 `test_before_acquire`(默认开启)会在借出前 ping 一下,但这会给每次借用增加一次往返。高性能场景可以关掉它并配合 `max_lifetime` 定期回收连接。

**错误信息泄露。** 上面「更好的写法」已经指出。

**没有超时控制。** 一条慢查询会一直占着连接。生产环境应该在数据库侧设 `statement_timeout`,或者用 `tokio::time::timeout` 包住查询。

**方式二会放大连接占用。** 步骤 2 验证过。**默认用方式一**,只在需要事务时用方式二。

**用了运行时 SQL 而非编译期宏。** 这放弃了 sqlx 最大的优势。生产项目应该用 `query!`,配合 `cargo sqlx prepare` 生成离线缓存。

## 动手练习

**1. 建一张真实的表,做一次完整的增查。**

提示:

```bash
docker exec -it axum-tutorial-pg psql -U postgres -c \
  'create table todos (id serial primary key, text text not null, completed bool not null default false);'
```

然后加两条路由:`POST /todos` 插入(用 `sqlx::query` 加 `.bind(text)` 传参),`GET /todos` 查全部(用 `query_as` 反序列化成 struct,需要给 struct 派生 `sqlx::FromRow`)。这一步做完,[第 23 章](23-todos.md)那个内存版 todos 就有了持久化版本。

**2. 把错误信息从响应里挪走。**

提示:改 `internal_error`,用 `tracing::error!(?err, "database error")` 记日志,响应固定返回 `"Internal Server Error"`。做完对比一下 curl 输出和终端日志——这是[第 14 章](14-error-handling.md)那套原则的直接应用。

**3. 用 `FromRef` 派生宏支持复合状态。**

提示:定义 `#[derive(Clone, FromRef)] struct AppState { pool: PgPool, greeting: String }`,`with_state(AppState { ... })`。你会发现 `DatabaseConnection` 提取器**一个字都不用改**就能工作,而且还能同时写 `State(greeting): State<String>` 的 handler。这就是 `FromRef` 的价值。

## 小结与下一章预告

连接池解决的是"建连接很贵、数据库连接数有限"这两个问题。`max_connections` 要按"所有实例之和不超过数据库上限"来定,`acquire_timeout` 必须设——池子满了要让请求快速失败,而不是无限排队拖死服务。`PgPool` 内部是 `Arc`,可以直接当 `State`,不用再包一层。

sqlx 是 SQL 优先而非 ORM,最大的卖点是**带 `!` 的宏能在编译期校验 SQL**。本章示例用的是运行时版本,放弃了这个优势——生产项目该用 `query!` 配合 `cargo sqlx prepare`。

**`FromRef` 让提取器只依赖状态的一部分**:约束写成 `PgPool: FromRef<S>` 而不是 `S = PgPool`,提取器就能用在任何"能取出 `PgPool`"的应用里。复合状态用 `#[derive(FromRef)]` 一行搞定。这是写可复用提取器的标准做法。

两种取连接的方式差别在占用时长:传池子只在查询期间占用,先借连接则整个 handler 都占着。本章验证过后者会耗尽连接池——**默认用前者,只在需要事务时用后者**。

本章用 sqlx 连 PostgreSQL。[第 25 章 tokio-postgres](25-tokio-postgres.md) 换一套组合:`tokio-postgres` 驱动加 `bb8` 连接池。代码结构几乎一样,所以那一章会以**对比**为主线——两套库在类型设计、参数绑定、错误处理上各有什么取舍,以及为什么 axum 官方要同时提供这两个示例。
