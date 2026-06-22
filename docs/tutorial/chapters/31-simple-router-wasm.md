# 31. simple-router-wasm

对应示例：`examples/simple-router-wasm`

本章目标：理解 Axum 不一定只能运行成传统 TCP HTTP 服务器，也可以把 `Router` 当成一个处理单次 `Request` 的 service，在 WASM 或 serverless 风格环境中使用。

前面所有示例几乎都是：

```text
TcpListener
-> axum::serve(listener, app)
```

这一章换一个角度：不启动端口，只构造一个 `Request`，把它交给 Axum Router，拿到 `Response`。

## 这个小项目在做什么

应用只有一个路由：

```text
GET /api/ -> <h1>Hello, World!</h1>
```

但它没有监听 `127.0.0.1:3000`。

它做的是：

```text
手动创建 Request
-> 调用 app(request)
-> app 内部创建 Router
-> router.call(request)
-> 得到 Response
-> assert status 是 200
```

这模拟了很多 serverless 或 WASM 运行时的模式：

```text
运行时收到一个请求
-> 调用你导出的函数
-> 你的函数返回一个响应
```

## 先理解传统服务器和本章模式的区别

传统 Axum 服务通常是：

```text
程序自己监听端口
程序自己接收 HTTP 连接
程序一直运行
```

代码像这样：

````rust
let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
axum::serve(listener, app).await.unwrap();
````

而本章更像：

```text
外部运行时负责接收请求
你的函数只负责把 Request 变成 Response
```

代码像这样：

````rust
let response = router.call(request).await.unwrap();
````

这就是理解 serverless 和部分 WASM 场景的关键。

## 为什么 WASM 要关闭 Axum 默认 feature

`Cargo.toml` 里写了：

````toml
axum = { path = "../../axum", default-features = false }
````

注释里解释了原因：

```text
tokio 的 IO 层 mio 不支持 wasm32-unknown-unknown
```

Axum 默认 feature 会拉入一些传统服务器运行时需要的能力。  
但 `wasm32-unknown-unknown` 目标没有常规操作系统 TCP IO。

所以这个 example 关闭 Axum 默认 feature：

```text
不使用 Axum 的传统 server 部分
只使用 routing、extractor、response、tower service 等核心能力
```

这也是本章重点：

```text
Axum Router 本身是 Tower service
不一定非要配合 TcpListener 使用
```

## 文件和依赖

这个 example 有两个主要文件：

1. `examples/simple-router-wasm/Cargo.toml`：关闭 Axum 默认 feature，并声明 futures-executor、http、tower-service。
2. `examples/simple-router-wasm/src/main.rs`：手动构造请求，调用 Router，并断言响应状态码。

关键依赖：

- `axum`：提供 Router、routing、Html、Response。
- `http`：提供通用 HTTP `Request` 类型。
- `tower-service`：提供 `Service` trait，让我们能调用 `router.call(request)`。
- `futures-executor`：提供 `block_on`，在同步 `main` 里执行 async 函数。
- `axum-extra`：本例代码没有直接使用它。example 故意把它加进依赖，是为了**验证 axum-extra 在 wasm 目标下也能编译**（关闭默认 feature 后）。你自己的项目里不需要加。

## 第一步：手动构造 Request

源码：

````rust
let request: Request<String> = Request::get("https://serverless.example/api/")
    .body("Some Body Data".into())
    .unwrap();
````

这里没有浏览器，也没有 curl。  
代码手动创建了一个 HTTP 请求。

请求方法是：

```text
GET
```

请求 URI 是：

```text
https://serverless.example/api/
```

body 是：

```text
Some Body Data
```

虽然这个路由不读取 body，但这样写可以说明：

```text
Request 本身就是一个方法、URI、header、body 的结构体
```

传统服务器只是帮你从网络连接里构造出这个 Request。  
本章直接手动构造。

## 第二步：用 block_on 调用 async app

源码：

````rust
let response: Response = block_on(app(request));
assert_eq!(200, response.status());
````

`app(request)` 是 async 函数，返回 future。  
普通同步 `main` 里不能直接 `.await`，所以这里用：

````rust
block_on(app(request))
````

`block_on` 会运行这个 future，直到拿到结果。

在真实 WASM 或 serverless 运行时里，通常是运行时负责调用 async 函数。  
这个 example 用 `main` 来模拟运行时。

## 第三步：app 函数像 serverless handler

源码：

````rust
async fn app(request: Request<String>) -> Response {
    let mut router = Router::new().route("/api/", get(index));
    let response = router.call(request).await.unwrap();
    response
}
````

这个函数接收一个请求：

````rust
Request<String>
````

返回一个响应：

````rust
Response
````

这很像 serverless 平台常见的函数签名：

```text
fn handle(request) -> response
```

它内部仍然可以使用 Axum Router：

````rust
Router::new().route("/api/", get(index))
````

也就是说：

```text
外面是 serverless 风格
里面仍然用 Axum 的路由和 handler
```

## 第四步：Router 是 Tower Service

源码里引入了：

````rust
use tower_service::Service;
````

这是为了能调用：

````rust
router.call(request).await
````

`Router` 实现了 Tower 的 `Service` trait。  
可以先把 Service 理解成：

```text
接收一个 Request
异步返回一个 Response
```

这就是为什么 Router 不一定要通过 `axum::serve` 使用。  
你也可以直接把请求交给它。

## 第五步：index handler 返回 HTML

源码：

````rust
async fn index() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
````

这和前面章节一样，是普通 Axum handler。

说明即使外层运行环境不同，handler 的写法仍然很熟悉：

```text
extractor
handler
response
Router
```

这些 Axum 核心能力依然可以用。

## 第六步：为什么 Router 要是 mut

源码：

````rust
let mut router = Router::new().route("/api/", get(index));
let response = router.call(request).await.unwrap();
````

`Service::call` 需要可变引用，因为 service 可能维护内部状态，例如 readiness、缓存或中间件状态。

所以这里要写：

````rust
let mut router = ...
````

如果去掉 `mut`，编译器会提示不能可变借用。

## 第七步：这个 example 和真正 WASM 的距离

这个文件可以普通运行：

````bash
cargo run -p example-simple-router-wasm
````

也应该能针对 wasm 目标编译：

````bash
cargo build -p example-simple-router-wasm --target wasm32-unknown-unknown
````

但它不是一个完整浏览器应用，也不是完整 Cloudflare Worker 示例。  
它展示的是最核心的适配点：

```text
把外部运行时给你的请求
交给 Axum Router
再把 Response 还给外部运行时
```

真实平台还需要处理平台自己的 request/response 类型转换。

## 函数职责速查

- `main`：模拟运行时，构造一个请求，调用 `app`，断言响应状态码。
- `app`：创建 Axum Router，把单次请求交给 Router 处理，返回响应。
- `index`：普通 Axum handler，返回 HTML。

这一章的关键不是函数数量，而是调用方式从 `axum::serve` 变成了 `router.call(request)`。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//!
//! ```not_rust
//! cargo run -p example-simple-router-wasm
//! ```
//!
//! 这个 example 展示 Axum 在 WASM 风格环境中的用法。
//! 它不启动 TCP 服务器，而是手动构造 Request，然后调用 Router。

use axum::{
    response::{Html, Response},
    routing::get,
    Router,
};
use futures_executor::block_on;
use http::Request;
use tower_service::Service;

fn main() {
    // 模拟 serverless 或 WASM 运行时收到的 HTTP 请求。
    let request: Request<String> = Request::get("https://serverless.example/api/")
        .body("Some Body Data".into())
        .unwrap();

    // app 是 async 函数。同步 main 里用 block_on 等它执行完。
    let response: Response = block_on(app(request));

    // 验证 Router 返回了 200。
    assert_eq!(200, response.status());
}

#[allow(clippy::let_and_return)]
async fn app(request: Request<String>) -> Response {
    // 创建一个普通 Axum Router。
    let mut router = Router::new().route("/api/", get(index));

    // Router 实现了 Tower Service，可以直接 call 一个 Request。
    let response = router.call(request).await.unwrap();

    response
}

// 普通 Axum handler。
async fn index() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
````

## 运行和验证

普通运行：

````bash
cargo run -p example-simple-router-wasm
````

如果没有输出并且程序正常退出，说明断言通过了：

```text
response.status() == 200
```

编译到 wasm 目标：

````bash
cargo build -p example-simple-router-wasm --target wasm32-unknown-unknown
````

如果本机没有安装 wasm target，可以先安装：

````bash
rustup target add wasm32-unknown-unknown
````

## 常见卡点

### 1. 这是不是浏览器里的前端 WASM？

不是。  
这个 example 主要展示 Axum Router 在 WASM 目标下可编译，以及 serverless 风格的 request/response 调用方式。

### 2. 为什么没有 tokio::main？

因为这个例子不启动 Tokio TCP server。  
它用 `futures_executor::block_on` 在同步 `main` 中执行 async 函数。

### 3. 为什么 Axum 要 default-features = false？

因为默认 feature 可能引入 Tokio IO，而 `wasm32-unknown-unknown` 不支持常规操作系统 IO。  
关闭默认 feature 后，可以只使用 Router、handler、extractor、response 这些核心能力。

### 4. 为什么要 use tower_service::Service？

因为 `router.call(request)` 来自 `Service` trait。  
不引入这个 trait，方法可能无法被调用。

### 5. 为什么 URI 是完整 URL，路由还能匹配 /api/？

HTTP Request 里可以带完整 URI。  
Router 匹配时关注的是路径部分：

```text
/api/
```

所以能匹配：

````rust
.route("/api/", get(index))
````

## 手写任务

建议按下面顺序自己敲一遍：

1. 创建一个普通 `Router::new().route("/api/", get(index))`。
2. 写 `index` handler，返回 `Html`。
3. 手动构造 `Request::get("https://serverless.example/api/")`。
4. 写 `async fn app(request) -> Response`。
5. 在 `app` 里用 `router.call(request).await`。
6. 在 `main` 里用 `block_on(app(request))`。
7. 断言状态码是 200。

加深练习：

1. 新增 `/api/health` 路由，手动构造请求验证状态码。
2. 把 handler 改成返回 JSON，观察 Response 仍然能正常返回。
3. 尝试编译 `wasm32-unknown-unknown`，理解哪些依赖不能用于该目标。

## 本章真正要记住什么

Axum 的核心不只是“监听端口”。  
更底层地看：

```text
Router 是一个 Tower Service
Service 接收 Request
Service 返回 Response
```

传统服务器里，`axum::serve` 负责从 TCP 连接中产生 Request。  
serverless 或 WASM 风格环境里，外部运行时可能已经给了你 Request，你只需要调用：

````rust
router.call(request).await
````

这让 Axum 的路由、extractor 和 response 能在更多运行环境里复用。

## 源码对照

本章手写版对应源码：

- `examples/simple-router-wasm/src/main.rs`
- `examples/simple-router-wasm/Cargo.toml`
