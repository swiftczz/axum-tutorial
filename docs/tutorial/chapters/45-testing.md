# 45. testing

对应示例：`examples/testing`

本章目标：学习 Axum HTTP 接口测试的几种常见写法，包括 `oneshot`、真实服务器测试、多请求 `ready/call`、以及 `MockConnectInfo`。

测试章节非常重要。  
后端不是“能跑起来”就结束了，接口行为要能被自动验证。

## 这个小项目在做什么

应用有三个接口：

```text
GET  /                      -> Hello, World!
POST /json                  -> 包装并返回请求 JSON
GET  /requires-connect-info -> 返回客户端地址
```

测试覆盖：

```text
GET / 正常响应
POST /json 正常解析和返回 JSON
不存在路径返回 404
启动真实服务器后用 HTTP client 请求
同一个 service 连续处理多个请求
Mock ConnectInfo
```

## 先理解 app() 的价值

源码：

````rust
fn app() -> Router {
    Router::new()
        .route("/", get(|| async { "Hello, World!" }))
        ...
}
````

把 Router 封装成 `app()` 是测试友好写法。

这样：

```text
main 可以 app() 后启动真实服务
test 可以 app() 后直接调用 Router
```

测试不一定需要监听端口。

## 文件和依赖

这个 example 有两个文件：

1. `examples/testing/Cargo.toml`：声明 Axum、http-body-util、hyper-util、tower。
2. `examples/testing/src/main.rs`：实现 app 和测试。

关键依赖：

- `tower::ServiceExt`：提供 `oneshot`、`ready`。
- `http-body-util::BodyExt`：读取响应 body。
- `hyper-util`：真实 HTTP client 测试。
- `MockConnectInfo`：测试需要 `ConnectInfo` 的 handler。

## 第一步：测试 GET /

源码：

````rust
let response = app
    .oneshot(Request::get("/").body(Body::empty()).unwrap())
    .await
    .unwrap();

assert_eq!(response.status(), StatusCode::OK);

let body = response.into_body().collect().await.unwrap().to_bytes();
assert_eq!(&body[..], b"Hello, World!");
````

`Router` 实现了 Tower `Service`。  
所以测试里可以直接把 `Request` 交给它：

```text
Request -> Router -> Response
```

不需要真的启动 HTTP 服务器。

## 第二步：测试 JSON 接口

源码：

````rust
let response = app
    .oneshot(
        Request::post("/json")
            .header(http::header::CONTENT_TYPE, mime::APPLICATION_JSON.as_ref())
            .body(Body::from(
                serde_json::to_vec(&json!([1, 2, 3, 4])).unwrap(),
            ))
            .unwrap(),
    )
    .await
    .unwrap();
````

这里手动构造 POST 请求：

- 设置 `content-type: application/json`。
- 把 JSON 序列化成 body。

然后读取响应 body，再反序列化：

````rust
let body: Value = serde_json::from_slice(&body).unwrap();
assert_eq!(body, json!({ "data": [1, 2, 3, 4] }));
````

这验证了 extractor 和响应 JSON。

## 第三步：测试 404

源码：

````rust
let response = app
    .oneshot(Request::get("/does-not-exist").body(Body::empty()).unwrap())
    .await
    .unwrap();

assert_eq!(response.status(), StatusCode::NOT_FOUND);
let body = response.into_body().collect().await.unwrap().to_bytes();
assert!(body.is_empty());
````

不仅检查状态码，也检查 body 是空的。  
测试应该尽量验证行为，而不只验证“没有 panic”。

## 第四步：启动真实服务器测试

源码：

````rust
let listener = TcpListener::bind("0.0.0.0:0").await.unwrap();
let addr = listener.local_addr().unwrap();

tokio::spawn(async move {
    axum::serve(listener, app()).await;
});
````

端口 `0` 让系统分配可用端口。

然后用 HTTP client 请求：

````rust
let client =
    hyper_util::client::legacy::Client::builder(hyper_util::rt::TokioExecutor::new())
        .build_http();

let response = client
    .request(
        Request::get(format!("http://{addr}"))
            .header("Host", "localhost")
            .body(Body::empty())
            .unwrap(),
    )
    .await
    .unwrap();
````

这种测试更接近真实网络调用。  
适合验证服务启动、协议层、真实 client 行为。

## 第五步：多次请求用 ready 和 call

源码：

````rust
let mut app = app().into_service();

let response = ServiceExt::<Request<Body>>::ready(&mut app)
    .await
    .unwrap()
    .call(request)
    .await
    .unwrap();
````

`oneshot` 会消耗 service。  
如果想用同一个 service 连续处理多个请求，可以：

```text
into_service
ready
call
```

这更接近 Tower 的底层使用方式。

## 第六步：测试 ConnectInfo

源码 handler：

````rust
get(|ConnectInfo(addr): ConnectInfo<SocketAddr>| async move { format!("Hi {addr}") })
````

正常运行时，`ConnectInfo` 来自：

````rust
into_make_service_with_connect_info
````

但单元测试直接调用 Router 时没有真实连接信息。  
所以测试加：

````rust
let mut app = app()
    .layer(MockConnectInfo(SocketAddr::from(([0, 0, 0, 0], 3000))))
    .into_service();
````

`MockConnectInfo` 在测试里模拟客户端地址。

## 函数职责速查

- `main`：启动真实服务。
- `app`：构造可复用 Router。
- `hello_world`：测试 GET `/`。
- `json`：测试 POST JSON 请求和响应。
- `not_found`：测试未知路径 404。
- `the_real_deal`：启动真实服务器，用 HTTP client 请求。
- `multiple_request`：演示同一个 service 连续处理请求。
- `with_into_make_service_with_connect_info`：演示 `MockConnectInfo`。


## 带中文注释的手写版

完整 `src/main.rs`：

````rust
//! Axum HTTP 测试示例。
//!
//! Run with
//!
//! ```not_rust
//! cargo test -p example-testing
//! ```

use std::net::SocketAddr;

use axum::{
    extract::ConnectInfo,
    routing::{get, post},
    Json, Router,
};
use tower_http::trace::TraceLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app()).await.unwrap();
}

// 把 Router 放进函数，main 和测试都能复用。
fn app() -> Router {
    Router::new()
        .route("/", get(|| async { "Hello, World!" }))
        .route(
            "/json",
            post(|payload: Json<serde_json::Value>| async move {
                Json(serde_json::json!({ "data": payload.0 }))
            }),
        )
        .route(
            "/requires-connect-info",
            get(|ConnectInfo(addr): ConnectInfo<SocketAddr>| async move { format!("Hi {addr}") }),
        )
        .layer(TraceLayer::new_for_http())
}

#[cfg(test)]
mod tests {
    use super::*;
    use axum::{
        body::Body,
        extract::connect_info::MockConnectInfo,
        http::{self, Request, StatusCode},
    };
    use http_body_util::BodyExt;
    use serde_json::{json, Value};
    use tokio::net::TcpListener;
    use tower::{Service, ServiceExt};

    #[tokio::test]
    async fn hello_world() {
        let app = app();

        // Router 是 Tower service，所以可以直接 oneshot 一个 Request。
        let response = app
            .oneshot(Request::get("/").body(Body::empty()).unwrap())
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::OK);

        let body = response.into_body().collect().await.unwrap().to_bytes();
        assert_eq!(&body[..], b"Hello, World!");
    }

    #[tokio::test]
    async fn json() {
        let app = app();

        let response = app
            .oneshot(
                Request::post("/json")
                    .header(http::header::CONTENT_TYPE, mime::APPLICATION_JSON.as_ref())
                    .body(Body::from(
                        serde_json::to_vec(&json!([1, 2, 3, 4])).unwrap(),
                    ))
                    .unwrap(),
            )
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::OK);

        let body = response.into_body().collect().await.unwrap().to_bytes();
        let body: Value = serde_json::from_slice(&body).unwrap();
        assert_eq!(body, json!({ "data": [1, 2, 3, 4] }));
    }

    #[tokio::test]
    async fn not_found() {
        let app = app();

        let response = app
            .oneshot(Request::get("/does-not-exist").body(Body::empty()).unwrap())
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::NOT_FOUND);
        let body = response.into_body().collect().await.unwrap().to_bytes();
        assert!(body.is_empty());
    }

    // 集成测试：启动真实 HTTP server，再用 HTTP client 请求。
    #[tokio::test]
    async fn the_real_deal() {
        let listener = TcpListener::bind("0.0.0.0:0").await.unwrap();
        let addr = listener.local_addr().unwrap();

        tokio::spawn(async move {
            axum::serve(listener, app()).await.unwrap();
        });

        let client =
            hyper_util::client::legacy::Client::builder(hyper_util::rt::TokioExecutor::new())
                .build_http();

        let response = client
            .request(
                Request::get(format!("http://{addr}"))
                    .header("Host", "localhost")
                    .body(Body::empty())
                    .unwrap(),
            )
            .await
            .unwrap();

        let body = response.into_body().collect().await.unwrap().to_bytes();
        assert_eq!(&body[..], b"Hello, World!");
    }

    // 同一个 service 连续处理多个请求时，可以用 ready + call。
    #[tokio::test]
    async fn multiple_request() {
        let mut app = app().into_service();

        let request = Request::get("/").body(Body::empty()).unwrap();
        let response = ServiceExt::<Request<Body>>::ready(&mut app)
            .await
            .unwrap()
            .call(request)
            .await
            .unwrap();
        assert_eq!(response.status(), StatusCode::OK);

        let request = Request::get("/").body(Body::empty()).unwrap();
        let response = ServiceExt::<Request<Body>>::ready(&mut app)
            .await
            .unwrap()
            .call(request)
            .await
            .unwrap();
        assert_eq!(response.status(), StatusCode::OK);
    }

    // 测试需要 ConnectInfo 的接口时，用 MockConnectInfo 注入模拟地址。
    #[tokio::test]
    async fn with_into_make_service_with_connect_info() {
        let mut app = app()
            .layer(MockConnectInfo(SocketAddr::from(([0, 0, 0, 0], 3000))))
            .into_service();

        let request = Request::get("/requires-connect-info")
            .body(Body::empty())
            .unwrap();
        let response = app.ready().await.unwrap().call(request).await.unwrap();
        assert_eq!(response.status(), StatusCode::OK);
    }
}
````

## 运行和验证

运行：

````bash
cargo test -p example-testing
````

如果项目依赖完整，应该看到这些测试通过：

```text
hello_world
json
not_found
the_real_deal
multiple_request
with_into_make_service_with_connect_info
```

## 常见卡点

### 1. oneshot 为什么不用启动服务？

因为 `Router` 本身是 Tower service。  
测试可以直接调用 service，不必走 TCP。

### 2. 为什么要 collect body？

响应 body 是异步 body。  
要断言内容，需要先 collect 成 bytes。

### 3. 什么时候用真实服务器测试？

当你要覆盖真实网络栈、client 行为、服务启动配置时，使用真实服务器测试。

### 4. ConnectInfo 测试为什么需要 MockConnectInfo？

直接调用 Router 没有真实 socket 地址。  
`MockConnectInfo` 用来在测试里补这个信息。

## 手写任务

1. 把 Router 封装成 `app()`。
2. 写 `GET /` 的 `oneshot` 测试。
3. 写 POST JSON 测试。
4. 写 404 测试。
5. 写一个真实服务器测试，端口用 0。
6. 给需要 `ConnectInfo` 的接口加 `MockConnectInfo` 测试。

## 本章真正要记住什么

Axum 测试的基础模型是：

```text
Router 是 Service
Request 进
Response 出
不一定要启动服务器
```

常用工具：

```text
oneshot
BodyExt::collect
ready + call
MockConnectInfo
真实服务器 + HTTP client
```

## 源码对照

本章手写版对应源码：

- `examples/testing/src/main.rs`
- `examples/testing/Cargo.toml`
