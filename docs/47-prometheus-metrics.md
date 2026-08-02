# 第 47 章 prometheus-metrics:指标采集与基数陷阱

## 本章你将做出什么

一个**把自己的运行状况暴露给监控系统**的服务:

```
127.0.0.1:3000    /fast   立刻返回
                  /slow   睡 1 秒再返回
127.0.0.1:3001    /metrics   Prometheus 格式的指标
```

打几次请求,然后 `curl 127.0.0.1:3001/metrics`,你会看到:

```
http_requests_total{method="GET",path="/fast",status="200"} 5
http_requests_duration_seconds_bucket{...,le="0.005"} 5
```

**请求量、延迟分布、错误率,全都有了。**

学完本章你将掌握:

- 日志、指标、链路追踪的分工,以及**指标解决日志解决不了的什么问题**
- Prometheus 的**拉模型**和文本暴露格式
- Counter / Gauge / Histogram / Summary 四种指标类型该怎么选
- `metrics` 门面 + `metrics-exporter-prometheus` 记录器的分层
- **直方图分桶**:为什么要自己配,以及配了之后输出格式会变
- **标签基数爆炸**:一个能把监控系统撑爆的经典陷阱
- **`MatchedPath`**:axum 为这个陷阱准备的解药
- `route_layer` 和 `layer` 的区别,以及它在这里为什么关键

## 前置知识

### 可观测性的三根支柱

| | 回答什么问题 | 数据形态 | 成本 |
|---|---|---|---|
| **日志(Logs)** | "**这一次**请求发生了什么" | 一条条文本/结构化事件 | 高(量大) |
| **指标(Metrics)** | "**整体上**现在什么状况" | 数字时间序列 | **低**(聚合过) |
| **链路追踪(Traces)** | "这次请求在**各个服务**里各花了多久" | 带父子关系的 span | 中(通常采样) |

[第 17 章](17-tracing-aka-logging.md)讲的是日志。日志很强,但**回答不了"现在 P99 延迟是多少"这类问题**——你得把海量日志聚合一遍,慢且贵。

**指标的本质是"在应用内部就聚合好"**:不记录每次请求耗时多少,而是维护"落在 5ms 桶里的请求有多少个"这样的计数器。**内存占用固定,查询极快**,代价是丢失了单次请求的细节。

**两者互补,不能互相替代。** 指标告诉你"错误率涨到 5% 了",日志告诉你"因为这个用户的这个参数"。

### Prometheus 是拉(pull)模型

大多数人的直觉是"应用把指标推给监控系统"。**Prometheus 反过来**:

```
Prometheus 服务器  ──每 15 秒──>  GET http://你的应用:3001/metrics
                  <──文本响应──
```

**应用只负责在一个 HTTP 端点上把当前值渲染成文本,剩下的都是 Prometheus 的事。**

这么设计的好处:

- **应用不需要知道监控系统在哪** —— 配置在 Prometheus 那边,靠服务发现找目标
- **抓取失败本身就是一个信号** —— `up` 指标会变成 0,天然的健康监控
- **应用侧无状态、无缓冲、无重试** —— 实现极其简单,就是一个 GET 接口

代价是:**短命任务(批处理、CronJob)可能还没被抓就结束了**,这类场景要用 Pushgateway 中转。

### 暴露格式长什么样

就是纯文本,一行一个时间序列:

```
# HELP http_requests_total 总请求数
# TYPE http_requests_total counter
http_requests_total{method="GET",path="/fast",status="200"} 5
http_requests_total{method="GET",path="/slow",status="200"} 2
```

结构是:

```
指标名{标签1="值",标签2="值"} 数值
```

**指标名 + 一组标签值 = 一个时间序列(series)。** 上面这两行是**两个**不同的 series。

**记住这句话**,后面讲基数爆炸全靠它。

### 四种指标类型

| 类型 | 语义 | 只增不减? | 例子 |
|---|---|---|---|
| **Counter** | 累计计数 |  是(重启归零) | 总请求数、总错误数 |
| **Gauge** | 瞬时值 |  可增可减 | 当前连接数、内存占用、队列长度 |
| **Histogram** | 分布(分桶计数) | 是 | 请求延迟、响应体大小 |
| **Summary** | 分布(客户端算分位数) | 是 | 同上,但**不能跨实例聚合** |

**Counter 为什么只增不减?** 因为 Prometheus 查询时用 `rate()` 算增长速率。计数器单调递增,`rate` 才有意义;重启归零也能被 `rate` 正确处理(它会识别出"计数器重置")。

**"总请求数"用 Counter,"当前正在处理的请求数"用 Gauge** —— 这是最容易混的一对。

**Histogram vs Summary 的关键区别**:

- **Histogram** 在应用侧只记录**每个桶的计数**,分位数在 Prometheus **查询时**用 `histogram_quantile()` 算。**能跨实例聚合**(把 10 个实例的桶加起来再算 P99)。
- **Summary** 在应用侧直接算好分位数。**不能跨实例聚合**(10 个实例的 P99 求平均没有意义)。

**多实例部署几乎总是该用 Histogram。** 本章步骤 4 会看到,`metrics-exporter-prometheus` **默认输出的是 Summary**,要配了分桶才变成 Histogram。

### 直方图的分桶

Histogram 需要你**预先定义边界**:

```rust
const EXPONENTIAL_SECONDS: &[f64] = &[
    0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0,
];
```

含义是"**小于等于 5ms 的有几个、小于等于 10ms 的有几个、……**"(累积计数,Prometheus 里叫 `le`,less-or-equal)。渲染出来是:

```
http_requests_duration_seconds_bucket{le="0.005"} 5
http_requests_duration_seconds_bucket{le="0.01"}  5
...
http_requests_duration_seconds_bucket{le="+Inf"}  7
http_requests_duration_seconds_sum   2.003
http_requests_duration_seconds_count 7
```

**分桶要贴合你的实际延迟分布。** 如果服务的响应普遍在 1~3ms,而最小的桶是 5ms,那所有请求都落在第一个桶里,**你什么也看不出来**。反过来,桶分得太细会增加 series 数量(每个桶都是一个 series)。

上面这组是**指数递增**的,覆盖 5ms 到 10s 共 4 个数量级,是一个通用的起点。

### `metrics` 门面 + 记录器

和 `tracing` / `tracing-subscriber` 完全一样的分层([第 17 章](17-tracing-aka-logging.md#那段日志初始化样板)见过这个模式):

```
你的代码  →  metrics::counter!(...)          ← 门面,只是记录"发生了什么"
                     ↓
           metrics-exporter-prometheus       ← 记录器,决定怎么存、怎么渲染
```

**好处是业务代码不绑定具体的监控系统。** 想换成 StatsD 或 OpenTelemetry,只换记录器,`counter!` 那些调用一行不用动。

 **没有安装记录器时,`counter!` 是静默的空操作。** 不报错、不 panic,数据直接丢弃。这个特性很方便(库作者可以随便埋点,不强迫使用者装记录器),但也意味着**忘了安装记录器时你不会收到任何提示**。

```rust
let recorder_handle = PrometheusBuilder::new().install_recorder().unwrap();
```

`install_recorder()` 把它注册成**全局记录器**,并返回一个 `PrometheusHandle`。这个 handle 的核心方法是:

```rust
recorder_handle.render()   // → String,就是那段文本
```

**所以 `/metrics` 路由的实现就一行:把 `render()` 的结果返回。**

### 标签基数:能把监控系统撑爆的东西

回到那句话:**指标名 + 一组标签值 = 一个 series**。

Prometheus **为每个 series 在内存里维护一份索引和最近的样本**。series 数量直接决定内存占用。算一笔账:

```
method(4 种) × path(20 条路由) × status(6 种)  =  480 个 series
```

完全没问题。但如果 `path` 用的是**原始 URL 路径**:

```
GET /user/1
GET /user/2
GET /user/3
...
```

**每一个 user id 都是一个新的 series。** 100 万用户就是 100 万个 series,乘上 method 和 status,再乘上直方图的 11 个桶……**Prometheus 会 OOM,或者被迫丢弃数据。**

这个问题有个专门的名字:**基数爆炸(cardinality explosion)**。它是指标系统头号事故来源。

**规则:标签的取值集合必须是有限且很小的。** 用户 ID、请求 ID、时间戳、UUID、原始路径、错误信息 —— **这些都绝对不能当标签**。它们属于日志。

### `MatchedPath`:axum 的解药

axum 提供了一个提取器,拿到的是**匹配上的路由模板**,而不是实际路径:

| 实际请求 | `uri().path()` | `MatchedPath` |
|---|---|---|
| `GET /user/1` | `/user/1` | `/user/{id}` |
| `GET /user/2` | `/user/2` | `/user/{id}` |
| `GET /user/999999` | `/user/999999` | `/user/{id}` |

**无论多少个用户,`MatchedPath` 都只有一个值。** 基数从"用户数"降到"路由数"。

它是在路由匹配时被放进请求扩展的,所以**中间件必须跑在路由之后**才能读到——这就是下面要用 `route_layer` 而不是 `layer` 的原因。

### `route_layer` vs `layer`

| | `Router::layer` | `Router::route_layer` |
|---|---|---|
| 对没匹配上的请求(404) | **也会运行** | **不运行** |
| 典型用途 | 日志、CORS、压缩 | **鉴权、指标** |

对指标来说 `route_layer` 是正确选择,有两个理由:

1. **能读到 `MatchedPath`**(虽然 `layer` 在 axum 里也能读到,因为路由匹配发生在中间件之前;但 `layer` 会对 404 也执行,那时 `MatchedPath` 不存在)
2. **扫描器的垃圾流量不会被统计** —— 公网上每天有大量机器人请求 `/wp-admin`、`/.env` 之类的路径,如果给这些 404 也记指标,又是一场基数爆炸

 **一个共同的坑:`layer` 和 `route_layer` 只对"在它之前注册的路由"生效。** 之后再 `.route(...)` 加的路由不会被包住。这个规则很容易忘,而且**失败方式是静默的**(那条路由的指标就是没有)。

## 项目搭建

```bash
cargo new prometheus-metrics
cd prometheus-metrics
```

```bash
cargo add axum
cargo add metrics --no-default-features
cargo add metrics-exporter-prometheus --no-default-features
cargo add tokio --features full
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`--no-default-features` 关掉了这两个 crate 里我们用不上的部分(比如 `metrics-exporter-prometheus` 自带的 HTTP 服务器——我们要用 axum 自己提供 `/metrics`)。

## 分步构建

### 步骤 1:两个服务,两个端口

**要做什么**:先把结构搭出来——业务服务在 3000,指标服务在 3001。

```rust
// src/main.rs
use axum::{routing::get, Router};
use std::time::Duration;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

fn metrics_app() -> Router {
    Router::new().route("/metrics", get(|| async { "not yet" }))
}

fn main_app() -> Router {
    Router::new()
        .route("/fast", get(|| async {}))
        .route(
            "/slow",
            get(|| async {
                tokio::time::sleep(Duration::from_secs(1)).await;
            }),
        )
}

async fn start_main_server() {
    let app = main_app();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn start_metrics_server() {
    let app = metrics_app();

    // NOTE: expose metrics endpoint on a different port
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3001")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

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

    // The `/metrics` endpoint should not be publicly available. If behind a reverse proxy, this
    // can be achieved by rejecting requests to `/metrics`. In this example, a second server is
    // started on another port to expose `/metrics`.
    let (_main_server, _metrics_server) = tokio::join!(start_main_server(), start_metrics_server());
}
```

**为什么要拆成两个端口?** 官方注释说得很清楚:**`/metrics` 不应该对外公开**。

指标里包含大量内部信息:有哪些路由、各自的调用量、错误率、依赖了哪些外部服务、甚至版本号。**这对攻击者是一份免费的侦察报告。**

两种常见做法:

- **单独端口**(本章):3000 对外,3001 只在内网/本机开放
- **反向代理拦截**([第 43 章](43-reverse-proxy.md)):在代理层直接拒绝 `/metrics` 的外部请求

k8s 里通常是前者:业务端口进 Service,指标端口只在 Pod 上声明,由 Prometheus 直接抓 Pod IP。

**`main_app()` / `metrics_app()` 拆成函数**,这是[第 21 章](21-testing.md)讲过的可测试结构。

**`tokio::join!` 同时跑两个服务** —— [第 44 章](44-serve-with-hyper.md#tokiojoin-vs-tokiospawn)讲过它和 `spawn` 的区别。

**跑起来看**:

```bash
cargo run
```

```bash
curl http://127.0.0.1:3000/fast      # 空响应,200
time curl http://127.0.0.1:3000/slow # 约 1 秒
curl http://127.0.0.1:3001/metrics   # not yet
```

三个端点都活着,骨架完成。

### 步骤 2:安装 Prometheus 记录器

**要做什么**:装上记录器,让 `/metrics` 返回真正的渲染结果。

```rust
use metrics_exporter_prometheus::{PrometheusBuilder, PrometheusHandle};
use std::future::ready;
```

```rust
fn setup_metrics_recorder() -> PrometheusHandle {
    PrometheusBuilder::new().install_recorder().unwrap()
}

fn metrics_app() -> Router {
    let recorder_handle = setup_metrics_recorder();
    Router::new().route("/metrics", get(move || ready(recorder_handle.render())))
}
```

**`get(move || ready(recorder_handle.render()))` 这个 handler 值得看两眼:**

```rust
move || ready(recorder_handle.render())
```

它**不是 `async` 闭包**,而是一个返回 future 的普通闭包。`std::future::ready(v)` 产生一个**立刻就绪、值为 `v`** 的 future。

[第 45 章](45-graceful-shutdown.md#步骤-1两个慢路由先看看粗暴退出长什么样)用过 `std::future::pending`(永不完成),这里是它的反面。**渲染指标是纯 CPU 操作,没有任何 IO,用 `ready` 比 `async move { ... }` 更直白地表达了"这里不会等待"。**

`move` 把 `recorder_handle` 的所有权移进闭包。**`PrometheusHandle` 是可克隆的共享句柄**,和[第 46 章](46-tls-graceful-shutdown.md#axum_serverhandle-能做什么)的 `axum_server::Handle` 一个模式。

**跑起来看**:

```bash
cargo run
curl http://127.0.0.1:3001/metrics
```

**输出是空的。** 因为还没有任何埋点。记录器装好了,但没人往里写数据。

### 步骤 3:埋点中间件

**要做什么**:写一个中间件,给每个请求记一次计数和一次耗时。

```rust
use axum::{
    extract::{MatchedPath, Request},
    middleware::{self, Next},
    response::IntoResponse,
};
use std::time::Instant;
```

```rust
async fn track_metrics(req: Request, next: Next) -> impl IntoResponse {
    let start = Instant::now();
    let path = if let Some(matched_path) = req.extensions().get::<MatchedPath>() {
        matched_path.as_str().to_owned()
    } else {
        req.uri().path().to_owned()
    };
    let method = req.method().clone();

    let response = next.run(req).await;

    let latency = start.elapsed().as_secs_f64();
    let status = response.status().as_u16().to_string();

    let labels = [
        ("method", method.to_string()),
        ("path", path),
        ("status", status),
    ];

    metrics::counter!("http_requests_total", &labels).increment(1);
    metrics::histogram!("http_requests_duration_seconds", &labels).record(latency);

    response
}
```

```rust
fn main_app() -> Router {
    Router::new()
        .route("/fast", get(|| async {}))
        .route(
            "/slow",
            get(|| async {
                tokio::time::sleep(Duration::from_secs(1)).await;
            }),
        )
        .route_layer(middleware::from_fn(track_metrics))
}
```

**这是一个标准的 `middleware::from_fn` 中间件**,结构和[第 09 章](09-consume-body-in-extractor-or-middleware.md)讲的洋葱一样:`next.run(req).await` 之前是"请求侧",之后是"响应侧"。

**逐块看:**

**① 为什么在 `next.run` 之前取 `path` 和 `method`?**

因为 **`req` 会被 `next.run(req)` 消耗掉**。之后想再读请求信息就没得读了。[第 09 章](09-consume-body-in-extractor-or-middleware.md)讲过同样的所有权问题。

**② `req.extensions().get::<MatchedPath>()`**

这里没用提取器语法,而是**直接从扩展里读**。因为中间件拿到的是裸 `Request`,提取器是 handler 的东西。`MatchedPath` 本质上就是路由匹配时插进扩展的一个值,提取器只是帮你读出来的语法糖。

**③ 那个 `else` 分支是个隐患。**

前置知识说过,`route_layer` 保证了中间件只在匹配上路由时运行,所以 `MatchedPath` 应该总是存在,`else` 是死代码。

> **更好的写法**:**`else` 分支不该用原始路径。** 万一将来有人把 `route_layer` 改成 `layer`(比如"想统计 404 数量"),这个分支立刻活过来,而公网上的扫描器每天会请求成千上万个不存在的路径(`/wp-admin`、`/.env`、`/../../etc/passwd`),**每一个都变成一个新的 series**。
>
> 安全的写法是给未匹配的请求一个固定标签:
>
> ```rust
> } else {
>     "<unmatched>".to_owned()
> };
> ```
>
> 这样即使有人改成 `layer`,基数也是可控的。**把安全性建立在"不会有人改这行代码"上是不牢靠的。**

**④ `status` 在 `next.run` 之后才能取** —— 显而易见,但正好说明为什么这个中间件必须包住 handler 而不是只做前置处理。

**⑤ 两个宏的写法:**

```rust
metrics::counter!("http_requests_total", &labels).increment(1);
metrics::histogram!("http_requests_duration_seconds", &labels).record(latency);
```

`counter!` / `histogram!` 返回一个**句柄**,再调 `.increment` / `.record` 才真正写入。这是 `metrics` 0.24 的 API 风格——句柄可以复用,避免高频路径上重复查找。

**跑起来看**:

```bash
cargo run
```

```bash
curl http://127.0.0.1:3000/fast
curl http://127.0.0.1:3000/fast
curl http://127.0.0.1:3000/slow
curl http://127.0.0.1:3001/metrics
```

```
# TYPE http_requests_total counter
http_requests_total{method="GET",path="/fast",status="200"} 2
http_requests_total{method="GET",path="/slow",status="200"} 1

# TYPE http_requests_duration_seconds summary
http_requests_duration_seconds{method="GET",path="/fast",status="200",quantile="0"} 0.000041
http_requests_duration_seconds{method="GET",path="/fast",status="200",quantile="0.5"} 0.000045
http_requests_duration_seconds{method="GET",path="/fast",status="200",quantile="0.9"} 0.000048
...
http_requests_duration_seconds_sum{method="GET",path="/fast",status="200"} 0.000086
http_requests_duration_seconds_count{method="GET",path="/fast",status="200"} 2
```

**注意 `# TYPE ... summary`** —— 前置知识说过,`metrics-exporter-prometheus` **默认把直方图渲染成 Summary**(应用侧算好的分位数)。下一步改掉它。

再试试 404:

```bash
curl http://127.0.0.1:3000/nope
curl http://127.0.0.1:3001/metrics
```

**指标里没有 `/nope`。** `route_layer` 生效了——没匹配上的请求根本不进这个中间件。

### 步骤 4:配置分桶,变成真正的 Histogram

**要做什么**:给延迟指标指定桶边界,并加上周期性维护。

```rust
use metrics_exporter_prometheus::Matcher;
```

```rust
fn setup_metrics_recorder() -> PrometheusHandle {
    const EXPONENTIAL_SECONDS: &[f64] = &[
        0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0,
    ];

    let recorder_handle = PrometheusBuilder::new()
        .set_buckets_for_metric(
            Matcher::Full("http_requests_duration_seconds".to_string()),
            EXPONENTIAL_SECONDS,
        )
        .unwrap()
        .install_recorder()
        .unwrap();

    let upkeep_handle = recorder_handle.clone();
    tokio::spawn(async move {
        loop {
            tokio::time::sleep(Duration::from_secs(5)).await;
            upkeep_handle.run_upkeep();
        }
    });

    recorder_handle
}
```

**`Matcher::Full(...)` 是精确匹配指标名。** 还有 `Matcher::Prefix`(前缀)和 `Matcher::Suffix`(后缀),后两个适合"所有以 `_seconds` 结尾的都用这套桶"这种批量配置。

**`run_upkeep()` 是周期性维护。** 记录器内部对某些指标(尤其是 Summary 类型的滚动分位数)维护着时间窗口,`run_upkeep` 推进这个窗口、清掉过期数据。**不调用的话,分位数会一直反映着很久以前的数据。**

5 秒一次是个合理的默认值,和 Prometheus 常见的 15 秒抓取间隔配合得不错。

**注意 `recorder_handle.clone()`** —— spawn 出去的任务需要一份自己的句柄,原件要返回给 `/metrics` 路由用。

**跑起来看**:

```bash
cargo run
curl http://127.0.0.1:3000/fast
curl http://127.0.0.1:3000/slow
curl http://127.0.0.1:3001/metrics
```

```
# TYPE http_requests_duration_seconds histogram
http_requests_duration_seconds_bucket{method="GET",path="/fast",status="200",le="0.005"} 1
http_requests_duration_seconds_bucket{method="GET",path="/fast",status="200",le="0.01"} 1
...
http_requests_duration_seconds_bucket{method="GET",path="/fast",status="200",le="+Inf"} 1
http_requests_duration_seconds_sum{method="GET",path="/fast",status="200"} 0.000045
http_requests_duration_seconds_count{method="GET",path="/fast",status="200"} 1

http_requests_duration_seconds_bucket{method="GET",path="/slow",status="200",le="0.005"} 0
...
http_requests_duration_seconds_bucket{method="GET",path="/slow",status="200",le="0.5"} 0
http_requests_duration_seconds_bucket{method="GET",path="/slow",status="200",le="1"} 1
http_requests_duration_seconds_bucket{method="GET",path="/slow",status="200",le="2.5"} 1
...
```

**`# TYPE` 从 `summary` 变成了 `histogram`。**

**`/slow` 那组桶把 1 秒的延迟展示得清清楚楚**:`le="0.5"` 是 0,`le="1"` 是 1 —— 说明这次请求耗时在 0.5s 到 1s 之间。这就是分桶的价值。

**数一下 series 数量**:一条路由的直方图就有 **11 个桶 + `_sum` + `_count` = 13 个 series**。乘上 method 和 status 的组合……**这就是为什么标签基数必须控制住**。

### 步骤 5:故意改坏,亲眼看基数爆炸

**要做什么**:临时加一条带路径参数的路由,然后把 `MatchedPath` 换成原始路径,对比两者的输出。

**先临时加一条路由**(验证完删掉):

```rust
        .route("/user/{id}", get(|| async {}))    // 临时,验证完删掉
```

放在 `.route_layer(...)` **之前**——前置知识提过,`route_layer` 只对它之前注册的路由生效。

**第一轮:保持 `MatchedPath` 不变。**

```bash
cargo run
```

```bash
for i in $(seq 1 5); do curl -s "http://127.0.0.1:3000/user/$i"; done
curl -s http://127.0.0.1:3001/metrics | grep http_requests_total
```

```
http_requests_total{method="GET",path="/user/{id}",status="200"} 5
```

**5 次请求,1 个 series。**

**第二轮:临时把中间件改成用原始路径。**

```rust
    // 故意改坏
    let path = req.uri().path().to_owned();
```

```bash
cargo run
for i in $(seq 1 5); do curl -s "http://127.0.0.1:3000/user/$i"; done
curl -s http://127.0.0.1:3001/metrics | grep http_requests_total
```

```
http_requests_total{method="GET",path="/user/1",status="200"} 1
http_requests_total{method="GET",path="/user/2",status="200"} 1
http_requests_total{method="GET",path="/user/3",status="200"} 1
http_requests_total{method="GET",path="/user/4",status="200"} 1
http_requests_total{method="GET",path="/user/5",status="200"} 1
```

**5 次请求,5 个 series。**

再数一下直方图:

```bash
curl -s http://127.0.0.1:3001/metrics | grep -c http_requests_duration_seconds_bucket
```

第一轮是 **12**(11 个桶 + `+Inf`),第二轮是 **60**。

**把 5 换成你的真实用户量:**

| 用户数 | series 数(仅这一条路由) |
|---|---|
| 5 | 65 |
| 1 万 | 13 万 |
| 100 万 | **1300 万** |

**Prometheus 单实例通常能扛几百万 series,1300 万就崩了。** 而且这还只是一条路由。

**关键在于:这个 bug 不会报错。** 应用照常运行,指标照常上报,**是监控系统那边先撑不住**——而且往往是在业务量上来之后才爆,也就是最不希望它爆的时候。

**这类事故在真实系统里的常见面貌:**

- 把 `user_id` 或 `tenant_id` 当标签
- 把错误信息当标签(`error="connection refused to 10.0.0.7:5432"`)
- 把 URL 的 query string 带进标签
- 把请求 ID([第 19 章](19-request-id.md))当标签

**记住那条铁律:标签的取值集合必须有限且很小。想记录高基数信息,请用日志。**

**把中间件改回 `MatchedPath` 版本,把临时路由删掉。**

## 完整代码

`src/main.rs`:

```rust
use axum::{
    extract::{MatchedPath, Request},
    middleware::{self, Next},
    response::IntoResponse,
    routing::get,
    Router,
};
use metrics_exporter_prometheus::{Matcher, PrometheusBuilder, PrometheusHandle};
use std::{
    future::ready,
    time::{Duration, Instant},
};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

fn metrics_app() -> Router {
    let recorder_handle = setup_metrics_recorder();
    Router::new().route("/metrics", get(move || ready(recorder_handle.render())))
}

fn main_app() -> Router {
    Router::new()
        .route("/fast", get(|| async {}))
        .route(
            "/slow",
            get(|| async {
                tokio::time::sleep(Duration::from_secs(1)).await;
            }),
        )
        .route_layer(middleware::from_fn(track_metrics))
}

async fn start_main_server() {
    let app = main_app();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn start_metrics_server() {
    let app = metrics_app();

    // NOTE: expose metrics endpoint on a different port
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3001")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

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

    // The `/metrics` endpoint should not be publicly available. If behind a reverse proxy, this
    // can be achieved by rejecting requests to `/metrics`. In this example, a second server is
    // started on another port to expose `/metrics`.
    let (_main_server, _metrics_server) = tokio::join!(start_main_server(), start_metrics_server());
}

fn setup_metrics_recorder() -> PrometheusHandle {
    const EXPONENTIAL_SECONDS: &[f64] = &[
        0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0,
    ];

    let recorder_handle = PrometheusBuilder::new()
        .set_buckets_for_metric(
            Matcher::Full("http_requests_duration_seconds".to_string()),
            EXPONENTIAL_SECONDS,
        )
        .unwrap()
        .install_recorder()
        .unwrap();

    let upkeep_handle = recorder_handle.clone();
    tokio::spawn(async move {
        loop {
            tokio::time::sleep(Duration::from_secs(5)).await;
            upkeep_handle.run_upkeep();
        }
    });

    recorder_handle
}

async fn track_metrics(req: Request, next: Next) -> impl IntoResponse {
    let start = Instant::now();
    let path = if let Some(matched_path) = req.extensions().get::<MatchedPath>() {
        matched_path.as_str().to_owned()
    } else {
        req.uri().path().to_owned()
    };
    let method = req.method().clone();

    let response = next.run(req).await;

    let latency = start.elapsed().as_secs_f64();
    let status = response.status().as_u16().to_string();

    let labels = [
        ("method", method.to_string()),
        ("path", path),
        ("status", status),
    ];

    metrics::counter!("http_requests_total", &labels).increment(1);
    metrics::histogram!("http_requests_duration_seconds", &labels).record(latency);

    response
}
```

## 代价与边界

**`/metrics` 端点没有认证。** 只靠"绑在 127.0.0.1 上"保护。生产环境要么放在内网,要么加认证([第 30 章](30-jwt.md)那类),要么由反向代理拦截([第 43 章](43-reverse-proxy.md))。**它泄露的信息量比大多数人以为的大。**

**记录器的安装时机有竞态。** `setup_metrics_recorder()` 在 `metrics_app()` 里调用,而 `metrics_app()` 是在 `start_metrics_server()` 里调用的。`tokio::join!` 先轮询 `start_main_server()` —— **理论上业务服务可能在记录器安装之前就开始接受请求**,那些请求的指标会被静默丢弃(前置知识说过,没有记录器时 `counter!` 是空操作)。

> **更好的写法**:把 `setup_metrics_recorder()` 提到 `main` 里,在两个服务启动之前调用,handle 再传给 `metrics_app()`:
>
> ```rust
> let recorder_handle = setup_metrics_recorder();
> tokio::join!(start_main_server(), start_metrics_server(recorder_handle));
> ```
>
> 实际项目里这个窗口极短,但**"启动瞬间的指标丢失"在排查冷启动问题时恰恰最要命**。

**指标名不符合 Prometheus 官方约定。** `http_requests_duration_seconds` 里的 `requests` 应该是单数:官方推荐 **`http_request_duration_seconds`**(计数器用复数 `http_requests_total`,直方图描述的是"单次请求的时长"所以用单数)。不影响功能,但会让熟悉约定的人看着别扭,也可能和现成的 Grafana 面板对不上。

**没有 in-flight gauge。** 只有"总量"和"延迟分布",没有"**当前正在处理多少个请求**"。这个指标在排查"服务卡住了"时特别有用——请求数不涨、in-flight 一直高,说明请求全堵在某个地方。加起来很简单,见「动手练习」。

**handler panic 时不记录任何指标。** `next.run(req).await` 如果 panic,后面那几行**根本不会执行**。所以**panic 在指标里是完全隐形的**——错误率看起来一切正常,实际上有请求在崩。要覆盖这种情况需要 `CatchPanicLayer`(`tower-http`)把 panic 转成 500 响应,这样中间件才能记录到。

**计时不包含外层中间件的耗时。** `Instant::now()` 是在这个中间件里取的,所以更外层的中间件(压缩、TLS 之类)花的时间没算进来。想测"用户感知的完整耗时",这个中间件要放在最外层——但那样又读不到 `MatchedPath` 了。**两者有天然的张力**,实践中通常接受这个误差。

**`route_layer` 只对之前注册的路由生效。** 前置知识提过。步骤 5 特意把临时路由加在 `route_layer` 之前就是这个原因。**加在后面的话,那条路由的指标会静默缺失。**

**没有业务指标。** 本章只有 HTTP 层面的通用指标。真实系统还需要:数据库查询耗时、缓存命中率、队列深度、外部依赖调用的成功率、以及业务本身的计数(订单数、注册数)。**通用指标告诉你"系统有没有病",业务指标告诉你"生意好不好"。**

**没有 `/health` 端点。** 和指标不同,健康检查是给负载均衡器和 k8s 探针用的。通常需要两个:liveness(进程还活着吗)和 readiness(能接流量吗)。[第 45 章](45-graceful-shutdown.md#代价与边界)提过 readiness 在优雅停机里的作用。

**日志和指标的关联缺失。** 看到 P99 飙高,想找到具体是哪些请求慢——需要 exemplar(把 trace ID 附在指标样本上)或者靠时间窗口去日志里捞。本章两者是割裂的。

## 动手练习

**1. 加一个 in-flight gauge。**

提示:`metrics::gauge!("http_requests_in_flight").increment(1.0)` 放在 `next.run` 之前,`.decrement(1.0)` 放在之后。 **注意 gauge 不要带 `path` 标签**,否则又是基数问题(而且 in-flight 通常只需要全局值)。验证:同时发 5 个 `/slow` 请求,期间去看 `/metrics`,应该看到 5。

**2. 把记录器的安装提到 `main` 里。**

提示:按「代价与边界」的写法改。`start_metrics_server` 加一个 `PrometheusHandle` 参数。**这个改动同时让 `metrics_app()` 变成纯函数,更好测试。**

**3. 加上 `CatchPanicLayer`,让 panic 也能被统计。**

提示:`cargo add tower-http --features catch-panic`,在 `route_layer` 里把它套在 `track_metrics` **内层**(这样 panic 被转成 500 之后中间件才能读到状态码)。加一条 `/panic` 路由验证:请求它之后 `/metrics` 里应该出现 `status="500"` 的计数。**「代价与边界」说过 panic 在指标里是隐形的,这个练习把它变可见。**

**4. 把 `else` 分支改成固定标签,然后把 `route_layer` 换成 `layer`。**

提示:先按步骤 3 的「更好的写法」改成 `"<unmatched>"`,再换成 `layer`。验证:`curl http://127.0.0.1:3000/wp-admin` 若干个不同路径,确认指标里只多出**一条** `path="<unmatched>"`。**这个练习让你既拿到 404 的统计,又不会基数爆炸。**

**5.(进阶)接上真正的 Prometheus 和 Grafana。**

提示:用 Docker 起一个 Prometheus,`prometheus.yml` 里配 `scrape_configs` 指向 `host.docker.internal:3001`。然后在 Prometheus 的表达式浏览器里试:

```promql
rate(http_requests_total[1m])
histogram_quantile(0.99, rate(http_requests_duration_seconds_bucket[5m]))
```

**第二条就是"P99 延迟"** —— 前置知识说的"分位数在查询时才算"在这里变成了实感。再用 `ab` 或 `wrk` 压一压,看曲线动起来。

## 小结与下一章预告

指标和日志是**两个观测维度**:日志回答"这一次请求发生了什么",指标回答"整体上现在什么状况"。指标的关键特性是**在应用内部就聚合好**,所以内存占用固定、查询极快,代价是丢失单次细节。

Prometheus 是**拉模型**:应用只提供一个返回文本的 HTTP 端点,抓取由监控侧发起。这让应用侧的实现简单到只有一行 `recorder_handle.render()`。

四种指标类型里,**多实例部署几乎总该用 Histogram 而不是 Summary**,因为只有 Histogram 的桶能跨实例相加。`metrics-exporter-prometheus` **默认渲染成 Summary**,步骤 4 配了 `set_buckets_for_metric` 之后才变成真正的 Histogram——`# TYPE` 那一行的变化是最直接的证据。

**全章最重要的一条是基数控制。** 指标名 + 一组标签值 = 一个 series,而 series 数量直接决定 Prometheus 的内存。步骤 5 用 5 个用户 ID 演示了 `MatchedPath` 和原始路径的差别:1 个 series vs 5 个,换成 100 万用户就是 1300 万 series。**而且这个 bug 完全不报错,是监控系统先崩**。铁律是:**标签取值必须有限且很小;高基数信息属于日志,不属于指标。**

`route_layer` 而不是 `layer`,让扫描器打的 404 不进统计——这是同一条铁律的另一面。但要记住它**只对之前注册的路由生效**,加在后面的路由会静默缺失指标。

还有几条容易被忽略的边界:**`/metrics` 不该公开**(它是一份免费的侦察报告)、**panic 在指标里是隐形的**(需要 `CatchPanicLayer`)、**记录器安装时机有竞态**(冷启动那一瞬的指标会丢)、**没有 in-flight gauge**(排查"卡住了"时最有用的那个指标)。

服务现在能优雅起停、能被观测了。[第 48 章 versioning](48-versioning.md) 转向一个纯粹的 API 设计问题:**接口改了,老客户端怎么办?** 你会看到四种版本化策略的取舍、一个从路径里提取版本的自定义提取器,以及一个比示例做法更该被采用的方案——`Router::nest`,以及示例那个 `{version}` 通配写法会带来的一个**静默的兼容性事故**。
