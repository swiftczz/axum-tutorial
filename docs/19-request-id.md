# 第 19 章 request-id:给每个请求一个身份

## 本章你将做出什么

一个服务,每个请求都会拿到一个唯一 ID。这个 ID 出现在三个地方:

- **请求头**:中间件生成后写进 `x-request-id`
- **日志 span**:该请求产生的每一条日志都带上它
- **响应头**:透传回客户端

```
INFO http_request{request_id="0198e2a1-..."}: Hello world!
```

客户端报错时报上这个 ID,你就能在日志里精确捞出那一次请求的全部记录。

学完本章你将掌握:

- 请求 ID 解决什么问题,以及为什么它是分布式系统的基础设施
- `SetRequestIdLayer` / `PropagateRequestIdLayer` 的分工
- 为什么这里**必须**用 `ServiceBuilder` 而不是链式 `.layer()`
- 透传上游 ID 与生成新 ID 的取舍

## 前置知识

### 一次操作,散落各处的日志

[第 17 章](17-tracing-aka-logging.md#tracingspan-与-event)讲过 span 能让日志自动带上请求上下文。但那个上下文只有方法和路径:

```
DEBUG http_request{method=GET matched_path="/users/{id}"}: fetching user
DEBUG http_request{method=GET matched_path="/users/{id}"}: cache miss
DEBUG http_request{method=GET matched_path="/users/{id}"}: querying database
```

高并发下这毫无用处——同一秒可能有几百个请求命中同一个路由,它们的日志交错在一起,**你没法判断哪几行属于同一次请求**。

更麻烦的是跨服务。一次用户下单可能经过网关 → 订单服务 → 库存服务 → 支付服务,日志分散在四个地方。用户说"我下单失败了",你得在四份日志里各自搜索、再靠时间戳猜测对应关系。

**请求 ID 就是解决这个问题的**:给每个请求分配一个唯一标识,让它贯穿整条链路。

- 日志按它过滤,一次请求的记录立刻聚拢
- 服务之间通过 HTTP 头部传递它,跨服务的日志能串起来
- 返回给客户端,用户报错时报上 ID,你能精确定位

这在分布式系统里叫**关联 ID**(correlation ID)或**追踪 ID**(trace ID)。完整的分布式追踪(OpenTelemetry、Jaeger)还会记录调用关系和耗时,但**请求 ID 是最低成本、收益最大的第一步**。

### x-request-id 这个头部

事实标准是 `x-request-id` 头部。它不在任何 RFC 里,但几乎所有网关、负载均衡器、云服务商都支持它:

- Nginx、Envoy、AWS ALB 会自动生成并注入
- 大多数 APM 和日志系统认识它

`X-` 前缀历史上表示"非标准头部",[RFC 6648](https://www.rfc-editor.org/rfc/rfc6648) 已经不建议再用了([第 07 章](07-handle-head-request.md)提过),但 `x-request-id` 因为用得太广泛,已经积重难返。**新项目沿用它比另起炉灶更实际**——上下游都认。

### 两个 Layer 的分工

`tower-http` 的 `request_id` 模块提供两个中间件,职责严格分开:

**`SetRequestIdLayer`** — 在**请求**进来时设置 ID。

```rust
SetRequestIdLayer::new(x_request_id.clone(), MakeRequestUuid)
```

两个参数:头部名、ID 生成器。`MakeRequestUuid` 是内置的生成器,产出 UUID v4。

**关键行为:如果请求已经带了这个头部,它不会覆盖。** 这个设计很重要——网关或上游服务已经生成过 ID 时,应该沿用而不是另起一个,否则链路就断了。

**`PropagateRequestIdLayer`** — 在**响应**返回时,把请求头里的 ID 复制到响应头。

```rust
PropagateRequestIdLayer::new(x_request_id)
```

只做这一件事。分开成两个 layer 而不是合并,是因为它们的作用方向不同(一个改请求、一个改响应),而且不总是要同时用——比如内部服务可能只需要设置不需要透传。

### 为什么必须用 ServiceBuilder

这是本章最值得注意的地方。三个中间件有**严格的顺序要求**:

1. `SetRequestIdLayer` 先跑,把 ID 写进请求头
2. `TraceLayer` 再跑,从请求头读出 ID 建 span
3. `PropagateRequestIdLayer` 在响应路上把 ID 复制到响应头

顺序错了就不工作:如果 `TraceLayer` 在 `SetRequestIdLayer` 之前,读请求头时 ID 还没生成,日志里就是 `could not extract request_id`。

[第 17 章](17-tracing-aka-logging.md#洋葱结构与包装顺序)讲过两种写法的顺序是**相反**的:

- `Router::layer` — 后调用的在外层,所以要**倒着写**
- `ServiceBuilder::layer` — 按书写顺序从外到内,**顺着写**

用 `Router::layer` 表达上面的顺序得写成:

```rust
let app = Router::new()
    .route("/", get(handler))
    .layer(PropagateRequestIdLayer::new(x_request_id.clone()))   // 最内
    .layer(TraceLayer::new_for_http() /* ... */)
    .layer(SetRequestIdLayer::new(x_request_id, MakeRequestUuid));  // 最外
```

**倒着读才是执行顺序**,很容易写错。而 `ServiceBuilder` 写出来和执行顺序一致:

```rust
let middleware = ServiceBuilder::new()
    .layer(SetRequestIdLayer::new(x_request_id.clone(), MakeRequestUuid))  // 先执行
    .layer(TraceLayer::new_for_http() /* ... */)
    .layer(PropagateRequestIdLayer::new(x_request_id));                    // 后执行

let app = Router::new().route("/", get(handler)).layer(middleware);
```

**顺序敏感的中间件组合,一律用 `ServiceBuilder`。** 这是官方文档明确推荐的做法,本章示例就是范例。

## 项目搭建

```bash
cargo new request-id
cd request-id
```

```bash
cargo add axum
cargo add tokio --features full
cargo add tower
cargo add tower-http --features request-id,trace
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

这一章第一次直接依赖 `tower` 本身(前面都是通过 axum 间接用),因为要用 `ServiceBuilder`。

`tower-http` 的 `request-id` feature 提供本章那两个 layer。

## 分步构建

### 步骤 1:先看没有 ID 时的样子

**要做什么**:挂一个只有 `TraceLayer` 的服务,确认日志无法区分请求。

```rust
// src/main.rs
use axum::{http::Request, response::Html, routing::get, Router};
use tower_http::trace::TraceLayer;
use tracing::{info, info_span};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

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

    let app = Router::new().route("/", get(handler)).layer(
        TraceLayer::new_for_http()
            .make_span_with(|_request: &Request<_>| info_span!("http_request")),
    );

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
```

**`handler` 里的 `info!("Hello world!")`** 是本章的观察点——它模拟业务代码里的日志。[第 17 章](17-tracing-aka-logging.md#tracingspan-与-event)讲过,这条 event 会自动继承所在 span 的字段。

**跑起来看**:

```bash
cargo run
```

并发发几个请求:

```bash
for i in 1 2 3; do curl -s http://127.0.0.1:3000/ > /dev/null & done; wait
```

```
DEBUG http_request: tower_http::trace::on_request: started processing request
DEBUG http_request: tower_http::trace::on_request: started processing request
INFO http_request: request_id: Hello world!
DEBUG http_request: tower_http::trace::on_request: started processing request
INFO http_request: request_id: Hello world!
DEBUG http_request: tower_http::trace::on_response: finished processing request latency=0 ms status=200
INFO http_request: request_id: Hello world!
DEBUG http_request: tower_http::trace::on_response: finished processing request latency=0 ms status=200
DEBUG http_request: tower_http::trace::on_response: finished processing request latency=0 ms status=200
```

**九条日志交错在一起,每条的 span 都叫 `http_request`,完全分不出谁是谁。** 这就是前置知识说的问题。

### 步骤 2:生成 ID 并写进 span

**要做什么**:加上 `SetRequestIdLayer`,让 span 带上请求 ID。

```rust
// src/main.rs
use axum::{
    http::{HeaderName, Request},                    // 改动:加上 HeaderName
    response::Html,
    routing::get,
    Router,
};
use tower::ServiceBuilder;                          // 新增
use tower_http::{
    request_id::{MakeRequestUuid, SetRequestIdLayer},   // 新增
    trace::TraceLayer,
};
use tracing::{error, info, info_span};               // 改动:加上 error
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 新增:头部名只写一次,避免拼写不一致
const REQUEST_ID_HEADER: &str = "x-request-id";

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    // 新增
    let x_request_id = HeaderName::from_static(REQUEST_ID_HEADER);

    // 改动:用 ServiceBuilder 组合两个中间件
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
        );

    // 改动:挂上组合好的中间件
    let app = Router::new().route("/", get(handler)).layer(middleware);

    // ...(listener 和 serve 不变)
}

// ...(handler 不变)
```

几个要点。

**`const REQUEST_ID_HEADER`** 把头部名抽成常量。它在三处用到(构造 `HeaderName`、`make_span_with` 里读、后面透传),写死字符串容易拼错,而拼错了不会报错——只会静默失效。

**`HeaderName::from_static`** 在编译期校验头部名合法(必须小写、只含合法字符),比运行时解析更早发现错误。

**`x_request_id.clone()`** 是因为它要在两个 layer 里用。步骤 3 会看到第二次使用。

**`make_span_with` 里读取而不是生成 ID。** 这是本章顺序问题的核心:`TraceLayer` 只负责**读**,ID 是 `SetRequestIdLayer` 在它之前写好的。所以两者顺序不能反。

**`None` 分支记 `error!`** 是防御性写法:正常情况下 ID 一定存在(前一个 layer 保证了),走到这个分支说明中间件顺序错了或者配置有问题。**用日志把这个不该发生的情况暴露出来**,比静默降级好。步骤 4 会故意触发它。

**`request_id = ?request_id`** 用 `?` 前缀按 `Debug` 记录。这里的值是 `HeaderValue` 类型,`Debug` 输出会带引号。

**跑起来看**:

```bash
cargo run
```

```bash
for i in 1 2 3; do curl -s http://127.0.0.1:3000/ > /dev/null & done; wait
```

```
DEBUG http_request{request_id="019a4f2c-8b7d-7e1f-9a2b-6c5d4e3f2a1b"}: tower_http::trace::on_request: started processing request
DEBUG http_request{request_id="019a4f2c-9c8e-7f2a-8b3c-7d6e5f4a3b2c"}: tower_http::trace::on_request: started processing request
INFO http_request{request_id="019a4f2c-8b7d-7e1f-9a2b-6c5d4e3f2a1b"}: request_id: Hello world!
DEBUG http_request{request_id="019a4f2c-ad9f-7a3b-9c4d-8e7f6a5b4c3d"}: tower_http::trace::on_request: started processing request
INFO http_request{request_id="019a4f2c-9c8e-7f2a-8b3c-7d6e5f4a3b2c"}: request_id: Hello world!
DEBUG http_request{request_id="019a4f2c-8b7d-7e1f-9a2b-6c5d4e3f2a1b"}: tower_http::trace::on_response: finished processing request latency=0 ms status=200
...
```

**现在每条日志都带着 ID,同一个请求的三条日志共享同一个 ID。** 交错不再是问题——按 ID 过滤就能捞出完整的一次请求:

```bash
cargo run 2>&1 | grep '019a4f2c-8b7d-7e1f-9a2b-6c5d4e3f2a1b'
```

真实环境里这一步由日志系统完成(Loki、Elasticsearch 的字段查询),但原理一样。

**验证"上游 ID 不被覆盖"。** 自己带一个 ID 发过去:

```bash
curl -s -H 'x-request-id: my-custom-id-12345' http://127.0.0.1:3000/ > /dev/null
```

```
DEBUG http_request{request_id="my-custom-id-12345"}: ...
INFO http_request{request_id="my-custom-id-12345"}: request_id: Hello world!
```

**用的是你传的 ID,没有生成新的。** 这就是前置知识说的行为——网关已经分配过 ID 时沿用它,链路才连得起来。

### 步骤 3:透传到响应头

**要做什么**:加上 `PropagateRequestIdLayer`,让客户端也能拿到 ID。

```rust
// src/main.rs
use tower_http::{
    request_id::{MakeRequestUuid, PropagateRequestIdLayer, SetRequestIdLayer},   // 改动
    trace::TraceLayer,
};
// ...(其余 use 不变)

#[tokio::main]
async fn main() {
    // ...(日志初始化和 x_request_id 不变)

    let middleware = ServiceBuilder::new()
        .layer(SetRequestIdLayer::new(
            x_request_id.clone(),
            MakeRequestUuid,
        ))
        .layer(
            TraceLayer::new_for_http().make_span_with(|request: &Request<_>| {
                // ...(和步骤 2 相同)
            }),
        )
        // 新增:send headers from request to response headers
        .layer(PropagateRequestIdLayer::new(x_request_id));

    // ...(其余不变)
}
```

这里用掉了 `x_request_id` 的所有权(没有 `.clone()`),因为是最后一次使用。

**注意它在 `ServiceBuilder` 里的位置是最后一个。** 按[第 17 章](17-tracing-aka-logging.md#洋葱结构与包装顺序)的规则,`ServiceBuilder` 里越后声明的越靠内层。它在响应路上工作,内层先返回、外层后处理——所以放最内层意味着它最早拿到响应,把 ID 写进去后再往外传。

**跑起来看**:

```bash
cargo run
```

```bash
curl -i -s http://127.0.0.1:3000/ | head -5
```

```
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
x-request-id: 019a4f3d-1e2f-7a3b-8c4d-5e6f7a8b9c0d
content-length: 22
date: Thu, 30 Jul 2026 11:52:33 GMT
```

**响应头里有 `x-request-id` 了。** 它和服务端日志里的 ID 完全一致:

```
INFO http_request{request_id="019a4f3d-1e2f-7a3b-8c4d-5e6f7a8b9c0d"}: request_id: Hello world!
```

**这条链路现在完整了**:客户端遇到问题 → 从响应头拿到 ID → 报给你 → 你用它去日志里精确定位。

自定义 ID 也会被透传:

```bash
curl -i -s -H 'x-request-id: trace-me-please' http://127.0.0.1:3000/ | grep -i request-id
```

```
x-request-id: trace-me-please
```

### 步骤 4:故意把顺序写反

**要做什么**:验证前置知识强调的顺序问题,让 `error!` 那个分支真的触发一次。

把 `SetRequestIdLayer` 和 `TraceLayer` 调换位置:

```rust
    let middleware = ServiceBuilder::new()
        .layer(
            TraceLayer::new_for_http().make_span_with(|request: &Request<_>| {
                // ...(不变)
            }),
        )
        .layer(SetRequestIdLayer::new(         // 临时移到后面
            x_request_id.clone(),
            MakeRequestUuid,
        ))
        .layer(PropagateRequestIdLayer::new(x_request_id));
```

重新运行:

```bash
cargo run
```

```bash
curl -i -s http://127.0.0.1:3000/ | head -3
```

服务端终端:

```
ERROR could not extract request_id
DEBUG http_request: tower_http::trace::on_request: started processing request
INFO http_request: request_id: Hello world!
```

**`ERROR could not extract request_id`,而且 span 里没有 `request_id` 字段了。** 因为 `TraceLayer` 现在跑在 `SetRequestIdLayer` 之前,读请求头时 ID 还没被写进去。

但响应头**仍然有** ID:

```
x-request-id: 019a4f4e-...
```

**这是最容易误判的地方**:响应头看起来一切正常,只有日志悄悄失去了关联能力。如果没有那个 `error!` 分支,你可能很久都发现不了——**这就是步骤 2 里说"把不该发生的情况暴露出来"的价值**。

改回正确顺序。

## 完整代码

`src/main.rs`:

```rust
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
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Html<&'static str> {
    info!("Hello world!");
    Html("<h1>Hello, World!</h1>")
}
```

## 代价与边界

**ID 的可信度问题。** `SetRequestIdLayer` 不覆盖已有的头部,意味着**客户端可以自己指定任意 ID**。步骤 2 里 `x-request-id: my-custom-id-12345` 就直接生效了。

这带来两个风险:

- **日志污染**:攻击者可以发送大量请求都用同一个 ID,或者用超长/含特殊字符的 ID 干扰日志系统
- **日志注入**:如果 ID 里含换行符,可能伪造出假的日志行

**在可信边界内部**(网关之后的内部服务)沿用上游 ID 是对的。**在边界上**(直接面向公网的服务)应该校验或直接覆盖:校验格式是否为合法 UUID、限制长度、或者干脆用 `SetRequestIdLayer` 的变体强制生成新 ID 并把客户端传来的值另存一个字段。

**`MakeRequestUuid` 生成的是 UUID v7 而非 v4。** `tower-http` 较新版本改用了 v7——它带时间戳前缀,所以**按字典序排序就是按时间排序**,对数据库索引和日志检索更友好。上面示例输出里那些以 `019a...` 开头的就是 v7 的时间戳部分。

**这只是追踪的起点,不是全部。** 请求 ID 能把单个服务内、以及跨服务的日志串起来,但它不记录调用关系(谁调用了谁)、不记录各段耗时。完整的分布式追踪需要 OpenTelemetry 那一套(trace ID + span ID + parent span ID)。**不过绝大多数团队从请求 ID 起步就已经解决了 80% 的问题**,而成本只有本章这几行。

**别忘了把 ID 传给下游。** 本章只做了"接收和透传给客户端"。如果你的服务还要调用其它服务,必须手动把 `x-request-id` 加到出站请求的头部里,否则链路在这里就断了。[第 41 章](41-reqwest-response.md)讲调用外部服务时会遇到这个需求。

## 动手练习

**1. 用 `Router::layer` 重写一遍,体会顺序的坑。**

提示:去掉 `ServiceBuilder`,改成三个链式 `.layer()`。记住 `Router::layer` 后注册的在外层,所以书写顺序要和 `ServiceBuilder` **完全相反**。写完验证日志里有没有 `request_id`——如果没有,说明顺序想反了。做完你就明白官方为什么推荐 `ServiceBuilder` 了。

**2. 在业务代码更深的地方记日志,验证 span 的继承。**

提示:在 `handler` 里调用一个普通的辅助函数(比如 `fn do_work() { tracing::info!("working"); }`),观察这条日志有没有带上 `request_id`。它会带上——因为 span 是**任务局部**的,同一个异步任务内的所有日志都继承它,不管调用有多深。这正是 tracing 相对传统日志的核心优势。

**3. 校验客户端传来的 ID,拒绝非法格式。**

提示:写一个自己的 `MakeRequestId` 实现(trait 在 `tower_http::request_id` 里),或者加一个前置中间件检查 `x-request-id` 是否为合法 UUID,不合法就删掉这个头部让 `SetRequestIdLayer` 重新生成。这是「代价与边界」里说的边界防护。验证:`curl -H 'x-request-id: <script>alert(1)</script>' ...` 应该被换成一个正常 UUID。

## 小结与下一章预告

请求 ID 是分布式系统里成本最低、收益最大的可观测性设施:给每个请求一个唯一标识,让它贯穿日志、跨越服务、返回给客户端。用户报错时报上 ID,你能精确捞出那一次请求的完整记录。

`tower-http` 把它拆成两个中间件:`SetRequestIdLayer` 在请求路上设置 ID(**已有则不覆盖**,这样才能沿用上游的 ID),`PropagateRequestIdLayer` 在响应路上把它复制到响应头。

**顺序敏感的中间件组合必须用 `ServiceBuilder`** —— 它的书写顺序和执行顺序一致,而 `Router::layer` 是倒着的。本章故意把顺序写反验证过后果:响应头一切正常,只有日志悄悄失去关联能力,如果没有那句 `error!` 兜底,这个问题可能很久都发现不了。

要注意 ID 的可信度:客户端能自己指定,所以在面向公网的边界上应该校验或强制覆盖。另外本章只做了"接收 + 透传给客户端",服务调用下游时还得手动把它带上,否则链路就断在这里。

「错误处理与工程化」阶段还剩两章。[第 20 章 cors](20-cors.md) 讲一个几乎每个前后端分离项目都会撞上的问题:浏览器的同源策略。你会看到为什么后端明明返回了数据、浏览器控制台却报错,预检请求(preflight)是怎么回事,以及 `CorsLayer` 的几个配置项各自对应哪一道检查。
