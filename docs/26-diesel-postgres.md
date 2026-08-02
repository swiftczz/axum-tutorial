# 第 26 章 diesel-postgres:ORM 与同步库的异步困境

## 本章你将做出什么

一个用 **diesel** 操作数据库的服务:

```
GET  /user/list     列出所有用户
POST /user/create   创建用户
```

和前两章最大的不同是:**这一章不写 SQL 字符串**。查询用 Rust 的类型和方法构造,SQL 由 diesel 生成。

还有一个绕不开的问题:**diesel 是同步库,而 axum 是异步的**。这一章会讲清这个矛盾怎么解决,以及为什么它值得单独占一章。

学完本章你将掌握:

- ORM 与 SQL 优先方案的根本差别
- diesel 的 schema 定义与类型安全查询
- **迁移(migration)是什么,以及为什么它比手动建表重要**
- 同步库在异步运行时里的正确用法:`spawn_blocking` 与 `interact`
- 三层错误嵌套:`Result<Result<T, E1>, E2>` 从哪来

## 前置知识

### ORM:用类型系统表达查询

前两章都是"写 SQL 字符串,库负责执行"。diesel 走的是另一条路——**查询本身是 Rust 代码**:

```rust
users::table
    .filter(users::name.eq("张三"))
    .limit(10)
    .load::<User>(conn)
```

diesel 把它翻译成:

```sql
SELECT id, name, hair_color FROM users WHERE name = $1 LIMIT 10
```

**好处是编译期检查覆盖面更广:**

- 表名、列名写错 → 编译失败(它们是 Rust 的标识符)
- 类型不匹配(拿 `i32` 和 `text` 列比较)→ 编译失败
- 用了不存在的字段 → 编译失败
- 拼错 SQL 关键字 → 不可能,因为你不写 SQL

[第 24 章](24-sqlx-postgres.md#sqlx不是-orm)讲的 sqlx `query!` 宏也有编译期校验,但两者机制不同:

| | sqlx `query!` | diesel |
|---|---|---|
| 校验方式 | 编译时连数据库校验 SQL | 纯类型系统推导 |
| 编译时需要数据库 | **需要**(或离线缓存) | 不需要 |
| 能表达的查询 | 任意 SQL | 受 DSL 限制 |
| 复杂 SQL(窗口函数、CTE) | 直接写 | 困难或需要 `sql_query` |
| 学习曲线 | 会 SQL 就会用 | 要学 DSL |

**diesel 的代价是表达力。** 简单的增删改查很舒服,但遇到复杂查询——多表 JOIN 加聚合、窗口函数、递归 CTE——DSL 会变得非常难写,甚至写不出来,最后还是要用 `diesel::sql_query` 回到写 SQL。

**这是选型的核心权衡**:diesel 用表达力换取更强的编译期保障和不依赖数据库的构建过程。

### schema:表结构的 Rust 表示

diesel 需要知道表结构才能做类型检查。这个信息用 `table!` 宏声明:

```rust
table! {
    users (id) {
        id -> Integer,
        name -> Text,
        hair_color -> Nullable<Text>,
    }
}
```

`users (id)` 表示表名和主键。每个字段声明它的 **SQL 类型**(不是 Rust 类型):`Integer`、`Text`、`Nullable<Text>`。

`Nullable<T>` 对应 SQL 的可空列,映射到 Rust 的 `Option<T>`。

**正常情况下这段代码是生成的,不是手写的。** diesel 的 CLI 工具会连上数据库,把表结构导出成 `src/schema.rs`:

```bash
diesel print-schema > src/schema.rs
```

官方示例的注释就写着 `// normally part of your generated schema.rs file` —— 它把生成的内容内联进 `main.rs` 只是为了让示例是单文件的。

**真实项目的流程是**:改迁移文件 → 跑迁移 → 重新生成 schema.rs → 编译。schema 和数据库结构由工具保证一致。

### 三个派生宏

```rust
#[derive(serde::Serialize, HasQuery)]
struct User {
    id: i32,
    name: String,
    hair_color: Option<String>,
}

#[derive(serde::Deserialize, Insertable)]
#[diesel(table_name = users)]
struct NewUser {
    name: String,
    hair_color: Option<String>,
}
```

**`HasQuery`** 让 `User` 能作为查询结果:提供 `User::query()` 构造查询、`User::as_returning()` 用于 `RETURNING` 子句。字段名和类型必须能对应上 schema 里的声明。

**`Insertable`** 让 `NewUser` 能被插入。`#[diesel(table_name = users)]` 指明插入到哪张表。

**注意 `User` 和 `NewUser` 是两个类型。** 这和[第 23 章](23-todos.md#步骤-2创建)讲的"请求类型和响应类型分开"是同一个道理:`NewUser` 没有 `id`,因为 id 由数据库生成,客户端不该指定。**diesel 用类型强制了这一点** —— 你根本没法把 `User` 插进去。

### 迁移:数据库结构的版本控制

这一章第一次出现**迁移**(migration),它比"手动执行 SQL 建表"重要得多。

迁移是一系列有序的 SQL 脚本,每个描述一次结构变更:

```
migrations/
└── 2023-03-14-180127_add_users/
    ├── up.sql     -- 应用这次变更
    └── down.sql   -- 撤销这次变更
```

`up.sql`:

```sql
CREATE TABLE "users"(
    "id" SERIAL PRIMARY KEY,
    "name" TEXT NOT NULL,
    "hair_color" TEXT
);
```

`down.sql`:

```sql
DROP TABLE "users";
```

**为什么需要它:**

**数据库结构要和代码一起进版本控制。** 手动建表的话,新同事拿到代码不知道该建什么表,测试环境和生产环境会慢慢分叉。

**变更要可追溯、可重放。** 迁移文件带时间戳前缀,按顺序执行。任何环境从空库开始跑一遍迁移,都能得到一致的结构。

**要能回滚。** `down.sql` 让失败的发布可以撤销。

diesel 用一张 `__diesel_schema_migrations` 表记录已执行的迁移版本,所以重复运行是安全的——已经跑过的会跳过。

**`embed_migrations!` 把迁移嵌进二进制:**

```rust
pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!("migrations/");
```

这个宏在**编译期**读取 `migrations/` 目录,把所有 SQL 变成字符串常量嵌入程序。[第 05 章](05-templates-minijinja.md#include_str-把文件内容嵌进二进制)的 `include_str!` 是同样的思路。

好处是**部署只需要一个二进制**,不用额外拷贝迁移目录。

### diesel 是同步的,这是个大问题

**这是本章最核心的话题。**

diesel 2.x 的核心 API 是**同步阻塞**的:`conn.execute()`、`query.load(conn)` 都会阻塞当前线程直到数据库返回。

而 axum 跑在 tokio 上,handler 是异步函数。在异步函数里直接调用阻塞操作会造成严重问题:

**tokio 的工作线程数量有限**(默认等于 CPU 核数)。一个线程被阻塞时,它上面排队的**所有其它任务**都跑不了——哪怕那些任务只是想返回一个静态字符串。

假设 8 核机器、8 个工作线程,同时来了 8 个数据库查询,每个阻塞 50ms:整个服务在这 50ms 内**完全无响应**,连健康检查都超时。

这和[第 22 章](22-key-value-store.md#持锁跨-await能让服务卡死的陷阱)那个"持锁跨 await"是同一类问题的两种表现:**都是在异步上下文里长时间占用工作线程。** 区别是那个有编译错误挡着,这个**编译完全通过**,只在压测或线上才暴露。

**标准解法是 `tokio::task::spawn_blocking`:**

```rust
tokio::task::spawn_blocking(move || {
    // 这里可以放心阻塞
    diesel_query(&mut conn)
}).await
```

它把闭包丢到一个**专门的阻塞线程池**(默认最多 512 个线程)执行,不占用工作线程。

**本章示例用的 `deadpool-diesel` 把这层封装成了 `interact`:**

```rust
conn.interact(|conn| {
    diesel::insert_into(users::table).values(new_user).get_result(conn)
}).await
```

`interact` 内部就是 `spawn_blocking` 加连接管理。**闭包里的代码在阻塞线程池执行**,外面的 `.await` 等它完成。

### 三层错误从哪来

`interact` 的调用后面跟着两个 `map_err`:

```rust
let res = conn
    .interact(|conn| { /* ... */ })
    .await
    .map_err(internal_error)?      // 第一层:interact 本身失败
    .map_err(internal_error)?;     // 第二层:闭包里的 diesel 查询失败
```

看起来很怪,但拆开就清楚了。`interact` 返回:

```rust
Result<Result<T, diesel::result::Error>, InteractError>
```

- **外层 `InteractError`** —— `interact` 这个机制本身出了问题:闭包 panic 了、连接池关闭了、任务被取消了
- **内层 `diesel::result::Error`** —— 闭包正常执行了,但查询失败:违反约束、表不存在、类型转换失败

**两层错误的语义完全不同**,所以不能合并。第一个 `map_err` 处理外层并解包,第二个处理内层。

这是"把同步代码包进异步"必然带来的复杂度——多一层抽象就多一层可能的失败。

## 项目搭建

### 数据库

沿用[第 24 章](24-sqlx-postgres.md#启动-postgresql)的容器,但需要一个专用数据库:

```bash
docker exec -it axum-tutorial-pg psql -U postgres -c 'create database diesel_demo;'
```

如果容器已经删了:

```bash
docker run --name axum-tutorial-pg \
  -e POSTGRES_PASSWORD=password \
  -p 5432:5432 \
  -d postgres:16
sleep 3
docker exec -it axum-tutorial-pg psql -U postgres -c 'create database diesel_demo;'
```

### 创建项目

```bash
cargo new diesel-postgres
cd diesel-postgres
```

```bash
cargo add axum --features macros
cargo add diesel --features postgres
cargo add diesel_migrations
cargo add deadpool-diesel --features postgres
cargo add serde --features derive
cargo add tokio --features full
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

**注意 `diesel` 需要系统上有 PostgreSQL 的客户端库**(`libpq`)。缺了会在编译时报链接错误:

```bash
# macOS
brew install libpq && brew link --force libpq

# Debian/Ubuntu
sudo apt install libpq-dev
```

这是 diesel 相对 sqlx / tokio-postgres 的一个额外门槛——**后两者是纯 Rust 实现,不依赖系统库**。

### 迁移文件

```bash
mkdir -p migrations/2023-03-14-180127_add_users
```

`migrations/2023-03-14-180127_add_users/up.sql`:

```sql
-- Your SQL goes here
CREATE TABLE "users"(
    "id" SERIAL PRIMARY KEY,
    "name" TEXT NOT NULL,
    "hair_color" TEXT
);
```

`migrations/2023-03-14-180127_add_users/down.sql`:

```sql
-- This file should undo anything in "up.sql"
DROP TABLE "users";
```

**目录名的格式是 `时间戳_名称`**,diesel 按它排序决定执行顺序。真实项目用 `diesel migration generate add_users` 生成,时间戳自动填。

### 环境变量

示例用 `std::env::var("DATABASE_URL").unwrap()`,**没有默认值**——不设置直接 panic。

```bash
export DATABASE_URL='postgres://postgres:password@localhost/diesel_demo'
```

## 分步构建

### 步骤 1:schema、模型与连接池

**要做什么**:声明表结构、定义两个模型类型、建立连接池并跑迁移。

```rust
// src/main.rs
use axum::{routing::get, Router};
use diesel::prelude::*;
use diesel_migrations::{embed_migrations, EmbeddedMigrations, MigrationHarness};
use std::net::SocketAddr;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// this embeds the migrations into the application binary
// the migration path is relative to the `CARGO_MANIFEST_DIR`
pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!("migrations/");

// normally part of your generated schema.rs file
table! {
    users (id) {
        id -> Integer,
        name -> Text,
        hair_color -> Nullable<Text>,
    }
}

#[derive(serde::Serialize, HasQuery)]
struct User {
    id: i32,
    name: String,
    hair_color: Option<String>,
}

#[derive(serde::Deserialize, Insertable)]
#[diesel(table_name = users)]
struct NewUser {
    name: String,
    hair_color: Option<String>,
}

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let db_url = std::env::var("DATABASE_URL").unwrap();

    // set up connection pool
    let manager = deadpool_diesel::postgres::Manager::new(db_url, deadpool_diesel::Runtime::Tokio1);
    let pool = deadpool_diesel::postgres::Pool::builder(manager)
        .build()
        .unwrap();

    // run the migrations on server startup
    {
        let conn = pool.get().await.unwrap();
        conn.interact(|conn| conn.run_pending_migrations(MIGRATIONS).map(|_| ()))
            .await
            .unwrap()
            .unwrap();
    }

    let app = Router::new()
        .route("/user/list", get(|| async { "not implemented yet" }))
        .with_state(pool);

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {addr}");
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

**`use diesel::prelude::*`** 一次导入 diesel 的常用 trait 和宏。ORM 库常这么做,因为查询 DSL 依赖大量 trait 方法。

**`deadpool_diesel::Runtime::Tokio1`** 告诉连接池用 tokio 作为运行时——它需要知道往哪个阻塞线程池丢任务。

**启动时跑迁移**,那个块表达式值得注意:

```rust
{
    let conn = pool.get().await.unwrap();
    conn.interact(|conn| conn.run_pending_migrations(MIGRATIONS).map(|_| ()))
        .await.unwrap().unwrap();
}
```

用块把它框起来,是为了让 `conn` 在块结束时归还连接池。这和[第 22 章](22-key-value-store.md#持锁跨-await能让服务卡死的陷阱)缩短临界区是同一个习惯。

两个 `.unwrap()` 就是前置知识讲的双层错误。启动阶段直接 panic 是对的——**迁移跑不通的服务不该启动**。

> **更好的写法**:启动时自动跑迁移对示例和小项目很方便,但在多实例部署时有隐患:多个实例同时启动会**并发执行迁移**。diesel 用数据库锁做了保护,不会损坏数据,但可能出现启动超时。更稳妥的做法是把迁移放在部署流程的独立步骤(CI 里执行 `diesel migration run`,或者用一个专门的 init container),应用启动时只校验版本是否匹配。

**跑起来看**:

```bash
export DATABASE_URL='postgres://postgres:password@localhost/diesel_demo'
cargo run
```

```
listening on 127.0.0.1:3000
```

验证迁移真的执行了:

```bash
docker exec -it axum-tutorial-pg psql -U postgres -d diesel_demo -c '\dt'
```

```
                 List of relations
 Schema |             Name              | Type  |  Owner
--------+-------------------------------+-------+----------
 public | __diesel_schema_migrations    | table | postgres
 public | users                         | table | postgres
```

**两张表:`users` 是迁移建的,`__diesel_schema_migrations` 是 diesel 用来记录版本的。**

看看记录了什么:

```bash
docker exec -it axum-tutorial-pg psql -U postgres -d diesel_demo -c 'select * from __diesel_schema_migrations;'
```

```
    version     |         run_on
----------------+-------------------------
 20230314180127 | 2026-07-30 12:03:11.221
```

**再跑一次 `cargo run`,不会重复建表** —— diesel 看到这条记录就跳过了。这就是迁移可重放的机制。

**验证 schema 的编译期检查。** 临时把 `User` 的字段名写错:

```rust
struct User {
    id: i32,
    nmae: String,        // 临时:拼错
    hair_color: Option<String>,
}
```

```bash
cargo build
```

编译失败,错误会指出 `nmae` 在 `users` 表里不存在。**这就是 ORM 的价值** —— 列名是 Rust 标识符,拼错就编译不过。改回来。

### 步骤 2:查询列表

**要做什么**:实现 `GET /user/list`,第一次用 diesel 的查询 DSL。

```rust
// src/main.rs
use axum::{
    extract::State,                          // 新增
    http::StatusCode,                        // 新增
    response::Json,                          // 新增
    routing::get,
    Router,
};
// ...(其余 use 不变)

#[tokio::main]
async fn main() {
    // ...(连接池和迁移不变)

    // 改动:挂上真正的 handler
    let app = Router::new()
        .route("/user/list", get(list_users))
        .with_state(pool);

    // ...
}

// 新增
async fn list_users(
    State(pool): State<deadpool_diesel::postgres::Pool>,
) -> Result<Json<Vec<User>>, (StatusCode, String)> {
    let conn = pool.get().await.map_err(internal_error)?;
    let res = conn
        .interact(|conn| User::query().load(conn))
        .await
        .map_err(internal_error)?
        .map_err(internal_error)?;
    Ok(Json(res))
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

**`User::query().load(conn)`** 是 `HasQuery` 派生宏提供的:构造一个"查询 `users` 表所有列并映射到 `User`"的查询,`load` 执行它返回 `Vec<User>`。

对比手写 SQL 的版本([第 24 章](24-sqlx-postgres.md)):

```rust
sqlx::query_as::<_, User>("select id, name, hair_color from users").fetch_all(&pool).await
```

diesel 版本不需要写列名,**加字段时不用改查询**。

**`conn.interact(...)`** 是前置知识讲的关键:闭包里的同步 diesel 代码跑在阻塞线程池上。

**两个 `map_err` 就是那两层错误。** 注意它们用的是同一个 `internal_error` 函数——[第 24 章](24-sqlx-postgres.md#步骤-1连上数据库用-state-拿池子)那个泛型辅助函数,这里直接复用。

> **更好的写法**:和[第 24](24-sqlx-postgres.md#步骤-1连上数据库用-state-拿池子)、[25 章](25-tokio-postgres.md)一样,`err.to_string()` 把数据库错误直接发给了客户端——会泄露表名、约束名。生产环境应响应固定文本、详情记日志。

**跑起来看**:

```bash
cargo run
```

```bash
curl -s http://127.0.0.1:3000/user/list
```

```json
[]
```

空表。手动插一条再看:

```bash
docker exec -it axum-tutorial-pg psql -U postgres -d diesel_demo \
  -c "insert into users (name, hair_color) values ('张三', '黑色'), ('李四', null);"
curl -s http://127.0.0.1:3000/user/list
```

```json
[{"id":1,"name":"张三","hair_color":"黑色"},{"id":2,"name":"李四","hair_color":null}]
```

**`hair_color` 是 `Nullable<Text>`,映射到 `Option<String>`,序列化成 JSON 就是 `null`。** 类型链条从数据库一路贯通到 JSON。

**验证 DSL 的类型检查。** 临时写一个类型不匹配的查询:

```rust
        .interact(|conn| {
            users::table.filter(users::id.eq("not a number")).load::<User>(conn)
        })
```

```bash
cargo build
```

编译失败——`users::id` 是 `Integer`,不能和字符串比较。**这类错误在 sqlx 的运行时版本和 tokio-postgres 里都要到运行时才发现。** 改回去。

### 步骤 3:插入数据

**要做什么**:实现 `POST /user/create`。

```rust
// src/main.rs
use axum::{
    extract::State,
    http::StatusCode,
    response::Json,
    routing::{get, post},                    // 改动:加上 post
    Router,
};
// ...

#[tokio::main]
async fn main() {
    // ...

    // 改动:加上创建路由
    let app = Router::new()
        .route("/user/list", get(list_users))
        .route("/user/create", post(create_user))
        .with_state(pool);

    // ...
}

// 新增
async fn create_user(
    State(pool): State<deadpool_diesel::postgres::Pool>,
    Json(new_user): Json<NewUser>,
) -> Result<Json<User>, (StatusCode, String)> {
    let conn = pool.get().await.map_err(internal_error)?;
    let res = conn
        .interact(|conn| {
            diesel::insert_into(users::table)
                .values(new_user)
                .returning(User::as_returning())
                .get_result(conn)
        })
        .await
        .map_err(internal_error)?
        .map_err(internal_error)?;
    Ok(Json(res))
}

// ...(list_users、internal_error 不变)
```

**`insert_into(users::table).values(new_user)`** —— `new_user` 是 `NewUser` 类型,靠 `Insertable` 派生宏知道怎么映射到列。

**`.returning(User::as_returning())`** 生成 SQL 的 `RETURNING` 子句,让插入直接返回完整的新行(包括数据库生成的 `id`)。**这比"插入后再查一次"少一次往返**,而且没有竞态。

`RETURNING` 是 PostgreSQL 的特性,MySQL 不支持——那边要用 `last_insert_id()`。

**`.get_result(conn)`** 执行并取一行结果。相关的还有 `execute`(不要结果)、`get_results`(多行)。

**`Json(new_user)` 必须是最后一个参数**([第 03 章](03-form.md#提取器axum-的核心机制)的规则)。

**跑起来看**:

```bash
cargo run
```

```bash
curl -s -X POST http://127.0.0.1:3000/user/create \
  -H 'Content-Type: application/json' \
  -d '{"name":"王五","hair_color":"棕色"}'
```

```json
{"id":3,"name":"王五","hair_color":"棕色"}
```

**返回了数据库生成的 `id`** —— `RETURNING` 的效果。

不带 `hair_color`(它是可空的):

```bash
curl -s -X POST http://127.0.0.1:3000/user/create \
  -H 'Content-Type: application/json' \
  -d '{"name":"赵六"}'
```

```json
{"id":4,"name":"赵六","hair_color":null}
```

`Option<String>` 字段缺席就是 `None`。

确认列表更新了:

```bash
curl -s http://127.0.0.1:3000/user/list
```

**验证类型强制的价值。** 试试在创建时指定 `id`:

```bash
curl -s -X POST http://127.0.0.1:3000/user/create \
  -H 'Content-Type: application/json' \
  -d '{"id":999,"name":"测试"}'
```

```json
{"id":5,"name":"测试","hair_color":null}
```

**`id: 999` 被忽略了** —— 因为 `NewUser` 结构体里根本没有 `id` 字段,serde 默认忽略多余的键。前置知识讲的"两个类型分开"在这里体现了价值:**客户端想指定 id 也做不到**。

## 完整代码

`migrations/2023-03-14-180127_add_users/up.sql`:

```sql
-- Your SQL goes here
CREATE TABLE "users"(
    "id" SERIAL PRIMARY KEY,
    "name" TEXT NOT NULL,
    "hair_color" TEXT
);
```

`migrations/2023-03-14-180127_add_users/down.sql`:

```sql
-- This file should undo anything in "up.sql"
DROP TABLE "users";
```

`src/main.rs`:

```rust
use axum::{
    extract::State,
    http::StatusCode,
    response::Json,
    routing::{get, post},
    Router,
};
use diesel::prelude::*;
use diesel_migrations::{embed_migrations, EmbeddedMigrations, MigrationHarness};
use std::net::SocketAddr;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// this embeds the migrations into the application binary
// the migration path is relative to the `CARGO_MANIFEST_DIR`
pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!("migrations/");

// normally part of your generated schema.rs file
table! {
    users (id) {
        id -> Integer,
        name -> Text,
        hair_color -> Nullable<Text>,
    }
}

#[derive(serde::Serialize, HasQuery)]
struct User {
    id: i32,
    name: String,
    hair_color: Option<String>,
}

#[derive(serde::Deserialize, Insertable)]
#[diesel(table_name = users)]
struct NewUser {
    name: String,
    hair_color: Option<String>,
}

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let db_url = std::env::var("DATABASE_URL").unwrap();

    // set up connection pool
    let manager = deadpool_diesel::postgres::Manager::new(db_url, deadpool_diesel::Runtime::Tokio1);
    let pool = deadpool_diesel::postgres::Pool::builder(manager)
        .build()
        .unwrap();

    // run the migrations on server startup
    {
        let conn = pool.get().await.unwrap();
        conn.interact(|conn| conn.run_pending_migrations(MIGRATIONS).map(|_| ()))
            .await
            .unwrap()
            .unwrap();
    }

    // build our application with some routes
    let app = Router::new()
        .route("/user/list", get(list_users))
        .route("/user/create", post(create_user))
        .with_state(pool);

    // run it with hyper
    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {addr}");
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn create_user(
    State(pool): State<deadpool_diesel::postgres::Pool>,
    Json(new_user): Json<NewUser>,
) -> Result<Json<User>, (StatusCode, String)> {
    let conn = pool.get().await.map_err(internal_error)?;
    let res = conn
        .interact(|conn| {
            diesel::insert_into(users::table)
                .values(new_user)
                .returning(User::as_returning())
                .get_result(conn)
        })
        .await
        .map_err(internal_error)?
        .map_err(internal_error)?;
    Ok(Json(res))
}

async fn list_users(
    State(pool): State<deadpool_diesel::postgres::Pool>,
) -> Result<Json<Vec<User>>, (StatusCode, String)> {
    let conn = pool.get().await.map_err(internal_error)?;
    let res = conn
        .interact(|conn| User::query().load(conn))
        .await
        .map_err(internal_error)?
        .map_err(internal_error)?;
    Ok(Json(res))
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

环境变量:`DATABASE_URL`(必需,没有默认值)

## 代价与边界

**同步库的额外开销。** 每次查询都要经过 `spawn_blocking`——跨线程调度、闭包移动、结果传回。单次开销是微秒级,但在极高 QPS 下会累积。而且阻塞线程池的线程数(默认 512)是另一个需要调优的参数。

**`interact` 的闭包不能捕获引用。** 闭包要 `'static`,所以外面的数据必须 `move` 进去或者克隆。这在需要传多个参数时会有点别扭。

**双层错误让代码变啰嗦。** 每次查询两个 `map_err`。真实项目通常封装一个辅助函数把两层压平。

**依赖系统的 libpq。** 项目搭建里提过,这会让 Docker 镜像构建和交叉编译变复杂。sqlx 和 tokio-postgres 是纯 Rust,没这个问题。

**DSL 的表达力天花板。** 前置知识讲过。复杂查询要么写得极其别扭,要么回退到 `diesel::sql_query` —— 那就失去了类型安全,还不如一开始就用 sqlx。

**启动时跑迁移的多实例问题。** 步骤 1 的「更好的写法」指出了。

**错误信息泄露。** 和前两章相同的问题。

**`schema.rs` 手动内联。** 示例把 `table!` 写在 `main.rs` 里,真实项目应该用 `diesel print-schema` 生成独立文件,并在 CI 里校验它和数据库一致——**手写的 schema 迟早会和数据库分叉**。

## 动手练习

**1. 加一条按名字查询的路由。**

提示:`GET /user/{name}`,用 `users::table.filter(users::name.eq(name)).first::<User>(conn).optional()`。`.optional()` 把"没找到"从错误变成 `Ok(None)`,正好对应 `404`。注意闭包要 `move` 捕获 `name`。

**2. 把双层错误压平成一个辅助函数。**

提示:写一个 `async fn run_query<T, F>(pool, f) -> Result<T, (StatusCode, String)>`,内部完成 `pool.get()`、`interact`、两次 `map_err`。两个 handler 各自就能缩到三行。做完对比一下可读性。

**3. 加一个迁移,给 users 表增加一列。**

提示:新建 `migrations/2026-01-01-000000_add_email/`,`up.sql` 写 `ALTER TABLE users ADD COLUMN email TEXT;`,`down.sql` 写 `ALTER TABLE users DROP COLUMN email;`。然后在 `table!` 里加 `email -> Nullable<Text>`,两个 struct 也加字段。重启服务观察迁移自动执行,再查 `__diesel_schema_migrations` 表看多了一条记录。**这个练习能完整体会迁移的工作流。**

## 小结与下一章预告

diesel 是真正的 ORM:查询用 Rust 类型和方法构造,SQL 由库生成。**表名、列名、类型不匹配全在编译期发现**,而且不像 sqlx 的宏那样需要编译时连数据库。代价是 DSL 的表达力有天花板,复杂查询会很难写,以及依赖系统的 libpq。

**迁移是数据库结构的版本控制**:带时间戳的 `up.sql`/`down.sql`,靠 `__diesel_schema_migrations` 表记录已执行版本,任何环境从空库跑一遍都能得到一致结构。`embed_migrations!` 把它们嵌进二进制,部署只需一个文件。

本章最核心的话题是**同步库在异步运行时里的困境**:diesel 的 API 是阻塞的,直接在 handler 里调用会占满 tokio 的工作线程,让整个服务卡住。而且**编译完全通过**,只在压测或线上暴露。解法是 `spawn_blocking`,`deadpool-diesel` 把它封装成了 `interact`——代价是多一层抽象,于是有了 `Result<Result<T, E1>, E2>` 那两层错误。

[第 27 章 diesel-async-postgres](27-diesel-async-postgres.md) 用 `diesel-async` 消掉这个困境:同样的 DSL、同样的 schema、同样的迁移,但查询是原生异步的——不再需要 `interact`,双层错误也变回一层。那一章会直接对比两份代码,看看"让 ORM 支持异步"这件事到底改变了什么。
