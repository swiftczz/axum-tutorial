# 23. request-id

对应示例：`examples/request-id`

本章目标：给每个 HTTP 请求加上 request id，并把这个 id 同时放进日志 span 和响应 header 里，理解它为什么能帮助后端排查问题。

第 21 章讲了 `TraceLayer`，第 22 章讲了 middleware 如何包住请求和响应。  
这一章把它们串起来：每次请求进入服务时，先给它一个唯一编号，然后后面的日志都带着这个编号。

## 这个小项目在做什么

应用只有一个接口：

```text
GET / -> <h1>Hello, World!</h1>
```

但它在路由外面加了三层 middleware：

```text
SetRequestIdLayer
TraceLayer
PropagateRequestIdLayer
```

请求主线是：

```text
客户端 GET /
-> SetRequestIdLayer 确保请求里有 x-request-id
-> TraceLayer 创建带 request_id 字段的日志 span
-> handler 打印 Hello world! 日志
-> PropagateRequestIdLayer 把 x-request-id 写进响应 header
-> 客户端收到响应，也能看到同一个 x-request-id
```

## 先理解 request id 解决什么问题

一个后端服务上线后，同一秒可能有很多请求：

```text
用户 A 请求 /
用户 B 请求 /
用户 C 请求 /
```

如果日志只有：

```text
started request
Hello world!
finished request
```

你很难判断哪几行属于同一个用户请求。

request id 的作用就是给每次请求贴一个编号：

```text
request_id=7a2...
  started request
  Hello world!
  finished request
```

这样排查问题时，你可以按一个 `request_id` 搜日志，看到这次请求从进入服务到返回响应的全过程。

## request id 和 trace id 的区别

这章用的是 HTTP request id，它通常放在 header 里：

```text
x-request-id: ...
```

它适合单个服务或简单链路。

大型微服务里还会用 trace id、span id、OpenTelemetry 等体系。  
它们解决的是跨多个服务的调用链追踪。

现在先记住：

```text
request id = 给一次 HTTP 请求一个可搜索的编号
```

这是理解链路追踪前最容易上手的一步。

## 文件和依赖

这个 example 有两个文件：

1. `examples/request-id/Cargo.toml`：声明 Axum、tower、tower-http request-id/trace、tracing。
2. `examples/request-id/src/main.rs`：实现 request id middleware、日志 span 和 HTML handler。

关键依赖：

- `axum`：提供 `Router`、`Request`、`Html`。
- `tower`：提供 `ServiceBuilder`，用来组合多层 middleware。
- `tower-http`：提供 `SetRequestIdLayer`、`PropagateRequestIdLayer`、`TraceLayer`。
- `tracing`：创建 span 和打印日志。
- `tracing-subscriber`：初始化日志系统。
- `tokio`：异步运行时。

`tower-http` 需要启用两个 feature：

````toml
tower-http = { version = "0.6", features = ["request-id", "trace"] }
````

## 第一步：初始化 tracing

源码：

````rust
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
````

这和第 21 章类似。  
没有设置 `RUST_LOG` 时，默认启用：

- 当前 example 的 debug 日志。
- `tower_http` 的 debug 日志。
- Axum rejection 的 trace 日志。

这一章的 handler 里有：

````rust
info!("Hello world!");
````

所以必须先初始化 `tracing_subscriber`，否则 `info!` 不会按预期输出。

## 第二步：定义 request id header 名称

源码：

````rust
const REQUEST_ID_HEADER: &str = "x-request-id";
````

HTTP header 名字用字符串表示。  
这里选择的是常见的 `x-request-id`。

后面还会把它转成 `HeaderName`：

````rust
let x_request_id = HeaderName::from_static(REQUEST_ID_HEADER);
````

为什么不每次都写字符串？

因为 middleware 需要的是强类型的 `HeaderName`。  
先转成变量后，可以在多个 layer 里复用，也避免拼写不一致。

## 第三步：用 ServiceBuilder 组合 middleware

源码：

````rust
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
````

这里有三层：

1. `SetRequestIdLayer`：保证请求 header 里有 `x-request-id`。
2. `TraceLayer`：创建日志 span，并把 `request_id` 写入 span 字段。
3. `PropagateRequestIdLayer`：把请求里的 `x-request-id` 复制到响应 header。

可以把它理解成：

```text
先给请求编号
再让日志读到这个编号
最后把编号返回给客户端
```

顺序很重要。  
`TraceLayer` 要放在能读到 request id 的位置，否则它创建 span 时就拿不到编号。

## 第四步：SetRequestIdLayer 生成编号

源码：

````rust
.layer(SetRequestIdLayer::new(
    x_request_id.clone(),
    MakeRequestUuid,
))
````

`SetRequestIdLayer` 做的事情是：

```text
检查请求 header 里有没有 x-request-id
没有的话，用 MakeRequestUuid 生成一个 UUID
然后把它写进请求 header
```

`MakeRequestUuid` 是 `tower-http` 提供的生成器。  
它适合教学和普通项目入门，因为你不用自己写 UUID 生成逻辑。

请求进入服务前可能没有 request id：

```text
GET /
```

经过这一层后，后续代码看到的是：

```text
GET /
x-request-id: 2f9b...
```

## 第五步：TraceLayer 把 request id 写入日志 span

核心源码：

````rust
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
})
````

`make_span_with` 会在每次请求进入时执行。  
它拿到 `Request`，从 header 里读出 `x-request-id`：

````rust
request.headers().get(REQUEST_ID_HEADER)
````

如果读到了，就创建这样的 span：

````rust
info_span!(
    "http_request",
    request_id = ?request_id,
)
````

`?request_id` 表示用 Debug 格式记录字段。  
这是因为 header value 不是普通字符串类型，直接用 Debug 更方便。

如果没读到 request id，就输出错误日志：

````rust
error!("could not extract request_id");
````

正常情况下，因为前面有 `SetRequestIdLayer`，这里应该能读到。

## 第六步：handler 中的日志会进入同一个请求上下文

源码：

````rust
async fn handler() -> Html<&'static str> {
    info!("Hello world!");
    Html("<h1>Hello, World!</h1>")
}
````

handler 本身没有读取 request id。  
但它的 `info!("Hello world!")` 会发生在 `TraceLayer` 创建的请求 span 里。

这就是 `TraceLayer` 的价值：

```text
handler 只关心业务
middleware 负责给日志补请求上下文
```

真实项目里，你不应该在每个 handler 里手动重复写：

```text
读取 request id
打印 request id
```

把这种通用逻辑放到 middleware 更清晰。

## 第七步：PropagateRequestIdLayer 把编号返回给客户端

源码：

````rust
.layer(PropagateRequestIdLayer::new(x_request_id));
````

`PropagateRequestIdLayer` 会把请求 header 里的 request id 复制到响应 header。

这样客户端可以看到：

```text
HTTP/1.1 200 OK
x-request-id: 2f9b...
```

这很实用。  
如果用户反馈“这个请求报错了”，前端或客户端可以把 response header 里的 `x-request-id` 一起上报，后端就能直接用这个 id 搜日志。

## 第八步：把 middleware 挂到 Router 上

源码：

````rust
let app = Router::new().route("/", get(handler)).layer(middleware);
````

这表示：

```text
整个 Router 都经过 request id middleware
```

只要以后你继续加路由：

````rust
.route("/users", get(list_users))
.route("/orders", get(list_orders))
````

它们都会自动拥有 request id、trace span 和响应 header。

## 函数职责速查

- `main`：初始化日志，创建 request id header，组合 middleware，注册路由并启动服务。
- `handler`：打印一条业务日志，返回 HTML。

这一章的关键逻辑主要不在函数数量，而在 middleware 的组合顺序。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-request-id
//! ```

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

// 统一定义 request id 使用的 header 名字。
// 客户端和服务端都可以通过这个 header 传递同一个请求编号。
const REQUEST_ID_HEADER: &str = "x-request-id";

#[tokio::main]
async fn main() {
    // 初始化 tracing 日志系统。
    // 如果没有设置 RUST_LOG，就使用这里给出的默认日志级别。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                // axum 内置 extractor 的 rejection 日志使用 axum::rejection target。
                // axum::rejection=trace 可以看到更详细的 extractor 失败信息。
                format!(
                    "{}=debug,tower_http=debug,axum::rejection=trace",
                    env!("CARGO_CRATE_NAME")
                )
                .into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 把字符串形式的 header 名字转成 HeaderName。
    // 后面的 request id layer 都需要使用这个类型。
    let x_request_id = HeaderName::from_static(REQUEST_ID_HEADER);

    // 用 ServiceBuilder 组合多层 middleware。
    let middleware = ServiceBuilder::new()
        // 第一层：确保请求里有 x-request-id。
        // 如果请求没有带这个 header，就用 UUID 生成一个。
        .layer(SetRequestIdLayer::new(
            x_request_id.clone(),
            MakeRequestUuid,
        ))
        .layer(
            // 第二层：为 HTTP 请求创建 tracing span。
            // 这个 span 会成为本次请求的日志上下文。
            TraceLayer::new_for_http().make_span_with(|request: &Request<_>| {
                // 从请求 header 里读取 request id。
                let request_id = request.headers().get(REQUEST_ID_HEADER);

                match request_id {
                    // 正常情况下，前面的 SetRequestIdLayer 已经保证这里能读到。
                    Some(request_id) => info_span!(
                        "http_request",
                        request_id = ?request_id,
                    ),
                    // 如果没有读到，说明 middleware 顺序或 header 处理出了问题。
                    None => {
                        error!("could not extract request_id");
                        info_span!("http_request")
                    }
                }
            }),
        )
        // 第三层：把请求里的 x-request-id 复制到响应 header。
        // 这样客户端也能看到本次请求的编号。
        .layer(PropagateRequestIdLayer::new(x_request_id));

    // 注册路由，并把 middleware 挂到整个 Router 上。
    let app = Router::new().route("/", get(handler)).layer(middleware);

    // 绑定本地端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());

    // 启动 Axum 服务。
    axum::serve(listener, app).await.unwrap();
}

// 一个简单的 GET handler。
async fn handler() -> Html<&'static str> {
    // 这条日志会出现在 TraceLayer 创建的请求 span 里。
    info!("Hello world!");

    // 返回 HTML 响应。
    Html("<h1>Hello, World!</h1>")
}
````

## 运行和验证

运行：

````bash
cargo run -p example-request-id
````

请求接口：

````bash
curl -i http://127.0.0.1:3000/
````

重点看响应 header：

````text
x-request-id: ...
````

你也可以自己带一个 request id：

````bash
curl -i -H 'x-request-id: demo-123' http://127.0.0.1:3000/
````

然后观察响应里是否也带有同一个 header。

终端日志里应该能看到类似字段：

````text
request_id=...
Hello world!
````

## 常见卡点

### 1. 为什么客户端也要拿到 request id？

因为线上排障经常从客户端反馈开始。  
客户端拿到 `x-request-id` 后，可以把它和错误信息一起上报，后端就不用在大量日志里盲找。

### 2. 为什么不用 handler 自己生成 request id？

request id 是横切逻辑。  
所有接口都需要它，所以放在 middleware 更合理。

如果每个 handler 都自己生成，很容易出现：

```text
有的接口忘了生成
有的接口 header 名字写错
有的接口日志字段不一致
```

### 3. 为什么 `TraceLayer` 里可能读不到 request id？

通常是 middleware 顺序不对，或者你移除了 `SetRequestIdLayer`。  
这一章的顺序是：

```text
SetRequestIdLayer -> TraceLayer -> PropagateRequestIdLayer -> handler
```

`TraceLayer` 创建 span 前，必须已经能从 request header 里读到 `x-request-id`。

### 4. request id 可以信任客户端传入的吗？

一般可以把它当成排障线索，但不要当成安全凭证。  
客户端传来的 header 可以被伪造，所以不能用 request id 判断用户身份或权限。

### 5. 每次都生成 UUID 会不会很重？

对普通 Web API 来说，这个成本通常很低。  
等你需要做全链路追踪时，可以再引入更完整的 trace id 方案。

## 手写任务

建议按下面顺序自己敲一遍：

1. 先写一个只有 `GET /` 的 Axum 服务。
2. 加上 `tracing_subscriber` 和 `info!("Hello world!")`。
3. 定义 `REQUEST_ID_HEADER` 和 `HeaderName`。
4. 用 `ServiceBuilder` 加 `SetRequestIdLayer`。
5. 再加 `TraceLayer`，从 request header 读取 request id。
6. 最后加 `PropagateRequestIdLayer`，用 `curl -i` 验证响应 header。

加深练习：

1. 用 `curl -H 'x-request-id: my-test-id'` 传入自定义 id。
2. 改变 middleware 顺序，观察日志是否还能读到 request id。
3. 给 `/health` 再加一个 handler，确认它也自动带 request id。

## 本章真正要记住什么

request id 的核心价值不是“生成 UUID”。  
它的核心价值是：

```text
让一次请求的日志、响应和客户端反馈能用同一个编号串起来
```

在 Axum 里，这件事适合用 middleware 完成：

```text
SetRequestIdLayer
-> TraceLayer
-> PropagateRequestIdLayer
```

这也是后端工程化里非常基础的一步。

## 源码对照

本章手写版对应源码：

- `examples/request-id/src/main.rs`
- `examples/request-id/Cargo.toml`
