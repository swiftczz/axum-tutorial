# 23. request-id

对应示例：`examples/request-id`

第 21 章讲 `TraceLayer`,第 22 章讲 middleware 如何包住请求和响应。这章把它们串起来:给每个 HTTP 请求加一个 request id,同时放进日志 span 和响应 header,方便后端排障。

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
tower = "0.5.2"
tower-http = { version = "0.6", features = ["request-id", "trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

tower-http 启用 `request-id` 和 `trace` feature。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

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
        .layer(PropagateRequestIdLayer::new(x_request_id));

    let app = Router::new().route("/", get(handler)).layer(middleware);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Html<&'static str> {
    info!("Hello world!");
    Html("<h1>Hello, World!</h1>")
}
````

## 运行

````bash
cd examples
cargo run -p example-request-id
````

请求接口:

````bash
curl -i http://127.0.0.1:3000/
````

重点看响应 header:

````text
x-request-id: ...
````

也可以自己带一个:

````bash
curl -i -H 'x-request-id: demo-123' http://127.0.0.1:3000/
````

响应里应带有同一个 header。终端日志:

````text
request_id=...
Hello world!
````

## 解读

### request id 解决什么问题

后端上线后同一秒有很多请求,日志只有 `started request` / `Hello world!` / `finished request` 时,你很难判断哪几行属于同一个用户请求。request id 给每次请求贴一个编号:

```text
request_id=7a2...
  started request
  Hello world!
  finished request
```

排查问题时按一个 `request_id` 搜日志,就能看到这次请求从进入到返回的全过程。

**request id vs trace id**:本章用的是 HTTP request id(放在 `x-request-id` header),适合单个服务或简单链路。大型微服务里还会用 trace id、span id、OpenTelemetry 等体系解决跨服务调用链追踪。先把 request id 理解为"给一次 HTTP 请求一个可搜索的编号"。

### 三层 middleware 顺序很关键

````rust
let middleware = ServiceBuilder::new()
    .layer(SetRequestIdLayer::new(x_request_id.clone(), MakeRequestUuid))   // 1. 确保有 id
    .layer(TraceLayer::new_for_http().make_span_with(...))                  // 2. span 读 id
    .layer(PropagateRequestIdLayer::new(x_request_id));                     // 3. id 写进响应
````

可以理解成:**先给请求编号 → 再让日志读到这个编号 → 最后把编号返回给客户端**。

`TraceLayer` 必须放在能读到 request id 的位置——创建 span 前必须已能从 header 读到 `x-request-id`,所以 `SetRequestIdLayer` 在前。

### `SetRequestIdLayer` 生成 id

```text
检查请求 header 里有没有 x-request-id
没有的话,用 MakeRequestUuid 生成一个 UUID
写进请求 header
```

`MakeRequestUuid` 是 tower-http 提供的生成器,适合教学和普通项目入门。请求进入前可能没 id,经过这层后后续代码看到 `x-request-id: 2f9b...`。

### `TraceLayer` 把 id 写进 span

````rust
.make_span_with(|request: &Request<_>| {
    let request_id = request.headers().get(REQUEST_ID_HEADER);
    match request_id {
        Some(request_id) => info_span!("http_request", request_id = ?request_id),
        None => { error!("could not extract request_id"); info_span!("http_request") }
    }
})
````

从 header 读出 `x-request-id` 写入 span 字段。`?request_id` 用 Debug 格式记录——header value 不是普通字符串,用 Debug 更方便。正常情况下前面有 `SetRequestIdLayer`,这里应该能读到。

### handler 不用关心 request id

````rust
async fn handler() -> Html<&'static str> {
    info!("Hello world!");  // 这条日志发生在 TraceLayer 创建的 span 里
    Html("<h1>Hello, World!</h1>")
}
````

handler 本身不读 request id,但它的 `info!("Hello world!")` 会自动带上 span 的 `request_id` 字段。这就是 TraceLayer 的价值:**handler 只关心业务,middleware 负责给日志补请求上下文**。不要在每个 handler 里手动重复读/打 request id。

### `PropagateRequestIdLayer` 返回给客户端

把请求 header 里的 request id 复制到响应 header。用户反馈"这个请求报错"时,前端可以把响应的 `x-request-id` 一起上报,后端直接用这个 id 搜日志。

### 常见问题

- **为什么客户端也要拿到 request id?** 线上排障常从客户端反馈开始,客户端拿到 id 后和错误一起上报,后端不用盲找日志。
- **为什么不用 handler 自己生成?** request id 是横切逻辑,所有接口都需要,放 middleware 更合理;handler 自己生成容易忘、名字写错、字段不一致。
- **可以信任客户端传入的 id 吗?** 可以当排障线索,但**不能当安全凭证**——header 可被伪造。
- **每次生成 UUID 重不重?** 普通 Web API 这个成本很低;需要全链路追踪时再引入 trace id 方案。

## 手写任务

按下面顺序敲一遍:

1. 写一个只有 `GET /` 的 axum 服务。
2. 加 `tracing_subscriber` 和 `info!("Hello world!")`。
3. 定义 `REQUEST_ID_HEADER` 和 `HeaderName`。
4. 用 `ServiceBuilder` 加 `SetRequestIdLayer`。
5. 再加 `TraceLayer`,从 request header 读 request id。
6. 最后加 `PropagateRequestIdLayer`,用 `curl -i` 验证响应 header。

加深练习:

1. 用 `curl -H 'x-request-id: my-test-id'` 传入自定义 id。
2. 改变 middleware 顺序,观察日志是否还能读到 request id。
3. 给 `/health` 加 handler,确认它也自动带 request id。

## 小结

- request id 的核心价值是让一次请求的日志、响应、客户端反馈能用同一个编号串起来。
- 三层 middleware 顺序:`SetRequestIdLayer` → `TraceLayer` → `PropagateRequestIdLayer`。
- handler 不用关心 request id,middleware 负责给日志补请求上下文。
- 客户端拿到响应里的 `x-request-id` 可上报,后端据此搜日志。
- request id 是排障线索不是安全凭证;跨服务追踪用 OpenTelemetry 等更完整方案。

## 源码对照

- `examples/request-id/Cargo.toml`
- `examples/request-id/src/main.rs`
