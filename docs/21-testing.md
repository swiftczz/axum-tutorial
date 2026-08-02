# 第 21 章 testing:不启动服务器也能测

## 本章你将做出什么

一套六个测试,覆盖同一个服务的不同测试手法:

```
running 6 tests
test tests::not_found ... ok
test tests::hello_world ... ok
test tests::json ... ok
test tests::multiple_request ... ok
test tests::the_real_deal ... ok
test tests::with_into_make_service_with_connect_info ... ok
```

**其中五个不启动任何服务器,不占用任何端口。** 因为[第 17 章](17-tracing-aka-logging.md#towerservice-与-layer)讲过的那件事:`Router` 本身就是一个 tower `Service`,你可以直接把请求"喂"给它。

这一章还清前面十几章欠下的债——那些"测试写法见第 21 章"的指路,终点在这里。

学完本章你将掌握:

- `oneshot`:一行代码对 `Router` 发一个请求
- 为什么发多个请求要用 `ready().call()` 而不是 `oneshot`
- 什么时候仍然需要启动真实服务器,以及怎么做
- `ConnectInfo` 这类依赖真实连接的提取器在测试里怎么模拟
- `fn app() -> Router` 这个模式的价值终于兑现

## 前置知识

### Router 是一个 Service

[第 17 章](17-tracing-aka-logging.md#towerservice-与-layer)提过一句:`axum::Router` 实现了 `tower::Service<Request<Body>>`。这不是细节,而是 axum 测试体验好的根本原因。

`Service` 的核心方法是:

```rust
fn call(&mut self, req: Request) -> Self::Future;
```

**输入一个 `Request`,输出一个 `Response`。** 全过程在内存里,不涉及 socket、不涉及端口、不涉及网络协议解析。

所以测试可以这样写:

```rust
let app = app();
let response = app.oneshot(request).await.unwrap();
assert_eq!(response.status(), StatusCode::OK);
```

对比其它语言常见的做法——起一个测试服务器、随机分配端口、用 HTTP 客户端连上去、测完关掉——省掉的不只是代码,还有:

**速度。** 没有 TCP 握手、没有报文序列化解析,一个测试是微秒级而不是毫秒级。

**可靠性。** 不会遇到"端口被占用"、不会有"服务器还没起来测试就开始了"的竞态,也不需要 `sleep` 等待。

**并行。** `cargo test` 默认并行跑测试,而这些测试之间没有任何共享资源(不抢端口),天然安全。

**这一层测的是什么。** 路由匹配、提取器、handler 逻辑、中间件、响应构造——**除了 HTTP 协议本身之外的一切**。协议层是 hyper 的责任,已经被 hyper 自己的测试覆盖了,你不需要重复验证。

### oneshot 与 ready().call()

两个方法来自 `tower::ServiceExt`,必须 `use tower::ServiceExt;` 才能调用(扩展 trait 的规矩,[第 09 章](09-consume-body-in-extractor-or-middleware.md#collect-与-bodyext)遇到过)。

**`oneshot`** 消耗 `self`,发一个请求:

```rust
pub fn oneshot(self, req: Request) -> Oneshot<Self, Request>
```

注意签名里是 `self` 而不是 `&mut self`——**用完就没了**。所以每个测试要么只发一个请求,要么重新 `app()` 一份。

**`ready().call()`** 借用 `&mut self`,可以连续用:

```rust
let mut app = app().into_service();
let r1 = app.ready().await.unwrap().call(req1).await.unwrap();
let r2 = app.ready().await.unwrap().call(req2).await.unwrap();
```

`ready()` 等待服务准备好(对应 `Service::poll_ready`,[第 17 章](17-tracing-aka-logging.md#towerservice-与-layer)提到的背压机制),`call()` 真正发请求。

**为什么不直接 `app.call(req)`?** 因为 tower 的约定是:调用 `call` 之前必须先确认 `poll_ready` 返回就绪。`Router` 永远是就绪的,所以这里 `ready()` 只是走个流程,但遵守约定能让代码在换成有背压的服务时仍然正确。

**`into_service()`** 把 `Router` 转成 `RouterIntoService`。`Router` 自身的 `Service` 实现要求 `&mut self` 但有些细节不便,官方在需要连续调用时统一用这个转换。

选择标准很简单:**发一个请求用 `oneshot`,发多个用 `ready().call()`**。

### 什么时候仍需真实服务器

内存测试覆盖不了这些:

- **HTTP 协议层行为** —— 报文格式、分块传输、keep-alive、HTTP/2 协商
- **真实客户端的行为** —— 某个客户端库发出的请求到底长什么样
- **`ConnectInfo`** —— 需要真实的 socket 地址
- **端到端集成** —— 从客户端到服务端的完整链路

这些场景下要起真实服务器。做法是绑定 `0.0.0.0:0`——**端口写 0 让操作系统分配一个空闲端口**,然后用 `local_addr()` 问出实际端口号。这样测试之间不会抢端口。

```rust
let listener = TcpListener::bind("0.0.0.0:0").await.unwrap();
let addr = listener.local_addr().unwrap();
tokio::spawn(async move { axum::serve(listener, app()).await.unwrap() });
// 然后用 HTTP 客户端连 addr
```

`tokio::spawn` 把服务器丢到后台任务。**不需要手动关闭**——测试函数结束时 tokio 运行时销毁,任务随之终止。

### MockConnectInfo

`ConnectInfo<SocketAddr>` 提取器给 handler 提供客户端地址。它的数据来源是真实连接,所以要靠 `into_make_service_with_connect_info` 启用:

```rust
axum::serve(listener, app.into_make_service_with_connect_info::<SocketAddr>())
```

**这在内存测试里用不了**——没有连接,哪来的地址。

axum 为此提供了 `MockConnectInfo`:

```rust
let app = app().layer(MockConnectInfo(SocketAddr::from(([0, 0, 0, 0], 3000))));
```

它是一个 `Layer`,往请求扩展里塞一个假的地址,让 `ConnectInfo` 提取器能正常工作。**这是"为可测试性提供的专门设施"的一个好例子**——框架意识到某个功能难测,就提供一个测试替身。

### fn app() -> Router 的价值兑现

从[第 03 章](03-form.md#步骤-1显示表单页面)起,好几章的示例都有这个函数,当时只说"为测试服务,第 21 章你会看到价值"。现在兑现:

```rust
/// Having a function that produces our app makes it easy to call it from tests
/// without having to create an HTTP server.
fn app() -> Router {
    Router::new()
        .route("/", get(|| async { "Hello, World!" }))
        // ...
}
```

如果路由构建写在 `main` 里,测试就没法复用它——只能在测试里再抄一遍路由表,然后两边慢慢不一致。抽成函数之后,`main` 和测试用的是**同一份**路由定义。

这是个通用的可测试性原则:**把"构造对象"和"运行它"分开**。`main` 负责运行,`app()` 负责构造,测试只需要构造。

## 项目搭建

```bash
cargo new testing
cd testing
```

运行时依赖:

```bash
cargo add axum
cargo add tokio --features full
cargo add tower-http --features trace
cargo add tracing
cargo add tracing-subscriber --features env-filter
cargo add serde_json
cargo add mime
cargo add hyper-util --features client,http1,client-legacy
```

**开发依赖**(只在测试时编译,用 `--dev`):

```bash
cargo add tower --dev --features util
```

`tower` 放在 `[dev-dependencies]` 里,因为 `ServiceExt` 只有测试用得到。这样正式构建不会带上它。

`hyper-util` 提供测试里那个真实 HTTP 客户端。`mime` 用来写 `Content-Type` 常量,避免手写字符串。

## 分步构建

### 步骤 1:被测的服务

**要做什么**:先写出要测的应用,注意路由构建抽成了 `app()`。

```rust
// src/main.rs
use std::net::SocketAddr;

use axum::{
    extract::ConnectInfo,
    routing::{get, post},
    Json, Router,
};
use tower_http::trace::TraceLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

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

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app()).await.unwrap();
}

/// Having a function that produces our app makes it easy to call it from tests
/// without having to create an HTTP server.
fn app() -> Router {
    Router::new()
        .route("/", get(|| async { "Hello, World!" }))
        .route(
            "/json",
            post(|payload: Json<serde_json::Value>| async move {
                Json(serde_json::json!({ "data": payload.0 }))
            }),
        )
        .route(
            "/requires-connect-info",
            get(|ConnectInfo(addr): ConnectInfo<SocketAddr>| async move { format!("Hi {addr}") }),
        )
        // We can still add middleware
        .layer(TraceLayer::new_for_http())
}
```

三条路由各有测试意图:

- **`/`** — 最简单的情况,验证基本流程
- **`/json`** — 需要构造带 body 和 `Content-Type` 的请求
- **`/requires-connect-info`** — 依赖 `ConnectInfo`,专门用来演示 `MockConnectInfo`

**注意 `main` 里 `axum::serve(listener, app())`** 直接调用 `app()`,没有中间变量。`main` 只管运行,构造交给 `app()`。

**`.layer(TraceLayer::new_for_http())`** 那句注释 "We can still add middleware" 是个重要提示:**中间件也会被测到**。`oneshot` 走的是完整的 `Service` 调用链,所有 layer 都会执行。这和"只调用 handler 函数"的单元测试不同——那种测法会绕过中间件。

**跑起来看**:

```bash
cargo run
```

```bash
curl -s http://127.0.0.1:3000/
curl -s -X POST http://127.0.0.1:3000/json -H 'Content-Type: application/json' -d '[1,2,3]'
```

```
Hello, World!
{"data":[1,2,3]}
```

> **更好的写法**: 官方示例的 `main` 函数在启动服务器时使用 `axum::serve(listener, app)`，未传入 `.into_make_service_with_connect_info::<SocketAddr>()`，导致生产环境中 `/requires-connect-info` 路由因缺少连接信息而返回 500。虽然测试中可以使用 `MockConnectInfo` 让单元测试通过，但生产服务中必须写为 `axum::serve(listener, app.into_make_service_with_connect_info::<SocketAddr>())`。官方示例保留原样是为了展示单元测试中使用 `MockConnectInfo` 替代真实连接信息的用法，实际项目中请务必保持生产启动代码与 handler 提取器需求一致。

第三条路由现在会失败——`ConnectInfo` 需要 `into_make_service_with_connect_info` 才能工作,而 `main` 里没用:

```bash
curl -i -s http://127.0.0.1:3000/requires-connect-info | head -1
```

```
HTTP/1.1 500 Internal Server Error
```

**这个失败是正常的**,官方示例的 `main` 就没启用它。步骤 4 会在测试里用 `MockConnectInfo` 让它工作。

### 步骤 2:三个 oneshot 测试

**要做什么**:写最基础的三个测试,全部不启动服务器。

在 `src/main.rs` 末尾追加:

```rust
// src/main.rs

#[cfg(test)]
mod tests {
    use super::*;
    use axum::{
        body::Body,
        http::{self, Request, StatusCode},
    };
    use http_body_util::BodyExt; // for `collect`
    use serde_json::{json, Value};
    use tower::ServiceExt; // for `oneshot`

    #[tokio::test]
    async fn hello_world() {
        let app = app();

        // `Router` implements `tower::Service<Request<Body>>` so we can
        // call it like any tower service, no need to run an HTTP server.
        let response = app
            .oneshot(Request::get("/").body(Body::empty()).unwrap())
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::OK);

        let body = response.into_body().collect().await.unwrap().to_bytes();
        assert_eq!(&body[..], b"Hello, World!");
    }

    #[tokio::test]
    async fn json() {
        let app = app();

        let response = app
            .oneshot(
                Request::post("/json")
                    .header(http::header::CONTENT_TYPE, mime::APPLICATION_JSON.as_ref())
                    .body(Body::from(
                        serde_json::to_vec(&json!([1, 2, 3, 4])).unwrap(),
                    ))
                    .unwrap(),
            )
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::OK);

        let body = response.into_body().collect().await.unwrap().to_bytes();
        let body: Value = serde_json::from_slice(&body).unwrap();
        assert_eq!(body, json!({ "data": [1, 2, 3, 4] }));
    }

    #[tokio::test]
    async fn not_found() {
        let app = app();

        let response = app
            .oneshot(Request::get("/does-not-exist").body(Body::empty()).unwrap())
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::NOT_FOUND);
        let body = response.into_body().collect().await.unwrap().to_bytes();
        assert!(body.is_empty());
    }
}
```

需要把 `http-body-util` 加进依赖:

```bash
cargo add http-body-util
```

逐块看。

**`#[cfg(test)] mod tests`** 是 Rust 的单元测试惯例:整个模块只在 `cargo test` 时编译,正式构建完全不包含。

**`use super::*`** 把父模块的项(包括 `app`)导入进来。

**`#[tokio::test]`** 是 `#[test]` 的异步版本,自动为每个测试创建一个运行时。默认是单线程运行时,需要多线程可以写 `#[tokio::test(flavor = "multi_thread")]`。

**`Request::get("/").body(Body::empty()).unwrap()`** 构造请求。`Request::get(...)` 返回一个 builder,`.body(...)` 收尾。`Body::empty()` 表示无请求体。

**`response.into_body().collect().await.unwrap().to_bytes()`** 读出响应体。这正是[第 09 章](09-consume-body-in-extractor-or-middleware.md#collect-与-bodyext)讲的 `BodyExt::collect`——**测试里读 body 和中间件里读 body 用的是同一套工具**。

**`mime::APPLICATION_JSON.as_ref()`** 比手写 `"application/json"` 好:拼错了编译不过。[第 08 章](08-parse-body-based-on-content-type.md#步骤-2实现-jsonorform)提过用 `mime` crate 正确处理媒体类型。

**`not_found` 那个测试断言 body 是空的。** 这印证了[第 06 章](06-global-404-handler.md#步骤-1一个正常的服务)看到的默认 404 行为——没配 fallback 时 body 为空。**测试在这里起到了"锁定当前行为"的作用**:哪天有人加了 fallback,这个测试会失败,提醒他确认这是有意的改动。

**跑起来看**:

```bash
cargo test
```

```
running 3 tests
test tests::not_found ... ok
test tests::hello_world ... ok
test tests::json ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**`finished in 0.00s`** —— 三个测试加起来不到 10 毫秒,而且没占用任何端口。

**验证测试确实在测真东西。** 临时把 `/` 的返回改成 `"Hello, Rust!"`:

```rust
        .route("/", get(|| async { "Hello, Rust!" }))
```

```bash
cargo test
```

```
test tests::hello_world ... FAILED

failures:

---- tests::hello_world stdout ----
thread 'tests::hello_world' panicked at src/main.rs:...:
assertion `left == right` failed
  left: [72, 101, 108, 108, 111, 44, 32, 82, 117, 115, 116, 33]
  right: [72, 101, 108, 108, 111, 44, 32, 87, 111, 114, 108, 100, 33]
```

**注意断言失败的输出是字节数组,很难读。** 因为 `assert_eq!(&body[..], b"Hello, World!")` 比较的是 `&[u8]`。改用字符串比较会好得多:

```rust
assert_eq!(std::str::from_utf8(&body).unwrap(), "Hello, World!");
```

这样失败信息会直接显示两个字符串。**官方示例用字节比较是为了少一层转换,但实际项目里可读的失败信息更值钱**——排查失败的测试时你会感谢自己。

改回 `"Hello, World!"`。

### 步骤 3:连续发多个请求

**要做什么**:验证 `oneshot` 消耗所有权的限制,然后用 `ready().call()` 解决。

先看限制。在 `hello_world` 测试里试着发第二个请求:

```rust
    #[tokio::test]
    async fn hello_world() {
        let app = app();

        let response = app
            .oneshot(Request::get("/").body(Body::empty()).unwrap())
            .await
            .unwrap();
        assert_eq!(response.status(), StatusCode::OK);

        // 临时加:再发一个
        let response2 = app
            .oneshot(Request::get("/").body(Body::empty()).unwrap())
            .await
            .unwrap();
    }
```

```bash
cargo test
```

```
error[E0382]: use of moved value: `app`
   |
   |         let app = app();
   |             --- move occurs because `app` has type `Router`, which does not implement the `Copy` trait
   |         let response = app
   |                        --- `app` moved due to this method call
   ...
   |         let response2 = app
   |                         ^^^ value used here after move
```

**`oneshot` 拿走了所有权。** 前置知识说过它的签名是 `fn oneshot(self, ...)`。

删掉临时代码,改用 `ready().call()` 写一个新测试:

```rust
// src/main.rs 的 tests 模块内
    use tower::{Service, ServiceExt}; // 改动:加上 Service

    // ...(前三个测试不变)

    // 新增:You can use `ready()` and `call()` to avoid using `clone()`
    // in multiple request
    #[tokio::test]
    async fn multiple_request() {
        let mut app = app().into_service();

        let request = Request::get("/").body(Body::empty()).unwrap();
        let response = ServiceExt::<Request<Body>>::ready(&mut app)
            .await
            .unwrap()
            .call(request)
            .await
            .unwrap();
        assert_eq!(response.status(), StatusCode::OK);

        let request = Request::get("/").body(Body::empty()).unwrap();
        let response = ServiceExt::<Request<Body>>::ready(&mut app)
            .await
            .unwrap()
            .call(request)
            .await
            .unwrap();
        assert_eq!(response.status(), StatusCode::OK);
    }
```

**`use tower::{Service, ServiceExt}`** —— `call` 来自 `Service`,`ready` 来自 `ServiceExt`,两个都要导入。

**`ServiceExt::<Request<Body>>::ready(&mut app)` 这个写法很别扭。** 为什么不能直接写 `app.ready()`?

因为 `RouterIntoService` 对多种请求类型都实现了 `Service`,编译器无法推断你想用哪个。用完全限定语法显式指定 `Request<Body>` 消除歧义。

**这是官方示例里最难读的一行**,但它的存在有意义:让你知道遇到"多个 impl 无法推断"时该怎么办。多数情况下 `app.ready()` 就能工作,只在编译器报歧义时才需要这么写。

**`into_service()`** 见前置知识。

**跑起来看**:

```bash
cargo test
```

```
running 4 tests
test tests::not_found ... ok
test tests::hello_world ... ok
test tests::json ... ok
test tests::multiple_request ... ok
```

### 步骤 4:真实服务器与 MockConnectInfo

**要做什么**:补上最后两个测试——一个启动真实服务器做端到端验证,一个用 `MockConnectInfo` 测那条依赖连接信息的路由。

```rust
// src/main.rs 的 tests 模块内
    use axum::extract::connect_info::MockConnectInfo;   // 新增
    use tokio::net::TcpListener;                        // 新增

    // ...(前四个测试不变)

    // 新增:You can also spawn a server and talk to it like any other HTTP server:
    #[tokio::test]
    async fn the_real_deal() {
        let listener = TcpListener::bind("0.0.0.0:0").await.unwrap();
        let addr = listener.local_addr().unwrap();

        tokio::spawn(async move {
            axum::serve(listener, app()).await.unwrap();
        });

        let client =
            hyper_util::client::legacy::Client::builder(hyper_util::rt::TokioExecutor::new())
                .build_http();

        let response = client
            .request(
                Request::get(format!("http://{addr}"))
                    .header("Host", "localhost")
                    .body(Body::empty())
                    .unwrap(),
            )
            .await
            .unwrap();

        let body = response.into_body().collect().await.unwrap().to_bytes();
        assert_eq!(&body[..], b"Hello, World!");
    }

    // 新增:Here we're calling `/requires-connect-info` which requires `ConnectInfo`
    //
    // That is normally set with `Router::into_make_service_with_connect_info` but we can't easily
    // use that during tests. The solution is instead to set the `MockConnectInfo` layer during
    // tests.
    #[tokio::test]
    async fn with_into_make_service_with_connect_info() {
        let mut app = app()
            .layer(MockConnectInfo(SocketAddr::from(([0, 0, 0, 0], 3000))))
            .into_service();

        let request = Request::get("/requires-connect-info")
            .body(Body::empty())
            .unwrap();
        let response = app.ready().await.unwrap().call(request).await.unwrap();
        assert_eq!(response.status(), StatusCode::OK);
    }
```

**`the_real_deal` 里的 `bind("0.0.0.0:0")`** 是前置知识讲的技巧:端口 0 让系统分配,`local_addr()` 问出实际值。**这样多个测试并行跑也不会抢端口。**

`tokio::spawn` 之后**没有任何同步等待**就直接发请求了。这能工作是因为 `TcpListener::bind` 已经完成——操作系统开始接受连接并放进队列,即使 `axum::serve` 还没开始 accept,客户端的连接也不会被拒绝。**这是个容易写错的地方**:如果先 spawn 再 bind,就需要等待或重试了。

`hyper_util::client::legacy::Client` 是 hyper 的 HTTP 客户端。名字里有 `legacy` 是因为 hyper 1.0 把客户端拆到了 `hyper-util` 并标记为过渡实现。真实项目里更常用 `reqwest`([第 41 章](41-reqwest-response.md)会用到),它基于 hyper 但 API 友好得多。

**`.header("Host", "localhost")`** 是必需的:HTTP/1.1 要求请求必须带 `Host` 头部。用高层客户端时它会自动加,这里手工构造请求就得自己写。

**注意这个测试也可以用 `oneshot` 写**——它测的 `/` 路由不需要真实连接。官方示例保留它是为了**演示"需要时怎么做"**,不是说这个场景必须这样测。

**`with_into_make_service_with_connect_info` 里的 `MockConnectInfo`** 就是前置知识讲的测试替身:塞一个假地址进去,让 `ConnectInfo` 提取器能工作。

注释解释得很清楚:正常要用 `into_make_service_with_connect_info`,但那个在测试里不好用(它返回的是 `MakeService` 而不是 `Service`,不能直接 `oneshot`),所以用 `MockConnectInfo` 替代。

这里 `app.ready()` 能直接写(不用完全限定语法),因为类型上下文足够明确。

**跑起来看**:

```bash
cargo test
```

```
running 6 tests
test tests::not_found ... ok
test tests::hello_world ... ok
test tests::json ... ok
test tests::multiple_request ... ok
test tests::with_into_make_service_with_connect_info ... ok
test tests::the_real_deal ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

**六个测试,0.01 秒。** 其中五个完全在内存里跑。

看单个测试的输出:

```bash
cargo test hello_world -- --nocapture
```

`--nocapture` 让 `println!` 和日志输出显示出来(默认被捕获)。

**验证 `MockConnectInfo` 确实在起作用。** 把那个 layer 临时注释掉:

```rust
        let mut app = app()
            // .layer(MockConnectInfo(SocketAddr::from(([0, 0, 0, 0], 3000))))
            .into_service();
```

```bash
cargo test with_into_make_service
```

```
test tests::with_into_make_service_with_connect_info ... FAILED

  left: 500
 right: 200
```

**`500`** —— `ConnectInfo` 提取器找不到连接信息,提取失败。这和步骤 1 里 curl 那条路由得到 500 是同一个原因。恢复那一行。

## 完整代码

`src/main.rs`:

```rust
use std::net::SocketAddr;

use axum::{
    extract::ConnectInfo,
    routing::{get, post},
    Json, Router,
};
use tower_http::trace::TraceLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

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

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app()).await.unwrap();
}

/// Having a function that produces our app makes it easy to call it from tests
/// without having to create an HTTP server.
fn app() -> Router {
    Router::new()
        .route("/", get(|| async { "Hello, World!" }))
        .route(
            "/json",
            post(|payload: Json<serde_json::Value>| async move {
                Json(serde_json::json!({ "data": payload.0 }))
            }),
        )
        .route(
            "/requires-connect-info",
            get(|ConnectInfo(addr): ConnectInfo<SocketAddr>| async move { format!("Hi {addr}") }),
        )
        // We can still add middleware
        .layer(TraceLayer::new_for_http())
}

#[cfg(test)]
mod tests {
    use super::*;
    use axum::{
        body::Body,
        extract::connect_info::MockConnectInfo,
        http::{self, Request, StatusCode},
    };
    use http_body_util::BodyExt; // for `collect`
    use serde_json::{json, Value};
    use tokio::net::TcpListener;
    use tower::{Service, ServiceExt}; // for `call`, `oneshot`, and `ready`

    #[tokio::test]
    async fn hello_world() {
        let app = app();

        // `Router` implements `tower::Service<Request<Body>>` so we can
        // call it like any tower service, no need to run an HTTP server.
        let response = app
            .oneshot(Request::get("/").body(Body::empty()).unwrap())
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::OK);

        let body = response.into_body().collect().await.unwrap().to_bytes();
        assert_eq!(&body[..], b"Hello, World!");
    }

    #[tokio::test]
    async fn json() {
        let app = app();

        let response = app
            .oneshot(
                Request::post("/json")
                    .header(http::header::CONTENT_TYPE, mime::APPLICATION_JSON.as_ref())
                    .body(Body::from(
                        serde_json::to_vec(&json!([1, 2, 3, 4])).unwrap(),
                    ))
                    .unwrap(),
            )
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::OK);

        let body = response.into_body().collect().await.unwrap().to_bytes();
        let body: Value = serde_json::from_slice(&body).unwrap();
        assert_eq!(body, json!({ "data": [1, 2, 3, 4] }));
    }

    #[tokio::test]
    async fn not_found() {
        let app = app();

        let response = app
            .oneshot(Request::get("/does-not-exist").body(Body::empty()).unwrap())
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::NOT_FOUND);
        let body = response.into_body().collect().await.unwrap().to_bytes();
        assert!(body.is_empty());
    }

    // You can also spawn a server and talk to it like any other HTTP server:
    #[tokio::test]
    async fn the_real_deal() {
        let listener = TcpListener::bind("0.0.0.0:0").await.unwrap();
        let addr = listener.local_addr().unwrap();

        tokio::spawn(async move {
            axum::serve(listener, app()).await.unwrap();
        });

        let client =
            hyper_util::client::legacy::Client::builder(hyper_util::rt::TokioExecutor::new())
                .build_http();

        let response = client
            .request(
                Request::get(format!("http://{addr}"))
                    .header("Host", "localhost")
                    .body(Body::empty())
                    .unwrap(),
            )
            .await
            .unwrap();

        let body = response.into_body().collect().await.unwrap().to_bytes();
        assert_eq!(&body[..], b"Hello, World!");
    }

    // You can use `ready()` and `call()` to avoid using `clone()`
    // in multiple request
    #[tokio::test]
    async fn multiple_request() {
        let mut app = app().into_service();

        let request = Request::get("/").body(Body::empty()).unwrap();
        let response = ServiceExt::<Request<Body>>::ready(&mut app)
            .await
            .unwrap()
            .call(request)
            .await
            .unwrap();
        assert_eq!(response.status(), StatusCode::OK);

        let request = Request::get("/").body(Body::empty()).unwrap();
        let response = ServiceExt::<Request<Body>>::ready(&mut app)
            .await
            .unwrap()
            .call(request)
            .await
            .unwrap();
        assert_eq!(response.status(), StatusCode::OK);
    }

    // Here we're calling `/requires-connect-info` which requires `ConnectInfo`
    //
    // That is normally set with `Router::into_make_service_with_connect_info` but we can't easily
    // use that during tests. The solution is instead to set the `MockConnectInfo` layer during
    // tests.
    #[tokio::test]
    async fn with_into_make_service_with_connect_info() {
        let mut app = app()
            .layer(MockConnectInfo(SocketAddr::from(([0, 0, 0, 0], 3000))))
            .into_service();

        let request = Request::get("/requires-connect-info")
            .body(Body::empty())
            .unwrap();
        let response = app.ready().await.unwrap().call(request).await.unwrap();
        assert_eq!(response.status(), StatusCode::OK);
    }
}
```

## 代价与边界

**内存测试测不到 HTTP 协议层。** `oneshot` 直接构造 `Request` 对象,跳过了报文解析。所以这些测试**不会发现**协议层面的问题:头部编码、分块传输、超大 URL、畸形报文。这些通常不是你的代码负责的(hyper 管),但如果你写了操作原始字节的中间件,就需要真实服务器测试。

**测试里的日志会打乱输出。** `TraceLayer` 在测试中同样工作,而 `tracing_subscriber` 的 `.init()` [第 17 章](17-tracing-aka-logging.md#代价与边界)说过全局只能调一次——多个测试各自初始化会 panic。本章示例没在测试里初始化订阅器,所以日志静默丢弃了。需要看日志时用 `tracing_subscriber::fmt().with_test_writer().try_init()`(`try_init` 失败返回 `Err` 而不 panic)。

**共享状态会让测试互相影响。** 本章的 `app()` 没有状态。一旦引入 `State`(比如[第 16 章](16-dependency-injection.md)那个 `InMemoryUserRepo`),就要注意:如果 `app()` 内部创建状态,每个测试拿到的是独立实例(好);如果状态是全局的或从外部传入同一份,测试之间就会串数据(坏)。**并行执行会让这类问题表现得随机而难查。**

**`b"Hello, World!"` 这种字节断言的失败信息很难读。** 步骤 2 演示过。实际项目建议先转成字符串再比较。

**没有测中间件的顺序。** 本章测试能验证"中间件生效了",但验证不了[第 19 章](19-request-id.md)那种"多个中间件的顺序对不对"。那需要构造更精细的断言,比如检查特定响应头是否存在。

**测试替身可能掩盖真实问题。** `MockConnectInfo` 让测试通过了,但它验证不了"生产环境真的调用了 `into_make_service_with_connect_info`"——本章示例的 `main` 其实就没调用,那条路由在生产环境是坏的(步骤 1 里 curl 到的 500)。**测试绿了不等于功能可用**,这是个很好的反面案例。

## 回头补上前面章节的测试

全书共有 11 个官方示例包含测试，其中在第 21 章之前涉及 5 个（当时按"第 21 章再讲"跳过了）。现在你有了全部工具,可以回头把它们读一遍——它们用的都是本章这几种模式:

| 章节 | 测试内容 | 用到的手法 |
|---|---|---|
| [03 form](03-form.md) | 表单页面渲染、表单提交解析 | `oneshot` + 构造 urlencoded body |
| [04 templates](04-templates.md) | 模板渲染结果 | `oneshot` + 断言 HTML |
| [07 handle-head-request](07-handle-head-request.md) | `GET` 有 body、`HEAD` 无 body | `oneshot` + `Request::head` |
| [12 validator](12-validator.md) | 四种输入的校验结果 | `oneshot` + 查询字符串 |
| [15 anyhow-error-response](15-anyhow-error-response.md) | 错误响应的状态码和 body | `oneshot` + 断言 500 |

**建议做法**:挑一两章回去看它的测试模块,对照本章学的内容读一遍。你会发现没有新东西——都是 `oneshot`、`collect`、`assert_eq!` 的组合。

## 动手练习

**1. 把字节断言改成字符串断言。**

提示:把 `assert_eq!(&body[..], b"Hello, World!")` 改成先 `std::str::from_utf8(&body).unwrap()` 再比较。然后故意改坏返回值,对比两种写法的失败信息。步骤 2 演示过差别有多大。

**2. 给 `/json` 补一个失败路径的测试。**

提示:发一个格式错误的 JSON(`Body::from("{bad")`),断言状态码是 `400`。再发一个不带 `Content-Type` 的,断言是 `415`。这两个状态码[第 08 章](08-parse-body-based-on-content-type.md#json收-json-与发-json)讲过。**测试错误路径往往比测试成功路径更有价值**——成功路径你手工点一次就发现问题了,错误路径容易被忽略。

**3. 给一个带 `State` 的服务写测试,验证状态隔离。**

提示:参照[第 16 章](16-dependency-injection.md)的 `InMemoryUserRepo`,让 `app()` 内部创建状态。写两个测试:一个创建用户、一个查询用户。**如果状态是每次 `app()` 新建的,第二个测试查不到第一个创建的用户**——这正是你想要的隔离。再试试把状态提到 `app()` 外面传进去,观察测试之间开始互相影响,而且并行执行下结果不稳定。

## 小结与下一章预告

`Router` 实现了 `tower::Service`,所以**不启动服务器、不占端口就能对它发请求**。`oneshot` 发一个请求(消耗所有权),`ready().call()` 可以连续发多个。这一层能测到路由匹配、提取器、handler、中间件、响应构造——除 HTTP 协议本身之外的一切,而且是微秒级、可并行、无竞态。

`fn app() -> Router` 这个从[第 03 章](03-form.md)就出现的模式,价值就在这里:**`main` 和测试共用同一份路由定义**,不会各写一份然后慢慢不一致。

需要真实服务器时(协议层行为、真实客户端、端到端),绑定 `0.0.0.0:0` 让系统分配端口,`tokio::spawn` 丢到后台,测试结束自动清理。`ConnectInfo` 这类依赖真实连接的提取器,用 `MockConnectInfo` 提供测试替身。

要留意的边界:测试替身可能掩盖真实问题(本章示例的 `main` 就没启用 `into_make_service_with_connect_info`,那条路由在生产环境是坏的,但测试是绿的);共享状态会让并行测试互相影响,而且表现随机;字节数组断言的失败信息很难读。

「错误处理与工程化」阶段到此结束。你已经能把一个 axum 应用组织得像样:错误有统一出口、日志有请求上下文、依赖可替换、代码可测试。

但所有这些章节的数据都是临时的——重启进程就没了。[第 22 章 key-value-store](22-key-value-store.md) 开始「状态与数据库」阶段,先把[第 05 章](05-templates-minijinja.md#state-与-with_state)欠下的债还上:**共享可变状态**。`Mutex` 和 `RwLock` 怎么选、`.lock().unwrap()` 为什么会 panic、以及一个能让服务彻底卡死的经典陷阱——持锁跨 `await`。
