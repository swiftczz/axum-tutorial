# 第 16 章 dependency-injection:trait 对象还是泛型

## 本章你将做出什么

一个用户存储服务,`POST /users` 创建、`GET /users/{id}` 查询。关键在于:**同一套业务逻辑挂了两遍**,一遍用 trait 对象注入依赖,一遍用泛型:

```
POST /dyn/users        GET /dyn/users/{id}
POST /generic/users    GET /generic/users/{id}
```

两组路由行为完全一致,差别只在依赖是怎么传进 handler 的。这一章把两种做法摆在一起,讲清各自的代价。

学完本章你将掌握:

- 依赖注入在 Rust Web 服务里具体解决什么问题
- 用 `Arc<dyn Trait>` 注入:写法、动态分发的开销、对象安全的限制
- 用 `T: Trait` 注入:写法、单态化的代价、turbofish 为什么是必需的
- `Router::nest` 挂载子路由,以及它和 `merge` 的区别
- 官方为什么推荐 trait 对象

## 前置知识

### 依赖注入解决什么问题

[第 05 章](05-templates-minijinja.md#为什么需要共享状态)讲 `State` 时解决的是"怎么把东西传给 handler"。这一章要解决的是下一层问题:**传进去的东西能不能换。**

假设 handler 直接用了具体类型:

```rust
async fn create_user(State(db): State<PostgresPool>, ...) { ... }
```

这段代码和 PostgreSQL **焊死**了。带来的麻烦是实际的:

**测试必须连真库。** 想测"创建用户"这段逻辑,得先起一个 PostgreSQL、建表、清数据。测试变慢、变脆,还没法在 CI 里轻松跑。

**换实现要改所有 handler。** 哪天要加一层缓存,或者迁到别的数据库,每个用到它的 handler 都要改签名。

**没法在开发期用假数据。** 前端联调时想返回固定的假用户,只能改代码。

**依赖注入**的思路是:**handler 只依赖一个接口(trait),具体用哪个实现由外部决定。**

```rust
trait UserRepo {
    fn get_user(&self, id: Uuid) -> Option<User>;
    fn save_user(&self, user: &User);
}
```

handler 只知道"有个东西能存取用户",不知道它是 PostgreSQL、Redis、内存 HashMap 还是测试用的假实现。生产环境注入真实现,测试注入假实现,handler 一个字不用改。

Java、C# 那些语言有专门的 DI 框架(Spring、Autofac)在运行时组装依赖。**Rust 不需要框架**——trait 加上 `State` 就够了,组装发生在 `main` 里,而且是编译期检查的。

### 两种注入方式

Rust 里"接受任何实现了某 trait 的类型"有两种表达,这是本章的核心对比。

**方式一:trait 对象(`dyn Trait`)。**

```rust
struct AppStateDyn {
    user_repo: Arc<dyn UserRepo>,
}
```

`dyn UserRepo` 表示"某个实现了 `UserRepo` 的类型,具体是哪个运行时才知道"。它的大小编译期未知,所以必须放在指针后面——这里用 `Arc`,顺便满足 `State` 要求的 `Clone`([第 05 章](05-templates-minijinja.md#state-与-with_state)讲过为什么)。

调用 `user_repo.get_user(id)` 时走**动态分发**:通过虚表(vtable)在运行时查到实际函数地址再跳转。

**方式二:泛型(`T: Trait`)。**

```rust
struct AppStateGeneric<T> {
    user_repo: T,
}
```

`T` 是具体类型,编译期确定。编译器为每个用到的 `T` **单态化**(monomorphize)出一份独立的代码,调用是直接的静态分发。

### 两者的代价对比

官方示例的注释直接列了取舍,展开说:

| | trait 对象 `Arc<dyn T>` | 泛型 `T: Trait` |
|---|---|---|
| 分发方式 | 动态(虚表跳转) | 静态(直接调用) |
| 运行时开销 | 有,但通常可忽略 | 无 |
| 能否内联优化 | 基本不能 | 能 |
| 类型参数扩散 | 无 | **会传染** |
| 编译产物大小 | 小 | 每个 `T` 一份,更大 |
| 编译速度 | 快 | 慢 |
| trait 的限制 | 必须**对象安全** | 无限制 |
| 运行时切换实现 | 能(存不同的 `Arc`) | 不能(类型编译期定死) |

**"类型参数扩散"是泛型最实际的代价。** `AppStateGeneric<T>` 一旦带上 `T`,所有碰它的东西都要跟着带:handler 签名要写 `<T>` 和 `where T: UserRepo`,存放它的结构体要带 `T`,返回它的函数也要带。项目一大,类型参数会顺着调用链爬满整个代码库。

**"对象安全"是 trait 对象的限制。** 一个 trait 要能写成 `dyn Trait`,得满足若干条件,常见的绊脚石是:

- 方法不能有泛型参数(`fn foo<T>(&self, x: T)`)
- 方法不能返回 `Self`
- 不能有关联常量

本章的 `UserRepo` 两个方法都很朴素,所以没问题。但如果你的 trait 需要泛型方法,就只能用泛型注入了。

**官方的结论是:除非确实需要泛型,否则用 trait 对象。** 理由是动态分发那点开销("likely to be negligible")远不如类型参数扩散带来的复杂度值得担心。这也是本章最该记住的一句话——**默认选简单的那个,除非有具体理由**。

### async trait 的现状

本章的 `UserRepo` 两个方法都是**同步**的:

```rust
trait UserRepo: Send + Sync {
    fn get_user(&self, id: Uuid) -> Option<User>;
    fn save_user(&self, user: &User);
}
```

真实的数据库操作显然是异步的,那样 trait 就得写成 `async fn get_user(...)`。Rust 1.75 起 trait 里可以直接写 `async fn`,但**带 `async fn` 的 trait 目前不是对象安全的**——不能写成 `dyn UserRepo`。

绕过的办法有两个:

- 用 `async-trait` crate(把 `async fn` 改写成返回 `Pin<Box<dyn Future>>`,牺牲一次堆分配换对象安全)
- 手动把方法签名写成 `fn get_user(&self, id: Uuid) -> Pin<Box<dyn Future<Output = Option<User>> + Send + '_>>`

示例用同步方法是为了避开这个话题。**实际项目里给数据库层做 trait 抽象时,这是你第一个会撞上的问题**,提前知道能省不少时间。

### Router::nest 与 merge 的区别

[第 02 章](02-routes-and-handlers-close-together.md#步骤-2把路由和处理器包进同一个函数)提过一句,这里说清楚。

```rust
let app = Router::new()
    .nest("/dyn", using_dyn)
    .nest("/generic", using_generic);
```

- **`merge`** — 平级合并,把另一个 `Router` 的路由**原样**搬过来。`/users` 合并后还是 `/users`。
- **`nest`** — 嵌套挂载,给另一个 `Router` 的所有路径**加前缀**。`/users` 挂到 `/dyn` 下变成 `/dyn/users`。

本章要让两套实现共存于同一个服务,路径必须区分开,所以用 `nest`。

`nest` 是按模块划分路径空间的标准手段,第 23 章 todos 和第 48 章 versioning 都会用到。

## 项目搭建

```bash
cargo new dependency-injection
cd dependency-injection
```

```bash
cargo add axum --features tracing,macros
cargo add serde --features derive
cargo add tokio --features full
cargo add tracing
cargo add tracing-subscriber --features env-filter
cargo add uuid --features serde,v4
```

`uuid` 的 `v4` feature 提供 `Uuid::new_v4()`(随机 UUID),`serde` feature 让它能序列化。用 UUID 而不是自增整数是 API 设计的常见选择——ID 不可猜测,也不暴露"你们一共有多少用户"这类信息。

`axum` 的 `tracing` feature 让 axum 内部的日志(比如提取器拒绝)能被 `tracing` 捕获。

## 分步构建

### 步骤 1:定义 trait 和一个实现

**要做什么**:先把"接口"和"实现"分开定义,不涉及 axum。

```rust
// src/main.rs
use std::{
    collections::HashMap,
    sync::{Arc, Mutex},
};

use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Serialize, Clone)]
struct User {
    id: Uuid,
    name: String,
}

#[derive(Deserialize)]
struct UserParams {
    name: String,
}

// 依赖的抽象:handler 只知道这个 trait
trait UserRepo: Send + Sync {
    fn get_user(&self, id: Uuid) -> Option<User>;

    fn save_user(&self, user: &User);
}

// 一个具体实现:内存存储
#[derive(Debug, Clone, Default)]
struct InMemoryUserRepo {
    map: Arc<Mutex<HashMap<Uuid, User>>>,
}

impl UserRepo for InMemoryUserRepo {
    fn get_user(&self, id: Uuid) -> Option<User> {
        self.map.lock().unwrap().get(&id).cloned()
    }

    fn save_user(&self, user: &User) {
        self.map.lock().unwrap().insert(user.id, user.clone());
    }
}

fn main() {
    // 下一步接上 axum
}
```

**`trait UserRepo: Send + Sync`** 里的 `Send + Sync` 是**超 trait 约束**,要求所有实现者都能跨线程使用。这是必需的——axum 的 handler 可能在任意 worker 线程上运行,而状态要在它们之间共享。漏了这两个约束,后面 `State` 那里会报一堆难懂的错误。

**两个方法都用 `&self` 而不是 `&mut self`。** 这是关键的设计选择:如果用 `&mut self`,`Arc<dyn UserRepo>` 就没法调用它们了(`Arc` 只给共享引用)。所以可变性被推到了实现内部——`InMemoryUserRepo` 用 `Mutex` 自己管理。

这个模式叫**内部可变性**,是 Rust 里共享可变状态的标准做法。

> **本章不深入**:`Arc<Mutex<HashMap<...>>>` 是共享可变状态。[第 05 章](05-templates-minijinja.md#state-与-with_state)的 `Arc` 只提供共享只读,套一层 `Mutex` 才能写。本章把它当存储用,不展开。`Mutex` 与 `RwLock` 怎么选、`.lock().unwrap()` 为什么会 panic、持锁跨 `await` 的陷阱,见[第 22 章](22-key-value-store.md)。

**`InMemoryUserRepo` 内部就有 `Arc`。** 所以它自己是廉价 `Clone` 的——克隆只复制那个 `Arc`,底层 `HashMap` 是共享的。这一点在步骤 3 很重要:泛型版本要求状态 `Clone`,而它天然满足。

**跑起来看**:这一步还没有服务器,只验证能编译:

```bash
cargo build
```

应该只有几个"未使用"的警告。

### 步骤 2:用 trait 对象注入

**要做什么**:把 `Arc<dyn UserRepo>` 放进 `State`,写出第一组路由。

```rust
// src/main.rs
use std::{
    collections::HashMap,
    sync::{Arc, Mutex},
};

use axum::{
    extract::{Path, State},                             // 新增
    http::StatusCode,                                   // 新增
    routing::{get, post},                               // 新增
    Json, Router,                                       // 新增
};
use serde::{Deserialize, Serialize};
use tokio::net::TcpListener;                            // 新增
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};   // 新增
use uuid::Uuid;

// 改动:换成真正的 main
#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let user_repo = InMemoryUserRepo::default();

    let using_dyn = Router::new()
        .route("/users/{id}", get(get_user_dyn))
        .route("/users", post(create_user_dyn))
        .with_state(AppStateDyn {
            user_repo: Arc::new(user_repo.clone()),
        });

    let app = Router::new().nest("/dyn", using_dyn);

    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

// 新增:装 trait 对象的状态
#[derive(Clone)]
struct AppStateDyn {
    user_repo: Arc<dyn UserRepo>,
}

// 新增
async fn create_user_dyn(
    State(state): State<AppStateDyn>,
    Json(params): Json<UserParams>,
) -> Json<User> {
    let user = User {
        id: Uuid::new_v4(),
        name: params.name,
    };

    state.user_repo.save_user(&user);

    Json(user)
}

// 新增
async fn get_user_dyn(
    State(state): State<AppStateDyn>,
    Path(id): Path<Uuid>,
) -> Result<Json<User>, StatusCode> {
    match state.user_repo.get_user(id) {
        Some(user) => Ok(Json(user)),
        None => Err(StatusCode::NOT_FOUND),
    }
}

// ...(User、UserParams、UserRepo、InMemoryUserRepo 不变)
```

几个要点。

**`#[derive(Clone)] struct AppStateDyn`** —— `State` 要求状态实现 `Clone`。这里能自动派生,是因为唯一的字段 `Arc<dyn UserRepo>` 本身就是 `Clone` 的(克隆 `Arc` 只增加引用计数)。

**`Arc::new(user_repo.clone())`** 这一行把具体类型"擦除"成了 trait 对象。`user_repo` 是 `InMemoryUserRepo`,包进 `Arc` 之后被强制转换(unsized coercion)成 `Arc<dyn UserRepo>`。**从这一刻起,类型信息就丢了**——后面只能通过 trait 的方法访问它。

这里的 `.clone()` 是因为步骤 3 还要用同一个 repo,让两组路由共享同一份数据。`InMemoryUserRepo` 内部是 `Arc`,克隆很廉价。

**handler 签名里完全看不到 `InMemoryUserRepo`。** 这就是依赖注入的效果:换成 `PostgresUserRepo` 只需要改 `main` 里那一行。

**`Path(id): Path<Uuid>`** 直接把路径参数解析成 `Uuid`。[第 04 章](04-templates.md#动态路径与路径参数)讲过 `Path<T>` 会做类型转换,转不出来返回 `400`。

**`Result<Json<User>, StatusCode>`** 是最简单的错误返回形式:找不到就给 `404`。[第 14 章](14-error-handling.md)讲的那套完整错误体系在这里是过度设计——**错误只有一种、也不需要额外信息时,直接返回 `StatusCode` 就够了**。

**跑起来看**:

```bash
cargo run
```

创建一个用户:

```bash
curl -s -X POST http://127.0.0.1:3000/dyn/users \
  -H 'Content-Type: application/json' \
  -d '{"name":"zhangsan"}'
```

```json
{"id":"3f2a1c4e-8b7d-4e1f-9a2b-6c5d4e3f2a1b","name":"zhangsan"}
```

把返回的 `id` 拿去查询(替换成你实际得到的):

```bash
curl -s http://127.0.0.1:3000/dyn/users/3f2a1c4e-8b7d-4e1f-9a2b-6c5d4e3f2a1b
```

```json
{"id":"3f2a1c4e-8b7d-4e1f-9a2b-6c5d4e3f2a1b","name":"zhangsan"}
```

查一个不存在的:

```bash
curl -i -s http://127.0.0.1:3000/dyn/users/00000000-0000-0000-0000-000000000000 | head -1
```

```
HTTP/1.1 404 Not Found
```

ID 不是合法 UUID 时,`Path<Uuid>` 会拒绝:

```bash
curl -i -s http://127.0.0.1:3000/dyn/users/not-a-uuid | head -1
```

```
HTTP/1.1 400 Bad Request
```

注意路径都带 `/dyn` 前缀——`nest` 的效果。

### 步骤 3:用泛型注入同一套逻辑

**要做什么**:再写一组 handler,依赖用泛型表达,挂到 `/generic` 下。

```rust
// src/main.rs
// ...(use 不变)

#[tokio::main]
async fn main() {
    // ...(日志初始化和 user_repo 不变)

    let using_dyn = Router::new()
        // ...(不变)
        .with_state(AppStateDyn {
            user_repo: Arc::new(user_repo.clone()),
        });

    // 新增:泛型版本
    let using_generic = Router::new()
        .route("/users/{id}", get(get_user_generic::<InMemoryUserRepo>))
        .route("/users", post(create_user_generic::<InMemoryUserRepo>))
        .with_state(AppStateGeneric { user_repo });

    // 改动:两组都挂上
    let app = Router::new()
        .nest("/dyn", using_dyn)
        .nest("/generic", using_generic);

    // ...(listener 和 serve 不变)
}

// ...(AppStateDyn 不变)

// 新增:带类型参数的状态
#[derive(Clone)]
struct AppStateGeneric<T> {
    user_repo: T,
}

// ...(create_user_dyn、get_user_dyn 不变)

// 新增
async fn create_user_generic<T>(
    State(state): State<AppStateGeneric<T>>,
    Json(params): Json<UserParams>,
) -> Json<User>
where
    T: UserRepo,
{
    let user = User {
        id: Uuid::new_v4(),
        name: params.name,
    };

    state.user_repo.save_user(&user);

    Json(user)
}

// 新增
async fn get_user_generic<T>(
    State(state): State<AppStateGeneric<T>>,
    Path(id): Path<Uuid>,
) -> Result<Json<User>, StatusCode>
where
    T: UserRepo,
{
    match state.user_repo.get_user(id) {
        Some(user) => Ok(Json(user)),
        None => Err(StatusCode::NOT_FOUND),
    }
}

// ...(其余不变)
```

**把两组 handler 并排看,函数体一模一样**,差别全在签名:

```rust
async fn get_user_dyn(State(state): State<AppStateDyn>, Path(id): Path<Uuid>) -> ...

async fn get_user_generic<T>(State(state): State<AppStateGeneric<T>>, Path(id): Path<Uuid>) -> ...
where T: UserRepo,
```

泛型版本多了 `<T>` 和 `where T: UserRepo`。**这就是前置知识说的"类型参数扩散"**:只是让依赖可替换,函数签名就多了两处噪音。真实项目里 handler 有几十个,每个都要带上这些。

**`get(get_user_generic::<InMemoryUserRepo>)` 里的 turbofish 是必需的。**

原因是编译器无法从上下文推断 `T`。`route()` 只知道要一个 `Handler`,不知道 `T` 该是什么;`with_state` 虽然给了 `AppStateGeneric<InMemoryUserRepo>`,但那是在链式调用的后面,推断不回来。所以必须显式写出来。

**这是泛型方案的一个实际摩擦点**:每注册一条路由都要重复写一遍具体类型。改实现时要改所有 turbofish,而不是像 trait 对象那样只改 `main` 里的一行。

**`AppStateGeneric { user_repo }`** 这里没有 `.clone()`,因为这是最后一次使用 `user_repo`,所有权直接移进去。

**跑起来看**:

```bash
cargo run
```

泛型版本的行为应该完全一致:

```bash
curl -s -X POST http://127.0.0.1:3000/generic/users \
  -H 'Content-Type: application/json' \
  -d '{"name":"lisi"}'
```

```json
{"id":"7d8e9f0a-1b2c-4d3e-8f5a-6b7c8d9e0f1a","name":"lisi"}
```

**现在验证一件有意思的事:两组路由共享同一份数据。** 用上面 `/generic` 创建的用户 ID,去 `/dyn` 查询:

```bash
curl -s http://127.0.0.1:3000/dyn/users/7d8e9f0a-1b2c-4d3e-8f5a-6b7c8d9e0f1a
```

```json
{"id":"7d8e9f0a-1b2c-4d3e-8f5a-6b7c8d9e0f1a","name":"lisi"}
```

**查到了。** 因为 `main` 里两组状态装的是同一个 `InMemoryUserRepo` 的克隆,而它内部的 `Arc<Mutex<HashMap>>` 是共享的——克隆只复制了指针。

反过来也成立:

```bash
curl -s -X POST http://127.0.0.1:3000/dyn/users \
  -H 'Content-Type: application/json' -d '{"name":"wangwu"}'
# 拿返回的 id 去 /generic 查
```

**这个验证顺带说明了一件事**:`InMemoryUserRepo` 的 `Clone` 是浅拷贝。如果它内部直接是 `HashMap` 而不是 `Arc<Mutex<HashMap>>`,两组路由就会各存各的,互相看不见。**"这个类型的 `Clone` 到底复制了什么"在共享状态场景下必须想清楚**。

## 该选哪个

前置知识那张表列了区别,这里给判断顺序。

**默认用 trait 对象。** 官方注释写得很直白:"Using trait objects is recommended unless you really need generics."

动态分发的开销在 Web 服务里可以忽略——一次虚表跳转是纳秒级,而一次数据库查询是毫秒级,差了六个数量级。为这点开销引入类型参数扩散不划算。

**下面这些情况才考虑泛型:**

- trait 不是对象安全的(有泛型方法、返回 `Self` 等)
- 依赖在极热的路径上被高频调用,而且你**测量过**动态分发确实是瓶颈
- 需要编译期就确定实现,比如用类型状态模式做静态保证

**注意"运行时切换实现"只有 trait 对象能做。** 比如根据配置决定用内存存储还是数据库:

```rust
let repo: Arc<dyn UserRepo> = if config.use_memory {
    Arc::new(InMemoryUserRepo::default())
} else {
    Arc::new(PostgresUserRepo::new(pool))
};
```

泛型版本写不出这个——两个分支的类型不同,编译不过。

**测试场景两者都支持**,但 trait 对象更省事:测试里注入 `Arc::new(FakeUserRepo)` 就行,不用改任何 turbofish。

## 完整代码

`src/main.rs`:

```rust
use std::{
    collections::HashMap,
    sync::{Arc, Mutex},
};

use axum::{
    extract::{Path, State},
    http::StatusCode,
    routing::{get, post},
    Json, Router,
};
use serde::{Deserialize, Serialize};
use tokio::net::TcpListener;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};
use uuid::Uuid;

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let user_repo = InMemoryUserRepo::default();

    // We generally have two ways to inject dependencies:
    //
    // 1. Using trait objects (`dyn SomeTrait`)
    //     - Pros
    //         - Likely leads to simpler code due to fewer type parameters.
    //     - Cons
    //         - Less flexible because we can only use object safe traits
    //         - Small amount of additional runtime overhead due to dynamic dispatch.
    //           This is likely to be negligible.
    // 2. Using generics (`T where T: SomeTrait`)
    //     - Pros
    //         - More flexible since all traits can be used.
    //         - No runtime overhead.
    //     - Cons:
    //         - Additional type parameters and trait bounds can lead to more complex code and
    //           boilerplate.
    //
    // Using trait objects is recommended unless you really need generics.

    let using_dyn = Router::new()
        .route("/users/{id}", get(get_user_dyn))
        .route("/users", post(create_user_dyn))
        .with_state(AppStateDyn {
            user_repo: Arc::new(user_repo.clone()),
        });

    let using_generic = Router::new()
        .route("/users/{id}", get(get_user_generic::<InMemoryUserRepo>))
        .route("/users", post(create_user_generic::<InMemoryUserRepo>))
        .with_state(AppStateGeneric { user_repo });

    let app = Router::new()
        .nest("/dyn", using_dyn)
        .nest("/generic", using_generic);

    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

#[derive(Clone)]
struct AppStateDyn {
    user_repo: Arc<dyn UserRepo>,
}

#[derive(Clone)]
struct AppStateGeneric<T> {
    user_repo: T,
}

#[derive(Debug, Serialize, Clone)]
struct User {
    id: Uuid,
    name: String,
}

#[derive(Deserialize)]
struct UserParams {
    name: String,
}

async fn create_user_dyn(
    State(state): State<AppStateDyn>,
    Json(params): Json<UserParams>,
) -> Json<User> {
    let user = User {
        id: Uuid::new_v4(),
        name: params.name,
    };

    state.user_repo.save_user(&user);

    Json(user)
}

async fn get_user_dyn(
    State(state): State<AppStateDyn>,
    Path(id): Path<Uuid>,
) -> Result<Json<User>, StatusCode> {
    match state.user_repo.get_user(id) {
        Some(user) => Ok(Json(user)),
        None => Err(StatusCode::NOT_FOUND),
    }
}

async fn create_user_generic<T>(
    State(state): State<AppStateGeneric<T>>,
    Json(params): Json<UserParams>,
) -> Json<User>
where
    T: UserRepo,
{
    let user = User {
        id: Uuid::new_v4(),
        name: params.name,
    };

    state.user_repo.save_user(&user);

    Json(user)
}

async fn get_user_generic<T>(
    State(state): State<AppStateGeneric<T>>,
    Path(id): Path<Uuid>,
) -> Result<Json<User>, StatusCode>
where
    T: UserRepo,
{
    match state.user_repo.get_user(id) {
        Some(user) => Ok(Json(user)),
        None => Err(StatusCode::NOT_FOUND),
    }
}

trait UserRepo: Send + Sync {
    fn get_user(&self, id: Uuid) -> Option<User>;

    fn save_user(&self, user: &User);
}

#[derive(Debug, Clone, Default)]
struct InMemoryUserRepo {
    map: Arc<Mutex<HashMap<Uuid, User>>>,
}

impl UserRepo for InMemoryUserRepo {
    fn get_user(&self, id: Uuid) -> Option<User> {
        self.map.lock().unwrap().get(&id).cloned()
    }

    fn save_user(&self, user: &User) {
        self.map.lock().unwrap().insert(user.id, user.clone());
    }
}
```

依赖:

- `axum` 0.8.x,features `tracing`、`macros`
- `serde` 1.x,feature `derive`
- `tokio` 1.x,feature `full`
- `tracing` 0.1.x
- `tracing-subscriber` 0.3.x,feature `env-filter`
- `uuid` 1.x,features `serde`、`v4`

## 动手练习

**1. 加一个假实现,体会依赖注入的价值。**

提示:写一个 `struct FakeUserRepo;`,`get_user` 永远返回一个固定用户,`save_user` 什么都不做。然后在 `main` 里把 `/dyn` 那组的 `Arc::new(user_repo.clone())` 换成 `Arc::new(FakeUserRepo)`。**注意你不需要改任何 handler**。再试试对 `/generic` 那组做同样的事——你会发现要改两处 turbofish,这就是两种方案在可替换性上的实际差距。

**2. 给 trait 加一个泛型方法,看看会发生什么。**

提示:在 `UserRepo` 里加 `fn find_by<F: Fn(&User) -> bool>(&self, pred: F) -> Vec<User>;`。编译会失败,错误信息会提到 `dyn UserRepo` 不是对象安全的。这就是前置知识说的限制——读懂这个错误,以后遇到就能立刻反应过来。做完删掉这个方法。

**3. 加一个 `DELETE /users/{id}`,两组路由都要支持。**

提示:trait 加一个 `fn delete_user(&self, id: Uuid) -> bool`,两组各写一个 handler。做完数一下:trait 对象那组改了几处,泛型那组改了几处?

## 小结与下一章预告

依赖注入让 handler 只依赖 trait 而非具体类型,换实现、写测试、用假数据都不必改业务代码。Rust 不需要 DI 框架——trait 加 `State`,组装写在 `main` 里,而且编译期检查。

两种表达方式各有代价:`Arc<dyn Trait>` 走动态分发,有微小的运行时开销、要求 trait 对象安全,但**类型参数不会扩散**,而且能在运行时切换实现;`T: Trait` 是静态分发、零开销,但类型参数会顺着调用链传染,每条路由还要写 turbofish。

**官方推荐默认用 trait 对象**,除非 trait 不是对象安全的、或者你测量过动态分发确实是瓶颈。这条建议背后是一个更普遍的判断:类型系统的复杂度是有成本的,而纳秒级的分发开销在 Web 服务里通常不值一提。

顺带认识了 `Router::nest`——给子路由加路径前缀,和 `merge` 的平级合并区分开。

到这里,前面章节欠下的 `TraceLayer` 和中间件顺序一直挂着"本章不深入"。[第 17 章 tracing-aka-logging](17-tracing-aka-logging.md) 把这笔债还上:tower 的 `Service`/`Layer` 到底是什么、多个 `.layer()` 为什么是"后注册的在外层"、`TraceLayer` 的每个回调在什么时机触发、span 和 event 的区别、以及 `RUST_LOG` 那套过滤语法怎么写。
