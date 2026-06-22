# 36. tokio-redis

对应示例：`examples/tokio-redis`

本章目标：用异步 Redis 客户端连接 Redis，理解 `redis::Client`、`bb8` 连接池、`AsyncCommands`，以及在 Axum handler 中读取 Redis 数据。

前几章连接的是 PostgreSQL。  
这一章连接 Redis。

## 这个小项目在做什么

应用只有一个路径：

```text
/
```

支持两个方法：

```text
GET  / -> 从 State<ConnectionPool> 拿 Redis 连接，读取 key foo
POST / -> 从自定义 DatabaseConnection extractor 拿 Redis 连接，读取 key foo
```

程序启动时会先写入：

```text
foo = bar
```

然后两个接口都会返回：

```text
bar
```

请求主线是：

```text
程序启动
-> 创建 redis::Client
-> 用 bb8 创建连接池
-> 启动前 set foo bar 并 get foo 验证连接
-> Router 保存 pool
-> 请求进入 handler
-> 从 pool 借 Redis 连接
-> conn.get("foo")
-> 返回字符串
```

## 先理解 Redis 和 PostgreSQL 的区别

PostgreSQL 是关系型数据库。  
它适合存结构化数据：

```text
用户表
订单表
文章表
复杂查询
事务
```

Redis 更常用作内存型 key-value 系统：

```text
缓存
登录会话
限流计数
排行榜
短期状态
消息队列辅助
```

本章只演示最基础的 key-value：

```text
SET foo bar
GET foo
```

先把它理解成：

```text
通过 key 找 value 的高速存储
```

## 文件和依赖

这个 example 有两个主要文件：

1. `examples/tokio-redis/Cargo.toml`：声明 Axum、bb8、redis、Tokio、tracing。
2. `examples/tokio-redis/src/main.rs`：创建 Redis client 和连接池，注册路由，读取 Redis key。

关键依赖：

- `redis`：Redis 客户端库。
- `bb8`：异步连接池。
- `axum`：提供 Router、State、FromRequestParts。
- `tokio`：异步运行时。
- `tracing` / `tracing-subscriber`：输出启动日志。

Redis 依赖配置：

````toml
redis = { version = "1", default-features = false, features = ["tokio-comp", "bb8"] }
````

这里启用了：

- `tokio-comp`：Tokio 异步支持。
- `bb8`：让 `redis::Client` 可以配合 bb8 连接池。

## 第一步：创建 Redis client

源码：

````rust
tracing::debug!("connecting to redis");
let client = redis::Client::open("redis://localhost").unwrap();
let pool = bb8::Pool::builder().build(client).await.unwrap();
````

`redis::Client::open("redis://localhost")` 创建 Redis client。  
它描述如何连接 Redis。

然后：

````rust
bb8::Pool::builder().build(client).await.unwrap()
````

用这个 client 创建连接池。

可以这样理解：

```text
Client 知道怎么连接 Redis
Pool 管理多条 Redis 连接
handler 从 Pool 借连接使用
```

### Redis 连接池 vs 单连接多路复用

这里有个初学者常问的问题：Redis 必须用连接池吗？

答案是不一定。Redis 本身是**单线程**处理命令的，一条连接的吞吐其实很大。所以 Redis 生态里有两种主流用法：

1. **连接池（本章用法）**：bb8 维护多条连接，每次请求借一条。适合命令执行时间较长、想提高并发度的场景。
2. **`MultiplexedConnection`（单连接多路复用）**：只开一条底层 TCP 连接，但通过 futures 分流，让多个逻辑任务**共享这一条连接**并发执行。很多生产 Redis 代码用这种，因为它连接数少、更省资源。

`MultiplexedConnection` 的用法大致是：

````rust
let conn = client.get_multiplexed_async_connection().await?;
// conn 可以被 clone（共享底层连接），多个任务并发用
let conn_clone = conn.clone();
conn.set("a", 1).await?;
````

本章用 bb8 池是 example 的选择。真实项目里，如果你的 Redis 命令都很轻量，`MultiplexedConnection` 通常更简单。两种都可以，理解了"Redis 单线程，连接不是瓶颈"这个本质，选哪个都心里有数。

另外注意：`redis::Client` 本身**不持有连接**，它只是一个"连接工厂"，几乎零成本。真正持有连接的是 Pool 或 MultiplexedConnection。

## 第二步：启动前 ping Redis

源码：

````rust
{
    let mut conn = pool.get().await.unwrap();
    conn.set::<&str, &str, ()>("foo", "bar").await.unwrap();
    let result: String = conn.get("foo").await.unwrap();
    assert_eq!(result, "bar");
}
tracing::debug!("successfully connected to redis and pinged it");
````

这段做了启动前检查：

1. 从连接池借一个 Redis 连接。
2. 执行 `SET foo bar`。
3. 执行 `GET foo`。
4. 确认结果是 `bar`。

如果 Redis 没启动或连接失败，程序会在启动阶段失败。  
这样比服务启动后才发现 Redis 不可用更清晰。

外层花括号让 `conn` 尽快离开作用域，还回连接池。

## 第三步：AsyncCommands 提供 get/set

源码导入：

````rust
use redis::AsyncCommands;
````

这一行很重要。  
`set`、`get` 这些异步方法来自 `AsyncCommands` trait。

如果忘记导入，编译器可能会提示：

```text
no method named get
no method named set
```

本章使用：

````rust
conn.set::<&str, &str, ()>("foo", "bar").await.unwrap();
let result: String = conn.get("foo").await.unwrap();
````

`set` 里写了类型参数：

````rust
::<&str, &str, ()>
````

可以理解成：

```text
key 是 &str
value 是 &str
返回值忽略为 ()
```

## 第四步：定义连接池类型别名

源码：

````rust
type ConnectionPool = bb8::Pool<redis::Client>;
````

这个类型表示：

```text
bb8 管理的 Redis 连接池
```

后面 handler 可以写：

````rust
State(pool): State<ConnectionPool>
````

比每次写完整泛型更清楚。

## 第五步：注册 GET 和 POST

源码：

````rust
let app = Router::new()
    .route(
        "/",
        get(using_connection_pool_extractor).post(using_connection_extractor),
    )
    .with_state(pool);
````

两个接口都读 Redis：

```text
GET  / -> using_connection_pool_extractor
POST / -> using_connection_extractor
```

连接池通过 `.with_state(pool)` 交给 Axum。

## 第六步：GET 直接使用 State<ConnectionPool>

源码：

````rust
async fn using_connection_pool_extractor(
    State(pool): State<ConnectionPool>,
) -> Result<String, (StatusCode, String)> {
    let mut conn = pool.get().await.map_err(internal_error)?;
    let result: String = conn.get("foo").await.map_err(internal_error)?;
    Ok(result)
}
````

这段逻辑很直接：

1. 从 state 里拿连接池。
2. 从连接池借连接。
3. 执行 `GET foo`。
4. 返回结果字符串。

注意 `conn` 是 `mut`：

````rust
let mut conn = ...
````

因为 Redis 命令需要可变连接引用。

## 第七步：自定义 DatabaseConnection extractor

源码：

````rust
struct DatabaseConnection(bb8::PooledConnection<'static, redis::Client>);
````

虽然名字叫 `DatabaseConnection`，这里包住的是 Redis 连接。  
Redis 也常被称为一种数据库，只是它和 PostgreSQL 的数据模型不同。

extractor 实现：

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

和前几章数据库 extractor 一样：

```text
从 state 取 pool
从 pool 借 owned connection
包装成自定义 extractor
```

它不读取请求 body，所以实现 `FromRequestParts`。

## 第八步：POST 使用自定义连接 extractor

源码：

````rust
async fn using_connection_extractor(
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<String, (StatusCode, String)> {
    let result: String = conn.get("foo").await.map_err(internal_error)?;

    Ok(result)
}
````

handler 不再显式写 `State(pool)`。  
它直接声明自己需要：

````rust
DatabaseConnection(mut conn)
````

Axum 会自动运行 extractor，把 Redis 连接准备好。

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

这里统一处理：

- 连接池获取失败。
- Redis get/set 失败。

返回：

```text
500 Internal Server Error
```

真实项目里建议把详细错误写日志，响应只返回通用错误。

## 函数职责速查

- `main`：初始化日志，连接 Redis，创建连接池，启动前写入并读取 `foo`，注册路由并启动服务。
- `using_connection_pool_extractor`：从 `State<ConnectionPool>` 借 Redis 连接并读取 `foo`。
- `DatabaseConnection::from_request_parts`：从 state 取连接池并借出 owned Redis 连接。
- `using_connection_extractor`：通过自定义 extractor 读取 Redis key。
- `internal_error`：把内部错误转成 HTTP 500。

## 带中文注释的手写版

````rust
//! Redis + Axum 示例。
//!
//! ```not_rust
//! cargo run -p example-tokio-redis
//! ```

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
    // 初始化日志。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 创建 Redis client 和 bb8 连接池。
    tracing::debug!("connecting to redis");
    let client = redis::Client::open("redis://localhost").unwrap();
    let pool = bb8::Pool::builder().build(client).await.unwrap();

    {
        // 服务启动前先验证 Redis 可用。
        let mut conn = pool.get().await.unwrap();

        // 写入 foo = bar。
        conn.set::<&str, &str, ()>("foo", "bar").await.unwrap();

        // 再读出 foo。
        let result: String = conn.get("foo").await.unwrap();
        assert_eq!(result, "bar");
    }
    tracing::debug!("successfully connected to redis and pinged it");

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

// Redis 连接池类型别名。
type ConnectionPool = bb8::Pool<redis::Client>;

// 写法一：直接从 State 中拿连接池。
async fn using_connection_pool_extractor(
    State(pool): State<ConnectionPool>,
) -> Result<String, (StatusCode, String)> {
    let mut conn = pool.get().await.map_err(internal_error)?;
    let result: String = conn.get("foo").await.map_err(internal_error)?;
    Ok(result)
}

// 自定义 extractor，包住一个 Redis 连接。
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

// 写法二：通过自定义 extractor 拿 Redis 连接。
async fn using_connection_extractor(
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<String, (StatusCode, String)> {
    let result: String = conn.get("foo").await.map_err(internal_error)?;

    Ok(result)
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

先启动 Redis。  
默认连接地址是：

```text
redis://localhost
```

运行：

````bash
cargo run -p example-tokio-redis
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
bar
```

如果启动失败，优先检查：

```text
Redis 是否启动
本机 6379 端口是否可访问
Redis 是否需要密码
连接地址是否应该改成 redis://:password@localhost
```

## 常见卡点

### 1. Redis 需要建表吗？

不需要。  
本章只是 key-value：

```text
SET foo bar
GET foo
```

Redis 不像 PostgreSQL 那样先建表。

### 2. 为什么要导入 AsyncCommands？

`get` 和 `set` 方法来自 `redis::AsyncCommands` trait。  
忘记导入时，方法可能找不到。

### 3. 为什么 Redis 也用连接池？

高并发服务里，每个请求都新建 Redis 连接也有成本。  
连接池可以复用连接，减少连接建立开销。

### 4. Redis 数据会不会永久保存？

取决于 Redis 配置。  
Redis 常作为内存数据库使用，持久化策略、过期时间、重启行为都要按项目配置。

### 5. Redis 能替代 PostgreSQL 吗？

通常不能。  
Redis 更适合缓存和快速 key-value 场景，PostgreSQL 更适合长期结构化数据和复杂查询。

## 手写任务

建议按下面顺序自己敲一遍：

1. 启动 Redis。
2. 创建 `redis::Client::open("redis://localhost")`。
3. 用 `bb8::Pool::builder().build(client).await` 创建连接池。
4. 启动时用 `set foo bar` 和 `get foo` 验证连接。
5. 把 pool 放进 `.with_state(pool)`。
6. 写 GET handler，用 `State<ConnectionPool>` 读取 `foo`。
7. 实现 `DatabaseConnection` extractor。
8. 写 POST handler，通过 extractor 读取 `foo`。

加深练习：

1. 新增 `POST /cache/{key}/{value}` 写入任意 key。
2. 新增 `GET /cache/{key}` 读取任意 key。
3. 给 key 设置过期时间，理解缓存失效。

## 本章真正要记住什么

Redis 接入 Axum 的核心模型和数据库连接池很像：

```text
启动时创建 client 和 pool
Router state 保存 pool
handler 从 pool 借连接
用 AsyncCommands 执行 Redis 命令
把结果返回给客户端
```

但 Redis 的使用场景不同。  
先把它当成：

```text
高速 key-value 存储
```

等理解 GET/SET 后，再继续学习过期时间、计数器、列表、集合、发布订阅等能力。

## 源码对照

本章手写版对应源码：

- `examples/tokio-redis/src/main.rs`
- `examples/tokio-redis/Cargo.toml`
