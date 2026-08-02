# 第 54 章 simple-router-wasm:没有服务器的 axum

## 本章你将做出什么

本教程最后一章,也是最反直觉的一章:**一个不监听任何东西的 axum 程序**。

没有 `axum::serve`,没有 `TcpListener`,没有 `#[tokio::main]`,**连 tokio 都不在依赖里**。

```rust
let request = Request::get("https://serverless.example/api/").body("Some Body Data".into())?;
let response = block_on(app(request));
assert_eq!(200, response.status());
```

**喂一个请求进去,拿一个响应出来。** 就这样。

而且它能编译到 WebAssembly:

```bash
cargo build --target wasm32-unknown-unknown
```

学完本章你将掌握:

- **`Router` 本质上只是一个函数**:请求进,响应出
- serverless 和边缘计算为什么正好需要这个形状
- `axum` 的 `default-features = false` 关掉了什么,以及为什么必须关
- `futures_executor::block_on`:不用 tokio 跑一个 future
- **`wasm32-unknown-unknown` 和 `wasm32-wasip1` 的区别**
- 一个只在编译到 wasm 时才暴露的依赖问题

## 前置知识

### 前 53 章一直在做的事,和这一章的区别

回顾一下[第 44 章](44-serve-with-hyper.md#axumserve-的五件事)拆开的那五件事:

```
① accept TCP 连接
② spawn 任务
③ 转换 IO
④ 转换 Service
⑤ 跑 HTTP 协议
       ↓
   Router 处理请求  ← 只有这一步是"你的业务"
```

**前 53 章有 52 章在讲怎么把请求送到 `Router` 面前。** 这一章把前五步全删掉,只留最后一个。

**为什么能这么干?** 因为 `Router` 从来就是一个 tower `Service`:

```rust
Service<Request<B>>:  Request  →  Response
```

[第 17 章](17-tracing-aka-logging.md#towerservice-与-layer)第一次讲 `Service`,[第 21 章](21-testing.md)用 `oneshot` 直接调用它做测试,[第 42 章](42-http-proxy.md#hyper-的-service-tower-的-service)把它塞进 hyper。**这一章只是把这件事说白了:那五步是可选的。**

### serverless:平台已经替你做了那五步

在 AWS Lambda、Cloudflare Workers、Fastly Compute 这类平台上,**你不能自己监听端口**。平台的模型是:

```
平台收到 HTTP 请求  →  唤起你的函数  →  你返回响应  →  平台发出去
```

**你要提供的就是一个 `fn(Request) -> Response`。**

对照一下:

| | 传统部署 | serverless |
|---|---|---|
| 谁监听端口 | **你的进程** | 平台 |
| 谁管连接 | 你(或 hyper) | 平台 |
| 进程生命周期 | 长期运行 | **按需唤起,随时冻结** |
| 你写的是 | 一个服务器 | **一个函数** |

**axum 的 `Router` 恰好就是那个函数的形状**,所以能原样搬过去——路由、提取器、中间件、`IntoResponse`,前 53 章学的东西一样都不浪费。

**这就是本章存在的意义:证明 axum 不绑定 tokio 服务器。**

### WebAssembly 和两个不同的目标

wasm 是一种可移植的字节码。Rust 能编到好几个 wasm 目标,**差别很大**:

| 目标 | 有什么系统接口 | 典型宿主 |
|---|---|---|
| **`wasm32-unknown-unknown`** | **什么都没有** —— 没有文件、没有时钟、没有网络、没有线程 | 浏览器、Cloudflare Workers |
| `wasm32-wasip1` / `wasip2` | WASI:文件、时钟、随机数、(p2 起)网络 | Fastly、Spin、wasmCloud、wasmtime |

**本章的目标是第一个,也是限制最狠的那个。**

"什么都没有"是字面意思:`std::time::Instant::now()` 会 panic,`std::fs` 全部不可用,**`std::net` 里的东西压根编译不过**。

**这直接决定了下一节的内容。**

### 为什么必须 `default-features = false`

axum 的默认 feature 里有一个 `tokio`,它会拉进 `tokio` 的网络能力,而 tokio 的 IO 层是 **`mio`** —— **`mio` 不支持 `wasm32-unknown-unknown`**(它要调 epoll / kqueue / IOCP,而 wasm 里这些都不存在)。

```toml
axum = { version = "0.8", default-features = false }
```

**关掉之后,`axum::serve` 就不存在了** —— 这正是你想要的,反正也用不上。

**但顺带被关掉的还有一批常用东西:**

| feature | 提供什么 | wasm 能不能开 |
|---|---|---|
| `tokio` | **`axum::serve`** | 否 |
| `http1` / `http2` | hyper 的协议支持 | (间接依赖 tokio) |
| `json` | **`Json<T>`** | 是 |
| `query` | **`Query<T>`** | 是 |
| `form` | **`Form<T>`** | 是 |
| `matched-path` | `MatchedPath`([第 47 章](47-prometheus-metrics.md)) | 是 |
| `original-uri` | `OriginalUri` | 是 |

**所以正确的做法不是全关,而是关掉需要 tokio 的、留下不需要的**:

```bash
cargo add axum --no-default-features --features json,query,form
```

官方示例全关了,因为它只用 `get(index)` 和 `Html`,**什么提取器都没用到**。步骤 4 会把这些加回来验证。

> 官方示例的 `Cargo.toml` 里还有个有意思的细节:它声明了 `axum-extra` 却从不使用,并在 `[package.metadata.cargo-machete]` 里显式忽略它——`cargo-machete` 是检查"声明了但没用到的依赖"的工具。配合源码注释「we don't strictly use axum-extra in this example but wanna make sure that works in wasm as well」可以看出来:**这个依赖存在的唯一目的是让 CI 验证 `axum-extra` 也能编到 wasm。**

### `futures_executor::block_on`

没有 tokio,谁来跑 future?

```rust
use futures_executor::block_on;
let response: Response = block_on(app(request));
```

`block_on` 是一个**最小执行器**:在当前线程上反复轮询这个 future,直到它完成。没有任务调度、没有定时器、没有 IO 事件循环——**只有一个 future 和一个循环**。

| | `#[tokio::main]` | `futures_executor::block_on` |
|---|---|---|
| 能跑多少任务 | 很多,可并发可并行 | **一个** |
| 定时器 | 是 | 否 |
| 网络 / 文件 IO | 是 | 否 |
| `spawn` | 是 | 否 |
| 依赖大小 | 大 | **很小** |
| wasm 支持 | 否 | 是 |

**它够用,是因为这里的 future 从头到尾不会真正挂起** —— 路由匹配、提取器、handler 全是纯计算,没有任何 IO。**第一次轮询就跑完了。**

**这也是它的边界**:如果 handler 里 `.await` 了一个真正需要等待的东西(网络请求、定时器),`block_on` 会**死等**,因为没有任何人会来唤醒它。真实的 serverless 运行时会提供自己的异步集成(Cloudflare Workers 的 `worker` crate、Lambda 的 `lambda_runtime`),**别把 `block_on` 当通用方案**。

### `router.call(request)`

```rust
let mut router = Router::new().route("/api/", get(index));
let response = router.call(request).await.unwrap();
```

三个细节:

**① `let mut`** —— 因为 tower 的 `Service::call` 取 `&mut self`([第 42 章](42-http-proxy.md#hyper-的-service-tower-的-service)那张表)。

**② `use tower_service::Service;`** —— `call` 是 trait 方法,trait 必须在作用域里。**忘了这个 import 会得到"方法不存在"的错误**,而不是"trait 未导入"的提示,容易懵。

**③ `.unwrap()`** —— `Router` 的错误类型是 `Infallible`([第 42 章](42-http-proxy.md#match-err-消灭-infallible)讲过),所以这个 unwrap 永远不会 panic。更严谨的写法是 `match err {}`。

**注意请求体的类型是 `String`,不是 `axum::body::Body`。** `Router` 的 `Service` 实现对请求体类型是泛型的,满足约束的类型都能传。**这一点对 serverless 很重要**——各家平台给你的请求体类型五花八门,不必先转成 axum 自己的 `Body`。真遇到不满足约束的类型,用 `Body::from(...)` 包一层即可。

## 项目搭建

```bash
cargo new simple-router-wasm
cd simple-router-wasm
```

```bash
cargo add axum --no-default-features
cargo add axum-extra --no-default-features
cargo add futures-executor
cargo add http
cargo add tower-service
```

装上 wasm 目标:

```bash
rustup target add wasm32-unknown-unknown
```

**注意直接依赖了 `http` crate。** 前 53 章都是用 `axum::http::...`(axum 把它重导出了),这里直接依赖是为了强调**这些类型不属于 axum**,它们是整个 Rust HTTP 生态的公共词汇——[第 43 章](43-reverse-proxy.md#反向代理的核心动作改写-uri)那个 `Uri`、[第 41 章](41-reqwest-response.md#两个-response不是一回事)那个 `StatusCode` 和 `HeaderMap`,都来自这个 crate。

## 分步构建

### 步骤 1:把 Router 当函数用

**要做什么**:构造一个请求,直接喂给 `Router`,拿到响应。

```rust
// src/main.rs
use axum::{
    response::{Html, Response},
    routing::get,
    Router,
};
use futures_executor::block_on;
use http::Request;
use tower_service::Service;

fn main() {
    let request: Request<String> = Request::get("https://serverless.example/api/")
        .body("Some Body Data".into())
        .unwrap();

    let response: Response = block_on(app(request));
    assert_eq!(200, response.status());
}

#[allow(clippy::let_and_return)]
async fn app(request: Request<String>) -> Response {
    let mut router = Router::new().route("/api/", get(index));
    let response = router.call(request).await.unwrap();
    response
}

async fn index() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
```

**`fn main()` 是同步的。** 没有 `#[tokio::main]`,因为没有 tokio。异步的边界被 `block_on` 收在了 `main` 内部。

**`async fn app(request) -> Response` 就是那个"serverless 函数"。** 官方注释说得很清楚:

> `main` 扮演的是 serverless 运行时的角色——它最初收到请求,然后调用 `app` 函数。

**在真实平台上,`main` 会被平台的 SDK 替换掉,而 `app` 原样保留。**

**`Request::get(...)` 里那个 URL 有两个信息位:**

- **路径 `/api/`** —— 这是路由匹配用的,必须和 `.route("/api/", ...)` 对上
- 主机名 `serverless.example` —— 这里没人用,但真实场景里可能有中间件读 `Host`

**`.body("Some Body Data".into())`** 塞了一段 body,虽然 `index` handler 根本不读它。**这是刻意的**:它证明了带 body 的请求也能走通这条路。

**`#[allow(clippy::let_and_return)]`** 是因为 `let response = ...; response` 这种写法 clippy 会建议直接返回。示例保留中间变量是为了可读性。

**跑起来看**:

```bash
cargo run
```

**没有任何输出,也没有 panic —— 就是成功。** 断言 `assert_eq!(200, response.status())` 通过了。

想看得更清楚一点,临时在 `main` 末尾加一句:

```rust
    println!("status = {}", response.status());
    println!("headers = {:?}", response.headers());
```

```
status = 200
headers = {"content-type": "text/html; charset=utf-8"}
```

**`Html` 设置的 `Content-Type` 也在** —— 整套 `IntoResponse` 机制([第 04 章](04-templates.md))在这里完全正常工作。

**再试试路由不匹配的情况**:把请求路径改成 `/nope`,`assert_eq!` 会失败并告诉你实际是 404。**路由匹配、404 兜底([第 06 章](06-global-404-handler.md))全都照常。**

### 步骤 2:编译到 WebAssembly

**要做什么**:验证这份代码真的能编到 wasm。

```bash
cargo build --target wasm32-unknown-unknown
```

```
    Finished `dev` profile [unoptimized + debuginfo] target(s)
```

**编译通过。** 产物在:

```bash
ls -lh target/wasm32-unknown-unknown/debug/*.wasm
```

**注意这个 wasm 文件不能直接"运行"**。`wasm32-unknown-unknown` 没有任何系统接口,连 `println!` 都没地方输出。它要被一个宿主环境加载——浏览器、Cloudflare Workers 的运行时、或者 `wasm-bindgen` 生成的胶水代码。

**本章验证的是"能不能编"**,这也正是官方示例的定位(源码注释:「This example should always compile with `--target wasm32-unknown-unknown`」)。

**顺带看看体积:**

```bash
cargo build --release --target wasm32-unknown-unknown
ls -lh target/wasm32-unknown-unknown/release/*.wasm
```

release 模式下小很多。**wasm 的体积直接影响冷启动时间**,serverless 场景下值得在意——`[profile.release] opt-level = "z"`、`lto = true`、`strip = true`,再配合 `wasm-opt` 还能继续压。

### 步骤 3:故意改坏,看看 tokio 为什么进不来

**要做什么**:把 axum 的默认 feature 打开,试着编到 wasm。

```bash
cargo remove axum
cargo add axum
```

(不带 `--no-default-features`,也就是打开默认 feature。)

**先确认原生还能跑:**

```bash
cargo run
```

正常。**这就是这个 bug 的可怕之处:本地开发完全看不出问题。**

**现在编到 wasm:**

```bash
cargo build --target wasm32-unknown-unknown
```

```
error: The wasm32-unknown-unknown targets are not supported by default; you may need to enable the "js" feature.
   ...
error: could not compile `mio` (lib) due to 1 previous error
```

(具体报错取决于版本,可能来自 `mio`、`socket2` 或 `tokio` 自身,但一定是**这条依赖链**上的某个 C 风格系统调用封装。)

**追一下这条链:**

```
axum(默认 feature `tokio`)
  └─ tokio(net)
       └─ mio
            └─ epoll / kqueue / IOCP   ← wasm 里不存在
```

**一个你根本没直接用到的 crate,挡住了整个构建。**

**这类问题的一般形态是:**

- 某个依赖的**默认 feature** 拉进了平台相关的东西
- 在你的开发平台上一切正常
- 换目标平台(wasm、嵌入式 `no_std`、交叉编译)才炸
- **报错发生在一个你没听说过的传递依赖里**

**排查手段是 `cargo tree`:**

```bash
cargo tree --target wasm32-unknown-unknown -i mio
```

`-i` 是反向查询:"谁依赖了 mio"。**它会把整条链打印出来**,让你知道该关哪个 feature。

**改回去:**

```bash
cargo remove axum
cargo add axum --no-default-features
cargo build --target wasm32-unknown-unknown     # 又通了
```

### 步骤 4:把能用的 feature 加回来

**要做什么**:全关太狠了,把不依赖 tokio 的提取器加回来。

```bash
cargo remove axum
cargo add axum --no-default-features --features json,query,form
```

改一下 handler 用上 `Query`:

```rust
use axum::extract::Query;
use std::collections::HashMap;
```

```rust
async fn index(Query(params): Query<HashMap<String, String>>) -> Html<String> {
    Html(format!("<h1>Hello, {:?}</h1>", params))
}
```

把请求的 URL 带上查询参数:

```rust
    let request: Request<String> = Request::get("https://serverless.example/api/?name=world")
        .body("Some Body Data".into())
        .unwrap();
```

**验证两件事:**

```bash
cargo run                                      # 原生跑通
cargo build --target wasm32-unknown-unknown    # wasm 也能编
```

**两个都过。** 说明 `json`、`query`、`form` 这几个 feature 是纯计算的,不碰系统接口。

**如果你手贱试试 `--features tokio`:**

```bash
cargo add axum --no-default-features --features tokio
cargo build --target wasm32-unknown-unknown
```

**又炸了。** 这就是前置知识那张表的实证。

**记住这个方法论:遇到目标平台不支持的依赖,不要放弃整个库,先去查它的 feature 表——通常只有一小部分是平台相关的。**

**改回步骤 1 的版本**,和官方示例保持一致。

## 完整代码

`src/main.rs`:

```rust
use axum::{
    response::{Html, Response},
    routing::get,
    Router,
};
use futures_executor::block_on;
use http::Request;
use tower_service::Service;

fn main() {
    let request: Request<String> = Request::get("https://serverless.example/api/")
        .body("Some Body Data".into())
        .unwrap();

    let response: Response = block_on(app(request));
    assert_eq!(200, response.status());
}

#[allow(clippy::let_and_return)]
async fn app(request: Request<String>) -> Response {
    let mut router = Router::new().route("/api/", get(index));
    let response = router.call(request).await.unwrap();
    response
}

async fn index() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
```

## 代价与边界

**这不是一个服务器,能力缺失是全方位的。** 没有并发、没有连接管理、没有 keep-alive、没有超时、没有优雅停机。**它只是一次函数调用。**

**没有 tokio 意味着 handler 里不能做任何真实 IO。** 不能查数据库([第 24 章](24-sqlx-postgres.md))、不能调外部 API([第 41 章](41-reqwest-response.md))、不能 `tokio::spawn`、不能用定时器。**serverless 场景里这些要靠平台提供的 SDK**(Workers 的 `fetch`、Lambda 的 AWS SDK),它们通常有自己的异步集成。

**大量中间件用不了。** `tower-http` 里凡是依赖 tokio 的都不行:`TimeoutLayer`([第 45 章](45-graceful-shutdown.md))要定时器、`ServeDir`([第 38 章](38-static-file-server.md))要文件系统。纯计算的中间件(`CorsLayer`、`SetResponseHeaderLayer`)一般没问题。

**`block_on` 遇到真正挂起的 future 会死等。** 前置知识说过。这个执行器没有唤醒机制。

**`wasm32-unknown-unknown` 上很多 `std` 会 panic 或不可用。** `Instant::now()`、`SystemTime::now()`、`std::fs`、`std::net`、`std::thread`。**需要时钟或文件就得换 `wasm32-wasip1`**,或者用宿主提供的接口。

> **顺带一个常见坑**:`uuid` 的 `v4`、`rand` 这类需要随机数的库,在 `wasm32-unknown-unknown` 上要显式开 `js` feature(从浏览器的 `crypto.getRandomValues` 取熵),否则**运行时才 panic**。[第 40 章](40-stream-to-file.md#动手练习)建议用 UUID 做文件名,搬到 wasm 上就会撞上这个。

**每次调用都重建 `Router`。**

```rust
async fn app(request: Request<String>) -> Response {
    let mut router = Router::new().route("/api/", get(index));   // ← 每次都建
    ...
}
```

演示代码这么写没问题,但**真实的 serverless 函数会被调用成千上万次**,每次重建路由表是纯浪费(路由匹配树的构建不便宜)。

> **更好的写法**:建一次,之后复用:
>
> ```rust
> use std::sync::OnceLock;
>
> fn router() -> Router {
>     static ROUTER: OnceLock<Router> = OnceLock::new();
>     ROUTER
>         .get_or_init(|| Router::new().route("/api/", get(index)))
>         .clone()      // Router 的 clone 是 Arc 引用计数,很便宜
> }
> ```
>
> **`Router` 的 `clone` 很便宜**(内部是 `Arc`,[第 44 章](44-serve-with-hyper.md#步骤-1手写第一个服务)说过),所以"建一次、每次 clone"的开销远小于"每次重建"。
>
>  但要注意 serverless 的**冷启动**:平台可能随时销毁并重建实例,`OnceLock` 里的东西不保证跨调用存活。**它是优化,不是缓存语义的保证。**

**没有真正的平台集成。** 这个示例只演示形状,没有连接任何一家 serverless 平台。真实项目里你需要:

- **AWS Lambda**:`lambda_http` crate,它提供了 tower 集成,可以直接 `lambda_http::run(app)`
- **Cloudflare Workers**:`worker` crate + `wasm-bindgen`
- **Fastly Compute**:`fastly` crate,目标是 `wasm32-wasip1`
- **Spin / wasmCloud**:WASI HTTP 接口

**这些 SDK 的共同点是:它们都接受一个 tower `Service`。** 所以本章学的东西是通用的。

**wasm 产物体积没有优化。** 步骤 2 提过,冷启动时间和体积直接相关,生产要配 `opt-level = "z"`、`lto`、`strip`,再过一遍 `wasm-opt`。

**`assert_eq!` 而不是真的测试。** 这是个演示程序,不是[第 21 章](21-testing.md)那种正经测试。想做成测试,`#[test] fn` 加上 `block_on` 就行——**而且这样的测试跑得飞快,因为完全没有 IO。**

## 动手练习

**1. 把它改成一个真正的测试。**

提示:把 `main` 里的逻辑挪进 `#[test] fn returns_200()`,再加几个用例:404 的路径、带 query 的请求、POST 到 GET-only 路由(应该是 405)。`cargo test` 跑一下——**注意有多快,因为一次 IO 都没有**。这也说明[第 21 章](21-testing.md)那种 `oneshot` 测法和本章是同一件事的两种写法。

**2. 用 `OnceLock` 缓存 `Router`。**

提示:按「代价与边界」的写法。**顺便想一想:如果 `Router` 带了 state([第 22 章](22-key-value-store.md)),`OnceLock` 里的 state 在 serverless 上意味着什么?**(答案:同一个实例的多次调用共享,冷启动后丢失——**所以绝对不能用它存必须持久的数据**。)

**3. 加回 `json` feature,做一个 JSON API。**

提示:`cargo add axum --no-default-features --features json`,加一条 `POST /api/echo` 路由收 `Json<T>` 返回 `Json<T>`。构造请求时 body 用 JSON 字符串,记得设 `Content-Type: application/json` 头。**每加一个 feature 就 `cargo build --target wasm32-unknown-unknown` 验证一次**,养成这个习惯。

**4. 用 `cargo tree` 找出所有平台相关的依赖。**

提示:

```bash
cargo tree --target wasm32-unknown-unknown
cargo tree --target wasm32-unknown-unknown -i mio      # 反查谁依赖了它
```

**对比一下开启和关闭 axum 默认 feature 时的依赖树大小。** 你会直观看到 `default-features = false` 砍掉了多少东西。

**5. 优化 wasm 体积。**

提示:在 `Cargo.toml` 的 `[profile.release]` 下加五个键:`opt-level = "z"`(为体积优化)、`lto = true`(跨 crate 链接期优化)、`codegen-units = 1`(单编译单元)、`strip = true`(剥离符号表)、`panic = "abort"`(panic 不展开栈)。对比优化前后的 `.wasm` 大小。再装个 `wasm-opt`(`binaryen` 包里)跑一遍 `wasm-opt -Oz`。**冷启动时间和这个数字直接相关。**

**6.(进阶)真正部署一次。**

提示:选一个平台走完全程。最容易上手的是 **AWS Lambda + `cargo lambda`**:

```bash
cargo install cargo-lambda
cargo add lambda_http
```

把 `main` 换成 `lambda_http::run(app).await`,`cargo lambda watch` 本地跑,`cargo lambda deploy` 部署。**你会发现前 53 章学的路由、提取器、中间件一行都不用改。**

## 小结与下一章预告

**`Router` 本质上只是一个函数:请求进,响应出。** 前 53 章里有 52 章在讲怎么把请求送到它面前——accept 连接、管理连接、跑 HTTP 协议。这一章把那些全删掉,证明了它们是可选的。

**serverless 和边缘计算平台要的正是这个形状**:你不能监听端口,平台唤起你的函数、你返回响应。axum 的 `Router` 原样就能用,**路由、提取器、中间件、`IntoResponse` 一样都不浪费**。

`default-features = false` 是必需的,因为 axum 的默认 feature 里有 `tokio`,而 tokio 的 IO 层 `mio` 不支持 `wasm32-unknown-unknown`。**但不该全关**——`json`、`query`、`form`、`matched-path` 这些是纯计算的,可以留着。步骤 4 验证了这一点。

步骤 3 那个实验值得记住:**打开默认 feature 之后,原生编译和运行完全正常,只有换目标平台才炸,而且报错来自一个你没听说过的传递依赖**。这是"默认 feature 拉进平台相关代码"这类问题的标准形态,排查工具是 `cargo tree -i <crate>`。

`futures_executor::block_on` 够用,是因为这里的 future **从头到尾不会真正挂起**——全是纯计算。**一旦 handler 里 `.await` 了真正需要等待的东西,它会死等**,因为没有唤醒机制。真实平台的 SDK 会提供自己的异步集成。

两个 wasm 目标的区别要分清:**`wasm32-unknown-unknown` 什么系统接口都没有**(时钟、文件、网络全无),`wasm32-wasip1` 有 WASI。需要随机数的库(`uuid` v4、`rand`)在前者上要显式开 `js` feature,否则运行时才 panic。

---

### 全书结语

**55 章,54 个官方示例,到这里全部走完了。**

回头看这条路线:

| 阶段 | 章节 | 你获得了什么 |
|---|---|---|
| **入门** | 00~05 | 从裸 `nc` 手敲 HTTP 请求开始,到路由、提取器、模板 |
| **请求与响应** | 06~13 | 提取器的完整机制,自定义提取器,body 只能读一次 |
| **错误处理与工程化** | 14~21 | 错误组织、依赖注入、tower 中间件、结构化日志、测试 |
| **状态与数据库** | 22~29 | 共享可变状态、四种数据库栈、完整 CRUD |
| **认证与安全** | 30~32 | JWT、OAuth、HTTPS |
| **实时通信** | 33~37 | SSE、WebSocket、聊天室 |
| **内容分发与代理** | 38~44 | 静态文件、压缩、流式、正向/反向代理、拆开 `axum::serve` |
| **生产部署** | 45~49 | 优雅停机、指标、版本化、热重载 |
| **专题** | 50~54 | 三种 TLS、Unix 域套接字、wasm |

**几条贯穿全书的主线,值得单独回顾:**

**① 类型系统是 axum 的核心。** `Handler`、`IntoResponse`、`FromRequest` / `FromRequestParts`、`FromRef` —— 这四个 trait 撑起了整个框架。理解它们比记住任何 API 都重要。

**② tower 的 `Service` 无处不在。** 从[第 17 章](17-tracing-aka-logging.md)第一次出现,到[第 21 章](21-testing.md)的测试、[第 42 章](42-http-proxy.md)的代理、[第 44 章](44-serve-with-hyper.md)的 `MakeService`,再到本章"`Router` 就是一个函数"——**这条线索串起了全书三分之一的内容**。

**③ 流式 vs 缓冲的取舍反复出现。** [第 09 章](09-consume-body-in-extractor-or-middleware.md)的 body、[第 13](13-multipart-form.md)/[40 章](40-stream-to-file.md)的上传、[第 33 章](33-sse.md)的 SSE、[第 41 章](41-reqwest-response.md)的转发、[第 39 章](39-compression.md)的压缩。**"要不要把整个东西读进内存"这个问题,你现在应该能条件反射地问出来。**

**④ 静默失败是最贵的 bug。** 全书设计了二十多个「故意改坏」的步骤,几乎每一个的共同点都是:**不报错、不 panic、只是结果悄悄不对**。漏掉 `.with_upgrades()`、漏掉 ALPN、漏掉 `set_nonblocking`、用错 `MatchedPath`、`{version}` 通配、`shutdown_future` 没人轮询……**养成"这个东西失败时会不会有声音"的思维习惯,比记住任何一个具体的坑都有用。**

**⑤ 官方示例不等于最佳实践。** 全书用 `> **更好的写法**` 标出了上百处:用错的状态码、丢掉的错误信息、不该抄的响应头、开放代理的 SSRF、`Html` 包纯文本、硬编码的 crate 名……**读别人的代码时保持这种审视,是从"会用"走向"用好"的关键一步。**

**接下来做什么?**

- **挑一个示例扩展成真项目。** [第 23 章](23-todos.md)的 todos、[第 37 章](37-chat.md)的 chat 都是好起点,把「代价与边界」里列的问题逐条补上。
- **把「动手练习」做完。** 全书两百多道,很多是把示例缺的东西补上——那正是真实项目要写的代码。
- **去读 axum 的源码。** 走完这 55 章之后,`Handler` 和 `FromRequest` 的实现不会再是天书。
- **关注 [axum 的 CHANGELOG](https://github.com/tokio-rs/axum/blob/main/axum/CHANGELOG.md)。** 本教程基于 0.8.x,而框架在演进——`serve` 的 `Listener` trait 就是 0.8 才有的。

祝写代码愉快。
