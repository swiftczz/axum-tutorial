# 18. dependency-injection

对应示例：`examples/dependency-injection`

真实项目里很重要的设计问题:handler 需要读写用户,但不应该关心用户存在内存、数据库还是远程服务里。本章用 trait object 和泛型**两种方式**注入 `UserRepo`,让 handler 依赖抽象而不是具体实现。

## Cargo.toml

````toml
[package]
name = "example-dependency-injection"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["tracing", "macros"] }
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
uuid = { version = "1.0", features = ["serde", "v4"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

````rust
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

    axum::serve(listener, app).await;
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
````

## 运行

````bash
cd examples
cargo run -p example-dependency-injection
````

dyn 版本创建用户:

````bash
curl -i -X POST http://127.0.0.1:3000/dyn/users \
  -H 'content-type: application/json' \
  -d '{"name":"Alice"}'
````

generic 版本:

````bash
curl -i -X POST http://127.0.0.1:3000/generic/users \
  -H 'content-type: application/json' \
  -d '{"name":"Bob"}'
````

两个响应都返回带 `id` 的用户 JSON。查询:

````bash
curl -i http://127.0.0.1:3000/dyn/users/<id>
curl -i http://127.0.0.1:3000/generic/users/<id>
````

## 解读

### 为什么要依赖注入

不依赖注入时 handler 直接依赖 `InMemoryUserRepo`,问题是:换数据库实现时 handler 要改,测试时不好替换成 fake repo,业务逻辑和存储细节耦合。依赖注入让 handler 只依赖 `UserRepo` trait:

```text
handler → UserRepo trait → 具体实现(内存/Postgres/Redis...)
```

### `UserRepo` trait

````rust
trait UserRepo: Send + Sync {
    fn get_user(&self, id: Uuid) -> Option<User>;
    fn save_user(&self, user: &User);
}
````

描述"用户仓库"必须提供的能力。`Send + Sync` 很重要——axum 并发处理请求,state 要能安全跨线程共享。

`InMemoryUserRepo` 实现这个 trait,内部用 `Arc<Mutex<HashMap<Uuid, User>>>`。真实项目可以实现 `PostgresUserRepo`,只要也实现 `UserRepo`,handler 不用大改。

### 方式一:trait object `Arc<dyn UserRepo>`

````rust
#[derive(Clone)]
struct AppStateDyn {
    user_repo: Arc<dyn UserRepo>,
}
````

`Arc<dyn UserRepo>` 表示"一个实现了 UserRepo 的对象,具体类型运行时动态分发"。handler 只调 trait 方法,不知道具体类型:

````rust
async fn create_user_dyn(
    State(state): State<AppStateDyn>,
    Json(params): Json<UserParams>,
) -> Json<User> {
    ...
    state.user_repo.save_user(&user);
    ...
}
````

优点:handler 类型简单,Router 组装直观。缺点:trait 必须 object safe,有很小的动态分发开销。

### 什么是 object safe

`dyn Trait` 是动态分发——编译器不知道具体类型,运行时通过虚表(vtable)找方法实现。trait 要支持这套机制必须 object safe,核心条件:

- **方法不能有泛型参数**(`fn foo<T>(&self)`)。泛型方法编译时要为每个 T 生成代码,但 `dyn` 只有一份。
- **方法不能用 `Self` 作返回类型或参数类型**(`fn clone(&self) -> Self`)。`dyn` 不知道具体类型。
- 不能有带 `Self` 约束的关联常量/类型。

`UserRepo` 没有泛型方法,不返回 `Self`,所以 object safe。反例:

````rust
trait BadRepo {
    fn get<T: DeserializeOwned>(&self, key: &str) -> Option<T>;  // ❌ 泛型方法
}
````

编译器会报 `this trait cannot be made into an object`。设计 trait 想用 `Arc<dyn Trait>` 时,避免泛型方法和返回 `Self`;实在需要泛型方法就用泛型注入。

### 方式二:泛型 `T: UserRepo`

````rust
#[derive(Clone)]
struct AppStateGeneric<T> {
    user_repo: T,
}

async fn create_user_generic<T>(
    State(state): State<AppStateGeneric<T>>,
    Json(params): Json<UserParams>,
) -> Json<User>
where
    T: UserRepo,
{
    ...
}
````

Router 组装时要写 turbofish:

````rust
.route("/users", post(create_user_generic::<InMemoryUserRepo>))
````

优点:没有动态分发开销,trait 不需要 object safe。缺点:handler 多类型参数和 trait bound,Router 要写 turbofish,代码更复杂。

### 两种方式怎么选

| 方式 | 适合场景 | 代价 |
| --- | --- | --- |
| `Arc<dyn UserRepo>` | 大多数 Web API,想让代码简单 | 需要 object safe,有动态分发 |
| `T: UserRepo` | 需要零动态分发、trait 不 object safe | 类型参数多,代码复杂 |

新手建议:先用 `Arc<dyn Trait>`,等真遇到限制再考虑泛型。

## 手写任务

跑通后做三个小改动:

1. 给 `User` 加 `email` 字段,让创建接口接收 email。
2. 新增 `FakeUserRepo`,让它总是返回固定用户,理解替换依赖。
3. 只保留 `/dyn` 版本,观察代码是否更简单。

## 小结

- 依赖注入让 handler 依赖抽象(trait)而不是具体实现,换存储时减少 handler 改动。
- `Arc<dyn Trait>` 简单,适合多数 Web API,但 trait 必须 object safe(无泛型方法、不返回 Self)。
- 泛型 `T: Trait` 灵活、零动态分发,但类型参数和 Router 组装(turbofish)更复杂。
- `State<T>` 是 axum 传递依赖的常用方式;`Send + Sync` 是 state 跨线程共享的硬性要求。

## 源码对照

- `examples/dependency-injection/Cargo.toml`
- `examples/dependency-injection/src/main.rs`
