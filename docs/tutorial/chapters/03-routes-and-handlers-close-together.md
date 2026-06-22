# 03. routes-and-handlers-close-together

对应示例：`examples/routes-and-handlers-close-together`

本章不增加新的 HTTP 功能，只解决一个代码组织问题：路由变多时，handler 怎么摆放更清楚。思路是把相关路由和 handler 放在同一个函数里返回一个小 `Router`，再用 `.merge(...)` 合成总应用。

## Cargo.toml

````toml
[package]
name = "example-routes-and-handlers-close-together"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 完整代码

````rust
use axum::{
    routing::{get, post, MethodRouter},
    Router,
};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .merge(root())
        .merge(get_foo())
        .merge(post_foo());

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    println!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app).await;
}

fn root() -> Router {
    async fn handler() -> &'static str {
        "Hello, World!"
    }

    route("/", get(handler))
}

fn get_foo() -> Router {
    async fn handler() -> &'static str {
        "Hi from `GET /foo`"
    }

    route("/foo", get(handler))
}

fn post_foo() -> Router {
    async fn handler() -> &'static str {
        "Hi from `POST /foo`"
    }

    route("/foo", post(handler))
}

fn route(path: &str, method_router: MethodRouter<()>) -> Router {
    Router::new().route(path, method_router)
}
````

## 运行

````bash
cd examples
cargo run -p example-routes-and-handlers-close-together
````

三条 curl，对应三个路由：

````bash
curl http://127.0.0.1:3000/
curl http://127.0.0.1:3000/foo
curl -X POST http://127.0.0.1:3000/foo
````

预期响应：

````text
Hello, World!
Hi from `GET /foo`
Hi from `POST /foo`
````

注意 `GET /foo` 和 `POST /foo` 路径相同但方法不同，可以共存。浏览器访问 `/foo` 默认是 GET，只会命中 `get_foo()`。

## 解读

### 为什么要把路由和 handler 放近

前两章把所有 handler 都堆在 `main.rs` 顶层。真实项目接口变多后，这种写法要跳来跳去。本章的思路：

```text
定义路由的地方，同时放处理这个路由的 handler。
```

### `main` 只负责合并

````rust
let app = Router::new()
    .merge(root())
    .merge(get_foo())
    .merge(post_foo());
````

`.merge(...)` 把另一个 Router 的所有路由合并进当前 Router。这让 `main` 不关心每个 handler 的细节，只关心应用由哪些路由模块组成。真实项目里常见 `users::routes()`、`todos::routes()` 这种按业务模块拆分的写法。

### 每组路由自己返回 Router

````rust
fn get_foo() -> Router {
    async fn handler() -> &'static str {
        "Hi from `GET /foo`"
    }

    route("/foo", get(handler))
}
````

`async fn handler` 定义在 `get_foo` 函数内部——这是 Rust 允许的"内部函数"。因为这个 handler 只服务于 `GET /foo` 这一条路由，没必要放到外层让整个文件都看见。

### helper `route`

````rust
fn route(path: &str, method_router: MethodRouter<()>) -> Router {
    Router::new().route(path, method_router)
}
````

`root`、`get_foo`、`post_foo` 都在重复 `Router::new().route(...)`，这个 helper 让它们更短。

注意 `MethodRouter<()>`：这里的 `()` 表示 Router 没有携带共享 state。后面学到 `State<T>` 后，Router 和 MethodRouter 会带上状态类型，写法会更具体。

## 手写任务

跑通后做两个小改动：

1. 新增 `GET /bar` 路由，返回 `Hi from GET /bar`。
2. 把 `get_foo` 函数改名为 `foo_get_route`，但保持访问路径仍然是 `/foo`。

第二个任务要确认一件事：函数名是 Rust 代码里的名字，URL 路径是 `.route("/foo", ...)` 里的字符串，改函数名不会自动改 URL。

## 小结

- `Router` 可以拆成多个小 Router，再用 `.merge(...)` 合成总应用。
- handler 可以定义在函数内部，让它和对应路由靠近。
- 相同路径可以绑定不同 HTTP 方法（`GET /foo` 和 `POST /foo` 共存）。
- `MethodRouter<()>` 表示 Router 不带 state；后面学 `State<T>` 后类型会更具体。

## 源码对照

- `examples/routes-and-handlers-close-together/Cargo.toml`
- `examples/routes-and-handlers-close-together/src/main.rs`
