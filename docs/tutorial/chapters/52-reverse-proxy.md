# 52. reverse-proxy

对应示例：`examples/reverse-proxy`

反向代理：客户端 → 代理 server → 后端 server。第 15 章用 reqwest 转发流式响应，这章用 `hyper-util` 的 client 做更底层的代理——**保留原始请求/响应**（method、uri、headers、body 原样转发），适合 API gateway、负载均衡、流量过滤。

分 2 步：先建 hyper-util `Client` 作为 State，再写 handler 把请求 URI 改写后转发。

相比前面章节新引入：**`hyper_util::client::legacy::Client`（hyper 原生 client）、`client.request(req)` 原样转发 Request、`req.uri_mut()` 改写目标**。

## Cargo.toml

````toml
[package]
name = "example-reverse-proxy"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
hyper = "1.0.0"
hyper-util = { version = "0.1", features = ["client", "client-legacy", "http1", "tokio"] }
tokio = { version = "1.0", features = ["full"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：建 hyper-util Client + 启动后端 demo

代理要转发到后端，所以要先有一个后端 server。这步同时启两个 server：监听 4000 的代理 + 监听 3000 的后端 demo。

````rust
use axum::{
    body::Body,
    routing::get,
    Router,
};
use hyper_util::{client::legacy::connect::HttpConnector, rt::TokioExecutor};

type Client = hyper_util::client::legacy::Client<HttpConnector, Body>;

#[tokio::main]
async fn main() {
    // 后台启后端 demo（端口 3000）
    tokio::spawn(server());

    // 建 hyper-util client（用 tokio executor + HttpConnector）
    let client: Client =
        hyper_util::client::legacy::Client::<(), ()>::builder(TokioExecutor::new())
            .build(HttpConnector::new());

    let app = Router::new().route("/", get(handler)).with_state(client);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:4000").await.unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

// 后端 demo（端口 3000）
async fn server() {
    let app = Router::new().route("/", get(|| async { "Hello, world!" }));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

# async fn handler(State(client): State<Client>, mut req: Request) -> Result<Response, StatusCode> {
#     // 下一步填
# }
````

> **新面孔：`hyper_util::client::legacy::Client`**
>
> hyper-util 提供的 HTTP client。比 reqwest 更底层（reqwest 内部也是基于 hyper），保留更多原始控制。
>
> - `Client::<(), ()>::builder(TokioExecutor::new())`：用 builder 模式创建（`<(), ()>` 是占位泛型参数，builder 推断具体类型）
> - `.build(HttpConnector::new())`：用 HTTP connector（明文 HTTP；HTTPS 要 `HttpsConnector`）
>
> Client 内部有连接池，**必须复用**（不要每次 handler 新建）——所以塞进 State。

> **新面孔：`HttpConnector`**
>
> hyper 的 plain HTTP connector——知道怎么建立 TCP 连接发 HTTP 请求。HTTPS 要 `hyper_rustls::HttpsConnector`（参考 ch15 reqwest 用 rustls）。

> **新面孔：`type Client = Client<HttpConnector, Body>`**
>
> 类型别名简化冗长的泛型。`Client<Connector, Body>` 是 hyper client 的完整类型——connector 决定怎么建连接，Body 决定请求 body 类型。

---

## 第二步：handler 转发请求

handler 拿到 client + 原始 request，改写 URI 后用 `client.request(req)` 原样转发——method/headers/body 全保留。

````rust
use axum::{
    extract::{Request, State},
    http::uri::Uri,
    response::{IntoResponse, Response},
};
use hyper::StatusCode;

async fn handler(State(client): State<Client>, mut req: Request) -> Result<Response, StatusCode> {
    // 1. 提取原 path + query
    let path = req.uri().path();
    let path_query = req
        .uri()
        .path_and_query()
        .map(|v| v.as_str())
        .unwrap_or(path);

    // 2. 改写 URI 到后端地址
    let uri = format!("http://127.0.0.1:3000{path_query}");
    *req.uri_mut() = Uri::try_from(uri).unwrap();

    // 3. 原样转发请求（method/headers/body 保留）
    Ok(client
        .request(req)
        .await
        .map_err(|_| StatusCode::BAD_REQUEST)?
        .into_response())
}
````

验证：

````bash
cd examples
cargo run -p example-reverse-proxy

curl http://127.0.0.1:4000/some/path?foo=bar
# Hello, world!
````

代理监听 4000，把请求转发到 3000（后端返回 "Hello, world!"）。

> **新面孔：`client.request(req)`**
>
> hyper client 的核心方法。接收完整 `Request`（method + uri + headers + body），原样发到目标 server。返回目标 server 的 `Response`。
>
> 比 reqwest 的 `client.get(url).send()` 更底层——reqwest 是高层封装（builder API），hyper client 是底层 API（直接传 Request）。

> **新面孔：`req.uri_mut()` 改写目标**
>
> 反向代理要改写请求 URI 让它指向后端，而不是原始客户端请求的地址。
>
> ```text
> 客户端请求：GET http://proxy:4000/foo?bar=1
> 改写后：   GET http://backend:3000/foo?bar=1
> ```
>
> `path_query` 保留原 path + query（如 `/foo?bar=1`），只换 host:port。method 和 headers 原样保留。

> **新面孔：保留原始 body**
>
> `client.request(req)` 把 `req` 整个转发（包括 body），不需要重新构造。`mut req: Request` 是因为要改 uri，body 部分原样。

### 和 ch15 reqwest 转发的对比

```text
ch15 reqwest 转发：
    client.get(url).send().await          → 自定义请求（要手动复制 headers/body）
    response.bytes_stream()                → 手动流式转发
    Response::builder().headers(...)       → 手动复制响应 headers

这章 hyper client 转发：
    client.request(req).await              → 原样转发（req 是完整 Request）
    自动保留所有 headers/body              → 简单
```

reqwest 适合"调下游 API"（如 OAuth 拿用户信息），hyper client 适合"反向代理"（原样转发）。

---

## 完整代码

````rust
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

type Client = hyper_util::client::legacy::Client<HttpConnector, Body>;

#[tokio::main]
async fn main() {
    tokio::spawn(server());

    let client: Client =
        hyper_util::client::legacy::Client::<(), ()>::builder(TokioExecutor::new())
            .build(HttpConnector::new());

    let app = Router::new().route("/", get(handler)).with_state(client);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:4000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}

async fn handler(State(client): State<Client>, mut req: Request) -> Result<Response, StatusCode> {
    let path = req.uri().path();
    let path_query = req
        .uri()
        .path_and_query()
        .map(|v| v.as_str())
        .unwrap_or(path);

    let uri = format!("http://127.0.0.1:3000{path_query}");

    *req.uri_mut() = Uri::try_from(uri).unwrap();

    Ok(client
        .request(req)
        .await
        .map_err(|_| StatusCode::BAD_REQUEST)?
        .into_response())
}

async fn server() {
    let app = Router::new().route("/", get(|| async { "Hello, world!" }));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}
````

## 运行

````bash
cd examples
cargo run -p example-reverse-proxy
````

测试代理（端口 4000）：

````bash
curl http://127.0.0.1:4000/
# Hello, world!（来自后端 3000）
````

## 解读

### 反向代理的常见用途

```text
客户端 ──> 代理 server ──> 后端 server (可能是多个)
              ↓
          - 负载均衡（按算法选后端）
          - SSL 终止（HTTPS → HTTP）
          - 缓存
          - 限流/熔断
          - 鉴权
          - 路由（按 path 转发到不同后端）
```

Nginx/HAProxy/Envoy/Caddy 都是反向代理。这章用 axum 实现一个最简版，可以扩展加任意业务逻辑。

### `path_query` 为什么要保留

```text
客户端：    GET /users/123?fields=name,email
代理收到：  GET /users/123?fields=name,email
代理改写：  GET http://backend/users/123?fields=name,email
            （path 和 query 保留，只换 host）
```

不保留 query 的话后端收不到参数。`path_and_query().map(|v| v.as_str()).unwrap_or(path)` 处理两种情况：有 query 时取 `path+query`，无 query fallback 到纯 path。

## 常见问题

**为什么用 hyper client 不用 reqwest？** reqwest 是高层 API，构造请求要 builder。这章要"原样转发 Request"，hyper client 的 `client.request(req)` 直接接收 `Request` 更适合。

**怎么转发所有路径（不只是 `/`）？** 把 handler 作为 fallback：`.fallback(handler)` 或 `.route("/{*path}", any(handler))`。这章只演示 `/`，生产用 fallback 接管所有路径。

**HTTPS 后端怎么办？** 用 `hyper_rustls::HttpsConnectorBuilder` 替代 `HttpConnector::new()`。

**怎么加多个后端做负载均衡？** 维护 `Vec<Uri>`（后端列表），每次请求按算法（轮询/随机/最少连接）选一个后端。

## 手写任务

1. 把 handler 改成 fallback（`.fallback(handler)`），转发所有路径。
2. 加负载均衡：维护 `Vec<String>` 后端列表，handler 每次轮询选一个。
3. 加重试：后端返回 5xx 时换一个后端重试。
4. 加日志：每个请求记录 client IP、目标后端、响应状态。

## 小结

这章用 2 步讲了反向代理：

1. **hyper-util Client**：建 `Client<HttpConnector, Body>` 作为 State，连接池复用。
2. **handler 转发**：改写 `req.uri_mut()` 指向后端，`client.request(req)` 原样转发（method/headers/body 保留）。

核心：反向代理用 `client.request(req)` 原样转发比 reqwest builder 简单（保留所有元数据）。这套模式可扩展成负载均衡、API gateway、流量过滤。

## 源码对照

- `examples/reverse-proxy/Cargo.toml`
- `examples/reverse-proxy/src/main.rs`
