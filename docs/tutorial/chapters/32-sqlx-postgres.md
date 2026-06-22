# 32. sqlx-postgres

对应示例：`examples/sqlx-postgres`

本章目标：用 SQLx 连接 PostgreSQL，理解连接池 `PgPool`、Axum `State`、以及自定义数据库连接 extractor。

从这一章开始进入数据库（第 32-37 章）。  
对不会后端的人来说，最重要的不是先背 SQLx API，而是先理解：

```text
Web 请求
-> handler
-> 从连接池拿数据库连接
-> 执行 SQL
-> 把结果变成 HTTP 响应
```

## 先理解：本系列 5 个数据库方案怎么选

第 32-37 章会讲 5 种数据库接入方式。在动手之前，先用这张表建立全局印象，避免"学了 5 章还不知道该用哪个"。

| 方案 | 章节 | 异步/同步 | 连接池来源 | 是否有编译期 SQL 校验 | ORM 程度 |
| --- | --- | --- | --- | --- | --- |
| **SQLx** | 32 | 原生异步 | 自带 `PgPool` | 有（`query!` 宏，需连数据库编译） | 半 ORM（写 SQL，自动映射） |
| **tokio-postgres + bb8** | 33 | 原生异步 | bb8 通用池 | 无 | 最底层（裸 SQL + 手动取列） |
| **Diesel（同步）+ deadpool** | 34 | 同步（用 `interact` 包到阻塞线程） | deadpool-diesel | 有（`table!` 宏 + 类型系统） | 重 ORM（DSL 写查询） |
| **Diesel Async + bb8** | 35 | 原生异步 | bb8 | 有（同 Diesel） | 重 ORM |
| **Redis（tokio + bb8）** | 36 | 原生异步 | bb8 或 `MultiplexedConnection` | 无 | 无（KV 命令式） |
| **MongoDB** | 37 | 原生异步 | `Client` 自带内部池 | 无 | 文档型（BSON） |

**怎么选？**

- **想要最省心、写 SQL 自动映射**：用 **SQLx**（32 章）。编译期校验 SQL 是它的杀手锏。
- **想完全控制、不要 ORM、不要宏**：用 **tokio-postgres**（33 章）。最轻量，但要手动取列、手动管池。
- **想要强类型 DSL 查询、不怕重**：用 **Diesel**（34 章同步 或 35 章异步）。类型安全最强，但学习曲线陡。
- **Diesel 同步还是异步？**：新项目优先 35 章 diesel-async（不用 `interact` 套阻塞调用，代码更直接）；老项目或必须用某些同步 API 时用 34 章。
- **缓存/计数器/排行榜**：用 **Redis**（36 章）。
- **文档型数据/灵活 schema**：用 **MongoDB**（37 章）。

**一个关键概念：同步 driver 在异步程序里怎么办？**

Diesel（34 章）是同步 driver——它的查询会阻塞当前线程。如果直接在 `async fn` 里调用，会卡住整个 Tokio worker 线程，导致其他请求都被阻塞。所以 34 章用 `pool.interact(|conn| ...)` 把同步查询扔到专门的阻塞线程池（底层就是 `tokio::task::spawn_blocking`）执行。这是同步 driver 接入异步 runtime 的标准做法。

而 SQLx（32 章）、tokio-postgres（33 章）、diesel-async（35 章）都是原生异步 driver，查询本身不会阻塞线程，可以直接 `.await`，代码更直观。这也是为什么新项目更倾向异步 driver。

理解了这张表，再看每一章就有了"它在整体里的位置"。下面正式进入 SQLx。

## 这个小项目在做什么

应用只有一个路径：

```text
/
```

但支持两个方法：

```text
GET  / -> 使用 State<PgPool> 直接拿连接池查询数据库
POST / -> 使用自定义 DatabaseConnection extractor 拿单个连接查询数据库
```

两个接口执行的 SQL 都是：

````sql
select 'hello world from pg'
````

返回结果都是：

```text
hello world from pg
```

请求主线是：

```text
程序启动
-> 读取 DATABASE_URL
-> 创建 PostgreSQL 连接池 PgPool
-> 把 pool 放进 Router state
-> 请求进入 handler
-> handler 用 pool 或 connection 执行 SQL
-> 返回字符串响应
```

## 先理解为什么需要连接池

后端连接数据库时，不能每来一个请求都重新建立一次数据库连接。

因为建立连接需要成本：

```text
TCP 连接
数据库认证
连接初始化
```

如果每次请求都重新连接，会很慢，也容易把数据库压垮。

连接池的做法是：

```text
程序启动时创建一组数据库连接
请求来了就借一个连接用
用完还回连接池
```

SQLx 的 `PgPool` 就是 PostgreSQL 连接池。

## 文件和依赖

这个 example 有两个主要文件：

1. `examples/sqlx-postgres/Cargo.toml`：声明 Axum、Tokio、tracing、SQLx。
2. `examples/sqlx-postgres/src/main.rs`：创建连接池、注册路由、执行数据库查询、自定义 extractor。

关键依赖：

- `axum`：提供 Router、State、FromRequestParts。
- `sqlx`：提供 PostgreSQL 连接池和查询 API。
- `tokio`：异步运行时。
- `tracing` / `tracing-subscriber`：输出启动日志。

SQLx 配置：

````toml
sqlx = { version = "0.9", features = ["runtime-tokio", "any", "postgres"] }
````

其中：

- `runtime-tokio`：让 SQLx 使用 Tokio 异步运行时。
- `postgres`：启用 PostgreSQL 支持。
- `any`：启用 SQLx 的通用数据库抽象（`AnyPool`/`AnyConnection`），支持运行时切换多种数据库。**本例代码实际没有用到它**（handler 里直接用 `PgPool`），这里只是 example 沿用的配置。你自己用 Postgres 时可以不加这个 feature。

## 第一步：读取 DATABASE_URL

源码：

````rust
let db_connection_str = std::env::var("DATABASE_URL")
    .unwrap_or_else(|_| "postgres://postgres:password@localhost".to_string());
````

数据库连接地址通常放在环境变量里：

```text
DATABASE_URL
```

这样代码不用写死不同环境的数据库地址。

例如：

```text
开发环境：postgres://postgres:password@localhost
测试环境：postgres://test:password@test-db
生产环境：由部署系统注入
```

如果没有设置环境变量，这个 example 使用默认值：

```text
postgres://postgres:password@localhost
```

## 第二步：创建 PgPool

源码：

````rust
let pool = PgPoolOptions::new()
    .max_connections(5)
    .acquire_timeout(Duration::from_secs(3))
    .connect(&db_connection_str)
    .await
    .expect("can't connect to database");
````

这段创建 PostgreSQL 连接池。

配置含义：

- `max_connections(5)`：连接池最多维护 5 个数据库连接。
- `acquire_timeout(Duration::from_secs(3))`：借连接最多等 3 秒。
- `connect(&db_connection_str).await`：真正连接数据库。

如果数据库没启动、密码错、地址不通，这里会失败并 panic：

````rust
.expect("can't connect to database")
````

真实项目里启动时连不上数据库通常也应该直接失败，因为服务没有数据库就无法正常工作。

## 第三步：把连接池放进 Router state

源码：

````rust
let app = Router::new()
    .route(
        "/",
        get(using_connection_pool_extractor).post(using_connection_extractor),
    )
    .with_state(pool);
````

`.with_state(pool)` 把连接池放进 Axum 应用状态里。

之后 handler 可以通过：

````rust
State(pool): State<PgPool>
````

拿到连接池。

这里同一路径 `/` 注册了两个方法：

```text
GET  -> using_connection_pool_extractor
POST -> using_connection_extractor
```

## 第四步：GET 使用 State<PgPool>

源码：

````rust
async fn using_connection_pool_extractor(
    State(pool): State<PgPool>,
) -> Result<String, (StatusCode, String)> {
    sqlx::query_scalar("select 'hello world from pg'")
        .fetch_one(&pool)
        .await
        .map_err(internal_error)
}
````

这是一种最直接的写法：

1. 从 `State` 里拿到 `PgPool`。
2. 执行 SQL。
3. 返回查询结果。

`query_scalar` 适合查询单个值。  
这条 SQL 只返回一个字符串：

````sql
select 'hello world from pg'
````

所以返回类型可以是：

```text
Result<String, ...>
```

`fetch_one(&pool)` 表示：

```text
从连接池里借一个连接
执行 SQL
拿一行结果
```

用完后连接会回到池里。

## 第五步：统一把数据库错误转成 HTTP 500

源码：

````rust
.await
.map_err(internal_error)
````

`sqlx` 查询失败时返回 SQLx 错误。  
但 handler 要返回 HTTP 响应。

所以这里用：

````rust
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
````

把任意数据库错误转成：

```text
500 Internal Server Error
错误文本
```

真实项目里通常不把数据库错误原文直接返回给用户。  
这里是 example，为了看清错误。

## 第六步：自定义 DatabaseConnection extractor

源码：

````rust
struct DatabaseConnection(sqlx::pool::PoolConnection<sqlx::Postgres>);
````

这个结构体包住一个数据库连接。

然后实现：

````rust
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

这表示：

```text
当 handler 参数里需要 DatabaseConnection 时
Axum 自动从应用 state 中找到 PgPool
然后从 pool 里 acquire 一个连接
最后把连接包装成 DatabaseConnection
```

这样 handler 就可以写得更干净：

````rust
async fn using_connection_extractor(
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<String, (StatusCode, String)> {
    ...
}
````

## 第七步：FromRequestParts 是什么

`FromRequestParts` 是 Axum 的 extractor trait。  
实现它之后，你的类型就可以作为 handler 参数。

为什么这里用 `FromRequestParts`，不是 `FromRequest`？

因为这个 extractor 只需要：

```text
请求 parts
应用 state
```

它不需要读取 request body。

这很重要。  
读取 body 的 extractor 会消费请求体，而数据库连接 extractor 不应该碰 body。

## 第八步：FromRef<S> 是什么

源码约束：

````rust
PgPool: FromRef<S>,
````

这表示：

```text
可以从应用 state S 中取出 PgPool
```

本章的 state 本身就是 `PgPool`：

````rust
.with_state(pool)
````

所以 `PgPool::from_ref(state)` 能拿到连接池。

这种写法的好处是以后 state 变复杂时也能扩展。  
例如真实项目可能是：

````rust
struct AppState {
    db: PgPool,
    config: AppConfig,
}
````

只要能从 `AppState` 里取出 `PgPool`，这个 extractor 仍然可以工作。

## 第九步：POST 使用 DatabaseConnection

源码：

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

这里 handler 不直接拿 `PgPool`。  
它拿的是已经 acquire 好的连接：

````rust
DatabaseConnection(mut conn): DatabaseConnection
````

执行查询时传入：

````rust
.fetch_one(&mut *conn)
````

因为 SQLx 查询需要一个可执行器。  
`&mut *conn` 把连接包装类型解引用成 SQLx 能使用的连接引用。

这种模式适合你想在 extractor 层统一处理：

```text
如何获取连接
获取失败如何返回错误
是否开启事务
是否做额外校验
```

## 第十步：两种写法怎么选择

### 直接用 State<PgPool>

优点：

```text
简单
直观
适合大多数普通查询
```

写法：

````rust
async fn handler(State(pool): State<PgPool>) { ... }
````

### 自定义 DatabaseConnection extractor

优点：

```text
handler 更聚焦业务
连接获取逻辑可以复用
以后能扩展事务或鉴权
```

写法：

````rust
async fn handler(DatabaseConnection(conn): DatabaseConnection) { ... }
````

初学阶段建议先掌握 `State<PgPool>`。  
等多个 handler 里重复出现连接获取逻辑时，再考虑自定义 extractor。

## 第十一步（重要）：SQLx 的 `query!` 宏——编译期 SQL 校验

本章用的是 `sqlx::query_scalar("select ...")`，这是**运行期**执行：SQL 字符串在运行时才发给 Postgres，列名写错、类型不匹配都要等运行时才报错。

但 SQLx 真正的招牌是 **`query!` 宏**，它在**编译期**就校验 SQL：

````rust
// 编译期会真的连数据库，执行 EXPLAIN，检查：
// 1. SQL 语法对不对
// 2. 表名、列名存不存在
// 3. 返回的列类型能不能映射到 Rust 类型
let row: (String,) = sqlx::query_as("select name from users where id = $1")
    .bind(user_id)
    .fetch_one(&pool)
    .await?;
````

如果列名写错了（比如 `select nam from users`），**编译就失败**，根本到不了运行时。这是 SQLx 相对 tokio-postgres（33 章）最大的优势。

使用 `query!` 宏的前提：

1. 编译时能连上数据库（通过 `DATABASE_URL` 环境变量），或
2. 用 `cargo sqlx prepare` 生成离线缓存文件（`.sqlx/` 目录），这样 CI/无数据库环境也能编译。

本例为什么没用 `query!`？因为 SQL 是常量 `select 'hello world from pg'`，没有列名/类型可校验，用 `query_scalar` 更简单。真实项目里查询带表名、列名时，**优先用 `query!` / `query_as!`**，把 SQL 错误消灭在编译期。

## 函数职责速查

- `main`：初始化日志，读取数据库地址，创建连接池，注册路由并启动服务。
- `using_connection_pool_extractor`：通过 `State<PgPool>` 直接执行数据库查询。
- `DatabaseConnection::from_request_parts`：从应用 state 中取出连接池，并 acquire 一个连接。
- `using_connection_extractor`：通过自定义 extractor 获得数据库连接并执行查询。
- `internal_error`：把数据库错误转换成 HTTP 500 响应。

## 带中文注释的手写版

````rust
//! SQLx + PostgreSQL 示例。
//!
//! Run with
//!
//! ```not_rust
//! cargo run -p example-sqlx-postgres
//! ```

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
    // 初始化日志。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 从环境变量读取数据库连接地址。
    // 如果没有设置，就使用本地 PostgreSQL 默认地址。
    let db_connection_str = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://postgres:password@localhost".to_string());

    // 创建 PostgreSQL 连接池。
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .acquire_timeout(Duration::from_secs(3))
        .connect(&db_connection_str)
        .await
        .expect("can't connect to database");

    // 注册路由，并把连接池放进应用 state。
    let app = Router::new()
        .route(
            "/",
            get(using_connection_pool_extractor).post(using_connection_extractor),
        )
        .with_state(pool);

    // 启动服务。
    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

// 写法一：直接从 State 中提取 PgPool。
async fn using_connection_pool_extractor(
    State(pool): State<PgPool>,
) -> Result<String, (StatusCode, String)> {
    sqlx::query_scalar("select 'hello world from pg'")
        .fetch_one(&pool)
        .await
        .map_err(internal_error)
}

// 自定义 extractor，内部包住一个 PostgreSQL 连接。
struct DatabaseConnection(sqlx::pool::PoolConnection<sqlx::Postgres>);

impl<S> FromRequestParts<S> for DatabaseConnection
where
    // 表示可以从应用 state S 中取出 PgPool。
    PgPool: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = (StatusCode, String);

    async fn from_request_parts(_parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        // 从应用 state 中拿连接池。
        let pool = PgPool::from_ref(state);

        // 从连接池借一个连接。
        let conn = pool.acquire().await.map_err(internal_error)?;

        // 包装成自定义 extractor 类型。
        Ok(Self(conn))
    }
}

// 写法二：handler 直接接收自定义数据库连接 extractor。
async fn using_connection_extractor(
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<String, (StatusCode, String)> {
    sqlx::query_scalar("select 'hello world from pg'")
        .fetch_one(&mut *conn)
        .await
        .map_err(internal_error)
}

// 把内部错误转换成 HTTP 500。
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
````

## 运行和验证

你需要先有一个 PostgreSQL 服务。  
例如本地连接地址：

```text
postgres://postgres:password@localhost
```

也可以通过环境变量指定：

````bash
export DATABASE_URL='postgres://postgres:password@localhost'
````

运行：

````bash
cargo run -p example-sqlx-postgres
````

验证 GET：

````bash
curl http://127.0.0.1:3000/
````

验证 POST：

````bash
curl -X POST http://127.0.0.1:3000/
````

两者都应该返回：

```text
hello world from pg
```

如果连接失败，重点检查：

```text
PostgreSQL 是否启动
DATABASE_URL 是否正确
用户名和密码是否正确
端口是否能访问
```

## 常见卡点

### 1. 为什么没有建表也能查询？

因为 SQL 是：

````sql
select 'hello world from pg'
````

它只是让 PostgreSQL 返回一个字符串常量，不需要任何表。

### 2. PgPool 是不是一个真实连接？

不是。  
`PgPool` 是连接池，里面管理多个连接。

handler 可以把 pool 传给 SQLx，SQLx 会从池里借连接执行查询。

### 3. max_connections 应该设置多少？

没有固定答案。  
它取决于数据库能力、服务实例数量、请求量和查询耗时。

入门时先理解：

```text
连接数太少 -> 请求可能排队
连接数太多 -> 数据库可能被压垮
```

### 4. 为什么自定义 extractor 不读取 body？

数据库连接和请求 body 没关系。  
它只需要 state，所以实现 `FromRequestParts` 更合适。

### 5. 能不能每个请求新建 PgPool？

不要。  
连接池应该在程序启动时创建一次，放进 state 里复用。

## 手写任务

建议按下面顺序自己敲一遍：

1. 先读取 `DATABASE_URL`。
2. 用 `PgPoolOptions::new()` 创建连接池。
3. 把 `pool` 放进 `.with_state(pool)`。
4. 写一个 GET handler，用 `State(pool): State<PgPool>`。
5. 在 handler 里执行 `sqlx::query_scalar(...)`。
6. 写 `internal_error`，把 SQLx 错误转成 500。
7. 再写 `DatabaseConnection` extractor。
8. 用 POST handler 验证自定义 extractor。

加深练习：

1. 把 SQL 改成 `select now()`，返回当前数据库时间。
2. 新建一张 `users` 表，尝试查询一行用户。
3. 把 state 改成 `AppState { db: PgPool }`，练习 `FromRef`。

## 本章真正要记住什么

数据库章节的第一层心智模型是：

```text
程序启动时创建连接池
Router state 保存连接池
handler 从 state 或 extractor 获得数据库能力
SQLx 执行 SQL
错误转换成 HTTP 响应
```

初学时优先掌握：

````rust
State(pool): State<PgPool>
````

再理解自定义 extractor：

````rust
impl<S> FromRequestParts<S> for DatabaseConnection
````

这样你能先写出可工作的数据库接口，再逐步把重复逻辑抽象出去。

## 源码对照

本章手写版对应源码：

- `examples/sqlx-postgres/src/main.rs`
- `examples/sqlx-postgres/Cargo.toml`
