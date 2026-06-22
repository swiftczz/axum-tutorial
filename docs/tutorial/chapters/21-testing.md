# 21. testing

对应示例：`examples/testing`

测试章节很重要——后端不是"能跑起来"就结束,接口行为要能被自动验证。本章学 axum HTTP 接口测试的几种常见写法:`oneshot`、真实服务器测试、多请求 `ready/call`、`MockConnectInfo`。

## Cargo.toml

````toml
[package]
name = "example-testing"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
http-body-util = "0.1.0"
hyper-util = { version = "0.1", features = ["client", "http1", "client-legacy"] }
mime = "0.3"
serde_json = "1.0"
tokio = { version = "1.0", features = ["full"] }
tower-http = { version = "0.6.1", features = ["trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

[dev-dependencies]
tower = { version = "0.5.2", features = ["util"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

````rust
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

    #[tokio::test]
    async fn multiple_request() {
        let mut app = app().into_service();

        let request = Request::get("/").body(Body::empty()).unwrap();
        let response = ServiceExt::<Request<Body>>::ready(&mut app)
            .await.unwrap().call(request).await.unwrap();
        assert_eq!(response.status(), StatusCode::OK);

        let request = Request::get("/").body(Body::empty()).unwrap();
        let response = ServiceExt::<Request<Body>>::ready(&mut app)
            .await.unwrap().call(request).await.unwrap();
        assert_eq!(response.status(), StatusCode::OK);
    }

    #[tokio::test]
    async fn with_into_make_service_with_connect_info() {
        let mut app = app()
            .layer(MockConnectInfo(SocketAddr::from(([0, 0, 0, 0], 3000))))
            .into_service();

        let request = Request::get("/requires-connect-info").body(Body::empty()).unwrap();
        let response = app.ready().await.unwrap().call(request).await.unwrap();
        assert_eq!(response.status(), StatusCode::OK);
    }
}
````

## 运行测试

````bash
cd examples
cargo test -p example-testing
````

测试:`hello_world`、`json`、`not_found`、`the_real_deal`、`multiple_request`、`with_into_make_service_with_connect_info` 都应通过。

## 解读

### `app()` 的价值(测试友好写法)

````rust
fn app() -> Router {
    Router::new().route("/", get(...))...
}
````

把 Router 封装成 `app()`,main 和测试都能复用:main 用 `app()` 启动真实服务,测试用 `app()` 直接调用 Router。**测试不一定需要监听端口**。

### `oneshot`:不启动服务器测试(基础模型)

````rust
let response = app
    .oneshot(Request::get("/").body(Body::empty()).unwrap())
    .await.unwrap();
````

`Router` 实现了 Tower `Service`(见第 29/31 章),测试直接把 `Request` 交给它:`Request → Router → Response`,不需要真实 HTTP 服务器。

### 读 body 要 `collect`

````rust
let body = response.into_body().collect().await.unwrap().to_bytes();
assert_eq!(&body[..], b"Hello, World!");
````

响应 body 是异步 body,要断言内容需先 `BodyExt::collect` 成 bytes。

### 测试 JSON 接口

````rust
let response = app.oneshot(
    Request::post("/json")
        .header(http::header::CONTENT_TYPE, mime::APPLICATION_JSON.as_ref())
        .body(Body::from(serde_json::to_vec(&json!([1, 2, 3, 4])).unwrap()))
        .unwrap()
).await.unwrap();

let body: Value = serde_json::from_slice(&body).unwrap();
assert_eq!(body, json!({ "data": [1, 2, 3, 4] }));
````

手动构造 POST:设 `content-type: application/json` + JSON 序列化 body;响应反序列化断言。验证 extractor 和响应 JSON。

### 测试 404

````rust
let response = app.oneshot(Request::get("/does-not-exist").body(Body::empty()).unwrap()).await.unwrap();
assert_eq!(response.status(), StatusCode::NOT_FOUND);
assert!(body.is_empty());
````

不只检查状态码,也检查 body 空——**测试应尽量验证行为,不只验证"没 panic"**。

### 启动真实服务器测试

````rust
let listener = TcpListener::bind("0.0.0.0:0").await.unwrap();   // 端口 0 系统分配
let addr = listener.local_addr().unwrap();
tokio::spawn(async move { axum::serve(listener, app()).await.unwrap(); });

let client = hyper_util::client::legacy::Client::builder(...).build_http();
let response = client.request(Request::get(format!("http://{addr}"))...).await.unwrap();
````

更接近真实网络调用,适合验证服务启动、协议层、真实 client 行为。端口 0 避免冲突。

### 多请求用 `ready` + `call`

````rust
let mut app = app().into_service();
let response = ServiceExt::<Request<Body>>::ready(&mut app).await.unwrap().call(request).await.unwrap();
````

`oneshot` 会消耗 service。想用同一个 service 连续处理多个请求用 `into_service` + `ready` + `call`,更接近 Tower 底层使用方式。

### `MockConnectInfo`

````rust
let mut app = app()
    .layer(MockConnectInfo(SocketAddr::from(([0, 0, 0, 0], 3000))))
    .into_service();
````

handler 里用 `ConnectInfo(addr)` 提取客户端地址(第 41 章),正常运行时来自 `into_make_service_with_connect_info`。单元测试直接调 Router 没真实连接信息,用 `MockConnectInfo` 模拟客户端地址。

## 常见问题

**`oneshot` 为什么不用启动服务?** `Router` 本身是 Tower service,测试直接调 service 不必走 TCP。

**为什么要 collect body?** 响应 body 是异步 body,断言内容要先 collect 成 bytes。

**什么时候用真实服务器测试?** 要覆盖真实网络栈、client 行为、服务启动配置时。

**ConnectInfo 测试为什么需 MockConnectInfo?** 直接调 Router 没真实 socket 地址,MockConnectInfo 补这个信息。

## 手写任务

1. Router 封装成 `app()`。
2. 写 `GET /` 的 `oneshot` 测试。
3. 写 POST JSON 测试。
4. 写 404 测试。
5. 写真实服务器测试,端口用 0。
6. 给需 `ConnectInfo` 的接口加 `MockConnectInfo` 测试。

## 小结

- axum 测试基础模型:Router 是 Service,Request 进 Response 出,不一定要启动服务器。
- 常用工具:`oneshot`(单请求,不启服务)、`BodyExt::collect`(读 body)、`ready`+`call`(多请求复用 service)、`MockConnectInfo`(模拟客户端地址)、真实服务器 + HTTP client(覆盖网络栈)。
- `app()` 封装让 main 和测试复用 Router;测试尽量验证行为不只验证"没 panic"。
- 端口 0 让系统分配避免冲突;`ConnectInfo` 测试用 `MockConnectInfo` 注入模拟地址。

## 源码对照

- `examples/testing/Cargo.toml`
- `examples/testing/src/main.rs`
