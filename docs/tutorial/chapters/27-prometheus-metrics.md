# 27. prometheus-metrics

对应示例：`examples/prometheus-metrics`

第 22 章讲了 tracing 日志，这章讲**指标（metrics）**——和日志互补的可观测性信号。日志记录单次请求细节，指标记录聚合数据（QPS、延迟分布、错误率），适合长期监控和告警。

本章用 `metrics` + `metrics-exporter-prometheus` 两个 crate，通过 axum middleware 给每个请求打点（请求总数、延迟分布），并在**独立端口**暴露 `/metrics` 端点供 Prometheus 拉取。

分 4 步：先装好 Prometheus recorder，再加 `/metrics` 端点，再加 track middleware，最后说明为何独立端口。

相比前面章节新引入：**`metrics` crate（counter/histogram 宏）、`PrometheusBuilder`、middleware 打点（`Instant`/`MatchedPath`）、独立 metrics 端口、`run_upkeep`**。

## Cargo.toml

````toml
[package]
name = "example-prometheus-metrics"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
metrics = { version = "0.24", default-features = false }
metrics-exporter-prometheus = { version = "0.18", default-features = false }
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：装 Prometheus recorder

`metrics` crate 只是 façade（接口），真正存储和导出指标要装一个 recorder。这步用 `PrometheusBuilder` 安装 Prometheus 格式的 recorder，返回一个 `PrometheusHandle` 用来导出指标。

````rust
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

    let handle = setup_metrics_recorder();
    println!("recorder installed: {}", handle.render().is_empty());
}
````

这步只装 recorder、不跑服务。运行后打印 "recorder installed: true"（指标为空所以渲染出空字符串）。

> **新面孔：`metrics` crate（façade 设计）**
>
> `metrics` crate 只提供 `counter!` / `histogram!` / `gauge!` 这些宏的**接口**，不存储数据。真正的存储和导出由你安装的 recorder 完成（这里用 `metrics-exporter-prometheus`）。
>
> 好处是：业务代码只调宏，不绑定具体后端；换后端（Prometheus、statsd、graphite）不用改业务代码。

> **新面孔：`PrometheusBuilder`**
>
> 安装 Prometheus 格式 recorder 的构造器：
> - `.set_buckets_for_metric(matcher, buckets)` 指定直方图的桶（延迟按哪些区间分箱）
> - `.install_recorder()` 装上 recorder，返回 `PrometheusHandle`（用来导出当前指标文本）
>
> `EXPONENTIAL_SECONDS` 是常见的指数桶（0.005s 到 10s），覆盖从快到慢的请求。桶设不好聚合数据失真，需根据真实延迟分布调整。

> **新面孔：`run_upkeep`**
>
> Prometheus 直方图内部用 lock-free 数据结构累积样本，需要定期"维护"（清理过期数据、聚合）。`run_upkeep()` 触发这个清理，例子用后台 task 每 5 秒跑一次。
>
> 不调 `run_upkeep` 短期没事，但长时间运行会内存膨胀。

---

## 第二步：暴露 `/metrics` 端点

recorder 装好后，需要把指标文本暴露成 HTTP 接口供 Prometheus 抓取。这步写一个独立的 `metrics_app`，绑定 `/metrics` 路由。

````rust
fn metrics_app() -> Router {
    let recorder_handle = setup_metrics_recorder();
    Router::new().route("/metrics", get(move || ready(recorder_handle.render())))
}

async fn start_metrics_server() {
    let app = metrics_app();

    // 注意：在另一个端口暴露 /metrics
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3001")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

# #[tokio::main]
# async fn main() {
#     // ...
    start_metrics_server().await;
# }
````

> **新面孔：`PrometheusHandle::render`**
>
> 返回当前所有指标的 Prometheus 文本格式字符串，类似：
>
> ```text
> # HELP http_requests_total Total HTTP requests
> http_requests_total{method="GET",path="/fast",status="200"} 42
> http_requests_duration_seconds_bucket{...,le="0.005"} 0
> http_requests_duration_seconds_bucket{...,le="0.01"} 5
> ...
> ```
>
> `/metrics` 路由就是 `get(|| ready(handle.render()))`——直接把字符串作为响应返回。`move` 捕获 handle 因为闭包需要拥有它。

验证：

````bash
cd examples
cargo run -p example-prometheus-metrics
````

````bash
curl http://127.0.0.1:3001/metrics
````

返回 Prometheus 文本（这时指标还没数据，因为还没打点）。下一步加打点。

---

## 第三步：track middleware 给每个请求打点

用 axum middleware 拦截每个请求，记录 method/path/status/latency。这步是 metrics 真正采集数据的地方。

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

async fn start_main_server() {
    let app = main_app();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
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
````

> **新面孔：`metrics::counter!` / `metrics::histogram!`**
>
> 两个核心打点宏：
> - `counter!("name", &labels).increment(1)`：计数器，只增不减（请求总数、错误总数）
> - `histogram!("name", &labels).record(value)`：直方图，记录数值分布（延迟分布、响应大小分布）
>
> `&labels` 是 `[("key", value), ...]` 切片，Prometheus 会按 label 维度聚合（如按 path 分别统计每个接口的 QPS）。

> **新面孔：middleware 打点模式**
>
> 打点 middleware 的典型结构：
> 1. **进 middleware 前**记录起始时间（`Instant::now()`）、提取 method/path（必须在 handler 前读，因为 `next.run(req)` 会消费 `req`）
> 2. **调用 handler**：`response = next.run(req).await`
> 3. **出 middleware 后**计算延迟、读响应状态、记录指标
>
> 注意 `MatchedPath` 从 `req.extensions()` 取——这是路由匹配后 axum 塞进去的扩展。`req.uri().path()` 是 fallback（没匹配到路由的情况，如 404）。用 `MatchedPath` 而非真实 URI 做标签：`/users/1`/`/users/2`/`/users/999` 会聚合成 `/users/{id}`，避免 label 爆炸。

> **新面孔：`route_layer` vs `layer`**
>
> `Router::route_layer` 给路由内的所有路由加 layer，**但不影响后续追加的路由**。这里给 `/fast` 和 `/slow` 加 track middleware。

验证：

````bash
curl http://127.0.0.1:3000/fast
curl http://127.0.0.1:3000/slow
curl http://127.0.0.1:3001/metrics   # 现在能看到 /fast 和 /slow 的计数和延迟
````

---

## 第四步：主服务和 metrics 服务并行跑

最终代码用 `tokio::join!` 并行跑两个服务：主服务在 3000 端口，metrics 服务在 3001 端口。

````rust
# async fn start_main_server() { /* ... */ }
# async fn start_metrics_server() { /* ... */ }

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(/* ... */)
        .with(tracing_subscriber::fmt::layer())
        .init();

    // /metrics 端点不应公开访问。反向代理可以拒绝 /metrics；这里用独立端口隔离
    let (_main_server, _metrics_server) = tokio::join!(start_main_server(), start_metrics_server());
}
````

> **新面孔：`tokio::join!` 并行跑多个 server**
>
> `tokio::join!(a, b)` 并发执行两个 future，等两个都完成。这里两个 server 都是无限循环，所以会一直跑。
>
> 为什么 `/metrics` 用独立端口？**安全**——指标数据可能包含敏感信息（QPS、路径分布），不应公开访问。生产环境常见做法：内部端口暴露 metrics，反向代理只放行公网端口；或限制访问源 IP。

---

## 完整代码

````rust
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
````

## 运行

````bash
cd examples
cargo run -p example-prometheus-metrics
````

发请求制造一些指标：

````bash
curl http://127.0.0.1:3000/fast
curl http://127.0.0.1:3000/slow
curl http://127.0.0.1:3000/fast
````

查看 Prometheus 格式输出：

````bash
curl http://127.0.0.1:3001/metrics
````

输出类似：

```text
http_requests_total{method="GET",path="/fast",status="200"} 2
http_requests_total{method="GET",path="/slow",status="200"} 1
http_requests_duration_seconds_bucket{...,le="0.005"} 2
http_requests_duration_seconds_bucket{...,le="+Inf"} 3
http_requests_duration_seconds_sum 1.0004
http_requests_duration_seconds_count 3
```

## 解读

### 为何 `/metrics` 独立端口

**安全**——指标数据可能包含敏感信息（QPS、路径分布），不应公开访问。生产做法：

- 独立内部端口暴露 `/metrics`（本例做法）
- 反向代理只放行公网端口，拒绝外部 `/metrics` 请求
- 限制 metrics 端口的访问源 IP

### 选桶是艺术

`EXPONENTIAL_SECONDS` 是常见的指数桶（0.005s 到 10s），覆盖从快到慢的请求。如果接口大多是 100ms 内，桶设太大（如 1s, 5s, 10s）就看不到差异；如果都是几秒级别，桶设太小（如 0.005s, 0.01s）全挤在最后一个桶。

## 常见问题

**`metrics` 和 `tracing` 啥关系？** 互补：`tracing` 记录单次事件细节（"请求来了，参数是 X，结果失败"），`metrics` 记录聚合数据（"过去 1 分钟 QPS=200，p99=850ms"）。日志适合调试，指标适合监控和告警。

**为什么 middleware 在 `next.run` 前后都要做事？** 前读请求（method/path）、记起始时间；后算延迟、读响应状态。handler 会消费 `req`，所以请求信息必须在 `next.run` 前提取。

**`run_upkeep` 必须吗？** 短期不调没事，但长时间运行内存会膨胀（直方图内部 lock-free 累积样本）。例子用后台 task 每 5 秒清理一次。

## 手写任务

1. 加一个 `/error` 路由返回 500，观察 `/metrics` 里 `status="500"` 的 counter。
2. 加一个 `gauge!` 指标记录当前活跃请求数（进 middleware +1，出 middleware -1）。
3. 把桶改成 `[0.1, 0.5, 1.0, 5.0]`，对比 `/fast` 和 `/slow` 的直方图分布。
4. 把 `track_metrics` 改成不记录 `path` label，理解高基数 label 的内存爆炸问题。

## 小结

这章用 4 步搭起 Prometheus 指标采集：

1. **装 recorder**：`PrometheusBuilder` 装 Prometheus 格式 recorder，配置直方图桶，启动 `run_upkeep` 后台清理。
2. **`/metrics` 端点**：`PrometheusHandle::render` 导出 Prometheus 文本，绑定独立端口（3001）。
3. **track middleware**：`counter!` 记请求总数、`histogram!` 记延迟分布，按 method/path/status 分维度。
4. **并行双服务**：`tokio::join!` 跑主服务和 metrics 服务，独立端口隔离 `/metrics`。

核心模式：**middleware 前读请求、后读响应**，打点宏 `counter!`/`histogram!` 由 `metrics` façade 转发给 recorder，业务代码和后端解耦。

## 源码对照

- `examples/prometheus-metrics/Cargo.toml`
- `examples/prometheus-metrics/src/main.rs`
