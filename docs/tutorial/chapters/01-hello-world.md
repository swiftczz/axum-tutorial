# 01. hello-world

对应示例：`examples/hello-world`

启动一个 HTTP 服务，访问 `GET /` 返回一段 HTML。这一章建立 axum 的最小闭环：`Router`、`route`、`handler`、`serve`。

## Cargo.toml

````toml
[package]
name = "example-hello-world"
version = "0.1.0"
edition = "2024"

[dependencies]
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }
````

两个依赖：

- `axum`：提供 `Router`、路由方法、handler 返回响应的能力。
- `tokio`：异步运行时，让 `async fn main`、监听端口这些异步操作能跑起来。`features = ["full"]` 开启全部功能，示例阶段图省事这么写。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

完整代码，复制即可运行：

````rust
use axum::{response::Html, routing::get, Router};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    println!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app).await;
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
````

注意代码组织：`use`、`async fn main`、`async fn handler` 都是顶层、平级的——handler 写在 `main` 外面，不是嵌在 `main` 里。整个教程都遵循这个结构。

## 运行

````bash
cd examples
cargo run -p example-hello-world
````

另开一个终端访问：

````bash
curl http://127.0.0.1:3000/
````

预期输出：

````html
<h1>Hello, World!</h1>
````

`curl` 是命令行 HTTP 客户端，和浏览器一样发 HTTP 请求，但输出更直接，适合验证后端接口。

常见卡点：

- 提示找不到 `../../axum/Cargo.toml`：当前仓库缺少本地 `axum/` crate，见 README 的运行前提。
- 提示 3000 端口被占用：先关掉占用端口的进程，或临时把源码里的 `127.0.0.1:3000` 改成别的端口。

## 解读

整个程序做一件事：浏览器访问 `GET /`，axum 把请求交给 `handler`，handler 返回一段 HTML。

```text
浏览器 GET /
  → TCP 连接进入 3000 端口
  → axum::serve 把请求交给 Router
  → Router 匹配 "/" 和 GET
  → handler 被调用
  → Html("<h1>...</h1>") 变成 HTTP 响应
```

### `use`

````rust
use axum::{response::Html, routing::get, Router};
````

从 axum 引入三样东西：`Html`（响应包装器）、`get`（GET 路由方法）、`Router`（路由表）。

### `#[tokio::main] async fn main`

````rust
#[tokio::main]
async fn main() {
    ...
}
````

axum 处理网络 I/O，网络 I/O 是异步的，所以 `main` 要写成 `async fn`。这里要把三件事一起理解：

- `async fn` 调用时**不立即执行**，而是返回一个 `Future`（"承诺以后给你结果"）。
- `.await` 负责驱动 `Future`，等待时让出线程给别的任务，数据准备好再回来。
- Rust 标准库**不提供**调度 `Future` 的运行时，`tokio` 就是这个运行时；`#[tokio::main]` 是个宏，帮你启动 Tokio 运行时来驱动 `async fn main` 这个 `Future`。

没有 `#[tokio::main]`，`async fn main` 返回的 `Future` 没人驱动，程序会直接退出。后面每一章都会看到这三件套，记住这条主线即可：

```text
async fn 返回 Future（不立即执行）
.await 驱动 Future，等待时让出线程
#[tokio::main] 启动 Tokio 运行时来调度所有 Future
```

### `Router::new().route("/", get(handler))`

````rust
let app = Router::new().route("/", get(handler));
````

- `Router::new()`：创建空路由表。
- `.route("/", ...)`：路径 `/` 怎么处理。
- `get(handler)`：只有 GET 请求交给 `handler`。

注意 `handler` 没有被手动调用——你只是把函数名交给 axum，真实请求来了 axum 自动调用它。

### `TcpListener::bind(...).await`

````rust
let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
    .await
    .unwrap();
````

向操作系统申请监听本机 3000 端口。`.await` 等 `Future` 完成（即端口绑定完成）；运行时在等待期间可以去处理别的任务。`.unwrap()` 失败就 panic，示例代码图省事这么写，真实项目要返回错误。

### `axum::serve(listener, app).await`

把 listener 收到的 HTTP 请求交给 `app` 这个 Router 处理。这一行之后程序就阻塞在这里，持续接收请求。

### `handler`

````rust
async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
````

- 无参数：不需要从请求里提取路径、查询、body。
- 返回 `Html<&'static str>`：响应体是一段 HTML。
- `&'static str` 是字符串字面量的类型。`'static` 是生命周期，表示"整个程序运行期间都有效"——字符串字面量在编译时固化进二进制，运行时一直存在，不需要手动管理内存。后面会反复看到 `'static`，基本都是这个意思。
- `Html(...)` 让响应带上 `text/html` 语义，浏览器知道这是 HTML 不是纯文本。

## 手写任务

跑通后做两个小改动确认理解：

1. 把首页内容改成自己的名字，例如 `<h1>Hello, Alice!</h1>`。
2. 新增 `GET /health` 路由返回 `ok`，用 `curl http://127.0.0.1:3000/health` 验证。

## 小结

- `Router` 是路由表，`.route("/", get(handler))` 把路径、方法、函数三者绑起来。
- handler 不用手动调用，请求来了 axum 自动调。
- handler 返回值只要能转成 HTTP 响应（如 `Html`），axum 就能发给客户端。
- `axum::serve(listener, app)` 把网络监听和路由表连起来。
- `async fn` + `.await` + `#[tokio::main]` 是 axum 服务的三件套。

## 源码对照

- `examples/hello-world/Cargo.toml`
- `examples/hello-world/src/main.rs`
