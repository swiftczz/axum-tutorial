# 32. auto-reload

对应示例：`examples/auto-reload`

解决开发体验问题:改完代码不想手动停/编译/重启。用 `cargo-watch`、`systemfd`、`listenfd` 配合 axum,实现改代码后自动重新编译重启,且端口由外部工具管理不释放。这章和业务功能关系不大,是开发工作流能力。

## Cargo.toml

````toml
[package]
name = "auto-reload"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
listenfd = "1.0.1"
tokio = { version = "1.0", features = ["full"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

````rust
use axum::{response::Html, routing::get, Router};
use listenfd::ListenFd;
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(handler));

    let mut listenfd = ListenFd::from_env();

    let listener = match listenfd.take_tcp_listener(0).unwrap() {
        Some(listener) => {
            listener.set_nonblocking(true).unwrap();
            TcpListener::from_std(listener).unwrap()
        }
        None => TcpListener::bind("127.0.0.1:3000").await.unwrap(),
    };

    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
````

## 运行

先装外部工具(`cargo-watch`、`systemfd` 是命令行工具,不是 Rust 依赖):

````bash
cargo install cargo-watch systemfd
````

普通运行:

````bash
cd examples
cargo run -p auto-reload
curl http://127.0.0.1:3000/
````

自动重载运行:

````bash
systemfd --no-pid -s http::3000 -- cargo watch -x 'run -p auto-reload'
````

验证:

1. 启动自动重载命令。
2. 修改 `handler` 返回的 HTML 文本。
3. 保存文件。
4. 观察终端自动重新编译并重启。
5. 重新 curl,确认响应变化。

## 解读

### 三个工具分工

容易混的三个名字:

- **`cargo-watch`**:监听文件变化(如 `src/main.rs` 改了)→ 自动重新执行 `cargo run`。
- **`systemfd`**:启动程序前**提前打开监听 socket**(`-s http::3000`),把它传给后面的程序。负责保管端口,让程序重启时复用。
- **`listenfd`**:**Rust 代码里的库**,负责从环境取出 `systemfd` 传进来的 listener。

一句话串起来:

```text
cargo-watch 重启程序
systemfd 保管端口
listenfd 让 axum 程序拿到这个端口 listener
```

### 为什么不只 cargo-watch 还要 systemfd

只用 `cargo-watch` 也能自动重启,但每次重启都重新绑定端口,可能遇到端口短暂占用或连接切换问题。`systemfd` 让监听 socket 由外部工具管理,服务进程重启时复用同一个 socket,新旧进程交接更顺。

### 关键代码:listener 来源二选一

````rust
let mut listenfd = ListenFd::from_env();

let listener = match listenfd.take_tcp_listener(0).unwrap() {
    Some(listener) => {                                  // systemfd 传了 listener
        listener.set_nonblocking(true).unwrap();         // Tokio 要非阻塞
        TcpListener::from_std(listener).unwrap()         // 转 Tokio listener
    }
    None => TcpListener::bind("127.0.0.1:3000").await.unwrap(),  // 没 systemfd 就自己 bind
};
````

`ListenFd::from_env()` 检查环境里有没有外部传进来的 listener,`take_tcp_listener(0)` 取第 0 个 TCP listener。

**为什么 `set_nonblocking(true)`?** Tokio 用异步 IO,从标准库拿到的 listener 要设成非阻塞,再 `TcpListener::from_std` 转成 `tokio::net::TcpListener`(axum 在 Tokio 运行时工作)。

**为什么有 `None` fallback?** 直接 `cargo run` 没有 systemfd 传 listener,程序走普通逻辑自己绑定 `127.0.0.1:3000`。这让 example 既支持自动重载启动,也支持普通启动。

### 不管 listener 来源,最终都一样

````rust
axum::serve(listener, app).await
````

对 axum 来说没区别:listener 可以是自己 bind 的,也可以是 systemfd 传进来的。

### 自动重载命令拆解

````bash
systemfd --no-pid -s http::3000 -- cargo watch -x 'run -p auto-reload'
````

- `--no-pid`:不记录 PID。
- `-s http::3000`:监听 3000 端口。
- `--`:后面是 systemfd 要启动的子命令。
- `cargo watch -x 'run -p auto-reload'`:文件变化时执行 `cargo run -p auto-reload`。

`-p auto-reload` 是因为这是 workspace,要指定包;在 example 包目录里运行 `cargo watch -x run` 就够。

## 常见问题

**为什么要 systemfd 不只用 cargo-watch?** systemfd 保留监听 socket,新旧进程端口交接更顺;只用 cargo-watch 每次重启重新绑定可能端口短暂占用。

**直接 `cargo run` 为什么也能工作?** 代码有 fallback:`None => TcpListener::bind(...)`,没 systemfd 时自己绑定。

**`cargo watch` 命令要不要 `-p`?** workspace 根目录运行 Cargo 需要知道运行哪个包,加 `-p`;在包目录里运行通常不用。

**自动重载适合生产吗?** 不适合。这是开发期工具,生产用 systemd/容器编排/进程管理器/滚动发布。

## 手写任务

按下面顺序敲:

1. 写一个普通 `GET /` axum 服务。
2. 引入 `listenfd::ListenFd`。
3. `ListenFd::from_env()` 创建读取器。
4. 调 `take_tcp_listener(0)`。
5. `Some(listener)` 分支设置非阻塞并转成 Tokio listener。
6. `None` 分支普通 bind 3000。
7. `systemfd --no-pid -s http::3000 -- cargo watch -x run` 验证自动重载。

加深练习:

1. 把返回 HTML 改成带版本号的文本,每次保存观察版本变化。
2. 新增 `/health` 路由,修改后确认自动重启生效。
3. 尝试只用 `cargo watch -x run`,对比和 systemfd 组合的区别。

## 小结

- 自动重载是开发工作流能力,不是 axum 业务能力。
- 三工具分工:`cargo-watch` 监听文件变化重启、`systemfd` 保管监听 socket、`listenfd` 让 Rust 程序接收 listener。
- 关键代码保留 `None` fallback:同一个程序既能开发期自动重载,也能普通 `cargo run`。
- 从标准库拿的 listener 要 `set_nonblocking(true)` 再 `TcpListener::from_std` 转 Tokio listener。
- 自动重载只适合开发,生产用 systemd/容器编排等。

## 源码对照

- `examples/auto-reload/Cargo.toml`
- `examples/auto-reload/src/main.rs`
- `examples/auto-reload/README.md`
