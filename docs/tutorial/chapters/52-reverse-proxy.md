# 52. reverse-proxy

对应示例：`examples/reverse-proxy`

理解反向代理的基本请求转发:客户端访问代理服务,代理把请求转发到后端,响应返回给客户端。上一章是正向代理(客户端显式配置),这章是反向代理(客户端以为在访问服务,代理在服务端内部转发)。



相比前面章节新引入：**`hyper_util::client::legacy::Client` + `State`、URI 改写、`X-Forwarded-*`**。

## Cargo.toml

````toml
[package]
name = "example-reverse-proxy"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
hyper = { version = "1.0.0", features = ["full"] }
hyper-util = { version = "0.1.1", features = ["client-legacy"] }
tokio = { version = "1", features = ["full"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 关键概念

> **新面孔：`hyper_util::client::legacy::Client`**
>
> 反向代理主动请求后端。`State<Client>` 共享 client。改写 URI 后 `client.request(req).await` 转发。


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

    let listener = tokio::net::TcpListener::bind("127.0.0.1:4000").await.unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}

async fn handler(State(client): State<Client>, mut req: Request) -> Result<Response, StatusCode> {
    let path = req.uri().path();
    let path_query = req.uri().path_and_query().map(|v| v.as_str()).unwrap_or(path);

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

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}
````

## 运行

````bash
cd examples
cargo run -p example-reverse-proxy
````

请求代理:

````bash
curl http://127.0.0.1:4000/
# Hello, world!(后端 3000 返回的)
````

## 解读

### 正向代理 vs 反向代理

```text
正向代理:客户端知道代理存在,代理帮客户端访问外部网站(第 51 章)
反向代理:客户端以为在访问服务,代理在服务端内部转给后端
```

Nginx/Envoy/网关/负载均衡器常扮演反向代理。

### 启动两个服务

```text
真实后端 127.0.0.1:3000
反向代理 127.0.0.1:4000
```

````rust
tokio::spawn(server());   // 后端后台运行
// 主任务启动代理服务 4000
````

### Hyper client 放 state

````rust
type Client = hyper_util::client::legacy::Client<HttpConnector, Body>;

let client: Client = hyper_util::client::legacy::Client::<(), ()>::builder(TokioExecutor::new())
    .build(HttpConnector::new());

let app = Router::new().route("/", get(handler)).with_state(client);
````

反向代理需主动请求后端,所以需 HTTP client。client 可复用(同第 15 章 reqwest Client),放进 state 后每个 handler 用同一个。

### 改写请求 URI(本章核心)

````rust
async fn handler(State(client): State<Client>, mut req: Request) -> Result<Response, StatusCode> {
    let path = req.uri().path();
    let path_query = req.uri().path_and_query().map(|v| v.as_str()).unwrap_or(path);

    let uri = format!("http://127.0.0.1:3000{path_query}");
    *req.uri_mut() = Uri::try_from(uri).unwrap();

    Ok(client.request(req).await.map_err(|_| StatusCode::BAD_REQUEST)?.into_response())
}
````

保留原请求 path 和 query(`/users?page=1`),改写 URI 到后端(`http://127.0.0.1:3000/users?page=1`),用 client 转发,响应直接转 axum response 返回。

### ⚠️ 本示例没改 Host header,生产代理必须改

示例**只改了 URI,没动 `Host` header**——后端收到的 `Host` 仍是 `127.0.0.1:4000`(代理地址)而非 `127.0.0.1:3000`。教学没问题,但生产会导致:

- 后端基于 Host 的路由(虚拟主机)失效
- 后端生成的绝对 URL(重定向/静态资源链接)错误
- 后端日志记录的访问来源不对

生产反向代理通常做两件事:

````rust
// 1. 改写 Host 成后端地址
req.headers_mut().insert(hyper::header::HOST, "127.0.0.1:3000".parse().unwrap());

// 2. 注入 X-Forwarded-* 让后端知道真实客户端
req.headers_mut().insert("x-forwarded-for", client_ip.parse().unwrap());      // 真实客户端 IP
req.headers_mut().insert("x-forwarded-proto", "https".parse().unwrap());      // 原始协议
````

`X-Forwarded-For`/`X-Forwarded-Proto`/`X-Forwarded-Host` 是反向代理的"增值"功能——让后端还原**经过代理前的真实请求信息**。Nginx/HAProxy 等生产代理都自动注入。本示例跳过了这些细节。

反向代理还必须处理 **hop-by-hop headers**(见第 51 章):`Connection`/`Transfer-Encoding`/`Keep-Alive` 等只在相邻两跳有效,不能原样透传。`hyper-util` client 在本示例自动处理了一部分,手写别的转发逻辑要按 RFC 7230 §6.1 剥离。

## 常见问题

**为什么改 URI?** HTTP client 需知道真正请求哪个后端地址,代理收到的 URI 是给代理自己的,要改成后端 URI。

**能用于生产吗?** 不能直接用。真实反向代理还处理 headers/负载均衡/超时/重试/body 大小/TLS/WebSocket。

**client 为什么放 state?** HTTP client 可复用,放进 state 后每个 handler 用同一个。

## 手写任务

1. 写后端服务监听 3000。
2. 写代理服务监听 4000。
3. 创建 Hyper client 放进 state。
4. handler 保留 path/query。
5. 改写 URI 到后端。
6. 转发请求返回响应。

## 小结

- 反向代理最小核心:接收客户端请求 → 改写目标 URI(保留 path/query)→ 用 HTTP client 请求后端 → 把响应返回客户端。
- 正向代理(客户端知道)vs 反向代理(客户端以为在访问服务);Nginx/网关/负载均衡常扮反向代理。
- HTTP client 放 state 复用(同 reqwest Client 思路)。
- **生产反向代理必须额外做**:改 Host header + 注入 `X-Forwarded-*`(For/Proto/Host)让后端还原真实请求信息 + 处理 hop-by-hop headers。
- 本示例是学习用最小模型,不是完整生产网关。

## 源码对照

- `examples/reverse-proxy/Cargo.toml`
- `examples/reverse-proxy/src/main.rs`
