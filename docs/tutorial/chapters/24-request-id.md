# 24. request-id

对应示例：`examples/request-id`

第 22 章的 tracing span 关联了同一请求的多条日志。这章给每个请求一个**唯一 ID**——日志按 ID 关联、客户端从响应头看到 ID 反馈问题时方便排查。生产环境这是必备功能（特别是分布式系统追踪）。

分 3 步：先用 `SetRequestIdLayer` 生成 ID 并注入请求头，再用 `PropagateRequestIdLayer` 把 ID 回写到响应头，最后把 ID 接到 tracing span 让日志带 ID。

相比前面章节新引入：**`tower-http` 的 `SetRequestIdLayer`、`MakeRequestUuid`（UUID 生成器）、`PropagateRequestIdLayer`、`ServiceBuilder` 组合 layer**。

## Cargo.toml

````toml
[package]
name = "example-request-id"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }
tower = "0.5"
tower-http = { version = "0.6", features = ["trace", "request-id"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

`tower-http` 要启用 `request-id` feature 才能用相关 layer。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：`SetRequestIdLayer` 生成 ID 注入请求头

每个请求来时给它生成一个 UUID，塞进 `x-request-id` 请求头。后续 layer/handler 都能从这个头读到 ID。

````rust
use axum::{
    http::{HeaderName, Request},
    response::Html,
    routing::get,
    Router,
};
use tower::ServiceBuilder;
use tower_http::request_id::{MakeRequestUuid, SetRequestIdLayer};

const REQUEST_ID_HEADER: &str = "x-request-id";

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(/* tracing 初始化 */)
        .with(tracing_subscriber::fmt::layer())
        .init();

    let x_request_id = HeaderName::from_static(REQUEST_ID_HEADER);

    let middleware = ServiceBuilder::new()
        .layer(SetRequestIdLayer::new(
            x_request_id.clone(),
            MakeRequestUuid,
        ));

    let app = Router::new()
        .route("/", get(handler))
        .layer(middleware);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
````

> **新面孔：`SetRequestIdLayer`**
>
> tower-http 提供的 layer，给每个请求生成 ID 并写入指定头。`SetRequestIdLayer::new(header_name, MakeRequestUuid)`：
> - `header_name`：写入的头名（这章用 `x-request-id`）
> - `MakeRequestUuid`：ID 生成器（UUID v4）
>
> 这步之后，每个请求的 headers 里有 `x-request-id: <uuid>`。

> **新面孔：`MakeRequestUuid`**
>
> ID 生成器 trait 的实现之一，生成 UUID。其他生成器：`MakeRequestUlid`（ULID，时间排序）、`MakeRequestInt`（递增整数）。
>
> `MakeRequestUuid` 是最常见的——UUID 全局唯一，不泄露请求数量（不像 Int）。

> **新面孔：`ServiceBuilder`**
>
> tower 的 layer 组合器。`ServiceBuilder::new().layer(A).layer(B)` 按顺序叠加 layer，比直接 `.layer(A).layer(B)` 在 Router 上更类型友好（layer 顺序也容易看出执行顺序）。
>
> 注意 layer 执行顺序：**后加的先执行**（洋葱模型外层）。

---

## 第二步：`PropagateRequestIdLayer` 把 ID 回写响应头

请求头里的 ID 客户端看不到（响应才发给客户端）。这步加 `PropagateRequestIdLayer` 把请求里的 ID 复制到响应头，客户端能看到自己的请求 ID。

````rust
use tower_http::request_id::PropagateRequestIdLayer;

# let middleware = ServiceBuilder::new()
#     .layer(SetRequestIdLayer::new(x_request_id.clone(), MakeRequestUuid))
#     .layer(TraceLayer::new_for_http())  // 中间可以插其他 layer
    .layer(PropagateRequestIdLayer::new(x_request_id));  // 把 ID 写回响应
````

验证：

````bash
cd examples
cargo run -p example-request-id

curl -i http://127.0.0.1:3000/
````

响应头里有：

```text
x-request-id: 550e8400-e29b-41d4-a716-446655440000
```

客户端拿到 ID，反馈问题时附上 ID 让 server 端按 ID 查日志。

> **新面孔：`PropagateRequestIdLayer`**
>
> 从请求头读 ID 复制到响应头。**注意顺序**：必须在 `SetRequestIdLayer` 之后（请求里要先有 ID），通常在 `TraceLayer` 之后（这样 trace span 能记录 ID）。
>
> 没有 propagate，客户端看不到 ID——日志里的 ID 只在 server 端可见。

---

## 第三步：把 ID 接到 tracing span（日志带 ID）

这是 ID 真正发挥威力的地方：让 tracing 的 span 包含 request_id，日志聚合后能按 ID 关联同一请求的所有日志。

````rust
use tower_http::trace::TraceLayer;
use tracing::{error, info, info_span};

# let middleware = ServiceBuilder::new()
#     .layer(SetRequestIdLayer::new(x_request_id.clone(), MakeRequestUuid))
    .layer(
        TraceLayer::new_for_http().make_span_with(|request: &Request<_>| {
            // 从请求头读出刚生成的 ID
            let request_id = request.headers().get(REQUEST_ID_HEADER);
            match request_id {
                Some(request_id) => info_span!(
                    "http_request",
                    request_id = ?request_id,
                ),
                None => {
                    error!("could not extract request_id");
                    info_span!("http_request")
                }
            }
        }),
    )
#     .layer(PropagateRequestIdLayer::new(x_request_id));
````

> **新面孔：tracing span 里嵌入 request_id**
>
> 第 22 章的 `TraceLayer::make_span_with` 创建每请求 span。这里从 `request.headers().get(REQUEST_ID_HEADER)` 读出 ID（上一步 `SetRequestIdLayer` 写进去的），记到 span 字段。
>
> 之后所有 tracing 事件（`info!`、`error!` 等）自动带这个 request_id——日志聚合按 ID 过滤能看完整请求生命周期。

> **新面孔：layer 顺序（洋葱模型）**
>
> ```text
> 请求进 → SetRequestId → TraceLayer → Propagate → handler
>   SetRequestId 先生成 ID
>   TraceLayer 在 ID 已存在时创建 span（能读到 ID）
>   Propagate 在响应回来时复制 ID 到响应头
> handler 返回 ← Propagate ← TraceLayer ← SetRequestId ← 响应出
> ```
>
> 顺序错了（比如 TraceLayer 在 SetRequestId 之前）span 里就读不到 ID。ServiceBuilder 的 layer 顺序要仔细想。

验证（看日志带 request_id）：

````bash
RUST_LOG=example_request_id=debug,tower_http=debug cargo run -p example-request-id
curl http://127.0.0.1:3000/
````

日志：

```text
INFO http_request request_id="550e8400-..." "Hello world!"
```

---

## 完整代码

````rust
use axum::{
    http::{HeaderName, Request},
    response::Html,
    routing::get,
    Router,
};
use tower::ServiceBuilder;
use tower_http::{
    request_id::{MakeRequestUuid, PropagateRequestIdLayer, SetRequestIdLayer},
    trace::TraceLayer,
};
use tracing::{error, info, info_span};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

const REQUEST_ID_HEADER: &str = "x-request-id";

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                // axum logs rejections from built-in extractors with the `axum::rejection`
                // target, at `TRACE` level. `axum::rejection=trace` enables showing those events
                format!(
                    "{}=debug,tower_http=debug,axum::rejection=trace",
                    env!("CARGO_CRATE_NAME")
                )
                .into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let x_request_id = HeaderName::from_static(REQUEST_ID_HEADER);

    let middleware = ServiceBuilder::new()
        .layer(SetRequestIdLayer::new(
            x_request_id.clone(),
            MakeRequestUuid,
        ))
        .layer(
            TraceLayer::new_for_http().make_span_with(|request: &Request<_>| {
                // Log the request id as generated.
                let request_id = request.headers().get(REQUEST_ID_HEADER);

                match request_id {
                    Some(request_id) => info_span!(
                        "http_request",
                        request_id = ?request_id,
                    ),
                    None => {
                        error!("could not extract request_id");
                        info_span!("http_request")
                    }
                }
            }),
        )
        // send headers from request to response headers
        .layer(PropagateRequestIdLayer::new(x_request_id));

    // build our application with a route
    let app = Router::new().route("/", get(handler)).layer(middleware);

    // run it
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}

async fn handler() -> Html<&'static str> {
    info!("Hello world!");
    Html("<h1>Hello, World!</h1>")
}
````

## 运行

````bash
cd examples
RUST_LOG=example_request_id=info cargo run -p example-request-id

curl -i http://127.0.0.1:3000/
````

响应头：

```text
x-request-id: 550e8400-e29b-41d4-a716-446655440000
```

服务端日志：

```text
INFO http_request{request_id="550e8400-..."}: Hello world!
```

## 解读

### request-id 的用途

```text
客户端发请求 → 服务端生成 request_id
                ↓
            日志/span 都带 request_id（server 端排查用）
                ↓
            响应头回写 request_id
                ↓
客户端收到响应 → 反馈问题时附上 request_id
                ↓
            server 端按 request_id 查日志（关联同一请求的所有事件）
```

分布式系统里 request_id 还会在服务间传递（A 服务调 B 服务时把 ID 透传过去），实现端到端追踪——这就是 OpenTelemetry/Datadog 的核心机制。

### layer 顺序（关键）

```text
正确的顺序：
SetRequestId → TraceLayer → PropagateRequestId
   先生成 ID    然后建 span    最后回写响应

错误顺序示例：
TraceLayer → SetRequestId → ...
   span 建时还没有 ID，span 字段为空
```

ServiceBuilder 的 layer 是"后加的先执行"（洋葱外层先 wrap），所以代码顺序是 SetRequestId 在最前面（最先 wrap），但实际执行是请求来时 SetRequestId 最先处理。

## 常见问题

**为什么客户端要看到 request_id？** 客户端报 bug 时附 ID，server 端按 ID 查日志定位问题——比"几点几分的请求"精确得多。

**为什么用 `x-` 前缀？** HTTP 头自定义前缀约定（虽然 RFC 6648 不再推荐，但业界仍普遍用）。

**怎么在 handler 里拿到 request_id？** 用 `HeaderMap` extractor 取 `x-request-id` 头，或定义 `RequestId(String)` extractor 自动提取。

## 手写任务

1. 写 `RequestId(String)` extractor 自动从 header 提取 ID，handler 拿到能记日志。
2. 改用 `MakeRequestUlid`，对比 ULID（时间排序）vs UUID。
3. 加一个 `LogRequestId` middleware，请求进出时都 log ID。
4. 把 request_id 透传到下游 reqwest 调用（用 `SetRequestIdLayer` 的 reqwest 版本）。

## 小结

这章用 3 步讲了 request-id：

1. **生成 + 注入**：`SetRequestIdLayer` + `MakeRequestUuid` 给每个请求生成 UUID 塞到 `x-request-id` 请求头。
2. **回写响应**：`PropagateRequestIdLayer` 把请求里的 ID 复制到响应头，客户端能看到。
3. **接到 tracing span**：`make_span_with` 从请求头读 ID 记到 span，日志带 ID 便于关联。

核心：request-id 是分布式追踪的基础。三个 layer 的顺序很关键（生成 → 建 span → 回写）。

## 源码对照

- `examples/request-id/Cargo.toml`
- `examples/request-id/src/main.rs`
