# 第 46 章 tls-graceful-shutdown:另一套停机 API

## 本章你将做出什么

[第 32 章](32-tls-rustls.md)那个 HTTPS 服务 + HTTP 重定向服务的组合,这次**两个都能优雅停机**:

```
https://localhost:3000    Hello, World!
http://localhost:7878     301 → https://localhost:3000
```

按 Ctrl-C(或者 `docker stop`),两个服务一起进入停机流程,在途请求做完,最多等 10 秒。

[第 45 章](45-graceful-shutdown.md)的 `axum::serve(...).with_graceful_shutdown(future)` 在这里用不了——因为 HTTPS 服务跑在 `axum-server` 上,它有**完全不同的一套停机 API**。

学完本章你将掌握:

- **两种停机 API 的形态差异**:传一个 future 进去(push)vs 拿一个 handle 出来(pull)
- `axum_server::Handle`:能干的不止是停机
- `graceful_shutdown(Some(d))` / `graceful_shutdown(None)` / `shutdown()` 三者的区别
- **一个信号 future 同时驱动两个服务器停机**的写法,以及它的**隐患**
- 泛型参数 `F: Future<Output = ()> + Send + 'static` 的用意

本章的 TLS 部分([第 32 章](32-tls-rustls.md)已讲透)只做简要回顾,重点全在停机上。

## 前置知识

### 回顾:为什么这里不能用 `axum::serve`

[第 32 章](32-tls-rustls.md#为什么不能用-axumserve)讲过:`axum::serve` 只接受一个 `TcpListener`,**没有任何地方能塞进 TLS 配置**。所以 HTTPS 要换成社区的 `axum-server`:

```rust
axum_server::bind_rustls(addr, config)
    .serve(app.into_make_service())
    .await
```

这就带来一个连锁问题:**`with_graceful_shutdown` 是 `axum::serve` 的方法,`axum-server` 上没有。** 停机得用它自己的机制。

### 两种停机 API:push vs pull

这是本章的核心对比:

| | `axum::serve`([第 45 章](45-graceful-shutdown.md)) | `axum_server`(本章) |
|---|---|---|
| 写法 | `.with_graceful_shutdown(future)` | `.handle(handle)` + 别处调 `handle.graceful_shutdown(...)` |
| 谁触发 | **future 完成时自动触发** | **你主动调用方法** |
| 形态 | 把"什么时候停"**推**给服务器 | 从服务器**拉**出一个遥控器 |
| 超时 | 服务器不管,靠 `TimeoutLayer` | **API 自带**:`graceful_shutdown(Some(d))` |
| 停机后能做别的吗 | 不能,future 完成就是全部 | 能,`handle` 还能查连接数等 |

**`Handle` 更像一个遥控器**:你先建好它,交给服务器,然后在**任何地方、任何时候**按下停机按钮。这带来两个 `axum::serve` 没有的能力:

1. **不只由信号触发** —— 管理接口的 `/admin/shutdown`、健康检查连续失败、配置热更新失败……任何代码路径都能停机
2. **超时内建** —— 不需要像[第 45 章](45-graceful-shutdown.md#步骤-5加上超时给等待封顶)那样靠 `TimeoutLayer` 间接实现

### `axum_server::Handle` 能做什么

```rust
let handle = axum_server::Handle::new();
```

它是**可克隆的共享句柄**(内部是 `Arc`,和[第 41 章](41-reqwest-response.md)的 `reqwest::Client` 一个模式)。主要能力:

| 方法 | 作用 |
|---|---|
| `graceful_shutdown(Some(d))` | 停止 accept,等在途请求,**最多等 `d`** |
| `graceful_shutdown(None)` | 停止 accept,等在途请求,**无限期等** |
| `shutdown()` | **立刻**关闭所有连接,不等 |
| `listening().await` | 等服务器真正开始监听,返回实际地址 |
| `connection_count()` | 当前连接数 |

**`listening()` 在测试里特别有用**:绑 `0.0.0.0:0` 让系统分配端口,然后 `handle.listening().await` 拿到真实端口。[第 21 章](21-testing.md)那种 `oneshot` 测法用不上,但需要真起一个服务器的集成测试就靠它。

**`graceful_shutdown(None)` 和[第 45 章](45-graceful-shutdown.md#步骤-4故意不加超时看看会卡成什么样)那个坑是同一个** —— 一个永不结束的请求会让进程永远退不出。**默认就该传 `Some(...)`。**

### 一个 future,两个服务器

本章最巧妙的一处设计是这三行:

```rust
let handle = axum_server::Handle::new();
let shutdown_future = shutdown_signal(handle.clone());
tokio::spawn(redirect_http_to_https(ports, shutdown_future));
```

`shutdown_signal` 这个 async fn 的函数体是:

```
等信号  →  打日志  →  handle.graceful_shutdown(Some(10s))  →  返回
```

而 `shutdown_future` 被交给了**重定向服务器**的 `with_graceful_shutdown`。于是信号到来时:

```
信号 →  future 里的 select! 完成
     →  handle.graceful_shutdown(...)   ← HTTPS 服务器开始停机
     →  future 返回
     →  with_graceful_shutdown 看到信号完成  ← HTTP 服务器开始停机
```

**一个 future 顺序触发了两套完全不同的停机机制**,而且顺序是确定的(HTTPS 先,HTTP 后)。

**但这个写法有个不小的隐患:HTTPS 服务器的停机,依赖于那个被注释成 "optional" 的重定向服务器。** 因为 `async fn` 调用只是**创建** future,不执行任何代码——**没有人 `.await` 它,里面的信号监听就永远不会开始**。步骤 5 会亲手演示删掉那行 `spawn` 之后会发生什么。

### 泛型的 future 参数

```rust
async fn redirect_http_to_https<F>(ports: Ports, signal: F)
where
    F: Future<Output = ()> + Send + 'static,
```

三个约束各有原因:

- **`Future<Output = ()>`** —— `with_graceful_shutdown` 要的就是这个形状:一个完成时不返回任何值的 future
- **`Send`** —— 因为整个函数会被 `tokio::spawn` 到别的线程上
- **`'static`** —— 同理,spawn 出去的任务不能借用栈上的东西

**为什么不写成 `signal: impl Future<Output = ()>`?** 也可以,效果一样,这里用显式泛型只是风格。**关键是别写成 `Box<dyn Future>`** —— 那会多一次堆分配,而且 `dyn Future` 还要 `Pin`,平白复杂。

### 那个 10 秒

```rust
handle.graceful_shutdown(Some(Duration::from_secs(10))); // 10 secs is how long docker will wait
                                                         // to force shutdown
```

官方注释直接点明了理由,和[第 45 章](45-graceful-shutdown.md#docker-和-kubernetes-是怎么停容器的)说的一致:**`docker stop` 默认给 10 秒宽限期,超过就 SIGKILL**。应用的等待上限要卡在这个数以内。

## 项目搭建

```bash
cargo new tls-graceful-shutdown
cd tls-graceful-shutdown
```

```bash
cargo add axum
cargo add axum-extra
cargo add axum-server --features tls-rustls
cargo add tokio --features full
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

和[第 32 章](32-tls-rustls.md#项目搭建)完全一样的依赖。

### 生成证书

同样和[第 32 章](32-tls-rustls.md#生成证书)一样,直接复用那一章的命令:

```bash
mkdir self_signed_certs
cd self_signed_certs

openssl req -x509 -newkey rsa:4096 \
  -keyout key.pem -out cert.pem \
  -days 365 -nodes \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

cd ..
echo 'self_signed_certs/*.pem' >> .gitignore
```

如果你还留着第 32 章的项目,直接把 `self_signed_certs/` 拷过来即可。**`subjectAltName` 是必需的**,理由见[第 32 章](32-tls-rustls.md#生成自签名证书)。

## 分步构建

### 步骤 1:把[第 32 章](32-tls-rustls.md)的 HTTPS 服务搭起来

**要做什么**:先把不带停机的版本跑通。这一步的代码[第 32 章](32-tls-rustls.md#步骤-1跑起-https-服务)全部讲过,这里只列出来,不再重复解释。

```rust
// src/main.rs
use axum::{routing::get, Router};
use axum_server::tls_rustls::RustlsConfig;
use std::{net::SocketAddr, path::PathBuf};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[derive(Clone, Copy)]
struct Ports {
    http: u16,
    https: u16,
}

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let ports = Ports {
        http: 7878,
        https: 3000,
    };

    // configure certificate and private key used by https
    let config = RustlsConfig::from_pem_file(
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("cert.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("key.pem"),
    )
    .await
    .unwrap();

    let app = Router::new().route("/", get(handler));

    // run https server
    let addr = SocketAddr::from(([127, 0, 0, 1], ports.https));
    tracing::debug!("listening on {addr}");
    axum_server::bind_rustls(addr, config)
        .serve(app.into_make_service())
        .await
        .unwrap();
}

async fn handler() -> &'static str {
    "Hello, World!"
}
```

 `let _ = ports;` 之类的警告先忽略,`ports.http` 下一步才用上。

**跑起来看**:

```bash
cargo run
```

```bash
curl -k https://localhost:3000/
```

```
Hello, World!
```

`-k` 是跳过证书校验(自签名证书,[第 32 章](32-tls-rustls.md#步骤-1跑起-https-服务)解释过)。

**顺便确认一下当前的停机行为**:另开终端 `curl -k https://localhost:3000/`,然后在服务端按 Ctrl-C —— 进程**立刻退出**。这次请求很快所以看不出问题,但和[第 45 章](45-graceful-shutdown.md#步骤-1两个慢路由先看看粗暴退出长什么样)一样,慢请求会被切断。

### 步骤 2:拿出遥控器

**要做什么**:创建 `Handle`,交给服务器。

```rust
    //Create a handle for our TLS server so the shutdown signal can all shutdown
    let handle = axum_server::Handle::new();
```

```rust
    axum_server::bind_rustls(addr, config)
        .handle(handle.clone())
        .serve(app.into_make_service())
        .await
        .unwrap();
```

**`.handle(...)` 插在 `bind_rustls` 和 `serve` 之间。** 这一步之后服务器行为完全不变——遥控器交出去了,但还没人按按钮。

**注意传的是 `handle.clone()`。** `Handle` 内部是 `Arc`,克隆出来的和原件指向同一个东西;要留一份给信号处理函数用。

### 步骤 3:按下按钮的那个函数

**要做什么**:写 `shutdown_signal`,它和[第 45 章](45-graceful-shutdown.md#步骤-2写出信号处理函数)那个几乎一样,只是**多了一个参数和最后两行**。

```rust
use std::time::Duration;
use tokio::signal;
```

```rust
async fn shutdown_signal(handle: axum_server::Handle<SocketAddr>) {
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

    tracing::info!("Received termination signal shutting down");
    handle.graceful_shutdown(Some(Duration::from_secs(10))); // 10 secs is how long docker will wait
                                                             // to force shutdown
}
```

**前面那一大段和[第 45 章](45-graceful-shutdown.md)逐字相同** —— SIGINT + SIGTERM + `select!` + Windows 的 `pending`。**这段代码是社区标准写法,值得记住。**

**新增的两行才是本章的重点:**

```rust
tracing::info!("Received termination signal shutting down");
handle.graceful_shutdown(Some(Duration::from_secs(10)));
```

**① 那条日志。** [第 45 章](45-graceful-shutdown.md#代价与边界)专门抱怨过官方示例没有停机日志,这一章有了。**排查"发版为什么慢"时,这条日志是起点。**

**② `graceful_shutdown` 是同步方法,不是 `.await`。** 它只是设置一个标志位并通知服务器,立刻返回。真正的等待发生在服务器那一侧。所以这个函数**打完日志、按完按钮就返回了**——这一点下一步很关键。

**③ `Handle<SocketAddr>` 带泛型参数。** `axum-server` 0.8 的 `Handle` 是泛型的(参数是连接的地址类型),因为它也支持 Unix 域套接字之类的其他地址。用 TCP 就写 `SocketAddr`。

### 步骤 4:重定向服务器 + 共用同一个 future

**要做什么**:把[第 32 章](32-tls-rustls.md#步骤-2http-重定向到-https)的重定向服务加回来,并让它**顺便驱动整个停机流程**。

```rust
use axum::{
    handler::HandlerWithoutStateExt,
    http::{StatusCode, Uri},
    response::Redirect,
    BoxError,
};
use std::future::Future;
```

```rust
async fn redirect_http_to_https<F>(ports: Ports, signal: F)
where
    F: Future<Output = ()> + Send + 'static,
{
    fn make_https(uri: Uri, https_port: u16) -> Result<Uri, BoxError> {
        let mut parts = uri.into_parts();

        parts.scheme = Some(axum::http::uri::Scheme::HTTPS);
        parts.authority = Some(format!("localhost:{https_port}").parse()?);

        if parts.path_and_query.is_none() {
            parts.path_and_query = Some("/".parse().unwrap());
        }

        Ok(Uri::from_parts(parts)?)
    }

    let redirect = move |uri: Uri| async move {
        match make_https(uri, ports.https) {
            Ok(uri) => Ok(Redirect::permanent(uri.to_string())),
            Err(error) => {
                tracing::warn!(%error, "failed to convert URI to HTTPS");
                Err(StatusCode::BAD_REQUEST)
            }
        }
    };

    let addr = SocketAddr::from(([127, 0, 0, 1], ports.http));
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    tracing::debug!("listening on {addr}");
    axum::serve(listener, redirect.into_make_service())
        .with_graceful_shutdown(signal)
        .await
        .unwrap();
}
```

**`make_https`、`Redirect::permanent`、`HandlerWithoutStateExt::into_make_service` 全部由[第 32 章](32-tls-rustls.md#步骤-2http-重定向到-https)讲过**,这里不重复。和那一章的差别只有两处:多了 `signal` 参数,以及末尾的 `.with_graceful_shutdown(signal)`。

然后在 `main` 里把它们串起来:

```rust
    let handle = axum_server::Handle::new();
    //save the future for easy shutting down of redirect server
    let shutdown_future = shutdown_signal(handle.clone());

    // optional: spawn a second server to redirect http requests to this server
    tokio::spawn(redirect_http_to_https(ports, shutdown_future));
```

**`let shutdown_future = shutdown_signal(handle.clone());` 这一行没有 `.await`。**

前置知识强调过:**调用 `async fn` 只是创建 future,函数体一行都没执行。** 此刻没有任何信号被监听。它只是被存进一个变量,然后 move 进 spawn 出去的任务里。

**只有当 `with_graceful_shutdown(signal)` 开始轮询它时,`signal::ctrl_c()` 才真正被调用。**

**跑起来看**:

```bash
cargo run
```

```
DEBUG tls_graceful_shutdown: listening on 127.0.0.1:3000
DEBUG tls_graceful_shutdown: listening on 127.0.0.1:7878
```

先确认基本功能([第 32 章](32-tls-rustls.md#步骤-2http-重定向到-https)验证过,快速过一遍):

```bash
curl -i http://localhost:7878/foo
```

```
HTTP/1.1 301 Moved Permanently
location: https://localhost:3000/foo
```

现在验证停机。按 **Ctrl-C**:

```
 INFO tls_graceful_shutdown: Received termination signal shutting down
```

**进程干净退出,而且留下了日志。**

**再验证"等在途请求"。** 本章的 handler 太快了看不出效果,**临时**给它加个延迟:

```rust
async fn handler() -> &'static str {
    tokio::time::sleep(Duration::from_secs(5)).await;   // 临时加,验证完删掉
    "Hello, World!"
}
```

```bash
cargo run
```

```bash
curl -k https://localhost:3000/
```

5 秒内按 Ctrl-C。**进程等到请求返回 `Hello, World!` 之后才退出**,日志里那条 `Received termination signal` 出现在返回之前。

**再试试超时封顶:** 把延迟改成 30 秒,重复上面的操作。这次 Ctrl-C 之后**只等 10 秒**,连接被强制关闭,curl 报:

```
curl: (52) Empty reply from server
```

**这就是 `Some(Duration::from_secs(10))` 的作用** —— 它不像 [第 45 章](45-graceful-shutdown.md#步骤-5加上超时给等待封顶)的 `TimeoutLayer` 那样给客户端一个体面的 408,而是**到点直接掐断**。两种做法各有取舍:`TimeoutLayer` 更友好,`graceful_shutdown` 的超时是最后的保险丝。

**把 handler 改回原样。**

### 步骤 5:故意改坏,暴露那个"optional"的隐患

**要做什么**:把那行 `tokio::spawn(redirect_http_to_https(...))` 注释掉——注释里写的是 "optional",看看它到底有多 optional。

```rust
    let handle = axum_server::Handle::new();
    let shutdown_future = shutdown_signal(handle.clone());

    // optional: spawn a second server to redirect http requests to this server
    // tokio::spawn(redirect_http_to_https(ports, shutdown_future));
```

(会有 unused 警告,忽略。)

**先把 handler 的 5 秒延迟加回来**,这样现象才看得见:

```bash
cargo run
```

```bash
curl -k https://localhost:3000/
```

5 秒内按 **Ctrl-C**。

**现象:进程立刻退出,curl 报 `curl: (52) Empty reply from server`。**

**而且日志里没有那条 `Received termination signal`。**

**为什么?** `shutdown_future` 被创建了,但**没有任何人轮询它**——它就是一个静静躺在变量里的状态机。于是:

- `signal::ctrl_c()` **从来没被调用过**,进程的 SIGINT 行为还是系统默认的"立刻终止"
- `handle.graceful_shutdown(...)` **从来没被执行过**,HTTPS 服务器压根不知道要停机

**"optional" 那个注释具有误导性:重定向服务器在功能上是可选的,但在这个写法下,它是整个停机机制的唯一驱动力。**

这是一类很典型的 Rust 陷阱:**future 是惰性的,创建 ≠ 执行**。同样的坑在别处也会出现——比如创建了一个 `async` 任务却忘了 `spawn` 或 `await`,代码能编译、能运行,只是那段逻辑**从来没跑过**。编译器只会给一个 `unused` 警告,而这里连警告都没有(它被 move 走了)。

> **更好的写法**:**把停机信号的监听和重定向服务器解耦。** 让信号处理独立 spawn,重定向服务器用自己的信号:
>
> ```rust
> let handle = axum_server::Handle::new();
>
> // 独立 spawn,不依赖任何"可选"组件
> tokio::spawn(shutdown_signal(handle.clone()));
>
> // 重定向服务器用 handle 的完成状态作为自己的停机信号
> tokio::spawn(redirect_http_to_https(ports, async move {
>     handle_clone.wait_graceful_shutdown().await   // 伪代码,示意
> }));
> ```
>
> 或者更简单直接:用一个 `tokio::sync::watch` 通道广播停机事件,两个服务器各自订阅。**核心原则是:主服务的停机能力不该依赖一个被标注为"可选"的组件。**

**把注释去掉,把 handler 的延迟删掉,恢复完整代码。**

## 完整代码

`src/main.rs`:

```rust
use axum::{
    handler::HandlerWithoutStateExt,
    http::{StatusCode, Uri},
    response::Redirect,
    routing::get,
    BoxError, Router,
};
use axum_server::tls_rustls::RustlsConfig;
use std::{future::Future, net::SocketAddr, path::PathBuf, time::Duration};
use tokio::signal;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[derive(Clone, Copy)]
struct Ports {
    http: u16,
    https: u16,
}

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let ports = Ports {
        http: 7878,
        https: 3000,
    };

    //Create a handle for our TLS server so the shutdown signal can all shutdown
    let handle = axum_server::Handle::new();
    //save the future for easy shutting down of redirect server
    let shutdown_future = shutdown_signal(handle.clone());

    // optional: spawn a second server to redirect http requests to this server
    tokio::spawn(redirect_http_to_https(ports, shutdown_future));

    // configure certificate and private key used by https
    let config = RustlsConfig::from_pem_file(
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("cert.pem"),
        PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("self_signed_certs")
            .join("key.pem"),
    )
    .await
    .unwrap();

    let app = Router::new().route("/", get(handler));

    // run https server
    let addr = SocketAddr::from(([127, 0, 0, 1], ports.https));
    tracing::debug!("listening on {addr}");
    axum_server::bind_rustls(addr, config)
        .handle(handle)
        .serve(app.into_make_service())
        .await
        .unwrap();
}

async fn shutdown_signal(handle: axum_server::Handle<SocketAddr>) {
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

    tracing::info!("Received termination signal shutting down");
    handle.graceful_shutdown(Some(Duration::from_secs(10))); // 10 secs is how long docker will wait
                                                             // to force shutdown
}

async fn handler() -> &'static str {
    "Hello, World!"
}

async fn redirect_http_to_https<F>(ports: Ports, signal: F)
where
    F: Future<Output = ()> + Send + 'static,
{
    fn make_https(uri: Uri, https_port: u16) -> Result<Uri, BoxError> {
        let mut parts = uri.into_parts();

        parts.scheme = Some(axum::http::uri::Scheme::HTTPS);
        parts.authority = Some(format!("localhost:{https_port}").parse()?);

        if parts.path_and_query.is_none() {
            parts.path_and_query = Some("/".parse().unwrap());
        }

        Ok(Uri::from_parts(parts)?)
    }

    let redirect = move |uri: Uri| async move {
        match make_https(uri, ports.https) {
            Ok(uri) => Ok(Redirect::permanent(uri.to_string())),
            Err(error) => {
                tracing::warn!(%error, "failed to convert URI to HTTPS");
                Err(StatusCode::BAD_REQUEST)
            }
        }
    };

    let addr = SocketAddr::from(([127, 0, 0, 1], ports.http));
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    tracing::debug!("listening on {addr}");
    axum::serve(listener, redirect.into_make_service())
        .with_graceful_shutdown(signal)
        .await
        .unwrap();
}
```

## 代价与边界

**HTTPS 服务器的停机依赖"可选"的重定向服务器。** 步骤 5 演示了。这是本章最值得记住的一条。

**`graceful_shutdown` 超时后是硬断,不是 408。** 步骤 4 验证过:到点直接关连接,客户端看到的是 `Empty reply from server`。想给客户端一个体面的响应,还是要配[第 45 章](45-graceful-shutdown.md#步骤-5加上超时给等待封顶)的 `TimeoutLayer`,而且**它的时长要小于 `graceful_shutdown` 的时长**,这样才有机会先返回 408。

**这一章根本没有 `TimeoutLayer`,也没有 `TraceLayer`。** 相比[第 45 章](45-graceful-shutdown.md),这里的中间件是空的:没有请求超时、没有访问日志。生产环境两个都要加。

**证书路径绑定到 `CARGO_MANIFEST_DIR`。** [第 32 章](32-tls-rustls.md#代价与边界)详细说过:编译好的二进制换台机器就找不到证书。

**重定向硬编码 `localhost`。** [第 32 章](32-tls-rustls.md#代价与边界)也说过,而且验证过 `Host: example.com` 的请求会被错误地导向 `localhost`。**这个 bug 在本章原样存在。**

**证书不会热重载。** [第 32 章](32-tls-rustls.md#代价与边界)提过 `RustlsConfig::reload_from_pem_file`。有意思的是:**有了 `Handle` 之后,"重启换证书"这条路更容易走了** —— 可以在证书文件变化时 `graceful_shutdown` 然后由 supervisor 拉起。但更好的还是热重载,不中断连接。

**后台任务、WebSocket、k8s 的 preStop —— [第 45 章](45-graceful-shutdown.md#代价与边界)列的那些边界这里全都适用。** `axum-server` 的 `graceful_shutdown` 同样只管连接,不管你自己 spawn 的东西。

**没处理 `spawn` 出去的重定向服务器的 panic。** `tokio::spawn` 返回的 `JoinHandle` 被丢弃了。如果 7878 端口被占用,`bind` 的 `unwrap()` 会 panic,而**这个 panic 只会打印在日志里,主服务继续跑**——看起来一切正常,实际上重定向服务没起来,**而且停机机制也跟着废了**(步骤 5 的问题)。

生产代码应该检查 `JoinHandle`,或者至少在 spawn 的任务里捕获错误并明确地让整个进程退出。

**监听 `127.0.0.1`。** 和[第 45 章](45-graceful-shutdown.md)的 `0.0.0.0` 相反。这一章只能本机访问,容器里跑要改。**两章的示例在这一点上不一致,说明官方示例本身也没有统一的约定** —— 这个值必须自己决定。

## 动手练习

**1. 解耦停机信号和重定向服务器。**

提示:按步骤 5 的「更好的写法」,用 `tokio::sync::watch::channel(false)` 广播停机事件。`shutdown_signal` 独立 spawn,收到信号后 `tx.send(true)` 并调 `handle.graceful_shutdown(...)`;重定向服务器的 `with_graceful_shutdown` 等 `rx.changed()`。**验证:注释掉重定向服务器的 spawn,Ctrl-C 仍然能优雅停机。**

**2. 加上 `TimeoutLayer`,让超时时返回 408 而不是断连。**

提示:`cargo add tower-http --features timeout`,给 `app` 加 `.layer(TimeoutLayer::with_status_code(StatusCode::REQUEST_TIMEOUT, Duration::from_secs(5)))`(需要 `use axum::http::StatusCode;`),注意**它要比 `graceful_shutdown` 的 10 秒短**。用一个 30 秒的慢 handler 验证:Ctrl-C 后 5 秒拿到 408,而不是等 10 秒后被掐断。

**3. 加一个管理端点触发停机。**

提示:把 `handle.clone()` 作为 state 交给一个 `POST /admin/shutdown` 路由,handler 里调 `handle.graceful_shutdown(Some(...))`。**这是 `Handle` 相比 `with_graceful_shutdown` 的独有能力**,试完你会理解 pull 式 API 的价值。 真实项目里这个端点必须有认证,而且不该暴露在公网。

**4. 用 `handle.connection_count()` 打印停机进度。**

提示:调完 `graceful_shutdown` 之后 spawn 一个循环,每秒打印一次剩余连接数。**这在排查"停机为什么慢"时非常有用** —— 你能直接看到是不是有连接赖着不走。

**5. 用 Docker 验证 `docker stop` 的 10 秒宽限期。**

提示:和[第 45 章](45-graceful-shutdown.md)的练习 5 一样,但这次把 `graceful_shutdown` 的参数改成 30 秒,观察容器在第 10 秒被 SIGKILL。**注释里那句 "10 secs is how long docker will wait" 就是这么来的。**

## 小结与下一章预告

`axum-server` 的停机是**另一套 API**:`axum::serve` 是把一个 future **推**给服务器(`with_graceful_shutdown`),`axum-server` 是从服务器**拉**出一个遥控器(`Handle`)。

遥控器的好处是**任何代码路径都能触发停机**(管理端点、健康检查失败、配置加载失败),而且**超时是 API 自带的**:`graceful_shutdown(Some(d))`。代价是超时后**硬断连接**,客户端看到 `Empty reply`,不像 `TimeoutLayer` 那样给一个体面的 408 —— 两者配合使用最好,而且 `TimeoutLayer` 的时长要更短。

`shutdown_signal` 的信号监听部分和[第 45 章](45-graceful-shutdown.md)逐字相同,只是末尾多了一条日志和一次 `graceful_shutdown` 调用。**注意 `graceful_shutdown` 是同步方法,按完按钮立刻返回**,等待发生在服务器那一侧。

本章最巧妙也最危险的是那三行:`let shutdown_future = shutdown_signal(handle.clone());` 创建 future 但不 await,再把它交给重定向服务器的 `with_graceful_shutdown`。一个 future 顺序触发了两套停机机制,很漂亮——**但也意味着 HTTPS 服务器的停机完全依赖一个被注释成 "optional" 的组件**。步骤 5 把那行 spawn 注释掉之后,Ctrl-C 立刻变回粗暴退出,连日志都没有。

背后的通用教训是:**Rust 的 future 是惰性的,创建不等于执行**。没人轮询它,里面一行代码都不会跑,而且编译器不会提醒你。

「生产部署」阶段接下来转向可观测性。服务能优雅地起、优雅地停了,但**它跑得怎么样?** [第 47 章 prometheus-metrics](47-prometheus-metrics.md) 讲指标采集:请求量、延迟分布、错误率怎么暴露给监控系统。你会看到一个和日志完全不同的观测维度,以及一个能把监控系统拖垮的经典陷阱——**标签基数爆炸**,而 axum 的 `MatchedPath` 正是为它准备的解药。
