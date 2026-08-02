# 第 01 章 hello-world:第一个 axum 服务

## 本章你将做出什么

一个 14 行的 axum 服务,访问 `http://127.0.0.1:3000/` 返回一段 HTML 标题。

功能上它和[第 00 章](00-web-basics.md)手写的服务器差不多,但代码量少一半,而且并发处理、请求解析、`Content-Length` 计算全都不用你管。

学完本章你将掌握:

- axum 的三个核心构件——`Router`、handler、`axum::serve`——各自负责什么
- 为什么一个普通的 `async fn` 能直接当 HTTP 处理器:`Handler` trait 的作用
- 返回值如何自动变成 HTTP 响应:`IntoResponse` 的作用,以及它如何替你设置 `Content-Type`

## 前置知识

### async 运行时与 tokio

Rust 的 `async fn` 本身不会自动执行。调用它只是得到一个 `Future`,这个 `Future` 需要交给**运行时**(runtime)去轮询、驱动、在等待 IO 时调度别的任务。标准库不提供运行时,得用第三方的,事实标准是 **tokio**。

Web 服务是 IO 密集型的典型场景:大部分时间在等网络。同步模型下每个连接占一个线程,几千个连接就把内存和上下文切换开销吃光了;async 模型下一个线程可以驱动成千上万个连接,谁的数据到了就处理谁。这是[第 00 章](00-web-basics.md#这个服务器还差什么)提到的"并发"缺口的答案。

`#[tokio::main]` 这个属性宏会把你的 `async fn main` 包一层:创建运行时,把 `main` 的 `Future` 扔进去跑。写它就相当于写了创建运行时的样板代码。

axum 是构建在 tokio 生态之上的,所以 tokio 是必需依赖。

### axum 在这个生态里的位置

从底往上看是四层:

- **tokio** — async 运行时和异步 IO(`tokio::net::TcpListener`)
- **hyper** — HTTP 协议实现,负责报文解析和序列化,也就是你在[第 00 章](00-web-basics.md)手写的那部分
- **tower** — 通用的服务抽象层,定义了 `Service` 和 `Layer` 两个 trait,中间件生态建在这上面
- **axum** — 路由、extractor、响应转换,让你用普通 Rust 函数写业务逻辑

axum 自己不实现 HTTP 协议,它把 hyper 包起来,给你一套好用的 API。这个分层意味着后面遇到需要底层控制的场合([第 44 章](44-serve-with-hyper.md) serve-with-hyper、第 50~52 章的底层 TLS),你可以掀开 axum 直接操作 hyper。

> **本章不深入**:tower 的 `Service` 和 `Layer` 是 axum 中间件体系的地基,日志、压缩、CORS、超时这些能力都靠它。本章还用不到,知道它在分层里的位置就够了。它的组合语义和执行顺序见[第 17 章](17-tracing-aka-logging.md)。

## 项目搭建

```bash
cargo new hello-world
cd hello-world
```

装两个依赖:

```bash
cargo add axum
cargo add tokio --features full
```

`tokio` 的 `full` feature 打开全部功能。生产项目里通常按需精简(比如只要 `rt-multi-thread`、`net`、`macros`),但教程阶段图省事,官方示例也是这么写的。

`axum` 不带 feature 就够本章用了。它的默认 feature 已经包含 `http1`、`json`、`tokio` 等常用项;后面章节需要额外 feature 时会明确说明(比如[第 34 章](34-websockets.md)的 `ws`)。

装完检查一下:

```bash
cargo tree --depth 1
```

应该能看到 `axum v0.8.x` 和 `tokio v1.x`。

## 分步构建

### 步骤 1:能跑起来的最小服务

**要做什么**:起一个 axum 服务,访问 `/` 返回一行纯文本。

把 `src/main.rs` 全部替换成:

```rust
// src/main.rs
use axum::{routing::get, Router};

#[tokio::main]
async fn main() {
    // 构建应用:一条路由
    let app = Router::new().route("/", get(handler));

    // 跑起来
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> &'static str {
    "Hello, World!"
}
```

十四行。逐块拆开看它们各自在干什么。

**`Router::new().route("/", get(handler))`**

`Router` 是 axum 的路由表:记录"什么路径 + 什么方法 → 交给哪个处理器"。`route` 接两个参数,第一个是路径,第二个是 `MethodRouter`。

`get(handler)` 这个调用构造出一个 `MethodRouter`,含义是"`GET` 方法交给 `handler` 处理"。同一路径上如果还要支持 `POST`,可以链式写成 `get(handler).post(other_handler)`——`MethodRouter` 是按方法分发的第二层路由。[第 02 章](02-routes-and-handlers-close-together.md)会用到这个特性。

这里可以对照[第 00 章](00-web-basics.md)的 `match (method, path)`:axum 把那个手写的两级分发拆成了两层类型——`Router` 管路径,`MethodRouter` 管方法。好处是路径匹配可以用高效的前缀树算法,方法不匹配时能自动返回正确的 `405`,不用你手写。

**`tokio::net::TcpListener::bind(...)`**

和[第 00 章](00-web-basics.md)的 `std::net::TcpListener::bind` 是同一件事,只是异步版本,所以要 `.await`。第一个 `.unwrap()` 处理绑定失败(最常见的原因是端口被占用)。

注意监听器是你自己创建、自己传给 axum 的,axum 不代你 bind。这个设计让你可以完全控制监听行为——换成 Unix socket([第 53 章](53-unix-domain-socket.md))、套一层 TLS([第 32 章](32-tls-rustls.md))、或者从 systemd 接管已有的 fd,都是替换这一行的事。

**`axum::serve(listener, app).await.unwrap()`**

把监听器和路由表接起来,开始接受连接。这个 `Future` 正常情况下永不完成,所以 `.await` 会一直阻塞在这里——服务器就该这样。

末尾的 `.unwrap()` 是必要的:`serve` 的 `Future` 解析出 `io::Result<()>`,不处理会有 `unused_must_use` 警告。文档里说明了它实际上永远不会返回错误(TCP 层面的错误会内部重试),但类型上仍是 `Result`。

> 如果你翻过 axum 仓库里的官方示例,会看到那里写的是 `axum::serve(listener, app).await;` 没有 `.unwrap()`。那是 main 分支上尚未发布的改动,对着 crates.io 上的 0.8.9 编译会有警告。本教程按已发布版本写。

**`async fn handler() -> &'static str`**

这就是**处理器**(handler)——一个普通的异步函数,返回一个字符串引用。没有实现任何 trait,没有继承任何基类,没有注解。

它凭什么能塞进 `get()` 里?这是 axum 最值得理解的设计,下一步骤会讲透。

**跑起来看**:

```bash
cargo run
```

```
listening on 127.0.0.1:3000
```

另开终端:

```bash
curl -i http://127.0.0.1:3000/
```

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 13
date: Thu, 30 Jul 2026 06:12:34 GMT

Hello, World!
```

和[第 00 章](00-web-basics.md)手写的响应几乎一样,但这次头部是 axum 生成的:

- `content-type: text/plain; charset=utf-8` — 你没写这个头部,axum 根据返回类型 `&'static str` 推断出来的
- `content-length: 13` — 你没算这个数,axum 数的
- `date` — HTTP 规范建议的头部,hyper 自动加的
- 头部名是小写的。HTTP 头部名大小写不敏感,HTTP/2 更是要求必须小写,所以 hyper 统一用小写

再试试[第 00 章](00-web-basics.md)留下的那个遗憾:

```bash
curl -i -X POST http://127.0.0.1:3000/
```

```
HTTP/1.1 405 Method Not Allowed
allow: GET
content-length: 0
date: Thu, 30 Jul 2026 06:12:51 GMT
```

**`405 Method Not Allowed`,而且带上了 `allow: GET`。** [第 00 章](00-web-basics.md)练习题 2 让你手写的逻辑,axum 白送。它知道 `/` 这个路径存在(`Router` 匹配上了),只是 `MethodRouter` 里没有 `POST` 分支,于是给出符合规范的回答。

不存在的路径依然是 404:

```bash
curl -i http://127.0.0.1:3000/nope
```

```
HTTP/1.1 404 Not Found
content-length: 0
date: Thu, 30 Jul 2026 06:13:02 GMT
```

body 是空的。想自定义 404 页面,那是[第 06 章](06-global-404-handler.md)的内容。

### 步骤 2:理解 Handler 与 IntoResponse

**要做什么**:不改功能,先把上一步"凭什么能跑"想清楚。这一步没有新代码,但它是本教程后面所有章节的地基。

#### Handler:为什么普通函数能当处理器

`get()` 的签名大致是这样(省略了无关的泛型约束):

```rust
pub fn get<H, T, S>(handler: H) -> MethodRouter<S>
where
    H: Handler<T, S>,
    // ...
```

它要求参数实现 `Handler` trait。而 `Handler` 的关键在于:**axum 用一组 impl 覆盖了各种形态的异步函数。**

简化后是这个意思:

```rust
// 零个参数的 async fn
impl<F, Fut, Res, S> Handler<((),), S> for F
where
    F: FnOnce() -> Fut,
    Fut: Future<Output = Res>,
    Res: IntoResponse,
{ /* ... */ }

// 一个参数的 async fn
impl<F, Fut, Res, S, T1> Handler<(T1,), S> for F
where
    F: FnOnce(T1) -> Fut,
    Fut: Future<Output = Res>,
    T1: FromRequest<S>,
    Res: IntoResponse,
{ /* ... */ }

// 两个参数、三个参数……一直写到 16 个
```

所以 `Handler` 不是要你去实现的 trait,而是 **axum 已经替所有"参数都能从请求里提取、返回值能变成响应"的异步函数实现好了**。你的 `async fn handler() -> &'static str` 命中了第一条:零个参数,返回值是 `&'static str`,而 `&'static str` 实现了 `IntoResponse`。于是它自动就是一个 `Handler`。

这个设计的分量在于:**你的业务函数不需要知道 axum 的存在。** 没有 `Request` 参数要接、没有 `Response` 要构造、没有框架类型侵入签名。函数签名只描述"我需要什么、我产出什么",axum 负责两头对接。

参数的部分要求实现 `FromRequest`,那就是**提取器**(extractor)机制——从请求里按类型取出你要的东西。本章的 handler 没有参数所以没用上,从[第 03 章](03-form.md)开始它会成为主角。

顺带说个实用推论:当你写的 handler 报出一长串看不懂的 trait bound 错误,十有八九是某个参数没实现 `FromRequest`,或者返回类型没实现 `IntoResponse`。axum 还提供了 `#[debug_handler]` 宏(需要 `macros` feature)专门用来把这类错误变得可读——[第 15 章](15-anyhow-error-response.md)碰到签名不匹配时会再见到编译器对它的提示。

#### IntoResponse:返回值如何变成 HTTP 响应

`IntoResponse` 就一个方法:

```rust
pub trait IntoResponse {
    fn into_response(self) -> Response;
}
```

axum 给大量常用类型实现了它。挑几个和本章相关的:

| 返回类型 | 状态码 | Content-Type |
|---|---|---|
| `&'static str` / `String` | 200 | `text/plain; charset=utf-8` |
| `Html<T>` | 200 | `text/html; charset=utf-8` |
| `Json<T>` | 200 | `application/json` |
| `StatusCode` | 你给的 | 无 body |
| `(StatusCode, T)` | 你给的 | 由 `T` 决定 |
| `()` | 200 | 无 body |

看这张表,[第 00 章](00-web-basics.md)手写 `Content-Type` 的活儿去哪了就很清楚了:**它被编码进了返回类型。** 你选 `Html` 就是 `text/html`,选 `Json` 就是 `application/json`。类型系统替你保证了头部和内容的一致性——不会出现"声明了 `application/json` 但 body 是 HTML"这种手写响应时很容易犯的错。

`(StatusCode, T)` 这一行值得留意:元组也实现了 `IntoResponse`,这让"改状态码但保留 body"变得很自然,写成 `(StatusCode::CREATED, Json(user))` 就行。[第 08 章](08-parse-body-based-on-content-type.md)之后会频繁用到这个组合。

**本步不改代码,不单独验证。** 步骤 1 已经跑通;`Handler` / `IntoResponse` 的效果下一步改返回类型时用 `curl -i` 对照就能看见。

### 步骤 3:改成返回 HTML

**要做什么**:把返回类型从 `&'static str` 换成 `Html<&'static str>`,观察 `Content-Type` 的变化。这一步会让你的代码抵达官方示例的最终形态。

改两处:

```rust
// src/main.rs
use axum::{response::Html, routing::get, Router};   // 改动:引入 Html

// ...(main 函数完全不变)

// 改动:返回类型换成 Html<&'static str>
async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
```

`main` 一个字都不用改。这一点本身就说明了 `IntoResponse` 的解耦程度:handler 的返回类型是它自己的事,`Router` 只要求它实现 `Handler`,而只要新返回类型实现了 `IntoResponse`,`Handler` 的 impl 就自动成立。

`Html` 是个薄薄的包装类型,定义大致是 `pub struct Html<T>(pub T);`。它不做任何 HTML 处理——不解析、不校验、不转义,只是给 `IntoResponse` 一个信号:"这段内容的 `Content-Type` 是 `text/html`"。

正因为它不转义,把用户输入直接塞进 `Html` 是有 XSS 风险的。要动态生成 HTML,该用模板引擎——那是[第 04 章](04-templates.md)的内容,模板引擎会替你做转义。

**跑起来看**:

```bash
cargo run
```

```bash
curl -i http://127.0.0.1:3000/
```

```
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 22

<h1>Hello, World!</h1>
```

`content-type` 从 `text/plain` 变成了 `text/html`,`content-length` 也跟着变了。**你只改了返回类型,头部自己跟着对了。**

用浏览器访问 `http://127.0.0.1:3000/`,你会看到一个渲染出来的大标题,而不是标签的原文。这和[第 00 章](00-web-basics.md)步骤 2 里手动改 `Content-Type` 的效果一致——只是这次是类型系统在替你保证正确性。

## 完整代码

`src/main.rs`:

```rust
use axum::{response::Html, routing::get, Router};

#[tokio::main]
async fn main() {
    // build our application with a route
    let app = Router::new().route("/", get(handler));

    // run it
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
```

## 动手练习

**1. 加一条 `GET /json` 路由,返回 `{"message":"Hello, World!"}`。**

提示:用 `axum::Json`。它要求内容实现 `serde::Serialize`,最省事的办法是 `Json(serde_json::json!({ "message": "Hello, World!" }))`,需要先 `cargo add serde_json`。用 `curl -i` 确认 `content-type` 是 `application/json`。想想为什么这里不需要你手写任何头部。

**2. 让 `/` 同时支持 `POST`,返回 `201 Created`。**

提示:`get(handler).post(create)` 可以在同一路径上挂两个方法。`create` 的返回类型用 `(StatusCode, &'static str)`。写完用 `curl -i -X POST` 和 `curl -i -X PUT` 各试一次,注意后者的 `allow` 头部变成了什么。

## 小结与下一章预告

axum 的服务由三块拼成:`Router` 记录路由表,handler 是普通的 `async fn`,`axum::serve` 把监听器和路由表接起来。`Handler` trait 让任意"参数可提取、返回值可转换"的异步函数自动成为处理器,`IntoResponse` 则把返回类型翻译成完整的 HTTP 响应——包括你在[第 00 章](00-web-basics.md)手写过的 `Content-Type` 和 `Content-Length`。

本章的路由表只有一条,`main` 函数里塞得下。但真实项目有几十上百条路由,全堆在 `main` 里、handler 散落在文件各处,很快就会难以维护。

[第 02 章 routes-and-handlers-close-together](02-routes-and-handlers-close-together.md) 讲一种把路由和处理器绑在一起的组织方式,顺便认识 `Router::merge` 和 `MethodRouter` 的更多用法。
