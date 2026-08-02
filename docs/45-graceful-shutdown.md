# 第 45 章 graceful-shutdown:让进程体面地退出

## 本章你将做出什么

一个**退出时会先把手头活干完**的服务:

```
GET /slow      睡 5 秒再返回
GET /forever   永远不返回
```

发起一个 `/slow` 请求,然后立刻在服务端按 Ctrl-C —— **服务不会立刻死掉**,它会等这个请求返回 200 之后才退出。

再发起一个 `/forever`,按 Ctrl-C —— 服务等 10 秒,给客户端一个 `408 Request Timeout`,然后退出。**不会无限期地等下去。**

学完本章你将掌握:

- **为什么"直接 Ctrl-C"在生产环境是不可接受的**
- SIGINT / SIGTERM / SIGKILL 的区别,以及 Docker 和 Kubernetes 是怎么停容器的
- `tokio::signal` 与 `#[cfg(unix)]` 条件编译
- `tokio::select!`:两个信号谁先来算谁
- `axum::serve(...).with_graceful_shutdown(...)` 具体做了什么
- **为什么优雅停机必须配一个硬超时**(不配会怎样,本章会亲手演示)
- `Router::layer` 接受**元组**:一次叠多层中间件

## 前置知识

### 为什么需要优雅停机

进程退出这件事,在生产环境里**每天要发生很多次**:

- 发版(滚动更新,每个实例都要被停掉重启)
- 自动扩缩容(缩容时随机挑几个实例杀掉)
- 节点维护、Pod 驱逐、机器重启
- 崩溃后的自动重启

如果进程是**被粗暴杀掉**的,后果是:

| 现象 | 用户看到什么 |
|---|---|
| 在途请求被切断 | 连接重置、`502 Bad Gateway`、空响应 |
| 数据库事务写到一半 | 数据不一致(如果没用事务的话) |
| 消息队列的消息取了没 ack | 消息被重投,或者丢失 |
| 文件写到一半 | 残缺文件([第 40 章](40-stream-to-file.md#代价与边界)讲过) |

**发一次版就有一小撮用户报错**,这在有 SLA 的系统里是不可接受的。

优雅停机要做的事很简单:**收到"该走了"的信号后,不再接新活,把手头的活干完,然后走。**

### 三个信号

Unix 用信号(signal)通知进程。和停机有关的是这三个:

| 信号 | 编号 | 谁发 | 能不能捕获 |
|---|---|---|---|
| **SIGINT** | 2 | 终端按 **Ctrl-C** |  能 |
| **SIGTERM** | 15 | `kill`、`docker stop`、k8s 停 Pod |  能 |
| **SIGKILL** | 9 | `kill -9`、宽限期超时后的系统 |  **不能** |

**SIGKILL 无法捕获,也无法忽略** —— 内核直接把进程干掉,没有任何机会做清理。所以优雅停机的全部功夫都花在**赶在 SIGKILL 之前把事情做完**。

**开发时按 Ctrl-C 发的是 SIGINT,生产环境收到的几乎都是 SIGTERM。** 两个都要处理——这就是本章 `shutdown_signal` 里 `select!` 两个分支的原因。

### Docker 和 Kubernetes 是怎么停容器的

理解这个流程,才知道超时该设多少:

```
docker stop:
    发 SIGTERM  →  等 10 秒(默认)  →  还没退出就发 SIGKILL

kubectl delete pod:
    从 Service 的 Endpoints 里摘除(异步!)
    执行 preStop hook(如果配了)
    发 SIGTERM  →  等 terminationGracePeriodSeconds(默认 30 秒)  →  SIGKILL
```

**关键结论:应用的停机超时必须小于平台的宽限期。** 本章用 10 秒,正好卡在 Docker 的默认值上——官方示例的注释里写的就是这个理由。如果应用要等 30 秒而 Docker 只给 10 秒,那后 20 秒的等待毫无意义,进程照样被 SIGKILL。

 上面 k8s 流程里那个 **"异步"** 三个字非常要命,「代价与边界」会专门说。

### `tokio::signal`

tokio 把信号处理封装成了 future:

```rust
tokio::signal::ctrl_c().await          // 等 SIGINT(跨平台)
```

```rust
tokio::signal::unix::signal(SignalKind::terminate())?  // 得到一个 SIGTERM 的流
    .recv().await
```

**注意两者的形态不同**:`ctrl_c()` 是一次性的 future,`unix::signal()` 返回的是一个可以反复 `recv()` 的**流**(信号可以来很多次)。

**`ctrl_c()` 跨平台,`unix::signal()` 只有 Unix 有。** Windows 没有 SIGTERM 概念,所以要用条件编译:

```rust
#[cfg(unix)]
let terminate = async { /* 等 SIGTERM */ };

#[cfg(not(unix))]
let terminate = std::future::pending::<()>();
```

`std::future::pending::<()>()` 是一个**永远不会完成**的 future。在 Windows 上让 `terminate` 分支永远挂着,`select!` 就只会被 `ctrl_c` 唤醒。**用"永不完成"来表达"这个平台上没有这回事",比写两套 `select!` 干净得多。**

### `tokio::select!`:谁先来算谁

[第 34 章](34-websockets.md#tokioselect谁先结束就收拾谁)用过 `select!` 同时等收发两个方向。这里的用法更简单:

```rust
tokio::select! {
    _ = ctrl_c => {},
    _ = terminate => {},
}
```

**两个 future 并发地等,任意一个完成就整体返回**,另一个被丢弃。语义就是"SIGINT 或 SIGTERM,来哪个都算"。

### "优雅"具体指哪几件事

`axum::serve(...).with_graceful_shutdown(signal)` 在 `signal` 完成后做三件事:

1. **停止 accept 新连接** —— 监听套接字关闭,新来的连接会被拒绝(或者由内核 backlog 挂着直到超时)
2. **通知每条已有连接进入 graceful 状态** —— 当前请求处理完就关闭,不再复用(HTTP/1.1 keep-alive 的下一个请求不会被处理)
3. **等所有连接关闭** —— 全关完了,`serve` 的 future 才 resolve,`main` 才返回

**空闲的 keep-alive 连接会立刻关闭**,只有正在处理请求的连接会被等。

 **注意第 3 步:它会一直等。** 如果有一个请求永远不返回(比如本章的 `/forever`,或者一条 WebSocket 长连接),**`serve` 永远不会 resolve,进程永远不退出**。这就是必须配超时的原因,步骤 4 会亲手演示。

### `Router::layer` 可以接受元组

前面章节叠中间件用的是链式调用或 `ServiceBuilder`([第 17 章](17-tracing-aka-logging.md#洋葱结构与包装顺序))。axum 还支持直接传一个**元组**:

```rust
.layer((
    TraceLayer::new_for_http(),
    TimeoutLayer::new(Duration::from_secs(10)),
))
```

**元组的顺序语义和 `ServiceBuilder` 一致:左边的在外层。** 所以这里是:

```
请求 →  TraceLayer  →  TimeoutLayer  →  handler
       (最外层)                        (最内层)
响应 ←              ←                ←
```

**Trace 在外面很重要**:超时产生的 408 响应是 `TimeoutLayer` 生成的,只有 Trace 在它外面才能记录到这个 408。反过来的话,日志里会看到一个"没有响应"的请求。[第 19 章](19-request-id.md)分析过同类的顺序问题。

元组最多支持 16 层,比链式 `.layer().layer()` 少写几个字,而且**顺序直觉和 `ServiceBuilder` 一致**(链式 `.layer()` 是反的,后加的在外层——[第 17 章](17-tracing-aka-logging.md)对比过)。

## 项目搭建

```bash
cargo new graceful-shutdown
cd graceful-shutdown
```

```bash
cargo add axum --features tracing
cargo add tokio --features full
cargo add tower-http --features timeout,trace
cargo add tracing-subscriber --features env-filter
```

**两个和以往不同的地方:**

**① `axum --features tracing`** —— 打开这个 feature 后,axum 自己会发出 tracing 事件(路由匹配、提取器拒绝等)。配合下面日志过滤里的 `axum=trace`,能看到 axum 内部的动作。

**② 没有 `cargo add tracing`** —— 本章的代码里没有任何 `tracing::debug!` 之类的宏调用,日志全部来自 `TraceLayer` 和 axum 自己。所以只需要 `tracing-subscriber`(订阅器),不需要 `tracing`(门面)。

## 分步构建

### 步骤 1:两个"慢"路由,先看看粗暴退出长什么样

**要做什么**:先写出服务和两个故意很慢的 handler,**不加**优雅停机,亲眼看看直接 Ctrl-C 的后果。

```rust
// src/main.rs
use std::time::Duration;

use axum::{routing::get, Router};
use tokio::net::TcpListener;
use tokio::time::sleep;
use tower_http::trace::TraceLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // Enable tracing.
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!(
                    "{}=debug,tower_http=debug,axum=trace",
                    env!("CARGO_CRATE_NAME")
                )
                .into()
            }),
        )
        .with(tracing_subscriber::fmt::layer().without_time())
        .init();

    // Create a regular axum app.
    let app = Router::new()
        .route("/slow", get(|| sleep(Duration::from_secs(5))))
        .route("/forever", get(std::future::pending::<()>))
        .layer(TraceLayer::new_for_http());

    // Create a `TcpListener` using tokio.
    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}
```

**两个 handler 都不是普通写法,值得说一下:**

```rust
get(|| sleep(Duration::from_secs(5)))
```

这个闭包**没有 `async`** —— 它直接返回 `sleep()` 产生的 future。[第 01 章](01-hello-world.md)讲过 `Handler` 要求的是"返回一个 future 的函数",`async` 闭包只是产生 future 的一种方式。`sleep` 的输出是 `()`,而 **`()` 实现了 `IntoResponse`**(生成一个空 body 的 200),所以这个 handler 合法。

```rust
get(std::future::pending::<()>)
```

更极端:**直接把 `pending` 这个函数本身当 handler**。`pending::<()>` 是一个不接受参数、返回 `Pending<()>` 的函数,正好满足 `Handler` 的形状。它产生的 future 永远不完成,所以这个路由**永远不返回响应**。

前置知识里 `#[cfg(not(unix))]` 分支用的是同一个 `pending`。**"永不完成的 future"是异步编程里一个很有用的原语**,这一章用了两次,用途完全不同。

**`.without_time()`** 去掉了日志里的时间戳。理由是:生产环境里日志通常被 systemd / Docker / k8s 收走,**收集方自己会打时间戳**,应用再打一份就是重复。

**验证(粗暴退出)**:

```bash
cargo run
```

另开一个终端:

```bash
curl -i http://localhost:3000/slow
```

**在这 5 秒之内**,回到服务端终端按 **Ctrl-C**。

curl 那边:

```
curl: (52) Empty reply from server
```

服务端日志里只有 `on_request`,**没有 `on_response`** —— 请求被从中间切断了。

**这就是要解决的问题。** 想象这是一个正在处理支付的请求。

### 步骤 2:写出信号处理函数

**要做什么**:实现 `shutdown_signal`,同时监听 SIGINT 和 SIGTERM。

```rust
use tokio::signal;
```

```rust
async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install signal handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
```

**这段代码几乎可以原样抄进任何 axum 项目**,它是社区里的标准写法。三点说明:

**① 两个 `let` 绑定的是 future,不是结果。** `async { ... }` 块只是**创建** future,里面的 `.await` 要等到这个 future 被轮询时才执行。所以写到 `select!` 之前不会阻塞。

**② `expect` 用在这里是合适的。** 装信号处理器失败意味着系统层面出了大问题(通常是权限或平台不支持),这时候**启动失败比带着"停不下来"的隐患跑起来更好**。这是少数几个 panic 比容错更合理的场景。

**③ 这个函数本身不做任何清理。** 它只负责"等到信号来",然后返回。真正的停机逻辑在 `with_graceful_shutdown` 里。**职责分离得很干净**——所以它能被复用给多个服务器([第 46 章](46-tls-graceful-shutdown.md)就会这么干)。

这一步只是加了一个函数,还没接上,`cargo run` 行为不变(会有一个 unused 警告)。

### 步骤 3:接上优雅停机

**要做什么**:一行调用,把信号和服务器连起来。

```rust
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
```

**验证**:

```bash
cargo run
```

```bash
curl -i http://localhost:3000/slow
```

**5 秒内**按 Ctrl-C。

**这次的现象完全不同:**

服务端**没有立刻退出**,而是停在那里。5 秒后:

curl 收到:

```
HTTP/1.1 200 OK
content-length: 0
```

**请求完整地完成了。** 然后服务端进程才退出。

服务端日志:

```
DEBUG request{method=GET uri=/slow version=HTTP/1.1}: tower_http::trace::on_request: started processing request
^C
DEBUG request{method=GET uri=/slow version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=5001 ms status=200
```

**`on_response` 出现在 `^C` 之后** —— 这就是"优雅"的全部含义。

再验证一下"不接新连接":Ctrl-C 之后**立刻**在另一个终端:

```bash
curl -i http://localhost:3000/slow
```

```
curl: (7) Failed to connect to localhost port 3000: Connection refused
```

**监听已经关了,新请求进不来,老请求还在跑。** 这正是滚动更新时想要的行为——负载均衡器发现连不上就把流量切给别的实例。

没有在途请求时按 Ctrl-C,进程**立刻退出**。

### 步骤 4:故意不加超时,看看会卡成什么样

**要做什么**:用 `/forever` 触发"永远等下去"的场景。**这一步不改代码**,当前代码就有这个问题。

```bash
cargo run
```

```bash
curl -i http://localhost:3000/forever
```

curl 挂着(意料之中,handler 永不返回)。现在在服务端按 **Ctrl-C**。

**现象:什么都没发生。** 进程还在,curl 还挂着。

**再按一次 Ctrl-C —— 还是什么都没发生。**

**这是本章最重要的一个坑,而且有两层:**

**第一层:`serve` 在等那条永不结束的连接。** 前置知识说过,优雅停机的第 3 步是"等所有连接关闭"。`/forever` 的连接永远不会关闭,所以这个等待永远不会结束。

**第二层(更阴险):第二次 Ctrl-C 也没用。** 因为 `tokio::signal::ctrl_c()` 一旦被调用,就**替换掉了进程的默认 SIGINT 行为**。默认行为是"终止进程",现在变成了"送进 tokio 的信号通道"。而我们的 `select!` 已经完成并被丢弃,**没有人再去读这个通道**,于是信号被静默吞掉。

**你唯一的出路是另开终端 `kill -9`:**

```bash
pkill -9 graceful-shutdown
```

**在生产环境,这个场景的表现是:发版卡住,直到平台的宽限期用完,所有实例被 SIGKILL —— 也就是说,你费劲加的优雅停机完全没起作用,而且还让发版慢了 10~30 秒。**

这是典型的静默失败:**没有错误、没有日志、只是不动了**。

### 步骤 5:加上超时,给等待封顶

**要做什么**:用 `TimeoutLayer` 给每个请求设一个上限,这样"等所有请求完成"就一定会结束。

```rust
use tower_http::timeout::TimeoutLayer;
```

```rust
    let app = Router::new()
        .route("/slow", get(|| sleep(Duration::from_secs(5))))
        .route("/forever", get(std::future::pending::<()>))
        .layer((
            TraceLayer::new_for_http(),
            // Graceful shutdown will wait for outstanding requests to complete. Add a timeout so
            // requests don't hang forever.
            TimeoutLayer::new(Duration::from_secs(10)),
        ));
```

**`.layer(( A, B ))` 传的是元组** —— 前置知识讲过,左边在外层。这里 `TraceLayer` 在外、`TimeoutLayer` 在内,所以超时产生的 408 会被日志记录到。

**`TimeoutLayer::new` 超时后返回的就是 `408 Request Timeout`**,这是 `tower-http` 的默认行为,不需要额外配置。

> 官方示例写的是 `TimeoutLayer::with_status_code(StatusCode::REQUEST_TIMEOUT, Duration::from_secs(10))`,显式指定了状态码。这个构造函数是 `tower-http` 较新版本才有的;用 crates.io 上的 `TimeoutLayer::new(...)` 结果完全一样,因为默认状态码就是 408。如果你想返回别的状态码(比如 `503 Service Unavailable`),再去查你所用版本是否支持定制。

**验证**:

```bash
cargo run
```

```bash
curl -i http://localhost:3000/forever
```

不用等,直接在服务端按 **Ctrl-C**。

**10 秒后**:

curl 收到:

```
HTTP/1.1 408 Request Timeout
content-length: 0
```

服务端日志:

```
DEBUG request{method=GET uri=/forever ...}: tower_http::trace::on_response: finished processing request latency=10002 ms status=408
```

**然后进程正常退出。** 问题解决。

**顺带验证一件事:超时是对每个请求生效的,和停机无关。** 不按 Ctrl-C,单独 `curl -i http://localhost:3000/forever`,10 秒后一样会拿到 408。

**这一点很重要:`TimeoutLayer` 不是"停机超时",而是"请求超时"。** 它恰好也解决了停机问题——因为停机等的就是这些请求。

 **10 秒这个值不是随便选的。** 前置知识说过,Docker 的默认宽限期就是 10 秒。设成 10 秒意味着:最坏情况下应用刚好在被 SIGKILL 的那一刻完成清理。实践中应该**设得比平台宽限期小一些**(比如平台 30 秒、应用 20 秒),留出余量。

### 步骤 6:验证 SIGTERM(生产环境真正会收到的信号)

**要做什么**:别只测 Ctrl-C。生产环境发的是 SIGTERM,得确认那条分支也工作。

```bash
cargo run
```

另一个终端:

```bash
curl -i http://localhost:3000/slow &
sleep 1
pkill -TERM graceful-shutdown
```

**现象和 Ctrl-C 完全一致**:进程等 4 秒,curl 拿到 200,然后进程退出。

用 `kill -TERM <pid>` 也一样。**`docker stop` 和 k8s 停 Pod 走的就是这条路。**

对比一下粗暴的:

```bash
curl -i http://localhost:3000/slow &
sleep 1
pkill -KILL graceful-shutdown
```

```
curl: (52) Empty reply from server
```

**SIGKILL 无法捕获,优雅停机在它面前毫无办法。** 这也再次说明为什么应用的超时必须小于平台的宽限期——超时了就轮到 SIGKILL 了。

## 完整代码

`src/main.rs`:

```rust
use std::time::Duration;

use axum::{routing::get, Router};
use tokio::net::TcpListener;
use tokio::signal;
use tokio::time::sleep;
use tower_http::timeout::TimeoutLayer;
use tower_http::trace::TraceLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // Enable tracing.
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!(
                    "{}=debug,tower_http=debug,axum=trace",
                    env!("CARGO_CRATE_NAME")
                )
                .into()
            }),
        )
        .with(tracing_subscriber::fmt::layer().without_time())
        .init();

    // Create a regular axum app.
    let app = Router::new()
        .route("/slow", get(|| sleep(Duration::from_secs(5))))
        .route("/forever", get(std::future::pending::<()>))
        .layer((
            TraceLayer::new_for_http(),
            // Graceful shutdown will wait for outstanding requests to complete. Add a timeout so
            // requests don't hang forever.
            TimeoutLayer::new(Duration::from_secs(10)),
        ));

    // Create a `TcpListener` using tokio.
    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

    // Run the server with graceful shutdown
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install signal handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
```

## 代价与边界

**第二次 Ctrl-C 不管用。** 步骤 4 详细说了:`tokio::signal::ctrl_c()` 接管了 SIGINT,停机开始后再按 Ctrl-C 会被静默吞掉,只能 `kill -9`。这在开发时非常烦人。

> **更好的写法**:让第二次信号强制退出。思路是把 `shutdown_signal` 改成"第一次触发优雅停机,之后继续监听,再来一次就 `std::process::exit(1)`":
>
> ```rust
> tokio::spawn(async {
>     // 等第二次信号
>     signal::ctrl_c().await.ok();
>     tracing::warn!("second signal received, forcing shutdown");
>     std::process::exit(1);
> });
> ```
>
> 在信号到达后 spawn 这个任务即可。**几乎所有成熟的服务端框架都提供这个逃生口**,官方示例没有。

**不等 `tokio::spawn` 出去的后台任务。** `with_graceful_shutdown` 只管**连接**。你自己 spawn 的定时任务、消费者、后台 worker —— 进程退出时会被直接丢弃。

正确做法是自己维护一个关闭通道(`tokio::sync::watch` 或 `Notify`)广播给所有后台任务,再用 `tokio_util::task::TaskTracker` 等它们结束。**这部分工作量不小,而且很容易被忘掉。**

**WebSocket 和 SSE 会把停机拖成"等超时"。** 这是最容易踩的实战问题:[第 33 章](33-sse.md)的 SSE 流和[第 34 章](34-websockets.md)的 WebSocket 连接**本来就是长期不结束的**。优雅停机会老老实实等着它们,直到超时。

一个挂着 100 个 WebSocket 客户端的服务,停机时会**每次都要等满超时**。正确做法是**主动通知这些连接关闭**:WebSocket 发一个 Close 帧,SSE 发一个约定的结束事件,然后主动断开。[第 37 章](37-chat.md)那种广播结构里,加一条"服务器要重启了"的广播消息就能做到。

**k8s 里光有优雅停机不够,还要 preStop。** 这条最反直觉:**Pod 被删除时,"从 Service 的 Endpoints 里摘除"和"发 SIGTERM"是并行发生的,而且摘除是异步传播的**(kube-proxy / Ingress 需要时间同步 iptables 或转发表)。

结果是:进程已经收到 SIGTERM 停止 accept 了,但**流量还在往这个 Pod 打**,那些请求会被拒绝(`Connection refused`)。

标准解法是加一个 `preStop` hook:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]
```

**先睡 5 秒什么都不做**,给 Endpoints 传播留时间,睡完才发 SIGTERM。同时把 readiness 探针配好,让摘除更快发生。

**这是"代码写对了但线上还是报错"的经典案例**,值得单独记住。

**没有关闭数据库连接池等资源。** [第 24 章](24-sqlx-postgres.md)的 `PgPool` 有 `close()` 方法,正常关闭会等在途查询完成并干净地断开连接。示例没调。虽然进程退出时操作系统会回收连接,但服务端会看到一堆异常断开的连接。

**`TimeoutLayer` 只覆盖到 handler 返回,不覆盖流式 body。** 和[第 41 章](41-reqwest-response.md#代价与边界)一样的问题:handler 返回一个流式响应之后就"完成"了,之后的传输时长不受这个超时管。**所以 `/stream` 这类路由仍然可能把停机拖住。**

**只处理了 SIGINT 和 SIGTERM。** SIGHUP(终端断开,传统上用于"重新加载配置")、SIGQUIT(某些平台用于"优雅停机+dump")没有处理。多数场景够用,但如果你的部署脚本用了别的信号,记得加上。

**监听 `0.0.0.0`。** [第 44 章](44-serve-with-hyper.md#代价与边界)说过这个决策需要显式做,别照抄。

**没有停机日志。** 收到信号后应该打一条日志说明"开始优雅停机",退出前再打一条"停机完成,耗时 X 秒"。没有这两条,排查"发版为什么慢"时抓瞎。[第 46 章](46-tls-graceful-shutdown.md)的示例就加了这条日志。

## 动手练习

**1. 加上停机日志。**

提示:在 `select!` 之后加 `tracing::info!("shutdown signal received")`,在 `serve(...).await` 之后加 `tracing::info!("shutdown complete")`(需要 `cargo add tracing`)。用 `/slow` 验证两条日志之间隔了 5 秒。**「代价与边界」说过没有这两条日志排查发版慢时会抓瞎。**

**2. 支持"第二次信号强制退出"。**

提示:按「代价与边界」的写法,在信号到达后 spawn 一个任务继续监听 `ctrl_c()`,第二次触发就 `std::process::exit(1)`。验证:`curl /forever` 后按两次 Ctrl-C,第二次应该立刻退出,不用等 10 秒。

**3. 让后台任务也参与停机。**

提示:`tokio::sync::watch::channel(false)` 建一个通道,spawn 一个每秒打印一次的后台任务,任务里用 `select!` 同时等 `sleep` 和 `rx.changed()`。信号到达时 `tx.send(true)`。验证:Ctrl-C 后后台任务的打印立刻停止,而不是被进程退出粗暴打断。

**4. 加一条 WebSocket 路由,亲眼看看它怎么拖住停机。**

提示:照[第 34 章](34-websockets.md)加一个简单的 echo WebSocket。用浏览器或 `websocat` 连上,然后 Ctrl-C —— 你会看到进程要等满 10 秒。**然后试着在停机时主动给所有连接发 Close 帧**,确认能立刻退出。

**5. 用 Docker 完整验证一遍。**

提示:写个最简单的 Dockerfile,`docker run` 起来后 `docker stop`(默认 10 秒宽限期),观察容器退出耗时。然后把 `TimeoutLayer` 改成 30 秒,再 `docker stop`,你会看到**容器在 10 秒时被 SIGKILL**。`docker stop -t 60` 能给更长的宽限期。**这个实验把"应用超时必须小于平台宽限期"变成了亲眼所见。**

## 小结与下一章预告

粗暴退出会切断在途请求,而进程退出在生产环境**每天要发生很多次**。优雅停机的定义是三件事:**停止 accept、通知已有连接处理完当前请求就关闭、等所有连接关闭**。

信号方面记住三条:**Ctrl-C 是 SIGINT,生产环境收到的是 SIGTERM,SIGKILL 捕获不了**。`shutdown_signal` 那段用 `select!` 同时等前两个,Windows 上用 `std::future::pending()` 把 SIGTERM 分支永远挂起——这段代码可以原样抄进任何项目。

**必须配超时。** 步骤 4 演示了不配的后果:一个永不返回的请求让进程**永远退不出**,而且因为 `tokio::signal::ctrl_c()` 接管了 SIGINT,**第二次 Ctrl-C 也没用**,只能 `kill -9`。生产环境的表现是发版卡住直到被 SIGKILL——优雅停机白加了。超时值必须**小于平台的宽限期**(Docker 默认 10 秒,k8s 默认 30 秒)。

`.layer((A, B))` 元组写法顺序和 `ServiceBuilder` 一致(左边在外层),这里让 `TraceLayer` 包住 `TimeoutLayer`,超时产生的 408 才会被记录。

最需要警惕的三条边界:**后台任务不在优雅停机的管辖范围内**、**WebSocket/SSE 会把停机拖到超时**、**k8s 里还要配 preStop sleep** 否则摘流量和发信号并行会导致新请求被拒。第三条是"代码写对了线上还是报错"的经典案例。

本章用的是 `axum::serve`。[第 46 章 tls-graceful-shutdown](46-tls-graceful-shutdown.md) 把优雅停机搬到 HTTPS 上——[第 32 章](32-tls-rustls.md)说过 `axum::serve` 不支持 TLS,得用 `axum-server`,而它的停机机制是**另一套 API**:不是传一个 future 进去,而是拿一个 `Handle` 出来主动调用。你还会看到一个巧妙的写法:**同一个信号 future 同时驱动两个服务器的停机**。
