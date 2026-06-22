# 18. dependency-injection

对应示例：`examples/dependency-injection`

真实项目里 handler 需要读写用户，但不应该关心用户存在内存还是 PostgreSQL。这章演示两种依赖注入方式，让 handler 依赖抽象（trait）而非具体实现。分 4 步搭。

相比前面章节新引入：**trait object（`Arc<dyn Trait>`）**和**泛型注入（`T: Trait`）**两种设计模式。

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

本章相比前面章节新增 `uuid`（生成用户唯一 ID，第 16 章已用过）。`axum` 启用 `tracing` feature（让 axum 内部日志走 tracing）和 `macros`（用 `#[derive(FromRequest)]` 等派生宏）。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：定义 trait 和内存实现

先定义"用户仓库"的抽象接口——handler 只依赖这个 trait：

````rust
use std::{
    collections::HashMap,
    sync::{Arc, Mutex},
};
use uuid::Uuid;

trait UserRepo: Send + Sync {
    fn get_user(&self, id: Uuid) -> Option<User>;
    fn save_user(&self, user: &User);
}
````

> **新面孔：`trait UserRepo: Send + Sync`**
>
> trait 描述"用户仓库"必须提供的能力：按 id 查询、保存用户。handler 后面只依赖这个 trait，不依赖具体存储。
>
> `Send + Sync` 很重要——axum 并发处理请求，state 要能跨线程共享。后面所有 trait bound 都会有这个约束。

然后写一个内存版实现：

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

`InMemoryUserRepo` 内部用 `Arc<Mutex<HashMap>>`（和 ch16/17 同理）。真实项目可以实现 `PostgresUserRepo`，只要也实现 `UserRepo`，handler 不用改。

---

## 第二步：方式一——trait object（`Arc<dyn UserRepo>`）

第一种注入方式：用 trait object。state 里存 `Arc<dyn UserRepo>`，运行时动态分发。

````rust
#[derive(Clone)]
struct AppStateDyn {
    user_repo: Arc<dyn UserRepo>,
}
````

> **新面孔：`Arc<dyn Trait>`（trait object）**
>
> `dyn UserRepo` 表示"一个实现了 UserRepo 的对象，具体类型运行时通过虚表（vtable）找方法"。`Arc` 让多个 handler 共享。
>
> trait 必须 **object safe** 才能用 `dyn`：不能有泛型方法、不能返回 `Self`。`UserRepo` 的两个方法都不违反，所以可以用。

handler 只调 trait 方法，不知道具体类型：

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

---

## 第三步：方式二——泛型（`T: UserRepo`）

第二种注入方式：用泛型。编译时确定类型，零动态分发开销。

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
    let user = User {
        id: Uuid::new_v4(),
        name: params.name,
    };
    state.user_repo.save_user(&user);
    Json(user)
}
````

> **trait object vs 泛型**
>
> | 方式 | 优点 | 缺点 |
> |---|---|---|
> | `Arc<dyn Trait>` | handler 类型简单，Router 组装直观 | trait 必须 object safe，有动态分发开销 |
> | `T: Trait` | 零开销，trait 不需 object safe | handler 多类型参数，Router 要写 turbofish |
>
> 新手先用 `Arc<dyn Trait>`，遇到限制再考虑泛型。

Router 组装时泛型要写 turbofish：

````rust
let using_generic = Router::new()
    .route("/users/{id}", get(get_user_generic::<InMemoryUserRepo>))
    .route("/users", post(create_user_generic::<InMemoryUserRepo>))
    .with_state(AppStateGeneric { user_repo });
````

---

## 第四步：两套路由 nest 到不同前缀

最终把两套路由挂到不同前缀，方便对比：

````rust
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
````

---

## 完整代码

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
    let user = User { id: Uuid::new_v4(), name: params.name };
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
where T: UserRepo {
    let user = User { id: Uuid::new_v4(), name: params.name };
    state.user_repo.save_user(&user);
    Json(user)
}

async fn get_user_generic<T>(
    State(state): State<AppStateGeneric<T>>,
    Path(id): Path<Uuid>,
) -> Result<Json<User>, StatusCode>
where T: UserRepo {
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

dyn 版本：

````bash
curl -X POST http://127.0.0.1:3000/dyn/users \
  -H 'content-type: application/json' -d '{"name":"Alice"}'
````

generic 版本：

````bash
curl -X POST http://127.0.0.1:3000/generic/users \
  -H 'content-type: application/json' -d '{"name":"Bob"}'
````

## 手写任务

1. 给 `User` 加 `email` 字段。
2. 新增 `FakeUserRepo`，让它总是返回固定用户。
3. 只保留 `/dyn` 版本，观察代码是否更简单。

## 小结

这章分 4 步演示了依赖注入：

1. **定义 trait**：`UserRepo` 描述"用户仓库"的能力，`Send + Sync` 保证跨线程。
2. **trait object**：`Arc<dyn UserRepo>`，简单但有动态分发，trait 需 object safe。
3. **泛型**：`T: UserRepo`，零开销但类型参数多、Router 要 turbofish。
4. **两种方式对比**：新手先用 `Arc<dyn Trait>`，遇到限制再考虑泛型。

## 源码对照

- `examples/dependency-injection/Cargo.toml`
- `examples/dependency-injection/src/main.rs`
