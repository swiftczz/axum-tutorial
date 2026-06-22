# 26. prometheus-metrics

对应示例：`examples/prometheus-metrics`

本章目标：用 middleware 统计 HTTP 请求次数和请求耗时，并通过 `/metrics` 暴露 Prometheus 格式的指标。

前面几章讲了日志、request id、CORS、压缩。  
这一章进入后端上线后的另一个基础能力：监控指标。

## 这个小项目在做什么

应用启动两个服务：

```text
主服务：http://127.0.0.1:3000
指标服务：http://127.0.0.1:3001/metrics
```

主服务有两个接口：

```text
GET /fast
GET /slow
```

`/fast` 立刻返回。  
`/slow` 会等待 1 秒再返回。

每次请求 `/fast` 或 `/slow` 时，middleware 会记录两类指标：

```text
http_requests_total
http_requests_duration_seconds
```

然后你可以访问：

```text
GET /metrics
```

看到 Prometheus 文本格式的指标。

请求主线是：

```text
客户端 GET /fast
-> track_metrics 记录开始时间、方法、路径
-> handler 执行业务
-> track_metrics 读取状态码和耗时
-> 写入 counter 和 histogram
-> 客户端访问 3001/metrics 查看指标
```

## 先理解指标和日志的区别

日志适合回答：

```text
某一次请求发生了什么？
为什么这个 request_id 报错？
```

指标适合回答：

```text
最近 5 分钟请求量是多少？
错误率是多少？
P95 耗时是多少？
/slow 比 /fast 慢多少？
```

指标不是一行行事件，而是一组可聚合的数字。  
比如：

```text
http_requests_total{method="GET",path="/fast",status="200"} 12
```

这表示：

```text
GET /fast 返回 200 的请求累计发生了 12 次
```

## Prometheus 暴露的是什么

Prometheus 通常会定时请求你的 `/metrics` 接口。  
这个接口返回的是文本，不是 JSON。

大概长这样：

```text
http_requests_total{method="GET",path="/fast",status="200"} 12
http_requests_duration_seconds_bucket{method="GET",path="/slow",status="200",le="1"} 3
```

Prometheus 把这些数据抓走后，才能在 Grafana 等工具里画图。

这一章不会搭建完整 Prometheus，只先学会：

```text
Axum 服务怎么产生 Prometheus 指标
```

## 文件和依赖

这个 example 有两个文件：

1. `examples/prometheus-metrics/Cargo.toml`：声明 Axum、metrics、metrics-exporter-prometheus、Tokio、tracing。
2. `examples/prometheus-metrics/src/main.rs`：实现主服务、指标服务、Prometheus recorder 和统计 middleware。

关键依赖：

- `axum`：提供 Router、middleware、`MatchedPath`。
- `metrics`：提供 `counter!` 和 `histogram!` 宏。
- `metrics-exporter-prometheus`：把 metrics 数据导出成 Prometheus 文本格式。
- `tokio`：同时运行主服务和指标服务。
- `tracing` / `tracing-subscriber`：输出启动日志。

## 第一步：主服务和指标服务分开

源码：

````rust
let (_main_server, _metrics_server) = tokio::join!(start_main_server(), start_metrics_server());
````

这个 example 同时启动两个服务：

- `start_main_server()`：监听 3000，提供业务接口。
- `start_metrics_server()`：监听 3001，提供 `/metrics`。

为什么把 `/metrics` 放到另一个端口？

因为指标接口通常不应该公开给所有用户。  
里面可能包含路径、状态码、流量、耗时等服务运行信息。

真实项目中可以通过内网端口、防火墙、反向代理规则等方式保护 `/metrics`。

## 第二步：主服务只挂业务路由和统计 middleware

源码：

````rust
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
````

主服务有两个路由：

```text
/fast -> 立刻返回
/slow -> sleep 1 秒后返回
```

`route_layer(middleware::from_fn(track_metrics))` 表示给这些路由加统计 middleware。

这里用的是 `route_layer`，不是 `layer`。  
可以先简单理解成：

```text
route_layer 更贴近已经匹配到的路由
```

这样 middleware 里更容易拿到 `MatchedPath`。

## 第三步：指标服务暴露 /metrics

源码：

````rust
fn metrics_app() -> Router {
    let recorder_handle = setup_metrics_recorder();
    Router::new().route("/metrics", get(move || ready(recorder_handle.render())))
}
````

`setup_metrics_recorder()` 会安装 Prometheus recorder，并返回一个 handle。  
这个 handle 可以把当前指标渲染成 Prometheus 文本：

````rust
recorder_handle.render()
````

`move || ready(...)` 是一个很小的 handler。  
它的意思是：

```text
收到 GET /metrics 时，立刻返回当前指标文本
```

这里用了 `std::future::ready`，因为 `render()` 本身不需要等待异步 IO。

## 第四步：安装 Prometheus recorder

源码：

````rust
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

    ...

    recorder_handle
}
````

`PrometheusBuilder::new()` 创建 Prometheus exporter。  
`install_recorder()` 把它安装成全局 metrics recorder。

安装之后，代码里调用：

````rust
metrics::counter!(...)
metrics::histogram!(...)
````

这些数据都会进入这个 recorder。

## 第五步：为耗时指标设置 buckets

源码：

````rust
const EXPONENTIAL_SECONDS: &[f64] = &[
    0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0,
];
````

histogram 不是只记录一个平均值。  
它会把请求耗时放进不同区间。

例如：

```text
<= 0.005 秒
<= 0.01 秒
<= 0.025 秒
...
<= 10 秒
```

这样后续可以计算：

```text
P50
P90
P95
P99
```

这比只看平均值更适合后端性能分析。  
因为平均值可能掩盖少量很慢的请求。

## 第六步：定期 run_upkeep

源码：

````rust
let upkeep_handle = recorder_handle.clone();
tokio::spawn(async move {
    loop {
        tokio::time::sleep(Duration::from_secs(5)).await;
        upkeep_handle.run_upkeep();
    }
});
````

这段启动了一个后台任务。  
每 5 秒调用一次：

````rust
run_upkeep()
````

它用于维护 exporter 内部状态。  
你可以先把它理解成 Prometheus exporter 的定期清理和维护工作。

## 第七步：track_metrics 记录开始时间

源码：

````rust
async fn track_metrics(req: Request, next: Next) -> impl IntoResponse {
    let start = Instant::now();
    ...
}
````

middleware 一开始先记录时间：

````rust
let start = Instant::now();
````

等后面的 handler 执行完，再计算：

````rust
let latency = start.elapsed().as_secs_f64();
````

这样得到的就是一次请求在服务端处理的耗时，单位是秒。

## 第八步：使用 MatchedPath 作为 path 标签

源码：

````rust
let path = if let Some(matched_path) = req.extensions().get::<MatchedPath>() {
    matched_path.as_str().to_owned()
} else {
    req.uri().path().to_owned()
};
````

这里优先使用 `MatchedPath`。  
原因和第 21 章一样：它能避免指标标签爆炸。

假设有路由：

```text
/users/{id}
```

真实请求可能是：

```text
/users/1
/users/2
/users/999
```

如果直接用真实 URI 做指标标签，Prometheus 会看到大量不同 path。  
这会让指标数量失控。

用 `MatchedPath` 时，标签会统一成：

```text
/users/{id}
```

这对监控系统非常重要。

## 第九步：调用后续 handler

源码：

````rust
let response = next.run(req).await;
````

这表示：

```text
把请求交给后面的路由 handler
```

必须等 handler 返回后，middleware 才能知道：

- 状态码是多少。
- 总耗时是多少。

所以指标记录发生在 `next.run(req).await` 之后。

## 第十步：写入 counter 和 histogram

源码：

````rust
let latency = start.elapsed().as_secs_f64();
let status = response.status().as_u16().to_string();

let labels = [
    ("method", method.to_string()),
    ("path", path),
    ("status", status),
];

metrics::counter!("http_requests_total", &labels).increment(1);
metrics::histogram!("http_requests_duration_seconds", &labels).record(latency);
````

这里记录两类指标。

### counter

````rust
metrics::counter!("http_requests_total", &labels).increment(1);
````

counter 只能增加。  
它适合记录累计次数。

例如：

```text
某个接口被请求了多少次
某个状态码出现了多少次
```

### histogram

````rust
metrics::histogram!("http_requests_duration_seconds", &labels).record(latency);
````

histogram 适合记录分布。  
这里记录请求耗时，后续可以分析慢请求比例和分位数。

### labels

labels 是指标的维度：

```text
method="GET"
path="/slow"
status="200"
```

通过 labels，你可以按接口、方法、状态码拆开看。

但 labels 不能乱加。  
不要把用户 ID、订单 ID、request id 放进 label，否则指标数量会失控。

## 函数职责速查

- `main`：初始化日志，同时启动主服务和指标服务。
- `start_main_server`：启动业务服务，监听 3000。
- `start_metrics_server`：启动指标服务，监听 3001。
- `main_app`：注册 `/fast`、`/slow`，并挂载统计 middleware。
- `metrics_app`：安装 Prometheus recorder，并暴露 `/metrics`。
- `setup_metrics_recorder`：配置 histogram buckets，安装 recorder，启动 upkeep 后台任务。
- `track_metrics`：统计单次请求的 method、path、status 和 latency。

## 带中文注释的手写版

````rust
//! 这个 example 演示如何手写 metrics middleware。
//!
//! Run with
//!
//! ```not_rust
//! cargo run -p example-prometheus-metrics
//! ```

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

// 指标服务。
// 它只负责暴露 /metrics。
fn metrics_app() -> Router {
    // 安装 Prometheus recorder，并拿到可以 render 指标的 handle。
    let recorder_handle = setup_metrics_recorder();

    // GET /metrics 时，返回 Prometheus 文本格式的指标。
    Router::new().route("/metrics", get(move || ready(recorder_handle.render())))
}

// 主业务服务。
fn main_app() -> Router {
    Router::new()
        // 一个很快的接口。
        .route("/fast", get(|| async {}))
        // 一个故意等待 1 秒的慢接口。
        .route(
            "/slow",
            get(|| async {
                tokio::time::sleep(Duration::from_secs(1)).await;
            }),
        )
        // 给业务路由加统计 middleware。
        .route_layer(middleware::from_fn(track_metrics))
}

// 启动主服务，监听 3000。
async fn start_main_server() {
    let app = main_app();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}

// 启动指标服务，监听 3001。
// 真实项目中，/metrics 通常不应该公开给外部用户。
async fn start_metrics_server() {
    let app = metrics_app();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3001")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}

#[tokio::main]
async fn main() {
    // 初始化日志。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 同时启动主服务和指标服务。
    let (_main_server, _metrics_server) = tokio::join!(start_main_server(), start_metrics_server());
}

// 配置并安装 Prometheus recorder。
fn setup_metrics_recorder() -> PrometheusHandle {
    // histogram 的耗时桶，单位是秒。
    const EXPONENTIAL_SECONDS: &[f64] = &[
        0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0,
    ];

    // 创建 Prometheus exporter，并给请求耗时指标设置 buckets。
    let recorder_handle = PrometheusBuilder::new()
        .set_buckets_for_metric(
            Matcher::Full("http_requests_duration_seconds".to_string()),
            EXPONENTIAL_SECONDS,
        )
        .unwrap()
        .install_recorder()
        .unwrap();

    // 定期维护 exporter 内部状态。
    let upkeep_handle = recorder_handle.clone();
    tokio::spawn(async move {
        loop {
            tokio::time::sleep(Duration::from_secs(5)).await;
            upkeep_handle.run_upkeep();
        }
    });

    recorder_handle
}

// 统计每次 HTTP 请求的 middleware。
async fn track_metrics(req: Request, next: Next) -> impl IntoResponse {
    // 记录请求开始时间。
    let start = Instant::now();

    // 优先使用匹配到的路由模板，避免 /users/1、/users/2 这种真实路径造成标签爆炸。
    let path = if let Some(matched_path) = req.extensions().get::<MatchedPath>() {
        matched_path.as_str().to_owned()
    } else {
        req.uri().path().to_owned()
    };

    // 记录 HTTP 方法。
    let method = req.method().clone();

    // 调用后续 handler。
    let response = next.run(req).await;

    // handler 返回后，才能知道耗时和状态码。
    let latency = start.elapsed().as_secs_f64();
    let status = response.status().as_u16().to_string();

    // 指标标签。注意不要把用户 ID、订单 ID、request id 这种高基数字段放进 label。
    let labels = [
        ("method", method.to_string()),
        ("path", path),
        ("status", status),
    ];

    // 累计请求次数。
    metrics::counter!("http_requests_total", &labels).increment(1);

    // 记录请求耗时分布，单位是秒。
    metrics::histogram!("http_requests_duration_seconds", &labels).record(latency);

    // 把原始响应继续返回给客户端。
    response
}
````

## 运行和验证

运行：

````bash
cargo run -p example-prometheus-metrics
````

请求几次业务接口：

````bash
curl http://127.0.0.1:3000/fast
curl http://127.0.0.1:3000/slow
curl http://127.0.0.1:3000/slow
````

查看指标：

````bash
curl http://127.0.0.1:3001/metrics
````

搜索这些名字：

```text
http_requests_total
http_requests_duration_seconds
```

你应该能看到带 labels 的指标，例如：

```text
method="GET"
path="/slow"
status="200"
```

## 常见卡点

### 1. 为什么 /metrics 要单独端口？

因为指标接口通常不应该公开给普通用户。  
单独端口更容易用网络规则、反向代理或容器配置保护起来。

### 2. 为什么不用 request id 当 label？

因为 request id 几乎每次请求都不同。  
如果放进 label，Prometheus 会产生海量时间序列，内存和查询都会出问题。

适合做 label 的是低基数字段：

```text
method
path 模板
status
```

### 3. 为什么 path 要用 MatchedPath？

为了避免路径标签爆炸。  
`/users/1`、`/users/2` 应该聚合成 `/users/{id}`。

### 4. counter 和 histogram 有什么区别？

counter 记录累计次数。  
histogram 记录数值分布。

请求总量适合 counter，请求耗时适合 histogram。

### 5. 为什么访问 /metrics 没看到业务指标？

你需要先访问业务接口。  
没有请求 `/fast` 或 `/slow` 时，相关指标可能还没有被创建。

## 手写任务

建议按下面顺序自己敲一遍：

1. 先写 `main_app()`，提供 `/fast` 和 `/slow`。
2. 写 `track_metrics`，先只记录 `Instant::now()` 和 `next.run(req).await`。
3. 补上 method、path、status、latency。
4. 调用 `metrics::counter!` 记录请求次数。
5. 调用 `metrics::histogram!` 记录请求耗时。
6. 写 `setup_metrics_recorder()`，安装 Prometheus recorder。
7. 写 `metrics_app()`，用 `/metrics` 返回 `recorder_handle.render()`。
8. 用 `tokio::join!` 同时启动两个服务。

加深练习：

1. 新增 `/error`，返回 500，观察 `status="500"` 指标。
2. 新增 `/users/{id}`，确认 path label 是模板而不是具体 id。
3. 修改 buckets，观察 `/slow` 落在哪些耗时区间。

## 本章真正要记住什么

后端监控指标不是日志的替代品。  
它们解决的是不同问题：

```text
日志：定位某一次请求的细节
指标：观察整体流量、错误率和耗时分布
```

Axum 中最常见的做法是：

```text
用 middleware 包住业务路由
-> 在 handler 前记录开始时间
-> 在 handler 后记录状态码和耗时
-> 用 Prometheus exporter 暴露 /metrics
```

另外一定要记住：指标 label 要低基数，优先使用 `MatchedPath`。

## 源码对照

本章手写版对应源码：

- `examples/prometheus-metrics/src/main.rs`
- `examples/prometheus-metrics/Cargo.toml`
