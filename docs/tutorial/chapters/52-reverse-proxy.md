# 52. reverse-proxy

对应示例：`examples/reverse-proxy`

本章目标：理解反向代理的基本请求转发：客户端访问代理服务，代理服务把请求转发到后端服务，并把响应返回给客户端。

上一章是正向代理：

```text
客户端显式配置代理
```

这一章是反向代理：

```text
客户端访问代理
代理背后转发到真实后端
```

## 这个小项目在做什么

程序启动两个服务：

```text
真实后端：127.0.0.1:3000
反向代理：127.0.0.1:4000
```

访问：

```text
http://127.0.0.1:4000/
```

代理会转发到：

```text
http://127.0.0.1:3000/
```

后端返回：

```text
Hello, world!
```

代理把这个响应返回给客户端。

## 正向代理和反向代理的区别

正向代理：

```text
客户端知道代理存在
代理帮客户端访问外部网站
```

反向代理：

```text
客户端以为自己在访问服务
代理在服务端内部把请求转给后端
```

Nginx、Envoy、网关、负载均衡器经常扮演反向代理角色。

## 文件和依赖

这个 example 有两个文件：

1. `examples/reverse-proxy/Cargo.toml`
2. `examples/reverse-proxy/src/main.rs`

关键依赖：

- `axum`：实现代理入口和后端服务。
- `hyper-util`：创建 HTTP client，把请求转发到后端。
- `hyper`：HTTP 类型和状态码。
- `tokio`：同时运行后端和代理。

## 第一步：启动后端服务

源码：

````rust
async fn server() {
    let app = Router::new().route("/", get(|| async { "Hello, world!" }));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await;
}
````

这是被代理的真实服务。  
它只返回：

```text
Hello, world!
```

主函数中用：

````rust
tokio::spawn(server());
````

把它放到后台运行。

## 第二步：创建 Hyper client

源码：

````rust
let client: Client =
    hyper_util::client::legacy::Client::<(), ()>::builder(TokioExecutor::new())
        .build(HttpConnector::new());
````

反向代理需要主动请求后端服务。  
所以它需要一个 HTTP client。

这里的 `Client` 类型别名：

````rust
type Client = hyper_util::client::legacy::Client<HttpConnector, Body>;
````

## 第三步：把 client 放进 state

源码：

````rust
let app = Router::new().route("/", get(handler)).with_state(client);
````

handler 需要复用 HTTP client。  
所以把 client 放进 Axum state。

## 第四步：读取原请求路径和 query

源码：

````rust
let path = req.uri().path();
let path_query = req
    .uri()
    .path_and_query()
    .map(|v| v.as_str())
    .unwrap_or(path);
````

反向代理需要保留客户端请求的 path 和 query。

例如：

```text
/users?page=1
```

应该转发到：

```text
http://127.0.0.1:3000/users?page=1
```

## 第五步：改写请求 URI

源码：

````rust
let uri = format!("http://127.0.0.1:3000{path_query}");

*req.uri_mut() = Uri::try_from(uri).unwrap();
````

这里把原始请求改成后端地址。

客户端请求代理：

```text
http://127.0.0.1:4000/
```

handler 内部转成：

```text
http://127.0.0.1:3000/
```

### ⚠️ 本示例没改 Host header，生产代理必须改

注意示例**只改了 URI，没动 `Host` header**。后端收到的 `Host` 仍然是 `127.0.0.1:4000`（代理地址），而不是 `127.0.0.1:3000`。这在教学里没问题，但在生产环境会导致：

- 后端基于 Host 的路由（虚拟主机）失效
- 后端生成的绝对 URL（重定向、静态资源链接）错误
- 后端日志记录的访问来源不对

生产反向代理通常要做两件事：

```text
1. 改写 Host header 成后端地址
2. 注入 X-Forwarded-* 系列头，让后端知道真实客户端信息
```

具体来说：

````rust
// 改写 Host
req.headers_mut().insert(
    hyper::header::HOST,
    "127.0.0.1:3000".parse().unwrap(),
);

// 注入客户端真实 IP（代理转发时，后端拿到的"客户端"其实是代理）
req.headers_mut().insert(
    "x-forwarded-for",
    client_ip.parse().unwrap(),
);

// 注入原始协议（如果代理本身是 HTTPS，后端是 HTTP，要让后端知道原始是 https）
req.headers_mut().insert(
    "x-forwarded-proto",
    "https".parse().unwrap(),
);
````

`X-Forwarded-For`、`X-Forwarded-Proto`、`X-Forwarded-Host` 这组 header 是反向代理的"增值"功能——它们让后端能还原出**经过代理前的真实请求信息**。Nginx、HAProxy 等生产代理都会自动注入这些。本示例作为学习模型，跳过了这些细节，但你要知道真实代理必须处理。

> 顺带一提：反向代理还必须处理 **hop-by-hop headers**（见第 51 章）。`Connection`、`Transfer-Encoding`、`Keep-Alive` 等 header 只在相邻两跳之间有效，不能原样透传。`hyper-util` 的 client 在本示例里自动处理了一部分，但如果你手写别的转发逻辑，要按 RFC 7230 §6.1 剥离这些 header。

## 第六步：转发请求并返回响应

源码：

````rust
Ok(client
    .request(req)
    .await
    .map_err(|_| StatusCode::BAD_REQUEST)?
    .into_response())
````

`client.request(req).await` 把修改后的请求发给后端。  
拿到后端响应后，直接转成 Axum response 返回给客户端。

这就是最小反向代理。

## 函数职责速查

- `main`：启动后端服务，创建代理 client，启动代理服务。
- `handler`：改写请求 URI，转发请求，返回后端响应。
- `server`：模拟真实后端服务。

## 带中文注释的手写版

完整 `src/main.rs`：

````rust
//! Reverse proxy listening in "localhost:4000" will proxy all requests to "localhost:3000"
//! endpoint.
//!
//! Run with
//!
//! ```not_rust
//! cargo run -p example-reverse-proxy
//! ```

use axum::{
    body::Body,
    extract::{Request, State},
    http::uri::Uri,
    response::{IntoResponse, Response},
    routing::get,
    Router,
};
use hyper::StatusCode;
use hyper_util::{client::legacy::connect::HttpConnector, rt::TokioExecutor};

// 代理需要主动请求后端，所以这里准备一个 Hyper client 类型别名。
type Client = hyper_util::client::legacy::Client<HttpConnector, Body>;

#[tokio::main]
async fn main() {
    // 先启动一个本地后端服务，监听 127.0.0.1:3000。
    tokio::spawn(server());

    // 创建 HTTP client。后面 handler 会用它把请求转发到后端。
    let client: Client =
        hyper_util::client::legacy::Client::<(), ()>::builder(TokioExecutor::new())
            .build(HttpConnector::new());

    // 代理服务监听 127.0.0.1:4000，并把 client 放进 state。
    let app = Router::new().route("/", get(handler)).with_state(client);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:4000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}

async fn handler(State(client): State<Client>, mut req: Request) -> Result<Response, StatusCode> {
    // 取出原请求的 path 和 query。
    let path = req.uri().path();
    let path_query = req
        .uri()
        .path_and_query()
        .map(|v| v.as_str())
        .unwrap_or(path);

    // 改写成后端服务地址。
    let uri = format!("http://127.0.0.1:3000{path_query}");
    *req.uri_mut() = Uri::try_from(uri).unwrap();

    // 用 Hyper client 转发请求，并把响应返回给客户端。
    Ok(client
        .request(req)
        .await
        .map_err(|_| StatusCode::BAD_REQUEST)?
        .into_response())
}

async fn server() {
    // 这是被代理的真实后端服务。
    let app = Router::new().route("/", get(|| async { "Hello, world!" }));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}
````

## 运行和验证

运行：

````bash
cargo run -p example-reverse-proxy
````

请求代理：

````bash
curl http://127.0.0.1:4000/
````

预期返回：

```text
Hello, world!
```

## 常见卡点

### 1. 为什么要改 URI？

HTTP client 需要知道真正请求哪个后端地址。  
代理收到的 URI 是给代理自己的，要改成后端服务 URI。

### 2. 这个代理能用于生产吗？

不能直接用。  
真实反向代理还要处理 headers、负载均衡、超时、错误重试、body 大小、TLS、WebSocket 等。

### 3. 为什么 client 放进 state？

HTTP client 可以复用。  
放进 state 后每个请求 handler 都能使用同一个 client。

## 手写任务

1. 先写一个后端服务监听 3000。
2. 写一个代理服务监听 4000。
3. 创建 Hyper client 并放进 state。
4. 在 handler 中保留 path/query。
5. 改写 URI 到后端。
6. 转发请求并返回响应。

## 本章真正要记住什么

反向代理最小核心是：

```text
接收客户端请求
改写目标 URI
用 HTTP client 请求后端
把后端响应返回客户端
```

这个 example 是学习用最小模型，不是完整生产网关。

## 源码对照

本章手写版对应源码：

- `examples/reverse-proxy/src/main.rs`
- `examples/reverse-proxy/Cargo.toml`
