# 18. dependency-injection

对应示例：`examples/dependency-injection`

本章目标：理解 trait object 与泛型两种依赖注入方式，让 handler 依赖抽象的 `UserRepo`，而不是直接依赖具体存储实现。

这是一个真实项目里很重要的设计问题：

```text
handler 需要读写用户
但 handler 不应该关心用户到底存在内存、数据库还是远程服务里
```

## 这个小项目在做什么

这个 example 实现一个简单用户 API，并用两种方式注入 `UserRepo`：

| 路径前缀 | 注入方式 | 状态类型 |
| --- | --- | --- |
| `/dyn` | trait object | `Arc<dyn UserRepo>` |
| `/generic` | 泛型 | `T where T: UserRepo` |

两组接口功能相同：

| 方法 | 路径 | 作用 |
| --- | --- | --- |
| `POST` | `/dyn/users` | 创建用户 |
| `GET` | `/dyn/users/{id}` | 查询用户 |
| `POST` | `/generic/users` | 创建用户 |
| `GET` | `/generic/users/{id}` | 查询用户 |

请求主线是：

```text
POST /dyn/users
-> Json<UserParams> 解析 name
-> create_user_dyn 生成 User
-> state.user_repo.save_user(&user)
-> 返回 Json(user)
```

不同点不在 API 行为，而在 handler 如何拿到依赖。

## 先理解依赖注入

不用依赖注入时，handler 可能直接写死：

```text
handler -> InMemoryUserRepo
```

这样的问题是：

- 想换数据库实现时，handler 要改。
- 测试时不好替换成 fake repo。
- 业务逻辑和存储细节耦合。

依赖注入的思路是：

```text
handler -> UserRepo trait -> 具体实现
```

handler 只知道“我需要一个能保存和读取用户的东西”。  
具体是内存、Postgres、Redis，交给外层组装。

## 文件和依赖

这个 example 有两个文件：

1. `examples/dependency-injection/Cargo.toml`：声明 Axum、Serde、Tokio、uuid 等依赖。
2. `examples/dependency-injection/src/main.rs`：实现 trait、内存 repo、两种 state 和两套路由。

关键依赖：

- `axum`：提供 `State`、`Path`、`Json`、`Router`。
- `serde`：解析请求 JSON，序列化响应 JSON。
- `uuid`：生成和解析用户 ID。
- `tokio`、`tracing`、`tracing-subscriber`：运行时和日志。

## 第一步：定义 UserRepo trait

源码：

````rust
trait UserRepo: Send + Sync {
    fn get_user(&self, id: Uuid) -> Option<User>;

    fn save_user(&self, user: &User);
}
````

这个 trait 描述“用户仓库”必须提供的能力：

- 根据 id 查询用户。
- 保存用户。

`Send + Sync` 很重要。  
Axum 会并发处理请求，state 需要能安全地跨线程共享。

handler 后面只依赖这个 trait，而不是直接依赖 `InMemoryUserRepo`。

## 第二步：实现内存版 UserRepo

源码：

````rust
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
````

内部结构是：

```text
Arc<Mutex<HashMap<Uuid, User>>>
```

- `HashMap` 存用户。
- `Mutex` 保护并发读写。
- `Arc` 允许 clone 后共享同一份 map。

这只是教学实现。真实项目可以实现另一个 `PostgresUserRepo`，只要它也实现 `UserRepo`，handler 就不用大改。

## 第三步：定义数据结构

响应类型：

````rust
#[derive(Debug, Serialize, Clone)]
struct User {
    id: Uuid,
    name: String,
}
````

创建用户的请求体：

````rust
#[derive(Deserialize)]
struct UserParams {
    name: String,
}
````

创建请求 body：

````json
{"name":"Alice"}
````

响应 body：

````json
{"id":"...","name":"Alice"}
````

## 第四步：方式一，trait object 注入

状态类型：

````rust
#[derive(Clone)]
struct AppStateDyn {
    user_repo: Arc<dyn UserRepo>,
}
````

组装路由：

````rust
let using_dyn = Router::new()
    .route("/users/{id}", get(get_user_dyn))
    .route("/users", post(create_user_dyn))
    .with_state(AppStateDyn {
        user_repo: Arc::new(user_repo.clone()),
    });
````

这里的 `Arc<dyn UserRepo>` 表示：

```text
一个实现了 UserRepo 的对象
具体类型在运行时通过动态分发调用
```

handler：

````rust
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
````

handler 不知道 repo 具体类型，只调用 trait 方法。

trait object 方式的特点：

- 优点：handler 类型简单，Router 组装也直观。
- 缺点：trait 必须 object safe，会有很小的动态分发开销。

### 什么是 object safe？为什么 trait object 要求它？

`Arc<dyn UserRepo>` 里的 `dyn` 表示"动态分发"——编译器不知道具体类型，运行时通过一个叫"虚表（vtable）"的东西找到真正的方法实现。为了让这套机制能工作，trait 必须满足一些条件，叫 **object safe**。

简单说，一个 trait 是 object safe 的，需要满足：

- **所有方法都不能有泛型参数**（如 `fn foo<T>(&self)`）。因为泛型方法在编译时要为每个 `T` 生成代码，但 `dyn Trait` 在运行时只有一份代码，没法处理无限种 `T`。
- **方法不能用 `Self` 作为返回类型或参数类型**（`fn clone(&self) -> Self`）。因为 `dyn Trait` 不知道具体类型，没法返回 `Self`。
- **不能有带 `Self` 约束的关联常量/类型**等细节条件。

本章的 `UserRepo` 是 object safe 的：

````rust
trait UserRepo: Send + Sync {
    fn get_user(&self, id: Uuid) -> Option<User>;
    fn save_user(&self, user: &User);
}
````

它没有泛型方法，没有返回 `Self`，所以可以用 `dyn`。

**反例**：如果 trait 写成这样就不 object safe：

````rust
trait BadRepo {
    // ❌ 有泛型方法，不是 object safe
    fn get<T: DeserializeOwned>(&self, key: &str) -> Option<T>;
}
````

编译器会报错：`this trait cannot be made into an object`。

所以当你设计 trait 想用 `Arc<dyn Trait>` 时，**避免泛型方法和返回 `Self`**。如果实在需要泛型方法，就用泛型注入（第五步那种）代替 trait object。

官方示例也建议：除非你真的需要泛型，否则 trait object 通常更简单。

## 第五步：方式二，泛型注入

状态类型：

````rust
#[derive(Clone)]
struct AppStateGeneric<T> {
    user_repo: T,
}
````

组装路由：

````rust
let using_generic = Router::new()
    .route("/users/{id}", get(get_user_generic::<InMemoryUserRepo>))
    .route("/users", post(create_user_generic::<InMemoryUserRepo>))
    .with_state(AppStateGeneric { user_repo });
````

handler：

````rust
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
````

泛型方式的特点：

- 优点：没有动态分发开销，trait 不需要 object safe，类型更静态。
- 缺点：handler 上会出现更多类型参数和 trait bound，Router 里也要写 turbofish，例如 `::<InMemoryUserRepo>`。

## 第六步：把两套路由 nest 到不同前缀

源码：

````rust
let app = Router::new()
    .nest("/dyn", using_dyn)
    .nest("/generic", using_generic);
````

最终接口是：

```text
/dyn/users
/dyn/users/{id}
/generic/users
/generic/users/{id}
```

这让你可以用同一个服务同时对比两种依赖注入写法。

## 两种方式怎么选

可以先用这张表：

| 方式 | 适合场景 | 代价 |
| --- | --- | --- |
| `Arc<dyn UserRepo>` | 大多数 Web API，想让代码简单 | 需要 object safe，有动态分发 |
| `T: UserRepo` | 需要泛型能力、零动态分发、trait 不 object safe | 类型参数更多，代码更复杂 |

新手建议：

```text
先用 Arc<dyn Trait>
等真的遇到限制，再考虑泛型
```

## 函数职责速查

- `main`：初始化日志，创建内存 repo，组装 dyn/generic 两套路由。
- `AppStateDyn`：保存 `Arc<dyn UserRepo>`。
- `AppStateGeneric<T>`：保存泛型 repo。
- `User`：用户响应结构体。
- `UserParams`：创建用户请求体。
- `create_user_dyn` / `get_user_dyn`：trait object 版本 handler。
- `create_user_generic` / `get_user_generic`：泛型版本 handler。
- `UserRepo`：用户仓库抽象。
- `InMemoryUserRepo`：内存版仓库实现。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-dependency-injection
//! ```

use std::{
    collections::HashMap,
    sync::{Arc, Mutex},
};

// 引入 Path、State、状态码、路由函数、Json 和 Router。
use axum::{
    extract::{Path, State},
    http::StatusCode,
    routing::{get, post},
    Json, Router,
};
// serde 用于请求反序列化和响应序列化。
use serde::{Deserialize, Serialize};
// TcpListener 绑定端口。
use tokio::net::TcpListener;
// tracing_subscriber 初始化日志。
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};
// UUID 用作用户 id。
use uuid::Uuid;

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 初始化日志系统。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 创建内存用户仓库。
    let user_repo = InMemoryUserRepo::default();

    // trait object 方式注入依赖。
    let using_dyn = Router::new()
        .route("/users/{id}", get(get_user_dyn))
        .route("/users", post(create_user_dyn))
        .with_state(AppStateDyn {
            user_repo: Arc::new(user_repo.clone()),
        });

    // 泛型方式注入依赖。
    let using_generic = Router::new()
        .route("/users/{id}", get(get_user_generic::<InMemoryUserRepo>))
        .route("/users", post(create_user_generic::<InMemoryUserRepo>))
        .with_state(AppStateGeneric { user_repo });

    // 把两套路由挂到不同前缀下，方便对比。
    let app = Router::new()
        .nest("/dyn", using_dyn)
        .nest("/generic", using_generic);

    // 绑定本机 3000 端口。
    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动 HTTP 服务。
    axum::serve(listener, app).await;
}

// trait object 方式的应用状态。
#[derive(Clone)]
struct AppStateDyn {
    user_repo: Arc<dyn UserRepo>,
}

// 泛型方式的应用状态。
#[derive(Clone)]
struct AppStateGeneric<T> {
    user_repo: T,
}

// 用户响应类型。
#[derive(Debug, Serialize, Clone)]
struct User {
    id: Uuid,
    name: String,
}

// 创建用户请求体。
#[derive(Deserialize)]
struct UserParams {
    name: String,
}

// trait object 版本：创建用户。
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

// trait object 版本：根据 id 查询用户。
async fn get_user_dyn(
    State(state): State<AppStateDyn>,
    Path(id): Path<Uuid>,
) -> Result<Json<User>, StatusCode> {
    match state.user_repo.get_user(id) {
        Some(user) => Ok(Json(user)),
        None => Err(StatusCode::NOT_FOUND),
    }
}

// 泛型版本：创建用户。
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

// 泛型版本：根据 id 查询用户。
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

// 用户仓库抽象。handler 依赖这个 trait，而不是具体存储实现。
trait UserRepo: Send + Sync {
    fn get_user(&self, id: Uuid) -> Option<User>;

    fn save_user(&self, user: &User);
}

// 内存版 UserRepo。
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
````

## 运行和验证

运行前先确认 `examples/dependency-injection/Cargo.toml` 里的 `axum = { path = "../../axum", features = ["tracing", "macros"] }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-dependency-injection
````

测试 dyn 版本创建用户：

````bash
curl -i -X POST http://127.0.0.1:3000/dyn/users \
  -H 'content-type: application/json' \
  -d '{"name":"Alice"}'
````

测试 generic 版本创建用户：

````bash
curl -i -X POST http://127.0.0.1:3000/generic/users \
  -H 'content-type: application/json' \
  -d '{"name":"Bob"}'
````

两个响应都会返回带 `id` 的用户 JSON。  
把返回的 id 填到查询请求里：

````bash
curl -i http://127.0.0.1:3000/dyn/users/<id>
curl -i http://127.0.0.1:3000/generic/users/<id>
````

常见卡点：

- `/dyn` 和 `/generic` 使用的是不同 Router，也各自有自己的 state。
- `Arc<dyn UserRepo>` 需要 trait object safe。
- 泛型 handler 需要在 Router 中写具体类型，例如 `get_user_generic::<InMemoryUserRepo>`。
- 这是内存仓库，服务重启后用户会丢失。

## 手写任务

完成本章代码后，试着做三个小改动：

1. 给 `User` 增加 `email` 字段，并让创建接口接收 email。
2. 新增一个 `FakeUserRepo`，让它总是返回固定用户，用来理解替换依赖。
3. 只保留 `/dyn` 版本，观察代码是否更简单。

## 本章真正要记住什么

- 依赖注入让 handler 依赖抽象，而不是依赖具体实现。
- `State<T>` 是 Axum 中传递依赖的常用方式。
- `Arc<dyn Trait>` 更简单，适合多数 Web API。
- 泛型 `T: Trait` 更灵活，但类型参数和 Router 组装更复杂。
- Repository trait 可以让存储实现从内存切换到数据库时减少 handler 改动。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/dependency-injection/Cargo.toml`
- `examples/dependency-injection/src/main.rs`
