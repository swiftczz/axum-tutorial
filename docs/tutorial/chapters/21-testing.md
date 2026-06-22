# 21. testing

对应示例：`examples/testing`

测试很重要——后端不是"能跑起来"就结束，接口行为要能被自动验证。这章学 axum HTTP 接口测试的几种写法，分 3 步从简单到复杂。

相比前面章节新引入：`oneshot`（不启动服务器测试）、`ServiceExt`、`BodyExt::collect`、真实服务器测试、`MockConnectInfo`。

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

---

## 第一步：`oneshot`——不启动服务器测试

axum 的 `Router` 实现了 Tower `Service` trait，所以测试可以直接把 `Request` 交给它，不需要监听端口。

先写一个被测的 `app()` 函数，然后写最简单的测试：

````rust
use axum::{body::Body, http::Request, routing::get, Router};

fn app() -> Router {
    Router::new().route("/", get(|| async { "Hello, World!" }))
}

#[tokio::test]
async fn hello_world() {
    let app = app();

    // Router 是 Tower service，直接 oneshot 一个 Request
    let response = app
        .oneshot(Request::get("/").body(Body::empty()).unwrap())
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::OK);

    // 响应 body 是异步的，要先 collect 成 bytes
    let body = response.into_body().collect().await.unwrap().to_bytes();
    assert_eq!(&body[..], b"Hello, World!");
}
````

> **新面孔：`oneshot` + `collect`**
>
> `oneshot` 来自 `tower::ServiceExt`：把一个 Request 交给 Router，拿回 Response。不需要启动 HTTP 服务器，不需要 TCP 连接。
>
> `response.into_body().collect().await` 把异步 body 收集成 bytes。响应 body 是流式的，要断言内容必须先 collect。

把 Router 封装成 `app()` 函数的好处：main 用 `app()` 启动真实服务，测试用 `app()` 直接调用 Router。**测试不一定需要监听端口**。

---

## 第二步：测试 JSON 接口和 404

同样用 `oneshot`，手动构造不同请求测试不同行为：

````rust
#[tokio::test]
async fn json() {
    let app = app();

    // 构造 POST + JSON body
    let response = app
        .oneshot(
            Request::post("/json")
                .header(http::header::CONTENT_TYPE, mime::APPLICATION_JSON.as_ref())
                .body(Body::from(serde_json::to_vec(&json!([1, 2, 3, 4])).unwrap()))
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
    assert!(body.is_empty());  // 404 body 应该是空
}
````

> **新面孔：手动构造请求**
>
> `Request::post("/json").header(...).body(...)` 手动构造 POST 请求——设 content-type header、序列化 JSON body。测试里没有浏览器或 curl，一切手动构造。
>
> 测试应该尽量**验证行为**（状态码 + body 内容），不只验证"没 panic"。

---

## 第三步：真实服务器测试 + `MockConnectInfo`

有些场景（验证真实网络栈、测试 `ConnectInfo`）需要启动真实服务器：

````rust
#[tokio::test]
async fn the_real_deal() {
    // 端口 0 让系统分配可用端口
    let listener = TcpListener::bind("0.0.0.0:0").await.unwrap();
    let addr = listener.local_addr().unwrap();

    tokio::spawn(async move {
        axum::serve(listener, app()).await.unwrap();
    });

    // 用真实 HTTP client 请求
    let client = hyper_util::client::legacy::Client::builder(hyper_util::rt::TokioExecutor::new())
        .build_http();

    let response = client
        .request(Request::get(format!("http://{addr}")).body(Body::empty()).unwrap())
        .await.unwrap();

    let body = response.into_body().collect().await.unwrap().to_bytes();
    assert_eq!(&body[..], b"Hello, World!");
}
````

测试 `ConnectInfo`（客户端地址）时，`oneshot` 没有真实连接信息，用 `MockConnectInfo` 模拟：

````rust
#[tokio::test]
async fn with_mock_connect_info() {
    let mut app = app()
        .layer(MockConnectInfo(SocketAddr::from(([0, 0, 0, 0], 3000))))
        .into_service();

    let request = Request::get("/requires-connect-info").body(Body::empty()).unwrap();
    let response = app.ready().await.unwrap().call(request).await.unwrap();
    assert_eq!(response.status(), StatusCode::OK);
}
````

> **新面孔：`MockConnectInfo` + 端口 0**
>
> `MockConnectInfo(addr)` 在测试里模拟客户端地址——直接调 Router 时没有真实 TCP 连接，用它补这个信息。
>
> 端口 `0` 让操作系统分配可用端口，避免固定端口冲突。

---

## 完整代码

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
            .await.unwrap();
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
                    .body(Body::from(serde_json::to_vec(&json!([1, 2, 3, 4])).unwrap()))
                    .unwrap(),
            )
            .await.unwrap();
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
            .await.unwrap();
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
            .await.unwrap();

        let body = response.into_body().collect().await.unwrap().to_bytes();
        assert_eq!(&body[..], b"Hello, World!");
    }

    #[tokio::test]
    async fn multiple_request() {
        let mut app = app().into_service();

        let request = Request::get("/").body(Body::empty()).unwrap();
        let response = ServiceExt::<Request<Body>>::ready(&mut app).await.unwrap().call(request).await.unwrap();
        assert_eq!(response.status(), StatusCode::OK);

        let request = Request::get("/").body(Body::empty()).unwrap();
        let response = ServiceExt::<Request<Body>>::ready(&mut app).await.unwrap().call(request).await.unwrap();
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

## 运行

````bash
cd examples
cargo test -p example-testing
````

## 手写任务

1. 把 Router 封装成 `app()`。
2. 写 `GET /` 的 `oneshot` 测试。
3. 写 POST JSON 测试。
4. 写 404 测试。
5. 写真实服务器测试，端口用 0。
6. 给需 `ConnectInfo` 的接口加 `MockConnectInfo` 测试。

## 小结

这章分 3 步学了 axum 测试：

1. **`oneshot`**：不启动服务器，Router 直接处理 Request。适合大多数测试。
2. **JSON 和 404**：手动构造不同请求，验证状态码 + body 内容。
3. **真实服务器 + `MockConnectInfo`**：端口 0 避免冲突；`MockConnectInfo` 模拟客户端地址。

核心工具：`oneshot`（单请求）、`collect`（读 body）、`ready`+`call`（多请求复用 service）、`MockConnectInfo`（模拟地址）。

## 源码对照

- `examples/testing/Cargo.toml`
- `examples/testing/src/main.rs`
