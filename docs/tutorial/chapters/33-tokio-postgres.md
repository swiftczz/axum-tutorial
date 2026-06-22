# 33. tokio-postgres

对应示例：`examples/tokio-postgres`

本章目标：使用 `tokio-postgres` 和 `bb8` 连接 PostgreSQL，理解连接管理器、连接池、查询 row 以及自定义数据库连接 extractor。

上一章使用 SQLx。  
这一章换成更底层一些的 `tokio-postgres`，并用 `bb8` 提供连接池。

## 这个小项目在做什么

应用只有一个路径：

```text
/
```

支持两个方法：

```text
GET  / -> 从 State 中拿连接池，借连接执行 SQL
POST / -> 用自定义 DatabaseConnection extractor 借连接执行 SQL
```

执行的 SQL 是：

````sql
select 1 + 1
````

返回：

```text
2
```

请求主线是：

```text
程序启动
-> 创建 PostgresConnectionManager
-> 用 bb8 创建连接池
-> 把连接池放进 Router state
-> 请求进入 handler
-> 从 pool 借连接
-> query_one 执行 SQL
-> 从 Row 里取出第一列
-> 返回字符串
```

## 先理解 tokio-postgres 和 SQLx 的区别

上一章 SQLx 里：

```text
PgPoolOptions -> PgPool -> query_scalar -> fetch_one
```

这一章 tokio-postgres 里：

```text
PostgresConnectionManager -> bb8 Pool -> conn.query_one -> row.try_get
```

可以先这样理解：

```text
SQLx 提供了连接池和查询 API
tokio-postgres 提供 PostgreSQL 客户端
bb8 负责连接池
```

所以这一章会多一个角色：

```text
连接管理器 manager
```

manager 告诉连接池如何创建 PostgreSQL 连接。

## 文件和依赖

这个 example 有两个主要文件：

1. `examples/tokio-postgres/Cargo.toml`：声明 Axum、bb8、bb8-postgres、tokio-postgres、Tokio、tracing。
2. `examples/tokio-postgres/src/main.rs`：创建连接池、注册路由、执行查询、自定义 extractor。

关键依赖：

- `tokio-postgres`：PostgreSQL 异步客户端。
- `bb8`：通用异步连接池。
- `bb8-postgres`：让 `bb8` 能管理 `tokio-postgres` 连接。
- `axum`：提供 Router、State、FromRequestParts。
- `tokio`：异步运行时。
- `tracing` / `tracing-subscriber`：输出启动日志。

## 第一步：创建连接管理器

源码：

````rust
let manager =
    PostgresConnectionManager::new_from_stringlike("host=localhost user=postgres", NoTls)
        .unwrap();
````

`PostgresConnectionManager` 负责告诉 `bb8`：

```text
怎么创建 PostgreSQL 连接
怎么检查连接是否可用
```

连接字符串是：

```text
host=localhost user=postgres
```

它表示连接本机 PostgreSQL，用户名是 `postgres`。

`NoTls` 表示不使用 TLS 加密连接。  
本地开发通常可以这样。生产环境是否需要 TLS 要看部署环境。

## 第二步：用 bb8 创建连接池

源码：

````rust
let pool = Pool::builder().build(manager).await.unwrap();
````

`Pool::builder()` 创建连接池构建器。  
`.build(manager).await` 用前面的 manager 真正创建连接池。

连接池会负责：

```text
维护数据库连接
请求来了借连接
用完后归还连接
连接不可用时重新创建
```

这一点和上一章的 `PgPool` 心智模型一样。

## 第三步：定义连接池类型别名

源码：

````rust
type ConnectionPool = Pool<PostgresConnectionManager<NoTls>>;
````

这个类型完整写出来比较长。  
所以源码给它起了一个别名：

```text
ConnectionPool
```

后面就可以写：

````rust
State(pool): State<ConnectionPool>
````

而不是每次都写完整泛型。

这是 Rust 代码里常见的可读性优化。

## 第四步：把 pool 放进 Router state

源码：

````rust
let app = Router::new()
    .route(
        "/",
        get(using_connection_pool_extractor).post(using_connection_extractor),
    )
    .with_state(pool);
````

和上一章一样，连接池是应用级共享资源。  
程序启动时创建一次，然后放进 state，所有请求共用。

同一路径 `/` 注册两个方法：

```text
GET  -> using_connection_pool_extractor
POST -> using_connection_extractor
```

## 第五步：GET 直接用 State<ConnectionPool>

源码：

````rust
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
````

这段分四步：

1. `State(pool)` 从 Axum state 里拿连接池。
2. `pool.get().await` 从连接池借一个连接。
3. `query_one("select 1 + 1", &[])` 执行 SQL，并要求返回一行。
4. `row.try_get(0)` 从第 0 列取出结果。

`&[]` 是 SQL 参数列表。  
这条 SQL 没有参数，所以传空数组。

## 第六步：Row 和 try_get

`tokio-postgres` 查询返回的是 `Row`。

可以把 `Row` 理解成：

```text
数据库返回的一行数据
```

这一行可能有多列。  
本章 SQL：

````sql
select 1 + 1
````

只有一列，所以用：

````rust
let two: i32 = row.try_get(0).map_err(internal_error)?;
````

含义是：

```text
从第 0 列取值，并尝试转成 i32
```

如果列不存在或类型不匹配，`try_get` 会返回错误。

## 第七步：自定义 DatabaseConnection extractor

源码：

````rust
struct DatabaseConnection(PooledConnection<'static, PostgresConnectionManager<NoTls>>);
````

这个结构体包住一个从 bb8 池里借出的连接。

然后实现：

````rust
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
````

逻辑和上一章非常像：

```text
从 state 中取连接池
从连接池借连接
包装成 DatabaseConnection
交给 handler
```

这里使用的是：

````rust
pool.get_owned().await
````

它返回 owned connection，适合放进 extractor 结构体里。

## 第八步：POST 使用自定义连接 extractor

源码：

````rust
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
````

handler 参数里直接写：

````rust
DatabaseConnection(conn): DatabaseConnection
````

Axum 就会自动调用刚才实现的 extractor。

handler 不需要知道：

```text
连接池在哪里
怎么从池里借连接
借连接失败怎么变成 HTTP 错误
```

它只关心拿到连接后执行 SQL。

## 第九步：错误转换

源码：

````rust
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
````

这个函数复用在多个地方：

- 借连接失败。
- SQL 查询失败。
- 从 row 取值失败。

统一转成：

```text
500 Internal Server Error
```

真实项目里更推荐：

```text
详细错误写日志
响应给用户通用错误
```

但 example 直接返回错误文本，更方便学习。

## 第十步：和 SQLx 版本怎么选

SQLx 版本更一体化：

```text
SQLx 自带 pool
查询 API 更高级
可以使用宏做 SQL 检查
```

tokio-postgres 版本更接近 PostgreSQL 客户端本身：

```text
bb8 管连接池
tokio-postgres 执行查询
手动从 Row 取列
```

初学建议：

```text
先学会连接池 + State + 查询 + 错误处理这个共同模型
再根据项目选择 SQLx 或 tokio-postgres
```

## 函数职责速查

- `main`：初始化日志，创建 PostgreSQL manager 和 bb8 连接池，注册路由并启动服务。
- `using_connection_pool_extractor`：直接从 `State<ConnectionPool>` 借连接并查询。
- `DatabaseConnection::from_request_parts`：从 state 中取连接池，并借出 owned connection。
- `using_connection_extractor`：使用自定义 extractor 提供的连接执行查询。
- `internal_error`：把内部错误转成 HTTP 500。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//!
//! ```not_rust
//! cargo run -p example-tokio-postgres
//! ```

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
    // 初始化日志。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 创建 PostgreSQL 连接管理器。
    let manager =
        PostgresConnectionManager::new_from_stringlike("host=localhost user=postgres", NoTls)
            .unwrap();

    // 用 bb8 创建连接池。
    let pool = Pool::builder().build(manager).await.unwrap();

    // 注册路由，并把连接池放进 state。
    let app = Router::new()
        .route(
            "/",
            get(using_connection_pool_extractor).post(using_connection_extractor),
        )
        .with_state(pool);

    // 启动服务。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

// 给较长的连接池类型起别名，后面代码更好读。
type ConnectionPool = Pool<PostgresConnectionManager<NoTls>>;

// 写法一：直接从 State 中拿连接池。
async fn using_connection_pool_extractor(
    State(pool): State<ConnectionPool>,
) -> Result<String, (StatusCode, String)> {
    // 从连接池借一个连接。
    let conn = pool.get().await.map_err(internal_error)?;

    // 执行 SQL，要求返回一行。
    let row = conn
        .query_one("select 1 + 1", &[])
        .await
        .map_err(internal_error)?;

    // 从第 0 列取出 i32。
    let two: i32 = row.try_get(0).map_err(internal_error)?;

    // 返回字符串响应。
    Ok(two.to_string())
}

// 自定义 extractor，包住一个从 bb8 池里借出的连接。
struct DatabaseConnection(PooledConnection<'static, PostgresConnectionManager<NoTls>>);

impl<S> FromRequestParts<S> for DatabaseConnection
where
    // 表示可以从应用 state S 中取出 ConnectionPool。
    ConnectionPool: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = (StatusCode, String);

    async fn from_request_parts(_parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        // 从 state 中取连接池。
        let pool = ConnectionPool::from_ref(state);

        // 借出一个 owned connection，方便放进 extractor 里。
        let conn = pool.get_owned().await.map_err(internal_error)?;

        Ok(Self(conn))
    }
}

// 写法二：直接从自定义 extractor 中拿连接。
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

// 把内部错误转成 HTTP 500。
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
````

## 运行和验证

你需要先启动 PostgreSQL，并确保这个连接字符串可用：

```text
host=localhost user=postgres
```

运行：

````bash
cargo run -p example-tokio-postgres
````

验证 GET：

````bash
curl http://127.0.0.1:3000/
````

验证 POST：

````bash
curl -X POST http://127.0.0.1:3000/
````

两个请求都应该返回：

```text
2
```

如果连接失败，重点检查：

```text
PostgreSQL 是否启动
用户 postgres 是否存在
本机连接是否需要密码
连接字符串是否要加 password 或 dbname
```

## 常见卡点

### 1. 为什么这个例子不用 DATABASE_URL？

源码直接写了：

```text
host=localhost user=postgres
```

这是为了让 example 更短。  
真实项目更推荐像上一章一样从环境变量读取连接字符串。

### 2. bb8 是什么？

`bb8` 是连接池库。  
`tokio-postgres` 负责连接 PostgreSQL，`bb8` 负责复用和管理多个连接。

### 3. query_one 和 try_get 分别做什么？

`query_one` 执行 SQL，并要求数据库返回一行。  
`try_get(0)` 从这一行里取第 0 列。

### 4. 为什么 SQL 参数是 &[]？

因为本章 SQL 没有参数：

````sql
select 1 + 1
````

如果 SQL 有参数，就需要把参数放进这个数组里。

### 5. get 和 get_owned 有什么区别？

`pool.get()` 借出普通连接，适合在当前函数里直接使用。  
`pool.get_owned()` 借出 owned connection，适合放进自定义 extractor 结构体。

## 手写任务

建议按下面顺序自己敲一遍：

1. 用 `PostgresConnectionManager::new_from_stringlike` 创建 manager。
2. 用 `Pool::builder().build(manager).await` 创建连接池。
3. 定义 `type ConnectionPool = ...`。
4. 把 pool 放进 `.with_state(pool)`。
5. 写 GET handler，从 `State<ConnectionPool>` 借连接。
6. 用 `query_one("select 1 + 1", &[])` 查询。
7. 用 `row.try_get(0)` 取出结果。
8. 再实现 `DatabaseConnection` extractor。

加深练习：

1. 把连接字符串改成从环境变量读取。
2. 把 SQL 改成带参数的查询，例如 `select $1::int + $2::int`。
3. 新建一张表，练习从 row 里取多列字段。

## 本章真正要记住什么

tokio-postgres 版本和 SQLx 版本的共同模型是一样的：

```text
启动时创建连接池
Router state 保存连接池
handler 或 extractor 借连接
执行 SQL
把数据库结果转成 HTTP 响应
```

不同点是：

```text
SQLx 自带 PgPool
tokio-postgres 需要 bb8 提供连接池
```

学会这个区别后，切换数据库库就不会只是在背 API，而是在复用同一套后端思路。

## 源码对照

本章手写版对应源码：

- `examples/tokio-postgres/src/main.rs`
- `examples/tokio-postgres/Cargo.toml`
