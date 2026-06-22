# 01. hello-world

对应示例：`examples/hello-world`

本章目标：手写第一个 Axum 服务，理解 `Router`、`route`、`handler`、`serve`。

## 这个小项目在做什么

这个项目只做一件事：启动一个 HTTP 服务，访问 `GET /` 时返回一段 HTML。

新手不要一开始背 Axum API，先抓住这一条请求路径：

```text
浏览器访问 http://127.0.0.1:3000/
-> TCP 连接进入监听端口
-> axum::serve 把请求交给 Router
-> Router 匹配 "/" 和 GET
-> handler 被调用
-> Html("<h1>Hello, World!</h1>") 变成 HTTP 响应
```

这就是 Axum 最小闭环。

## 先理解后端最小概念

不会后端时，先把这几个词对上就够了：

- 客户端：发请求的一方，可以是浏览器，也可以是命令行里的 `curl`。
- 服务端：接收请求并返回响应的一方，这一章里就是我们写的 Axum 程序。
- 地址：`127.0.0.1` 表示本机，也就是只访问你自己的电脑。
- 端口：`3000` 是这个服务监听的入口。电脑上可以同时运行很多服务，端口用来区分它们。
- 路径：URL 里的 `/`、`/users`、`/health` 这类部分，用来表示你要访问哪个功能。
- 方法：`GET` 表示读取资源，后面会看到 `POST` 表示提交数据。
- 响应：服务端处理完请求后返回给客户端的内容，包括状态码、响应头和响应体。

所以访问 `http://127.0.0.1:3000/` 可以拆成：

```text
http://        使用 HTTP 协议
127.0.0.1      访问本机
:3000          找到监听 3000 端口的服务
/              请求路径是首页
GET            浏览器默认用 GET 读取页面
```

## 文件和依赖

这个 example 只有两个文件：

1. `Cargo.toml`：声明包名和依赖。
2. `src/main.rs`：写完整 HTTP 服务。

关键依赖只有两个：

- `axum`：提供 `Router`、路由方法、handler 返回响应的能力。
- `tokio`：提供异步运行时，让 `async fn main`、监听端口、处理请求这些异步操作能运行。

## 第一步：先写服务入口

先写这个壳：

````rust
#[tokio::main]
async fn main() {
}
````

为什么 `main` 要这样写？

- `async fn main` 表示入口函数里可以使用 `.await`。
- 普通 Rust 程序没有异步运行时，`#[tokio::main]` 会帮你创建 Tokio 运行时。
- Axum 底层处理网络 I/O，网络 I/O 是异步的，所以需要 Tokio。

新手可以先记住：

```text
写 Axum 服务，main 通常是 #[tokio::main] async fn main()
```

## 第二步：创建 Router

源码里是：

````rust
let app = Router::new().route("/", get(handler));
````

这一行很重要。

拆开看：

- `Router::new()`：创建一个空路由表。
- `.route("/", ...)`：告诉 Axum，路径 `/` 应该怎么处理。
- `get(handler)`：告诉 Axum，只有 GET 请求交给 `handler`。

所以它表达的是：

```text
如果请求是 GET /
就调用 handler 函数
```

这里的 `handler` 不是手动调用的。你没有写 `handler()`。  
你只是把函数名交给 Axum，等真实请求来了，Axum 会自动调用它。

## 第三步：绑定端口并启动服务

源码：

````rust
let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
    .await
    .unwrap();
````

这段做的是：

- `127.0.0.1:3000`：只监听本机 3000 端口。
- `bind(...)`：向操作系统申请这个端口。
- `.await`：绑定端口是异步操作，要等待完成。
- `.unwrap()`：示例代码为了简单，失败就直接 panic。真实项目里通常要返回错误或打印更清楚的日志。

最后：

````rust
axum::serve(listener, app).await;
````

意思是：

```text
把 listener 收到的 HTTP 请求交给 app 这个 Router 处理。
```

## 第四步：写 handler

源码：

````rust
async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
````

这是第一个请求处理函数。

为什么可以这样写？

- handler 是普通 Rust 函数，只是它是 `async`。
- 没有参数，说明它不需要从请求里提取路径、查询、body、状态。
- 返回 `Html<&'static str>`，表示响应体是一段 HTML。
- `Html(...)` 实现了 Axum 的响应转换能力，Axum 会把它变成 HTTP response。

如果只返回字符串，也能工作；这里用 `Html` 是为了告诉浏览器：

```text
这不是普通文本，这是 HTML。
```

## 每个函数解读

### `main`

角色：程序入口。

它负责四件事：

1. 创建 Router。
2. 注册路由。
3. 监听端口。
4. 启动 HTTP 服务。

新手理解重点：

```text
main 不处理具体业务，它只负责把服务搭起来。
```

### `handler`

角色：业务处理函数。

它负责：

1. 接收请求。
2. 生成响应内容。
3. 把 HTML 返回给 Axum。

新手理解重点：

```text
handler 的返回值只要能转换成 HTTP 响应，Axum 就能把它发给客户端。
```

## 带中文注释的手写版

这份代码适合你第一遍手写时使用，注释帮助理解每行代码为什么存在。

````rust
//! 这是 crate 级文档注释，说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-hello-world
//! ```

// 从 axum 引入 Html 响应包装器、GET 路由函数和 Router 路由表。
use axum::{response::Html, routing::get, Router};

// 创建 Tokio 异步运行时，并允许 main 函数使用 async/await。
#[tokio::main]
async fn main() {
    // 创建应用路由表：GET / 请求会交给 handler 函数。
    let app = Router::new().route("/", get(handler));

    // 监听本机 3000 端口；await 表示等待端口绑定完成。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 打印实际监听地址，方便运行后确认服务已经启动。
    println!("listening on {}", listener.local_addr().unwrap());

    // 把 TCP listener 和 Router 连接起来，开始处理 HTTP 请求。
    axum::serve(listener, app).await;
}

// 处理 GET / 请求，返回一段 HTML。
async fn handler() -> Html<&'static str> {
    // Html 包装器会让响应带上 text/html 语义。
    Html("<h1>Hello, World!</h1>")
}
````

## 运行和验证

运行前先确认 `examples/hello-world/Cargo.toml` 里的 `axum = { path = "../../axum" }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

运行：

````bash
cd examples
cargo run -p example-hello-world
````

访问：

````bash
curl http://127.0.0.1:3000/
````

`curl` 是一个命令行 HTTP 客户端。它和浏览器一样会发送 HTTP 请求，但输出更直接，适合验证后端接口。

预期看到：

````html
<h1>Hello, World!</h1>
````

常见卡点：

- 如果提示找不到 `../../axum/Cargo.toml`，说明当前仓库缺少本地 `axum/` crate。
- 如果提示 3000 端口被占用，先关闭占用端口的进程，或者临时把源码里的 `127.0.0.1:3000` 改成其他端口。

## 手写任务

完成本章代码后，做两个小改动确认自己真的理解了：

1. 把首页返回内容改成自己的名字，例如 `<h1>Hello, Alice!</h1>`。
2. 新增一个 `GET /health` 路由，返回 `ok`，再用 `curl http://127.0.0.1:3000/health` 验证。

## 本章真正要记住什么

- `Router` 是路由表。
- `.route("/", get(handler))` 是把路径、HTTP 方法和函数绑定起来。
- handler 不需要你手动调用，请求来了 Axum 会调用。
- handler 返回的值只要实现响应转换能力，就能成为 HTTP 响应。
- `axum::serve(listener, app)` 是把网络监听和路由表连接起来。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/hello-world/Cargo.toml`
- `examples/hello-world/src/main.rs`
