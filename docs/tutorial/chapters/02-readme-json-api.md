# 02. readme-json-api

对应示例：`examples/readme`

本章目标：手写 GET/POST JSON API，理解 `Json<T>`、`serde` 和 `IntoResponse`。

## 这个小项目在做什么

上一章只有一个 `GET /`，这一章开始像真实 API：

- `GET /` 返回普通文本。
- `POST /users` 接收 JSON 请求体。
- 服务端把 JSON 解析成 Rust 结构体。
- 创建一个 `User`。
- 返回 `201 Created` 和 JSON 响应。

请求主线是：

```text
客户端 POST /users
-> 请求 body 是 JSON
-> Json<CreateUser> 把 JSON 解析成 Rust 结构体
-> create_user 生成 User
-> (StatusCode::CREATED, Json(user)) 变成 HTTP 响应
```

这一章的重点不是数据库，而是先理解 HTTP JSON API 的输入和输出。

## 先看懂一次 POST 请求

`POST /users` 不是只有一个路径，它通常由几部分组成：

```text
方法：POST
路径：/users
Header：content-type: application/json
Body：{"username":"alice"}
```

逐个看：

- `POST`：表示客户端要提交数据，让服务端创建或处理资源。
- `/users`：表示这次提交和用户资源有关。
- `content-type: application/json`：告诉服务端 body 里的内容是 JSON。
- `{"username":"alice"}`：真正提交的数据。

所以这一章的 handler 要做两件事：

```text
从请求 body 里拿 JSON
把 JSON 转成 Rust 结构体
```

`Json<CreateUser>` 正是在做这件事。

## 文件和依赖

这个项目有两个文件：

1. `Cargo.toml`：声明 Axum、Serde、Tokio、tracing。
2. `src/main.rs`：写路由、handler、请求结构体和响应结构体。

关键依赖：

- `axum`：提供路由、`Json` extractor、响应转换。
- `serde`：把 JSON 和 Rust struct 互相转换。
- `tokio`：异步运行时。
- `tracing` / `tracing-subscriber`：打印结构化日志。

## 第一步：写 main，创建两个路由

核心代码：

````rust
let app = Router::new()
    .route("/", get(root))
    .route("/users", post(create_user));
````

这一段表达的是：

```text
GET /        -> root
POST /users -> create_user
```

为什么要把不同路径交给不同函数？

因为 handler 应该只关心一个明确的业务动作。  
`root` 只负责返回欢迎文本，`create_user` 只负责创建用户。

## 第二步：写最简单的 GET handler

````rust
async fn root() -> &'static str {
    "Hello, World!"
}
````

这个函数没有参数，说明它不需要读取请求内容。  
返回 `&'static str`，Axum 会自动把字符串转换成 HTTP 响应 body。

新手先记住：

```text
handler 的返回值不一定非得是 Response。
只要 Axum 知道怎么把它转成 Response，就可以返回。
```

## 第三步：定义输入结构体

````rust
#[derive(Deserialize)]
struct CreateUser {
    username: String,
}
````

这表示客户端发来的 JSON 应该长这样：

````json
{
  "username": "alice"
}
````

为什么要 `Deserialize`？

因为方向是：

```text
JSON 请求体 -> Rust struct
```

这个方向叫反序列化，所以需要 `Deserialize`。

## 第四步：定义输出结构体

````rust
#[derive(Serialize)]
struct User {
    id: u64,
    username: String,
}
````

为什么要 `Serialize`？

因为方向是：

```text
Rust struct -> JSON 响应体
```

这个方向叫序列化，所以需要 `Serialize`。

## 第五步：写 POST handler

源码：

````rust
async fn create_user(
    Json(payload): Json<CreateUser>,
) -> impl IntoResponse {
    let user = User {
        id: 1337,
        username: payload.username,
    };

    (StatusCode::CREATED, Json(user))
}
````

这里有几个关键点。

### `Json(payload): Json<CreateUser>`

这不是普通函数参数。

它的意思是：

```text
请 Axum 从请求 body 里读取 JSON，
再把 JSON 解析成 CreateUser，
最后把解析结果绑定到 payload 变量。
```

如果 JSON 格式不对，Axum 会在进入 handler 之前返回错误。

### `-> impl IntoResponse`

这个返回类型的意思是：

```text
我不直接写死返回类型，只保证返回的东西能转换成 HTTP 响应。
```

### `(StatusCode::CREATED, Json(user))`

这是 Axum 很常见的返回方式。

它表示：

```text
HTTP 状态码：201 Created
响应 body：user 转成 JSON
```

## 每个函数解读

### `main`

角色：程序入口。

它负责初始化日志、创建 Router、注册两个路由、绑定端口、启动服务。

为什么这样写：

- 初始化日志放在最前面，后续启动和请求处理才能输出调试信息。
- 路由都集中在 `Router::new()` 后面，服务有哪些入口一眼能看到。
- `axum::serve(listener, app)` 让 Router 真正接收请求。

### `root`

角色：最简单的 GET handler。

它证明 handler 可以非常简单：没有参数，只返回一个字符串。

### `create_user`

角色：JSON API handler。

它演示了 Axum 最核心的模式：

```text
用 extractor 声明输入
用普通 Rust 代码处理业务
用 IntoResponse 返回输出
```

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-readme
//! ```

// 引入状态码、响应转换、GET/POST 路由函数、Json 和 Router。
use axum::{
    http::StatusCode,
    response::IntoResponse,
    routing::{get, post},
    Json, Router,
};

// serde 用来在 JSON 和 Rust struct 之间转换。
use serde::{Deserialize, Serialize};

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 初始化日志系统。
    tracing_subscriber::fmt::init();

    // 创建 Router，并注册两个接口。
    let app = Router::new()
        // GET / 请求交给 root。
        .route("/", get(root))
        // POST /users 请求交给 create_user。
        .route("/users", post(create_user));

    // 绑定本机 3000 端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 输出监听地址。
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动服务。
    axum::serve(listener, app).await;
}

// GET / 的 handler，返回静态字符串。
async fn root() -> &'static str {
    "Hello, World!"
}

// POST /users 的 handler。
async fn create_user(
    // Json<CreateUser> 会把请求 body 解析成 CreateUser。
    Json(payload): Json<CreateUser>,
) -> impl IntoResponse {
    // 这里模拟创建用户；真实项目通常会写入数据库。
    let user = User {
        id: 1337,
        username: payload.username,
    };

    // 返回 201 状态码和 JSON 响应体。
    (StatusCode::CREATED, Json(user))
}

// 请求输入类型：客户端 JSON 需要能反序列化成这个结构体。
#[derive(Deserialize)]
struct CreateUser {
    username: String,
}

// 响应输出类型：这个结构体需要能序列化成 JSON。
#[derive(Serialize)]
struct User {
    id: u64,
    username: String,
}
````

## 运行和验证

运行前先确认 `examples/readme/Cargo.toml` 里的 `axum = { path = "../../axum" }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动服务：

````bash
cd examples
cargo run -p example-readme
````

测试 GET：

````bash
curl http://127.0.0.1:3000/
````

预期响应体：

````text
Hello, World!
````

测试 POST：

````bash
curl -i -X POST http://127.0.0.1:3000/users \
  -H 'content-type: application/json' \
  -d '{"username":"alice"}'
````

预期状态码：

````text
HTTP/1.1 201 Created
````

预期响应体类似：

````json
{"id":1337,"username":"alice"}
````

常见卡点：

- `Json<CreateUser>` 依赖 `content-type: application/json` 和合法 JSON body；如果少了 header 或 JSON 写错，Axum 会在进入 `create_user` 前返回错误。
- 这个 example 只是模拟创建用户，`id: 1337` 是写死的，不代表已经接入数据库。

## 手写任务

完成本章代码后，试着做两个小改动：

1. 给 `CreateUser` 增加 `email: String` 字段，并在 curl 的 JSON body 里传入 email。
2. 给 `User` 也增加 `email: String` 字段，让响应 JSON 里返回 email。

改完后，预期响应类似：

````json
{"id":1337,"username":"alice","email":"alice@example.com"}
````

## 本章真正要记住什么

- `Json<T>` 既可以做请求提取器，也可以做响应包装器。
- `Deserialize` 用于输入：JSON -> Rust。
- `Serialize` 用于输出：Rust -> JSON。
- `(StatusCode, Json(value))` 可以直接作为响应返回。
- `impl IntoResponse` 让 handler 返回“能变成响应的东西”，不用手写底层 `Response`。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/readme/Cargo.toml`
- `examples/readme/src/main.rs`
