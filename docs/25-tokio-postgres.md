# 第 25 章 tokio-postgres:换一套驱动与连接池

## 本章你将做出什么

和[第 24 章](24-sqlx-postgres.md)几乎一样的服务——两条路由、两种取连接方式、同一个 PostgreSQL——但底层换成 `tokio-postgres` 驱动加 `bb8` 连接池。

**本章以对比为主线。** 结构和上一章重合的部分不再重复讲,重点是两套方案的实际差别:类型设计、参数绑定、结果取值、以及为什么 axum 官方要同时提供这两个示例。

学完本章你将掌握:

- `tokio-postgres` 与 `sqlx` 的定位差别
- 驱动与连接池分离的设计,以及 `bb8` 的角色
- 手动从 `Row` 取值:`try_get` 与类型推断
- `get_owned` 与生命周期:为什么提取器里不能用 `get()`
- 选型时该看什么

## 前置知识

### 驱动与连接池的分离

[第 24 章](24-sqlx-postgres.md#为什么需要连接池)的 sqlx 是**一体化**的:驱动、连接池、查询构造、类型映射全在一个 crate 里,`PgPoolOptions` 直接就能用。

`tokio-postgres` 走的是另一条路——**它只做驱动**,负责协议实现和查询执行,不提供连接池。想要池化就得自己配一个:

| 组件 | 本章用的 | 职责 |
|---|---|---|
| 驱动 | `tokio-postgres` | PostgreSQL 协议、查询执行 |
| 连接池 | `bb8` | 通用异步连接池 |
| 适配器 | `bb8-postgres` | 把 `tokio-postgres` 接到 `bb8` 上 |

**`bb8` 是通用的**——它不知道 PostgreSQL 的存在,只定义了"如何创建、检查、回收一个连接"这套接口(`ManageConnection` trait)。Redis、MongoDB、甚至自定义的 TCP 服务都能用它池化。

这种分离的好处是**换组件不用换全套**:想从 `bb8` 换成 `deadpool`(另一个流行的异步连接池),只需要改连接池那一层。

代价是**配置更零碎**:

```rust
let manager = PostgresConnectionManager::new_from_stringlike("host=localhost user=postgres", NoTls).unwrap();
let pool = Pool::builder().build(manager).await.unwrap();
```

两步——先造 manager(告诉它怎么建连接),再造 pool。对比 sqlx 的一步到位:

```rust
let pool = PgPoolOptions::new().max_connections(5).connect(&url).await?;
```

### 连接串格式不同

这是个容易踩的小坑。

**sqlx 用 URL 格式**:

```
postgres://postgres:password@localhost/dbname
```

**tokio-postgres 用键值对格式**:

```
host=localhost user=postgres password=xxx dbname=mydb
```

后者是 libpq(PostgreSQL 官方 C 客户端)的传统格式。`tokio-postgres` 其实两种都支持(`new_from_stringlike` 会自动判断),但官方示例用的是键值对格式。

**注意本章示例的连接串里没有密码**:

```rust
"host=localhost user=postgres"
```

而[第 24 章](24-sqlx-postgres.md#启动-postgresql)那个 Docker 容器设了 `POSTGRES_PASSWORD=password`。**所以直接跑会认证失败** —— 项目搭建那一节会说明怎么处理。

### NoTls 是什么

```rust
PostgresConnectionManager::new_from_stringlike("...", NoTls)
```

第二个参数是 **TLS 配置**。`tokio-postgres` 把 TLS 做成了可插拔的:

- `NoTls` — 明文连接
- `tokio-postgres-rustls` / `tokio-postgres-native-tls` — 加密连接

**本地开发用 `NoTls` 没问题,但生产环境连接远程数据库必须启用 TLS。** 数据库流量里有你的全部数据和凭据,明文传输在公网上等于裸奔。云数据库(RDS、Cloud SQL)通常强制要求 TLS。

sqlx 的处理方式不同:它把 TLS 做成 feature(`tls-rustls`),连接串里写 `?sslmode=require` 就启用。

### 手动从 Row 取值

这是两套库在使用体验上最明显的差别。

**sqlx 有 `query_scalar`、`query_as`** 帮你做类型转换:

```rust
let s: String = sqlx::query_scalar("select 'hello'").fetch_one(&pool).await?;
```

**tokio-postgres 只给你 `Row`**,得自己取:

```rust
let row = conn.query_one("select 1 + 1", &[]).await?;
let two: i32 = row.try_get(0)?;
```

`try_get` 接受列索引(`0`)或列名(`"count"`),返回 `Result`。**类型靠左边的标注推断** —— 写成 `let two: i64` 就会运行时失败,因为 PostgreSQL 的 `1 + 1` 结果是 `int4`(对应 Rust 的 `i32`)。

**这是 tokio-postgres 最容易出错的地方:类型不匹配只有运行时才发现。** PostgreSQL 的类型和 Rust 类型有一套映射关系,记不住就得查文档:

| PostgreSQL | Rust |
|---|---|
| `int2` / `smallint` | `i16` |
| `int4` / `integer` | `i32` |
| `int8` / `bigint` | `i64` |
| `text` / `varchar` | `String` / `&str` |
| `bool` | `bool` |
| `timestamptz` | `chrono::DateTime<Utc>`(需 feature) |
| `uuid` | `uuid::Uuid`(需 feature) |

还有个 `get()` 方法(不带 `try_`),它在类型不匹配时直接 panic。**永远用 `try_get`**。

### 参数绑定的差别

**sqlx 用链式 `.bind()`**:

```rust
sqlx::query("select * from users where id = $1 and name = $2")
    .bind(user_id)
    .bind(name)
    .fetch_one(&pool).await?
```

**tokio-postgres 用切片**:

```rust
conn.query_one("select * from users where id = $1 and name = $2", &[&user_id, &name]).await?
```

那个 `&[]` 就是参数列表,本章示例的查询没有参数所以是空的。

注意**切片里每个元素都要再加一层 `&`** —— 因为参数类型是 `&[&(dyn ToSql + Sync)]`,即"trait 对象引用的切片"。写成 `&[user_id]` 编译不过,这是常见困惑点。

sqlx 的 `.bind()` 更直观,但每次调用都消耗并返回 query 对象,链式写法在参数多时会很长。两种风格各有拥趸。

### get_owned 与生命周期

本章提取器里有个细节值得注意:

```rust
struct DatabaseConnection(PooledConnection<'static, PostgresConnectionManager<NoTls>>);

let conn = pool.get_owned().await.map_err(internal_error)?;
```

**`bb8` 的 `get()` 返回的连接借用了池子**,类型是 `PooledConnection<'a, M>`,生命周期 `'a` 绑定到 `&'a self`。这在普通函数里没问题,但**提取器要把连接存进一个 struct 返回出去**,借用就活不过 `from_request_parts` 的作用域了。

`get_owned()` 返回 `PooledConnection<'static, M>` —— 它内部持有池子的 `Arc` 克隆,不借用任何东西,所以能自由移动。

**对比 sqlx**:`pool.acquire()` 返回的 `PoolConnection<DB>` 本身就没有生命周期参数,不存在这个问题。**这是 sqlx 一体化设计带来的便利。**

## 项目搭建

### 数据库

沿用[第 24 章](24-sqlx-postgres.md#启动-postgresql)那个容器。如果已经删了:

```bash
docker run --name axum-tutorial-pg \
  -e POSTGRES_PASSWORD=password \
  -p 5432:5432 \
  -d postgres:16
```

**官方示例的连接串没有密码**,而容器要求密码认证,所以直接跑会失败:

```
Error: Db(DbError { severity: "FATAL", code: SqlState(E28P01), message: "password authentication failed for user \"postgres\"" })
```

两种处理方式:

**方式一(推荐):起一个不要密码的容器。**

```bash
docker run --name axum-tutorial-pg-trust \
  -e POSTGRES_HOST_AUTH_METHOD=trust \
  -p 5433:5432 \
  -d postgres:16
```

注意端口映射到 `5433` 避免和上一章的容器冲突,连接串要相应改成 `host=localhost port=5433 user=postgres`。

**方式二:改代码加上密码。**

把连接串改成 `"host=localhost user=postgres password=password"`。这偏离了官方示例,但更贴近真实用法。

下面的验证步骤按方式一(端口 5433)写。

### 创建项目

```bash
cargo new tokio-postgres
cd tokio-postgres
```

```bash
cargo add axum
cargo add tokio --features full
cargo add tokio-postgres
cargo add bb8
cargo add bb8-postgres
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

三个数据库相关的 crate 对应前置知识那张表。

## 分步构建

### 步骤 1:建池子并查询

**要做什么**:配好 `bb8` + `tokio-postgres`,写第一条路由。

```rust
// src/main.rs
use axum::{extract::State, http::StatusCode, routing::get, Router};
use bb8::Pool;
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

    let app = Router::new()
        .route("/", get(using_connection_pool_extractor))
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

/// Utility function for mapping any error into a `500 Internal Server Error`
/// response.
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
```

**`type ConnectionPool = Pool<PostgresConnectionManager<NoTls>>`** —— 这个别名是必需的,原始类型太长,每个 handler 签名都写一遍不现实。对比 sqlx 的 `PgPool` 一个词搞定,**这是分层设计的代价**。

**`Pool::builder().build(manager)`** 没有配任何参数,用的是 bb8 的默认值(`max_size = 10`)。[第 24 章](24-sqlx-postgres.md#为什么需要连接池)讲的 `max_connections` 和超时在这里同样重要,只是官方示例省略了。

> **更好的写法**:`Pool::builder()` 没有设置任何参数,连接池上限、获取超时、空闲回收全用默认值。[第 24 章](24-sqlx-postgres.md#为什么需要连接池)讲过 `acquire_timeout` 为什么必须设——池子满了不设超时就会无限排队,请求堆积直到服务卡死。生产环境至少要写成 `Pool::builder().max_size(5).connection_timeout(Duration::from_secs(3)).build(manager)`。示例省略是为了突出主线,不是说默认值够用。

**`.unwrap()` 出现了两次**:`new_from_stringlike` 解析连接串失败会 panic,`build` 建池失败也会。[第 24 章](24-sqlx-postgres.md#步骤-1连上数据库用-state-拿池子)讲过启动时快速失败是对的。

**handler 里四步**:借连接 → 执行查询 → 取值 → 转字符串。对比 sqlx 那个一行链式,**啰嗦是明显的**,但每一步都很直白。

**`row.try_get(0)`** 按索引取第一列,类型由 `let two: i32` 推断。前置知识讲过为什么必须是 `i32`。

**跑起来看**:

按项目搭建里说的,连接串要改成带端口的形式(如果用方式一的容器):

```rust
    let manager = PostgresConnectionManager::new_from_stringlike(
        "host=localhost port=5433 user=postgres",
        NoTls,
    )
    .unwrap();
```

```bash
cargo run
```

```bash
curl http://127.0.0.1:3000/
```

```
2
```

数据库算出来的 `1 + 1`。

**验证类型不匹配的后果。** 临时把 `i32` 改成 `i64`:

```rust
    let two: i64 = row.try_get(0).map_err(internal_error)?;
```

```bash
cargo run
curl -s http://127.0.0.1:3000/
```

```
error deserializing column 0: cannot convert between the Rust type `i64` and the Postgres type `int4`
```

**编译完全通过,运行时才失败。** 这就是前置知识说的那个陷阱——`try_get` 的类型正确性没有任何编译期保障。

对比 sqlx 的 `query_scalar!` 宏(带 `!` 的版本):它在编译期就知道 `1 + 1` 返回 `int4`,类型写错直接编译失败。**这是选 sqlx 的主要理由。**

改回 `i32`。

### 步骤 2:自定义连接提取器

**要做什么**:和[第 24 章](24-sqlx-postgres.md#步骤-2自定义连接提取器)一样的提取器,但要处理 `bb8` 的生命周期问题。

```rust
// src/main.rs
use axum::{
    extract::{FromRef, FromRequestParts, State},        // 改动
    http::{request::Parts, StatusCode},                 // 改动
    routing::get,
    Router,
};
use bb8::{Pool, PooledConnection};                      // 改动:加上 PooledConnection
// ...(其余 use 不变)

#[tokio::main]
async fn main() {
    // ...(连接池不变)

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

// 新增
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

// ...(internal_error 不变)
```

结构和[第 24 章](24-sqlx-postgres.md#步骤-2自定义连接提取器)完全一致:`FromRequestParts` + `FromRef` 约束。`FromRef` 的作用见那一章,不再重复。

**两处差别值得注意。**

**`PooledConnection<'static, ...>` 和 `get_owned()`** —— 前置知识讲过。用 `get()` 会编译失败:

```
error[E0597]: `pool` does not live long enough
```

因为 `get()` 返回的连接借用了局部变量 `pool`,而它要被移进 `DatabaseConnection` 返回出去。

**`DatabaseConnection(conn)` 没有 `mut`** —— 对比 sqlx 那边的 `DatabaseConnection(mut conn)`。因为 `tokio-postgres` 的查询方法接受 `&self`,不需要可变借用;sqlx 的执行器接口要求 `&mut PgConnection`。

这个区别源于两者对"连接状态"的建模不同:`tokio-postgres` 把连接的可变状态藏在内部(实际的 IO 由后台任务处理),而 sqlx 让它显式体现在类型上。

**跑起来看**:

```bash
cargo run
```

```bash
curl http://127.0.0.1:3000/
curl -X POST http://127.0.0.1:3000/
```

```
2
2
```

两条路由行为一致,和[第 24 章](24-sqlx-postgres.md#步骤-2自定义连接提取器)的结论相同:方式一(传池子)只在查询期间占用连接,方式二(先借连接)整个 handler 都占着。**默认用方式一。**

### 步骤 3:两套方案的横向对比

**要做什么**:不写代码,把两章摆在一起看清差别。

| | sqlx([第 24 章](24-sqlx-postgres.md)) | tokio-postgres + bb8(本章) |
|---|---|---|
| **定位** | 一体化 SQL 工具包 | 驱动 + 独立连接池 |
| **crate 数量** | 1 | 3 |
| **编译期 SQL 校验** | **有**(`query!` 宏) | 无 |
| **编译期类型校验** | **有**(宏版本) | 无,`try_get` 运行时失败 |
| **连接池配置** | 一步 | 两步(manager + pool) |
| **池类型名** | `PgPool` | `Pool<PostgresConnectionManager<NoTls>>`(需别名) |
| **连接串格式** | URL | 键值对(也支持 URL) |
| **参数绑定** | `.bind(x).bind(y)` | `&[&x, &y]` |
| **取值** | `query_scalar` / `query_as` 自动 | `row.try_get(i)` 手动 |
| **多数据库支持** | PostgreSQL / MySQL / SQLite / MSSQL | 仅 PostgreSQL |
| **编译速度** | 慢(宏 + 多驱动) | 快 |
| **连接池可替换** | 否 | **是**(可换 deadpool 等) |

**怎么选:**

**默认选 sqlx。** 编译期 SQL 校验是实打实的价值——它能在 `cargo build` 阶段抓出"表名拼错""列不存在""类型对不上"这类错误,而这些在 tokio-postgres 里全是运行时问题,可能到线上才暴露。

**这些情况考虑 tokio-postgres:**

- 需要 PostgreSQL 的**特有能力**,比如 `COPY` 批量导入、`LISTEN/NOTIFY` 订阅、复制协议。sqlx 对这些支持有限。
- 追求**编译速度**。sqlx 的宏和多驱动支持让它编译很慢,大项目里感受明显。
- 已经在用某个特定的连接池(deadpool、mobc),不想引入 sqlx 自带的那套。
- 需要**极致的性能控制**,比如复用预处理语句、精细管理连接状态。

**这就是官方同时提供两个示例的原因** —— 不是"新旧版本"或"推荐/不推荐"的关系,而是两种取舍。绝大多数应用用 sqlx,少数有特殊需求的用 tokio-postgres。

顺带一提,[第 26 章](26-diesel-postgres.md) 的 diesel 是第三条路——真正的 ORM,用 Rust 的类型系统构造查询而不写 SQL 字符串。三者的完整对比放在那一章。

## 完整代码

`src/main.rs`:

```rust
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
```

## 代价与边界

**连接池没有任何配置。** 上面「更好的写法」已经指出。默认 `max_size = 10`,没有获取超时——这在生产环境是隐患。

**`NoTls` 只适合本地。** 前置知识讲过,连接远程数据库必须加密。

**连接串里的凭据。** 本章示例把连接串硬编码在源码里(甚至连[第 24 章](24-sqlx-postgres.md#步骤-1连上数据库用-state-拿池子)那个 `env::var` 都没有)。生产环境必须从环境变量或密钥管理系统读取。

**`try_get` 的类型错误只在运行时暴露。** 步骤 1 验证过。缓解办法是给每张表定义一个 struct 和一个 `from_row` 函数,把类型转换集中在一处——出错时至少容易定位。

**没有查询超时。** 和[第 24 章](24-sqlx-postgres.md#代价与边界)一样,慢查询会一直占着连接。

**错误信息直接返回给客户端。** `internal_error` 的问题在[第 24 章](24-sqlx-postgres.md#步骤-1连上数据库用-state-拿池子)已经指出,本章代码完全相同。

## 动手练习

**1. 给连接池加上配置。**

提示:`Pool::builder().max_size(5).connection_timeout(Duration::from_secs(3)).build(manager)`。然后照[第 24 章](24-sqlx-postgres.md#步骤-2自定义连接提取器)那个办法验证:在 handler 里 `sleep` 占住连接,并发发 6 个请求,看第 6 个是不是 3 秒后失败。

**2. 写一个带参数的查询,体会参数绑定的差别。**

提示:建表 `create table users (id serial primary key, name text not null);`,插入几条数据。然后写 `GET /users/{id}`,用 `conn.query_opt("select name from users where id = $1", &[&id])`。注意 `id` 要先从 `Path<i32>` 提取,而且切片里要写 `&id` 而不是 `id`。`query_opt` 返回 `Option<Row>`,正好对应"可能查不到"。

**3. 把 `try_get` 的类型故意写错,再用 sqlx 的宏对比。**

提示:本章步骤 1 演示过运行时失败。如果你还留着[第 24 章](24-sqlx-postgres.md)的项目,把那边的 `query_scalar` 换成 `query_scalar!`(带感叹号),故意写错类型,看编译器怎么说。**这个对比能让你直观感受到编译期校验值多少钱。** 注意用宏需要设置 `DATABASE_URL` 环境变量让编译时能连上数据库。

## 小结与下一章预告

`tokio-postgres` 只做驱动,连接池要自己配(本章用 `bb8` + `bb8-postgres` 适配器)。这种分离让组件可替换,代价是配置更零碎、类型名更长(必须起别名)。

两套方案在使用上的主要差别:**sqlx 有 `query_scalar`/`query_as` 自动转换类型,tokio-postgres 要手动 `row.try_get(i)` 且类型错误只在运行时暴露**。本章验证过把 `i32` 写成 `i64` 能编译通过但运行时失败——这类问题在 sqlx 的宏版本里是编译错误。

提取器里必须用 `get_owned()` 而不是 `get()`:后者返回的连接借用了池子,活不过 `from_request_parts` 的作用域。sqlx 没有这个问题,因为 `PoolConnection` 本身不带生命周期参数。

**选型上默认用 sqlx**,编译期 SQL 校验是实打实的价值。需要 PostgreSQL 特有能力(`COPY`、`LISTEN/NOTIFY`)、追求编译速度、或者要用特定连接池时,才考虑 tokio-postgres。**官方同时提供两个示例不是新旧关系,是两种取舍。**

前两章都是"写 SQL 字符串"的路子。[第 26 章 diesel-postgres](26-diesel-postgres.md) 走第三条路——真正的 ORM:用 Rust 的类型系统构造查询,`users.filter(name.eq("x")).load()`,SQL 由库生成。你会看到 schema 怎么定义、迁移怎么管理、以及一个绕不开的问题:**diesel 是同步库,怎么在 async 的 axum 里用它**。
