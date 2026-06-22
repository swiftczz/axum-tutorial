# 03. routes-and-handlers-close-together

对应示例：`examples/routes-and-handlers-close-together`

本章目标：学会按功能组织路由和 handler，让相关代码靠近。

## 这个小项目在做什么

这个 example 仍然很小，但它开始解决一个真实项目问题：

```text
路由越来越多时，所有 handler 都堆在 main.rs 顶层会很乱。
```

所以它演示一种组织方式：

- `root()` 函数返回 `/` 的 Router。
- `get_foo()` 函数返回 `GET /foo` 的 Router。
- `post_foo()` 函数返回 `POST /foo` 的 Router。
- `main()` 用 `.merge(...)` 把多个小 Router 合成一个应用。

这章的重点不是功能复杂，而是代码组织。

如果你刚学完前两章，可以这样定位本章：

```text
本章不增加新的 HTTP 功能。
本章只解决代码变多以后，路由和 handler 怎么摆放更清楚。
```

也就是说，请求、响应、端口、handler 的基本逻辑都和前两章一样，只是代码结构换了一种写法。

## 新手先抓住主线

请求处理流程还是一样：

```text
请求进入服务
-> 总 Router 收到请求
-> 总 Router 里合并过 root/get_foo/post_foo 三组路由
-> 匹配路径和方法
-> 调用对应局部 handler
```

和前两章不同的是：

```text
handler 不再全部放在顶层。
handler 被放到和路由定义靠近的位置。
```

这就是文件名里的含义：routes and handlers close together。

## 文件和依赖

这个 example 有两个文件：

1. `examples/routes-and-handlers-close-together/Cargo.toml`：声明包名和依赖。
2. `examples/routes-and-handlers-close-together/src/main.rs`：演示如何把路由和 handler 放在一起组织。

关键依赖：

- `axum`：提供 `Router`、`get`、`post` 和 `MethodRouter`。
- `tokio`：提供异步运行时，让服务可以监听端口并处理请求。

## 为什么要把路由和 handler 放近

在小项目里，你可以这样写：

````rust
Router::new()
    .route("/", get(root))
    .route("/foo", get(get_foo))
    .route("/foo", post(post_foo))
````

但真实项目会有很多接口，例如：

```text
GET /users
POST /users
GET /users/:id
PATCH /users/:id
DELETE /users/:id
```

如果所有 handler 都散在文件各处，你看一个路由时要跳来跳去。  
这个 example 的思路是：

```text
定义路由的地方，同时放处理这个路由的 handler。
```

## 第一步：main 只负责合并路由

源码：

````rust
let app = Router::new()
    .merge(root())
    .merge(get_foo())
    .merge(post_foo());
````

`merge` 的意思是：

```text
把另一个 Router 的路由合并进当前 Router。
```

这让 `main` 很干净：

```text
main 不关心每个 handler 的细节。
main 只关心应用由哪些路由模块组成。
```

## 第二步：每组路由自己返回 Router

例如：

````rust
fn get_foo() -> Router {
    async fn handler() -> &'static str {
        "Hi from `GET /foo`"
    }

    route("/foo", get(handler))
}
````

这里有一个新手容易疑惑的点：

```rust
async fn handler()
```

这个 handler 定义在 `get_foo` 函数里面。  
这是 Rust 允许的“内部函数”写法。

为什么这样写？

因为这个 handler 只服务于 `get_foo()` 这一个路由，没有必要放到外面让整个文件都看见。

## 第三步：封装公共 route 函数

源码：

````rust
fn route(path: &str, method_router: MethodRouter<()>) -> Router {
    Router::new().route(path, method_router)
}
````

这只是一个小工具函数。

它把：

````rust
Router::new().route(path, method_router)
````

封装成：

````rust
route(path, method_router)
````

为什么这么做？

因为 `root()`、`get_foo()`、`post_foo()` 都在重复创建一个小 Router。  
这个 helper 让每个局部路由函数更短。

但也要注意它的边界：

- 这里的 `MethodRouter<()>` 适合当前没有共享 state 的小 example。后面学到 `State<T>` 以后，Router 和 MethodRouter 会带上状态类型，写法会变得更具体。
- 内部 handler 适合“这个 handler 只服务于这一条路由”的场景。真实项目接口变多后，更常见的是按业务模块拆文件，例如 `users::routes()`、`todos::routes()`。

## 函数职责速查

- `main`：组装多个小 Router，绑定端口并启动服务。
- `root`：返回 `GET /` 的小 Router，内部 handler 返回 `"Hello, World!"`。
- `get_foo`：返回 `GET /foo` 的小 Router，内部 handler 只处理 GET 请求。
- `post_foo`：返回 `POST /foo` 的小 Router，和 `get_foo` 路径相同但方法不同。
- `route`：减少 `Router::new().route(...)` 的重复代码。

这里的重点不是每个 handler 的业务逻辑，而是观察路由和 handler 如何靠近、多个小 Router 如何通过 `.merge(...)` 合成一个应用。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-routes-and-handlers-close-together
//! ```

// 引入 get/post 路由函数、MethodRouter 类型和 Router。
use axum::{
    routing::{get, post, MethodRouter},
    Router,
};

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 创建总 Router，并把多个小 Router 合并进来。
    let app = Router::new()
        .merge(root())
        .merge(get_foo())
        .merge(post_foo());

    // 绑定监听端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 打印监听地址。
    println!("listening on {}", listener.local_addr().unwrap());

    // 启动 HTTP 服务。
    axum::serve(listener, app).await;
}

// 返回处理 GET / 的小 Router。
fn root() -> Router {
    // 这个 handler 只属于 root 路由，所以定义在 root 函数内部。
    async fn handler() -> &'static str {
        "Hello, World!"
    }

    // 把 GET / 绑定到内部 handler。
    route("/", get(handler))
}

// 返回处理 GET /foo 的小 Router。
fn get_foo() -> Router {
    // 这个 handler 只处理 GET /foo。
    async fn handler() -> &'static str {
        "Hi from `GET /foo`"
    }

    // 把 GET /foo 绑定到内部 handler。
    route("/foo", get(handler))
}

// 返回处理 POST /foo 的小 Router。
fn post_foo() -> Router {
    // 这个 handler 只处理 POST /foo。
    async fn handler() -> &'static str {
        "Hi from `POST /foo`"
    }

    // 把 POST /foo 绑定到内部 handler。
    route("/foo", post(handler))
}

// 小工具函数：用路径和 MethodRouter 创建一个只包含一条路由的 Router。
fn route(path: &str, method_router: MethodRouter<()>) -> Router {
    Router::new().route(path, method_router)
}
````

## 运行和验证

运行前先确认 `examples/routes-and-handlers-close-together/Cargo.toml` 里的 `axum = { path = "../../axum" }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-routes-and-handlers-close-together
````

测试：

````bash
curl http://127.0.0.1:3000/
curl http://127.0.0.1:3000/foo
curl -X POST http://127.0.0.1:3000/foo
````

三条命令的预期响应体分别是：

````text
Hello, World!
Hi from `GET /foo`
Hi from `POST /foo`
````

常见卡点：

- `GET /foo` 和 `POST /foo` 路径相同但方法不同，所以可以共存。
- 如果用浏览器直接访问 `/foo`，浏览器发的是 GET，只会命中 `get_foo()`，不会命中 `post_foo()`。

## 手写任务

完成本章代码后，试着做两个小改动：

1. 新增一个 `GET /bar` 路由，返回 `Hi from GET /bar`。
2. 把 `get_foo` 函数改名为 `foo_get_route`，但保持访问路径仍然是 `/foo`。

第二个任务要确认一件事：

```text
函数名是 Rust 代码里的名字。
URL 路径是 .route("/foo", ...) 里的字符串。
改函数名不会自动改变 URL 路径。
```

## 本章真正要记住什么

- `Router` 可以拆成多个小 Router。
- `.merge(...)` 可以把小 Router 合并成大 Router。
- handler 可以定义在函数内部，让它和对应路由靠近。
- 相同路径可以绑定不同 HTTP 方法。
- 代码组织本身也是后端项目的重要能力。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/routes-and-handlers-close-together/Cargo.toml`
- `examples/routes-and-handlers-close-together/src/main.rs`
