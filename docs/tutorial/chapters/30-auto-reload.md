# 30. auto-reload

对应示例：`examples/auto-reload`

本章目标：理解开发期自动重载的工作方式，并学会用 `cargo-watch`、`systemfd` 和 `listenfd` 配合 Axum 服务。

这章和业务功能关系不大。  
它解决的是开发体验问题：改完代码后，不想手动停止、重新编译、重新启动服务。

## 这个小项目在做什么

应用本身只有一个接口：

```text
GET / -> <h1>Hello, World!</h1>
```

真正的重点是启动方式：

```text
systemfd --no-pid -s http::3000 -- cargo watch -x run
```

它实现的效果是：

```text
你修改 Rust 源码
-> cargo-watch 发现文件变化
-> 自动重新 cargo run
-> systemfd 保持监听端口
-> 新进程通过 listenfd 接收已有 listener
-> 服务重新启动
```

## 先理解为什么需要 auto reload

开发后端时，你会频繁做这个循环：

```text
改代码
编译
启动服务
浏览器或 curl 验证
再改代码
```

如果每次都手动执行：

```text
Ctrl+C
cargo run
```

会很低效。

`cargo-watch` 能监听文件变化并自动执行命令。  
但如果每次进程重启都重新绑定端口，可能遇到端口短暂占用或连接切换问题。

所以这个 example 加了：

```text
systemfd + listenfd
```

让监听 socket 由外部工具管理，服务进程重启时可以复用。

## cargo-watch、systemfd、listenfd 分别做什么

这三个名字容易混。

### cargo-watch

`cargo-watch` 负责监听文件变化。

例如：

```text
src/main.rs 改了
Cargo.toml 改了
```

它就重新执行：

```text
cargo run
```

### systemfd

`systemfd` 负责提前打开监听 socket。

例如：

```text
-s http::3000
```

表示监听 3000 端口，并把这个 listener 传给后面的程序。

### listenfd

`listenfd` 是 Rust 代码里的库。  
它负责从环境里取出 `systemfd` 传进来的 listener。

一句话串起来：

```text
cargo-watch 重启程序
systemfd 保管端口
listenfd 让 Axum 程序拿到这个端口 listener
```

## 文件和依赖

这个 example 有三个主要文件：

1. `examples/auto-reload/Cargo.toml`：声明 Axum、listenfd、Tokio。
2. `examples/auto-reload/src/main.rs`：实现 listenfd listener 接收和 Axum 服务启动。
3. `examples/auto-reload/README.md`：说明需要安装的开发工具和运行命令。

关键依赖：

- `axum`：提供 Router、Html response。
- `listenfd`：从外部环境接收已经打开的 listener。
- `tokio`：异步运行时和 `TcpListener`。

外部命令行工具：

```text
cargo-watch
systemfd
```

它们不是 Rust 代码依赖，需要安装到本机命令行。

## 第一步：安装开发工具

示例 README 写的是：

````bash
cargo install cargo-watch systemfd
````

安装后，你应该能运行：

````bash
cargo watch --version
systemfd --version
````

如果命令找不到，检查 Cargo 的 bin 目录是否在 PATH 里。  
通常是：

```text
~/.cargo/bin
```

## 第二步：写普通 Axum 路由

源码：

````rust
let app = Router::new().route("/", get(handler));
````

handler：

````rust
async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
````

这一部分和第一章非常像。  
自动重载不要求你改变业务 handler。

重点在后面的 listener 创建方式。

## 第三步：从环境读取 listener

源码：

````rust
let mut listenfd = ListenFd::from_env();
let listener = match listenfd.take_tcp_listener(0).unwrap() {
    Some(listener) => {
        listener.set_nonblocking(true).unwrap();
        TcpListener::from_std(listener).unwrap()
    }
    None => TcpListener::bind("127.0.0.1:3000").await.unwrap(),
};
````

这段是本章核心。

`ListenFd::from_env()` 会检查环境里有没有外部传进来的 listener。

然后：

````rust
listenfd.take_tcp_listener(0)
````

尝试取第 0 个 TCP listener。

有两种情况。

## 第四步：有 systemfd listener 时复用它

源码：

````rust
Some(listener) => {
    listener.set_nonblocking(true).unwrap();
    TcpListener::from_std(listener).unwrap()
}
````

如果程序是通过 `systemfd` 启动的，这里会拿到一个标准库的 TCP listener。

接下来要做两件事：

1. `set_nonblocking(true)`：设置成非阻塞模式。
2. `TcpListener::from_std(listener)`：转换成 Tokio 的异步 listener。

Axum 在 Tokio 运行时里工作，所以最终需要的是：

```text
tokio::net::TcpListener
```

## 第五步：没有 systemfd 时普通绑定端口

源码：

````rust
None => TcpListener::bind("127.0.0.1:3000").await.unwrap(),
````

如果你直接运行：

````bash
cargo run -p auto-reload
````

没有 `systemfd` 传 listener，程序就走普通逻辑，自己绑定：

```text
127.0.0.1:3000
```

这让 example 既支持自动重载启动，也支持普通启动。

## 第六步：用拿到的 listener 启动 Axum

源码：

````rust
println!("listening on {}", listener.local_addr().unwrap());
axum::serve(listener, app).await;
````

不管 listener 来自哪里，最后都交给：

````rust
axum::serve(listener, app)
````

所以对 Axum 来说没有区别：

```text
listener 可以是自己 bind 的
listener 也可以是 systemfd 传进来的
```

## 第七步：自动重载运行命令

示例 README 给出的命令：

````bash
systemfd --no-pid -s http::3000 -- cargo watch -x run
````

拆开看：

```text
systemfd
  --no-pid
  -s http::3000
  --
  cargo watch -x run
```

`-s http::3000` 表示监听 3000 端口。  
`--` 后面是 systemfd 要启动的子命令。  
`cargo watch -x run` 表示文件变化时自动执行 `cargo run`。

如果你在 workspace 根目录运行，并且需要指定包，可以改成：

````bash
systemfd --no-pid -s http::3000 -- cargo watch -x 'run -p auto-reload'
````

具体用哪条，取决于你当前所在目录和 Cargo workspace 结构。

## 函数职责速查

- `main`：创建 Router，从 `listenfd` 或本地 bind 获得 listener，启动 Axum 服务。
- `handler`：返回一个简单 HTML 页面。

这一章函数很少，因为主要知识点在开发工具和 listener 交接。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//!
//! ```not_rust
//! cargo run -p auto-reload
//! ```

use axum::{response::Html, routing::get, Router};
use listenfd::ListenFd;
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    // 创建一个普通 Axum 应用。
    let app = Router::new().route("/", get(handler));

    // 从环境变量中读取 systemfd 传进来的 listener。
    let mut listenfd = ListenFd::from_env();

    let listener = match listenfd.take_tcp_listener(0).unwrap() {
        // 如果 systemfd 提供了第 0 个 TCP listener，就复用它。
        Some(listener) => {
            // Tokio 需要非阻塞 listener。
            listener.set_nonblocking(true).unwrap();

            // 把标准库 listener 转成 Tokio listener。
            TcpListener::from_std(listener).unwrap()
        }
        // 如果没有 systemfd，就退回普通启动方式。
        None => TcpListener::bind("127.0.0.1:3000").await.unwrap(),
    };

    // 启动服务。
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

// 普通页面 handler。
async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
````

## 运行和验证

先安装工具：

````bash
cargo install cargo-watch systemfd
````

普通运行：

````bash
cargo run -p auto-reload
````

请求：

````bash
curl http://127.0.0.1:3000/
````

自动重载运行：

````bash
systemfd --no-pid -s http::3000 -- cargo watch -x run
````

如果在 workspace 根目录需要指定包：

````bash
systemfd --no-pid -s http::3000 -- cargo watch -x 'run -p auto-reload'
````

验证方式：

1. 启动自动重载命令。
2. 修改 `handler` 返回的 HTML 文本。
3. 保存文件。
4. 观察终端是否自动重新编译并重启。
5. 刷新浏览器或重新 curl，确认响应变化。

## 常见卡点

### 1. 为什么要用 systemfd，不只用 cargo-watch？

只用 `cargo-watch` 也能自动重启。  
`systemfd` 的额外价值是保留监听 socket，让新旧进程之间的端口交接更顺。

### 2. 为什么要 set_nonblocking(true)？

Tokio 使用异步 IO。  
从标准库拿到的 listener 需要设置成非阻塞，再转换成 `tokio::net::TcpListener`。

### 3. 为什么直接 cargo run 也能工作？

因为代码里有 fallback：

````rust
None => TcpListener::bind("127.0.0.1:3000").await.unwrap()
````

没有 systemfd 时，程序就自己绑定端口。

### 4. 为什么 cargo watch 命令可能需要加 -p？

如果你在 workspace 根目录运行，Cargo 可能需要知道要运行哪个包。  
这时可以用：

````bash
cargo watch -x 'run -p auto-reload'
````

如果你就在 example 包目录里运行，`cargo watch -x run` 通常就够了。

### 5. 自动重载适合生产环境吗？

不适合。  
这章是开发期工具。生产环境应该用 systemd、容器编排、进程管理器、滚动发布等方式管理服务。

## 手写任务

建议按下面顺序自己敲一遍：

1. 先写一个普通的 `GET /` Axum 服务。
2. 引入 `listenfd::ListenFd`。
3. 用 `ListenFd::from_env()` 创建读取器。
4. 调用 `take_tcp_listener(0)`。
5. `Some(listener)` 分支里设置非阻塞并转成 Tokio listener。
6. `None` 分支里普通 bind 3000。
7. 用 `systemfd --no-pid -s http::3000 -- cargo watch -x run` 验证自动重载。

加深练习：

1. 把返回 HTML 改成带版本号的文本，每次保存观察版本变化。
2. 新增 `/health` 路由，修改后确认自动重启生效。
3. 尝试只用 `cargo watch -x run`，对比和 systemfd 组合的区别。

## 本章真正要记住什么

自动重载不是 Axum 的业务能力，而是开发工作流能力。

核心分工是：

```text
cargo-watch -> 监听文件变化并重新运行命令
systemfd    -> 保持监听 socket
listenfd    -> Rust 程序接收 systemfd 传入的 listener
Axum        -> 用这个 listener 启动 HTTP 服务
```

代码里最重要的是保留普通启动 fallback。  
这样同一个程序既能开发期自动重载，也能普通 `cargo run`。

## 源码对照

本章手写版对应源码：

- `examples/auto-reload/src/main.rs`
- `examples/auto-reload/README.md`
- `examples/auto-reload/Cargo.toml`
