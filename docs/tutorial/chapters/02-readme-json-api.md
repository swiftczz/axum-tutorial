# 02. readme-json-api

对应示例：`examples/readme`

本章在上一章的 `GET /` 基础上，加一个 `POST /users`：接收 JSON 请求体，解析成 Rust 结构体，返回 `201 Created` 和 JSON 响应。重点不是数据库，而是理解 HTTP JSON API 的输入和输出。

## Cargo.toml

````toml
[package]
name = "example-readme"
version = "0.1.0"
edition = "2024"

[dependencies]
axum = "0.8"
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

- `axum`：路由、`Json` extractor、响应转换。
- `serde`：JSON 和 Rust struct 互转。`features = ["derive"]` 让你能用 `#[derive(Serialize)]`。
- `tokio`：异步运行时。
- `tracing` / `tracing-subscriber`：结构化日志。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

````rust
use axum::{
    http::StatusCode,
    response::IntoResponse,
    routing::{get, post},
    Json, Router,
};
use serde::{Deserialize, Serialize};

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt::init();

    let app = Router::new()
        .route("/", get(root))
        .route("/users", post(create_user));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app).await;
}

async fn root() -> &'static str {
    "Hello, World!"
}

async fn create_user(
    Json(payload): Json<CreateUser>,
) -> impl IntoResponse {
    let user = User {
        id: 1337,
        username: payload.username,
    };

    (StatusCode::CREATED, Json(user))
}

#[derive(Deserialize)]
struct CreateUser {
    username: String,
}

#[derive(Serialize)]
struct User {
    id: u64,
    username: String,
}
````

## 运行

````bash
cd examples
cargo run -p example-readme
````

测试 GET：

````bash
curl http://127.0.0.1:3000/
````

预期：

````text
Hello, World!
````

测试 POST：

````bash
curl -i -X POST http://127.0.0.1:3000/users \
  -H 'content-type: application/json' \
  -d '{"username":"alice"}'
````

预期状态码 `201 Created`，响应体：

````json
{"id":1337,"username":"alice"}
````

常见卡点：`Json<CreateUser>` 依赖 `content-type: application/json` header 和合法 JSON body。少了 header 或 JSON 写错，axum 会在进入 `create_user` 前返回错误。

## 解读

本章有两个 handler，演示 axum JSON API 的完整闭环：

```text
POST /users
  → Json<CreateUser> 把请求 body 的 JSON 解析成 CreateUser 结构体
  → create_user 生成 User
  → (StatusCode::CREATED, Json(user)) 变成 201 + JSON 响应
```

### 两个路由

````rust
let app = Router::new()
    .route("/", get(root))
    .route("/users", post(create_user));
````

`GET /` → `root`，`POST /users` → `create_user`。每个 handler 只管一个明确的业务动作，这样代码好读也好测。

### `#[derive(Deserialize)]` / `#[derive(Serialize)]`

`#[derive(X)]` 是 Rust 的 derive 宏：编译器自动帮你实现 `X` 这个 trait，不用手写 impl 块。教程里会反复看到 `Serialize`、`Clone`、`Debug` 等，都是这个模式。

serde 的两个方向要分清：

```text
Deserialize：JSON 请求体 → Rust struct（输入，所以 CreateUser 用它）
Serialize：  Rust struct → JSON 响应体（输出，所以 User 用它）
```

### `Json(payload): Json<CreateUser>`

这不是普通参数。它告诉 axum：从请求 body 读 JSON，解析成 `CreateUser`，绑定到 `payload`。这个写法叫 **extractor**——用参数声明输入，axum 负责提取。

### `impl IntoResponse`

返回类型只要求"能转成 HTTP 响应"，不写死具体类型。`&'static str`、`Json<T>`、`(StatusCode, Json<T>)` 都实现了 `IntoResponse`，所以都能作为 handler 返回值。

### `(StatusCode::CREATED, Json(user))`

元组返回：状态码 `201`，响应体是 `user` 序列化成的 JSON。这是 axum 很常见的返回方式。

### `root`

GET handler 可以非常简单：无参数，返回 `&'static str`，axum 自动转成响应 body。

## 手写任务

跑通后做两个小改动：

1. 给 `CreateUser` 加 `email: String` 字段，curl 的 JSON body 也传入 email。
2. 给 `User` 加 `email: String` 字段，让响应 JSON 里返回 email。

预期响应：

````json
{"id":1337,"username":"alice","email":"alice@example.com"}
````

## 小结

- `Json<T>` 既能做请求 extractor，也能做响应包装器。
- `Deserialize` 用于输入（JSON → Rust），`Serialize` 用于输出（Rust → JSON）。
- extractor 用函数参数声明输入：`Json(payload): Json<CreateUser>`。
- `(StatusCode, Json(value))` 可以直接作为响应返回。
- `impl IntoResponse` 让 handler 返回"能变成响应的东西"，不用手写底层 `Response`。

## 源码对照

- `examples/readme/Cargo.toml`
- `examples/readme/src/main.rs`
