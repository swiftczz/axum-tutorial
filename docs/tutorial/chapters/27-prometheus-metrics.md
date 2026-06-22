# 27. prometheus-metrics

对应示例：`examples/prometheus-metrics`

用 middleware 统计 HTTP 请求次数和耗时,通过 `/metrics` 暴露 Prometheus 格式指标。日志回答"某次请求发生了什么",指标回答"最近 5 分钟请求量/错误率/P95 耗时是多少"。

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

## src/main.rs

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
    axum::serve(listener, app).await;
}

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
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

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

请求几次业务接口:

````bash
curl http://127.0.0.1:3000/fast
curl http://127.0.0.1:3000/slow
curl http://127.0.0.1:3000/slow
````

查看指标:

````bash
curl http://127.0.0.1:3001/metrics
````

搜索:

````text
http_requests_total
http_requests_duration_seconds
````

能看到带 labels 的指标,如 `method="GET"`、`path="/slow"`、`status="200"`。

## 解读

### 主服务和指标服务分开

```text
主服务 3000:/fast、/slow
指标服务 3001:/metrics
```

为什么分开端口?指标接口通常不应公开给所有用户(可能含路径、状态码、流量、耗时等服务运行信息),单独端口更容易用网络规则/反向代理/容器配置保护。

### Prometheus exporter 安装

````rust
fn setup_metrics_recorder() -> PrometheusHandle {
    let recorder_handle = PrometheusBuilder::new()
        .set_buckets_for_metric(Matcher::Full("http_requests_duration_seconds".to_string()), EXPONENTIAL_SECONDS)
        .unwrap()
        .install_recorder()
        .unwrap();
    ...
}
````

`install_recorder()` 把它安装成全局 metrics recorder,之后代码里 `counter!`/`histogram!` 的数据都进入这个 recorder。`recorder_handle.render()` 把当前指标渲染成 Prometheus 文本。

### histogram buckets

````rust
const EXPONENTIAL_SECONDS: &[f64] = &[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0];
````

histogram 不只记平均值,而是把请求耗时放进不同区间(<=0.005 秒、<=0.01 秒...<=10 秒),后续可计算 P50/P90/P95/P99。这比平均值更适合后端性能分析——平均值会掩盖少量很慢的请求。

### middleware 统计

````rust
async fn track_metrics(req: Request, next: Next) -> impl IntoResponse {
    let start = Instant::now();              // 记录开始时间
    let path = ...;                          // 拿 MatchedPath
    let method = req.method().clone();
    let response = next.run(req).await;      // 调用 handler,必须等它返回才能拿状态码和耗时
    let latency = start.elapsed().as_secs_f64();
    let status = response.status().as_u16().to_string();
    ...
    metrics::counter!("http_requests_total", &labels).increment(1);
    metrics::histogram!("http_requests_duration_seconds", &labels).record(latency);
    response
}
````

### `MatchedPath` 避免标签爆炸

用 `MatchedPath` 而非真实 URI 做标签:`/users/1`/`/users/2`/`/users/999` 会聚合成 `/users/{id}`。直接用真实 URI 做 label 会让 Prometheus 指标数量失控。原因和第 22 章一样。

### counter vs histogram

- **counter**(`http_requests_total`):只能增加,记累计次数(接口被请求多少次、某状态码出现多少次)。
- **histogram**(`http_requests_duration_seconds`):记数值分布,这里记请求耗时,后续分析慢请求比例和分位数。

### labels 必须低基数

labels 是指标维度(`method="GET"`、`path="/slow"`、`status="200"`),可以按接口/方法/状态码拆开看。但**不能乱加**——不要把用户 ID、订单 ID、request id 放进 label,否则指标数量爆炸。适合做 label 的是低基数字段:method、path 模板、status。

## 常见问题

**为什么 `/metrics` 要单独端口?** 指标接口通常不应公开给普通用户,单独端口便于用网络规则/反向代理/容器配置保护。

**为什么不用 request id 当 label?** request id 每次都不同,放进 label 会产生海量时间序列,内存和查询都出问题。

**访问 `/metrics` 没看到业务指标?** 需要先访问业务接口,没请求 `/fast` 或 `/slow` 时相关指标还没被创建。

## 手写任务

按下面顺序敲:

1. 写 `main_app()` 提供 `/fast` 和 `/slow`。
2. 写 `track_metrics`,先只记 `Instant::now()` 和 `next.run(req).await`。
3. 补上 method、path、status、latency。
4. 调用 `metrics::counter!` 记请求次数。
5. 调用 `metrics::histogram!` 记请求耗时。
6. 写 `setup_metrics_recorder()` 安装 Prometheus recorder。
7. 写 `metrics_app()` 用 `/metrics` 返回 `render()`。
8. 用 `tokio::join!` 同时启动两个服务。

加深练习:

1. 新增 `/error` 返回 500,观察 `status="500"` 指标。
2. 新增 `/users/{id}`,确认 path label 是模板而非具体 id。
3. 修改 buckets,观察 `/slow` 落在哪些耗时区间。

## 小结

- 日志定位某次请求细节,指标观察整体流量/错误率/耗时分布,两者解决不同问题。
- 用 middleware 包住业务路由:handler 前记开始时间,handler 后记状态码和耗时,用 Prometheus exporter 暴露 `/metrics`。
- counter 记累计次数,histogram 记数值分布(用 buckets 算分位数);请求量用 counter,请求耗时用 histogram。
- **指标 label 必须低基数**:优先用 `MatchedPath`,不要把用户 ID/request id 放进 label。
- `/metrics` 放单独端口,便于保护。

## 源码对照

- `examples/prometheus-metrics/Cargo.toml`
- `examples/prometheus-metrics/src/main.rs`
