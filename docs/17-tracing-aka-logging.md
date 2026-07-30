# 第 17 章 tracing-aka-logging:中间件与结构化日志

## 本章你将做出什么

一个只有一条路由的服务,但挂着一个把六个回调全部展开的 `TraceLayer`。你会看到一次请求在终端产生这样的输出:

```
DEBUG http_request{method=GET matched_path="/"}: started processing request
DEBUG http_request{method=GET matched_path="/"}: sending body chunk
DEBUG http_request{method=GET matched_path="/"}: finished processing request
```

这一章把前面几章欠下的债一次还清:**tower 的 `Service`/`Layer` 到底是什么、`.layer()` 的顺序为什么是"后注册的在外层"、`TraceLayer` 每个回调在什么时机触发、span 和 event 有什么区别、`RUST_LOG` 那串过滤语法怎么写。**

从[第 03 章](03-form.md#步骤-1显示表单页面)起你一直在照抄那段日志初始化样板,这一章解释它。

学完本章你将掌握:

- tower 的 `Service` 与 `Layer`:两个 trait 撑起整个中间件生态
- 中间件的洋葱结构与 `.layer()` 的包装顺序,以及 `ServiceBuilder` 为什么能反转它
- span 与 event 的区别,以及为什么日志要有层次
- `TraceLayer` 六个回调的触发时机
- `EnvFilter` 的过滤语法:target、级别、`axum::rejection=trace` 是什么意思

## 前置知识

### tower:Service 与 Layer

[第 01 章](01-hello-world.md#axum-在这个生态里的位置)介绍生态分层时提过 tower,当时挂了"本章不深入"。现在展开。

**`Service` 是"接收请求、返回响应"的抽象:**

```rust
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;
    fn call(&mut self, req: Request) -> Self::Future;
}
```

注意它**没有绑定 HTTP**。`Request` 是泛型,可以是 HTTP 请求、gRPC 调用、数据库查询——任何"请求—响应"形态的东西。这个泛化让 tower 的中间件能跨协议复用。

`poll_ready` 用于**背压**(backpressure):服务可以说"我现在忙,等会儿再给我请求"。写应用时很少直接用到,但它是限流、连接池这类中间件的基础。

**关键认识:`Router` 本身就是一个 `Service`。** 这不是比喻——`axum::Router` 实现了 `Service<Request<Body>>`。[第 21 章](21-testing.md)的测试之所以能直接对 `Router` 发请求而不启动服务器,靠的就是这一点。

**`Layer` 是"把一个 `Service` 包成另一个 `Service`"的抽象:**

```rust
pub trait Layer<S> {
    type Service;
    fn layer(&self, inner: S) -> Self::Service;
}
```

就一个方法:给我一个内层服务,还你一个包装后的服务。**这就是中间件的全部定义**——没有生命周期钩子、没有拦截器注册表,就是函数式的层层包装。

[第 09 章](09-consume-body-in-extractor-or-middleware.md#中间件在请求前后插入逻辑)用的 `middleware::from_fn` 是 axum 提供的便利入口:把一个 `async fn(Request, Next) -> Response` 包装成 `Layer`。它省掉了手写 `Service` 的样板,代价是灵活性稍低(比如没法实现自定义的 `poll_ready`)。

### 洋葱结构与包装顺序

理解顺序的关键是记住 `Layer::layer(inner)` 这个签名:**它接收内层,返回外层。**

```rust
Router::new()
    .route("/", get(handler))
    .layer(LayerA)      // 包住 router
    .layer(LayerB)      // 包住 (LayerA + router)
    .layer(LayerC)      // 包住 (LayerB + LayerA + router)
```

结果是这样一个洋葱:

```
LayerC
└── LayerB
    └── LayerA
        └── router → handler
```

**所以后调用的 `.layer()` 在更外层。** 请求从外往内穿过 C → B → A → handler,响应从内往外穿回 A → B → C。

这解释了[第 14 章](14-error-handling.md#日志需要请求上下文)那个"顺序不能反"的细节:`TraceLayer` 必须后注册(在外层),这样它建立的 span 才能覆盖内层中间件的日志。

**`ServiceBuilder` 可以反转这个直觉。** `tower::ServiceBuilder` 的 `.layer()` 是**从外往内**声明的:

```rust
let middleware = ServiceBuilder::new()
    .layer(LayerC)      // 最外层
    .layer(LayerB)
    .layer(LayerA);     // 最内层

let app = Router::new().route("/", get(handler)).layer(middleware);
```

写出来的顺序和执行顺序一致,读起来更自然。多个中间件有严格顺序要求时,官方推荐用 `ServiceBuilder`——[第 19 章](19-request-id.md)的请求 ID 就必须这么写,因为"生成 ID"必须在"记录 ID"之前。

一句话记忆:**`Router::layer` 由内向外堆,`ServiceBuilder::layer` 由外向内写。**

### tracing:span 与 event

`tracing` 和传统日志库(`log`、`log4j`)最大的区别是**它有层次结构**。

**event(事件)** 是一个时间点上发生的事,对应传统日志的一行:

```rust
tracing::info!("user created");
tracing::error!(?err, "failed to save");
```

**span(跨度)** 是一段有始有终的上下文:

```rust
let span = tracing::info_span!("http_request", method = "GET");
let _guard = span.enter();
// 这个作用域内记录的所有 event 都会带上 span 的字段
```

关键在于:**span 内部记录的 event 会自动继承 span 的字段**。所以你在业务代码深处写 `tracing::error!("failed")`,输出会自动变成:

```
ERROR http_request{method=GET matched_path="/users/{id}"}: failed
```

不需要把 `request_id`、`user_id` 这些信息一路当参数传下去。这是 tracing 相对传统日志的核心价值,也是[第 14 章](14-error-handling.md#日志需要请求上下文)能让错误日志自动带上请求信息的原因。

**字段语法**前面章节零散用过,这里汇总:

| 写法 | 含义 |
|---|---|
| `tracing::info!(count = 5, "msg")` | 字段值实现了 `Value`(数字、bool、字符串) |
| `tracing::info!(?err, "msg")` | 按 `Debug` 记录,字段名就是变量名 |
| `tracing::info!(%uri, "msg")` | 按 `Display` 记录 |
| `tracing::info!(user.id = 42, "msg")` | 带点号的层级字段名 |
| `tracing::field::Empty` | 占位,稍后用 `span.record(...)` 填 |

span 的宏是 `trace_span!`、`debug_span!`、`info_span!` 等,和 event 的宏一一对应。

### 那段日志初始化样板

从第 03 章起一直在抄的这段,现在逐块解释:

```rust
tracing_subscriber::registry()
    .with(
        tracing_subscriber::EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| "...".into()),
    )
    .with(tracing_subscriber::fmt::layer())
    .init();
```

**`registry()`** 创建一个订阅器(subscriber)。tracing 的设计是**记录与输出分离**:代码里的 `tracing::info!` 只是产生事件,得有订阅器接收才会真正输出。没注册订阅器,所有日志宏都静默丢弃。

**`.with(...)`** 往订阅器上叠加处理层。注意这里的 "layer" 是 `tracing_subscriber` 的概念,**和 tower 的 `Layer` 完全无关**,只是重名。

**`EnvFilter`** 决定哪些日志被放行,规则从 `RUST_LOG` 环境变量读取。

**`fmt::layer()`** 负责格式化并打到终端。

**`.init()`** 把它设为全局订阅器。这个操作全局只能做一次,重复调用会 panic。

### EnvFilter 的过滤语法

`RUST_LOG` 的值是一串用逗号分隔的指令,每条形如 `target=level`。

**target** 默认是产生日志的模块路径(crate 名 + 模块名)。**level** 从低到高是 `trace < debug < info < warn < error`,写 `debug` 表示"debug 及以上都放行"。

看本章示例的默认值:

```
tracing_aka_logging=debug,tower_http=debug,axum::rejection=trace
```

三条指令:

- `tracing_aka_logging=debug` — 本 crate 的 debug 及以上
- `tower_http=debug` — `tower-http` 的 debug 及以上,这样 `TraceLayer` 的日志才看得见
- `axum::rejection=trace` — **这条最值得记住**

axum 会把内置提取器的拒绝以 `axum::rejection` 为 target、`TRACE` 级别记录下来。开启它之后,[第 03 章](03-form.md#步骤-3观察提取失败时的行为)那些 `415`、`422` 在服务端就有日志了——**你不用再靠 curl 的输出猜为什么提取失败**。调试提取器问题时这是第一手段,可惜知道的人不多。

其它常用写法:

```bash
RUST_LOG=debug                      # 全局 debug
RUST_LOG=info,my_crate=trace        # 全局 info,本 crate trace
RUST_LOG=my_crate::handlers=debug   # 只放行某个模块
RUST_LOG=[http_request]=debug       # 按 span 名过滤
```

**注意 crate 名里的连字符要写成下划线**:crate 叫 `tracing-aka-logging`,target 是 `tracing_aka_logging`。示例用 `env!("CARGO_CRATE_NAME")` 自动取,就是为了避免手写出错。

## 项目搭建

```bash
cargo new tracing-aka-logging
cd tracing-aka-logging
```

```bash
cargo add axum --features tracing
cargo add tokio --features full
cargo add tower-http --features trace
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`axum` 的 `tracing` feature 让 axum 内部的日志(尤其是提取器拒绝)接入 tracing,配合上面说的 `axum::rejection=trace` 使用。

## 分步构建

### 步骤 1:默认配置的 TraceLayer

**要做什么**:挂一个不做任何配置的 `TraceLayer`,看它默认给什么。

```rust
// src/main.rs
use axum::{response::Html, routing::get, Router};
use tokio::net::TcpListener;
use tower_http::trace::TraceLayer;
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

    // build our application with a route
    let app = Router::new()
        .route("/", get(handler))
        .layer(TraceLayer::new_for_http());

    // run it
    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
```

`TraceLayer::new_for_http()` 是针对 HTTP 预设好的构造函数——它知道怎么从 `Response` 里读状态码、怎么判断哪些响应算失败(默认 5xx)。

**跑起来看**:

```bash
cargo run
```

```bash
curl -s http://127.0.0.1:3000/ > /dev/null
```

服务端终端:

```
2026-07-30T11:02:14.331847Z DEBUG request{method=GET uri=/ version=HTTP/1.1}: tower_http::trace::on_request: started processing request
2026-07-30T11:02:14.332103Z DEBUG request{method=GET uri=/ version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=0 ms status=200
```

**两条日志,都带着 `request{...}` span 前缀。** 这是 `TraceLayer` 的默认 span,字段是方法、URI、HTTP 版本。

试试 404:

```bash
curl -s http://127.0.0.1:3000/nope > /dev/null
```

```
DEBUG request{method=GET uri=/nope version=HTTP/1.1}: tower_http::trace::on_request: started processing request
DEBUG request{method=GET uri=/nope version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=0 ms status=404
```

**验证 `RUST_LOG` 的作用。** 关掉 `tower_http` 的日志:

```bash
RUST_LOG=tracing_aka_logging=debug cargo run
```

再发请求,两条 `TraceLayer` 日志消失了,只剩启动那条 `listening on ...`。因为过滤指令里没有 `tower_http=debug`,它的日志被拦掉了。

再试试只看 `info` 以上:

```bash
RUST_LOG=info cargo run
```

启动日志也没了——`tracing::debug!` 低于 `info`。

### 步骤 2:自定义 span

**要做什么**:用 `make_span_with` 换掉默认 span,加上 `MatchedPath`。

```rust
// src/main.rs
use axum::{
    extract::MatchedPath,                    // 新增
    http::Request,                           // 新增
    response::Html,
    routing::get,
    Router,
};
use tokio::net::TcpListener;
use tower_http::trace::TraceLayer;
use tracing::info_span;                      // 新增
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    // 改动:自定义 span
    let app = Router::new()
        .route("/", get(handler))
        .layer(
            TraceLayer::new_for_http().make_span_with(|request: &Request<_>| {
                // Log the matched route's path (with placeholders not filled in).
                // Use request.uri() or OriginalUri if you want the real path.
                let matched_path = request
                    .extensions()
                    .get::<MatchedPath>()
                    .map(MatchedPath::as_str);

                info_span!(
                    "http_request",
                    method = ?request.method(),
                    matched_path,
                    some_other_field = tracing::field::Empty,
                )
            }),
        );

    // ...(listener 和 serve 不变)
}

// ...(handler 不变)
```

**`make_span_with` 接收一个闭包**,每个请求调用一次,返回该请求的 span。

**`MatchedPath` 从请求扩展里取。** [第 10 章](10-customize-extractor-error.md#matchedpath拿到路由模式而非实际路径)讲过它和 `Uri` 的区别:`MatchedPath` 给的是路由**模式**(`/users/{id}`),`Uri` 给的是实际路径(`/users/42`)。

注释也提醒了这一点。做监控指标时必须用前者——用实际路径当标签会产生无限多的时间序列,把 Prometheus 撑爆。这在[第 47 章](47-prometheus-metrics.md)会正经讲。

**`method = ?request.method()`** 用 `?` 前缀按 `Debug` 记录。

**`some_other_field = tracing::field::Empty`** 声明一个**空占位字段**。span 的字段集合在创建时就固定了,后面不能新增——但可以给已声明的空字段填值:

```rust
span.record("some_other_field", 42);
```

用途是"创建 span 时还不知道值,处理过程中才算出来"。比如响应大小、数据库耗时、用户 ID。下一步的回调里会看到怎么填。

**跑起来看**:

```bash
cargo run
```

```bash
curl -s http://127.0.0.1:3000/ > /dev/null
```

```
DEBUG http_request{method=GET matched_path="/"}: tower_http::trace::on_request: started processing request
DEBUG http_request{method=GET matched_path="/"}: tower_http::trace::on_response: finished processing request latency=0 ms status=200
```

**span 名从 `request` 变成了 `http_request`,字段也换了。** `some_other_field` 没显示,因为它还是 `Empty`——空字段不会输出。

访问不存在的路径:

```bash
curl -s http://127.0.0.1:3000/nope > /dev/null
```

```
DEBUG http_request{method=GET}: tower_http::trace::on_request: started processing request
```

**`matched_path` 不见了。** 因为路由没匹配上,axum 没有往扩展里放 `MatchedPath`,`.map(...)` 得到 `None`,而值为 `None` 的字段不会输出。这个行为合理——没匹配上就没有"路由模式"可言。

### 步骤 3:展开全部六个回调

**要做什么**:把 `TraceLayer` 的每个回调都接上,看清一次请求的完整生命周期。

```rust
// src/main.rs
use axum::{
    body::Bytes,                                          // 新增
    extract::MatchedPath,
    http::{HeaderMap, Request},                           // 改动:加上 HeaderMap
    response::{Html, Response},                           // 改动:加上 Response
    routing::get,
    Router,
};
use std::time::Duration;                                  // 新增
use tokio::net::TcpListener;
use tower_http::{classify::ServerErrorsFailureClass, trace::TraceLayer};   // 改动
use tracing::{info_span, Span};                           // 改动:加上 Span
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    let app = Router::new()
        .route("/", get(handler))
        // `TraceLayer` is provided by tower-http so you have to add that as a dependency.
        // It provides good defaults but is also very customizable.
        //
        // See https://docs.rs/tower-http/0.1.1/tower_http/trace/index.html for more details.
        //
        // If you want to customize the behavior using closures here is how.
        .layer(
            TraceLayer::new_for_http()
                .make_span_with(|request: &Request<_>| {
                    // ...(和步骤 2 相同)
                })
                // 新增:以下五个回调
                .on_request(|_request: &Request<_>, _span: &Span| {
                    // You can use `_span.record("some_other_field", value)` in one of these
                    // closures to attach a value to the initially empty field in the info_span
                    // created above.
                    tracing::debug!("started processing request")
                })
                .on_response(|_response: &Response, _latency: Duration, _span: &Span| {
                    tracing::debug!("finished processing request")
                })
                .on_body_chunk(|_chunk: &Bytes, _latency: Duration, _span: &Span| {
                    tracing::debug!("sending body chunk")
                })
                .on_eos(
                    |_trailers: Option<&HeaderMap>, _stream_duration: Duration, _span: &Span| {
                        tracing::debug!("stream closed")
                    },
                )
                .on_failure(
                    |_error: ServerErrorsFailureClass, _latency: Duration, _span: &Span| {
                        tracing::error!("something went wrong")
                    },
                ),
        );

    // ...(listener 和 serve 不变)
}

// ...(handler 不变)
```

六个回调的触发时机:

| 回调 | 何时触发 | 典型用途 |
|---|---|---|
| `make_span_with` | 请求到达,处理开始前 | 建立上下文 |
| `on_request` | span 建立后,进入内层前 | 记录请求开始 |
| `on_response` | 内层返回响应头之后 | 记录状态码、耗时 |
| `on_body_chunk` | **每发送一块响应体** | 统计流量,流式响应时会多次触发 |
| `on_eos` | 流结束(HTTP/2 trailers) | 记录流式响应真正的结束 |
| `on_failure` | 响应被判定为失败 | 告警、错误统计 |

几个容易踩的点:

**`on_response` 不代表响应发完了。** 它在响应**头**准备好时触发,body 可能还在流式发送。所以那个 `latency` 参数是"到响应头就绪"的耗时,不含 body 传输时间。真正的完整耗时要看 `on_eos`。

**`on_body_chunk` 在流式响应下会触发很多次。** 本章的响应很小,只会触发一次;但 SSE([第 33 章](33-sse.md))或大文件下载会触发成百上千次。**在这个回调里写日志要非常小心**——它可能瞬间产生海量输出。生产环境通常只在这里累加计数器,不打日志。

**"失败"的判定由 classifier 决定。** `ServerErrorsFailureClass` 是 `new_for_http()` 的默认分类器:5xx 算失败,4xx 不算。这个默认值符合[第 14 章](14-error-handling.md#4xx-与-5xx-决定了两件事)讲的原则——客户端错误是正常流量,不该触发告警。

[第 14 章](14-error-handling.md#步骤-4给日志加上请求上下文)用 `.on_failure(())` 把这个回调关掉了,因为那一章要自己记录错误,不想重复。传 `()` 表示"什么都不做",这是 `tower-http` 的约定。

**所有回调都能拿到 `&Span`**,所以能用 `span.record(...)` 填步骤 2 声明的空字段。注释就是在说这件事。

**跑起来看**:

```bash
cargo run
```

```bash
curl -s http://127.0.0.1:3000/ > /dev/null
```

```
DEBUG http_request{method=GET matched_path="/"}: tracing_aka_logging: started processing request
DEBUG http_request{method=GET matched_path="/"}: tracing_aka_logging: sending body chunk
DEBUG http_request{method=GET matched_path="/"}: tracing_aka_logging: finished processing request
```

**三条,而且注意 target 变了**:从 `tower_http::trace::on_request` 变成了 `tracing_aka_logging`。因为日志现在是**你的闭包**记录的,target 自然是你的 crate。

顺序也值得看:`started` → `sending body chunk` → `finished`。`on_body_chunk` 出现在 `on_response` 之前,印证了上面说的"`on_response` 触发时 body 还没发完"。

**验证 `on_failure`。** 临时加一条会返回 500 的路由:

```rust
use axum::http::StatusCode;

    let app = Router::new()
        .route("/", get(handler))
        .route("/boom", get(|| async { StatusCode::INTERNAL_SERVER_ERROR }))
        // ...(layer 不变)
```

```bash
curl -s http://127.0.0.1:3000/boom > /dev/null
```

```
DEBUG http_request{method=GET matched_path="/boom"}: tracing_aka_logging: started processing request
DEBUG http_request{method=GET matched_path="/boom"}: tracing_aka_logging: finished processing request
ERROR http_request{method=GET matched_path="/boom"}: tracing_aka_logging: something went wrong
```

**多了一条 ERROR。** 而 404 不会触发它:

```bash
curl -s http://127.0.0.1:3000/nope > /dev/null
```

只有 `started` 和 `finished`,没有 ERROR——4xx 不算失败。

**再验证 `axum::rejection=trace` 的价值。** 加一条需要 JSON 的路由:

```rust
use axum::Json;

        .route("/json", axum::routing::post(|Json(v): Json<serde_json::Value>| async move {
            format!("{v}")
        }))
```

需要 `cargo add serde_json`。发一个格式错误的请求:

```bash
curl -s -X POST http://127.0.0.1:3000/json \
  -H 'Content-Type: application/json' -d '{bad' > /dev/null
```

```
DEBUG http_request{method=POST matched_path="/json"}: tracing_aka_logging: started processing request
TRACE http_request{method=POST matched_path="/json"}: axum::rejection: rejecting request rejection=JsonSyntaxError(...)
DEBUG http_request{method=POST matched_path="/json"}: tracing_aka_logging: finished processing request
```

**中间那条 `axum::rejection` 日志直接告诉你提取器为什么拒绝了请求。** 这就是默认过滤器里那条 `axum::rejection=trace` 的作用。前面十几章调试提取器都只能看 curl 的输出,现在服务端也有据可查了。

试试关掉它:

```bash
RUST_LOG=tracing_aka_logging=debug,tower_http=debug cargo run
```

再发同样的请求,那条 TRACE 就没了。

把临时加的两条路由删掉,回到官方示例的形态。

## 代价与边界

**`on_body_chunk` 里打日志在流式场景会失控。** 上面提过,SSE 或大文件下载会触发成千上万次。本章示例把它接上是为了演示,**生产环境请谨慎**——要么删掉这个回调,要么只在里面累加计数器。

**日志本身是有成本的。** 每条日志都要格式化、写 IO。`debug` 级别在高 QPS 服务上产生的日志量可能超过业务流量本身,还会因为 IO 阻塞拖慢响应。生产环境通常设 `info` 或 `warn`,只在排查问题时临时调高。

**span 会带来内存开销。** 每个请求一个 span,字段值要保存到 span 结束。字段里放大对象(比如整个请求体)会显著增加内存占用。

**日志里不要放敏感信息。** 这是最容易犯的错:span 字段里放了完整 URI,而 URI 的查询字符串里可能有 token;记录请求头时可能连 `Authorization` 和 cookie 一起记了。日志通常会被转发到集中式系统、长期保留、被很多人访问——**放进去的东西等于公开**。真实项目应该有明确的脱敏规则。

**`.init()` 全局只能调一次。** 在测试里尤其容易踩:多个测试各自初始化订阅器会 panic。测试中用 `tracing_subscriber::fmt().with_test_writer().try_init()`(`try_init` 失败返回 `Err` 而非 panic)。

## 完整代码

`src/main.rs`:

```rust
use axum::{
    body::Bytes,
    extract::MatchedPath,
    http::{HeaderMap, Request},
    response::{Html, Response},
    routing::get,
    Router,
};
use std::time::Duration;
use tokio::net::TcpListener;
use tower_http::{classify::ServerErrorsFailureClass, trace::TraceLayer};
use tracing::{info_span, Span};
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

    // build our application with a route
    let app = Router::new()
        .route("/", get(handler))
        // `TraceLayer` is provided by tower-http so you have to add that as a dependency.
        // It provides good defaults but is also very customizable.
        //
        // See https://docs.rs/tower-http/0.1.1/tower_http/trace/index.html for more details.
        //
        // If you want to customize the behavior using closures here is how.
        .layer(
            TraceLayer::new_for_http()
                .make_span_with(|request: &Request<_>| {
                    // Log the matched route's path (with placeholders not filled in).
                    // Use request.uri() or OriginalUri if you want the real path.
                    let matched_path = request
                        .extensions()
                        .get::<MatchedPath>()
                        .map(MatchedPath::as_str);

                    info_span!(
                        "http_request",
                        method = ?request.method(),
                        matched_path,
                        some_other_field = tracing::field::Empty,
                    )
                })
                .on_request(|_request: &Request<_>, _span: &Span| {
                    // You can use `_span.record("some_other_field", value)` in one of these
                    // closures to attach a value to the initially empty field in the info_span
                    // created above.
                    tracing::debug!("started processing request")
                })
                .on_response(|_response: &Response, _latency: Duration, _span: &Span| {
                    tracing::debug!("finished processing request")
                })
                .on_body_chunk(|_chunk: &Bytes, _latency: Duration, _span: &Span| {
                    tracing::debug!("sending body chunk")
                })
                .on_eos(
                    |_trailers: Option<&HeaderMap>, _stream_duration: Duration, _span: &Span| {
                        tracing::debug!("stream closed")
                    },
                )
                .on_failure(
                    |_error: ServerErrorsFailureClass, _latency: Duration, _span: &Span| {
                        tracing::error!("something went wrong")
                    },
                ),
        );

    // run it
    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
```

依赖:

- `axum` 0.8.x,feature `tracing`
- `tokio` 1.x,feature `full`
- `tower-http` 0.6.x,feature `trace`
- `tracing` 0.1.x
- `tracing-subscriber` 0.3.x,feature `env-filter`

## 动手练习

**1. 给那个空字段填上值。**

提示:在 `on_response` 回调里调用 `_span.record("some_other_field", _latency.as_millis() as u64)`(记得把参数名前的下划线去掉)。重新运行后观察日志——`some_other_field` 会出现在 span 的字段里。想想为什么必须先在 `info_span!` 里用 `Empty` 声明,不能直接 `record` 一个没声明过的字段。

**2. 验证中间件的洋葱顺序。**

提示:用 `middleware::from_fn` 写两个中间件 A 和 B,各自在 `next.run` 前后打一条日志(`A-in`/`A-out`、`B-in`/`B-out`)。挂成 `.layer(from_fn(a)).layer(from_fn(b))`,然后看日志顺序。你应该看到 `B-in → A-in → A-out → B-out`——后注册的 B 在外层。再换成 `ServiceBuilder::new().layer(from_fn(a)).layer(from_fn(b))`,顺序会反过来。

**3. 改用 JSON 格式输出日志。**

提示:把 `tracing_subscriber::fmt::layer()` 换成 `tracing_subscriber::fmt::layer().json()`。日志会变成每行一个 JSON 对象,span 字段作为嵌套结构。这是把日志接入 ELK、Loki 这类集中式系统的标准做法——机器解析比正则匹配文本可靠得多。

## 小结与下一章预告

tower 的整个中间件生态建立在两个 trait 上:`Service` 是"接收请求返回响应"(`Router` 自己就是一个 `Service`),`Layer` 是"把一个 `Service` 包成另一个"。中间件就是层层包装的洋葱,**`Router::layer` 后注册的在外层,`ServiceBuilder::layer` 则按书写顺序从外到内**——记住这个区别,顺序敏感的中间件才不会写反。

tracing 区别于传统日志的核心是 **span**:span 内部记录的 event 自动继承 span 的字段,所以请求上下文不用一路当参数传下去。字段用 `?` 前缀走 `Debug`、`%` 走 `Display`,`field::Empty` 占位后可用 `span.record` 填值。

`TraceLayer` 的六个回调覆盖请求的完整生命周期。要注意 `on_response` 触发时 body 还没发完,`on_body_chunk` 在流式响应下会触发极多次(**别在里面打日志**),而"失败"默认只算 5xx——4xx 是正常流量,不该告警。

`EnvFilter` 的 `axum::rejection=trace` 值得单独记住:开启后提取器为什么拒绝请求会直接写进服务端日志,调试提取器问题不用再靠猜。

`TraceLayer` 记录的是请求的元信息。有时你需要看到**请求和响应的完整内容**——尤其是对接第三方、排查"我发的到底是什么"这类问题时。[第 18 章 print-request-response](18-print-request-response.md) 写一个把两个方向的 body 都打印出来的中间件,它是[第 09 章](09-consume-body-in-extractor-or-middleware.md)那个"拆开—读取—重组"手法的完整版:这次响应体也要同样处理。
