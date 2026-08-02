# 第 10 章 customize-extractor-error:定制提取器错误的三种做法

## 本章你将做出什么

一个有三条路由的服务,每条都接收 JSON,但**用三种不同的技术手段**把 axum 默认的纯文本错误换成结构化 JSON:

```json
{"message":"Expected request with `Content-Type: application/json`","origin":"custom_extractor","path":"/custom-extractor"}
```

三条路由功能完全一样,区别只在实现方式。这是本教程第一个**多文件示例**——四个源文件,三个模块各演示一种做法,最后横向对比取舍。

学完本章你将掌握:

- 手写 `FromRequest` 包装内置提取器,以及如何在提取失败时借助其它提取器补充错误上下文
- `#[derive(FromRequest)]` 派生宏:用 `From` 转换替代手写实现
- `axum_extra::extract::WithRejection`:不定义新提取器,只替换拒绝类型
- 三种做法各自的代价,以及项目里该选哪个
- `thiserror` 定义错误类型的基本用法

## 前置知识

### 默认拒绝为什么不够用

[第 03 章](03-form.md#步骤-3观察提取失败时的行为)见过 `Form` 提取失败的响应:

```
HTTP/1.1 415 Unsupported Media Type
content-type: text/plain; charset=utf-8

Form requests must have `Content-Type: application/x-www-form-urlencoded`
```

`Json` 也一样,失败时返回纯文本。对人类调试足够,但对 API 客户端不够:

**前端没法统一处理。** 业务成功时返回 JSON,提取失败时返回纯文本,前端得先判断 `Content-Type` 再决定怎么解析,或者干脆用 try-catch 兜住 JSON 解析异常。接口契约不一致。

**丢失上下文。** 错误信息里没有出错的路径、没有错误码、没有请求 ID,排查线上问题时只有一句干巴巴的描述。

**跨服务不一致。** 团队约定错误格式是 `{"code":..., "message":...}`,但框架的默认拒绝完全不遵守这个约定。

所以真实 API 项目几乎一定要接管提取器的拒绝响应。axum 提供了三条路子,本章全部走一遍。

### JsonRejection:内置拒绝类型

`axum::Json` 提取失败时产生的类型是 `axum::extract::rejection::JsonRejection`,它是个 enum,涵盖各种失败原因:

| 变体 | 含义 | 状态码 |
|---|---|---|
| `MissingJsonContentType` | 没有 `Content-Type: application/json` | 415 |
| `JsonDataError` | JSON 合法但不匹配目标类型(缺字段、类型不对) | 422 |
| `JsonSyntaxError` | JSON 语法就是错的 | 400 |
| `BytesRejection` | 读 body 时出错(如超过大小限制) | 视情况 |

它提供两个关键方法,本章三种做法都要用:

- **`.status()`** — 返回该错误对应的 `StatusCode`
- **`.body_text()`** — 返回默认的纯文本描述

**这两个方法的存在意味着定制工作可以很轻。** 你不需要自己判断"什么错该返回什么状态码"、也不需要重写错误描述——axum 已经算好了,你只要把它们重新包装成自己的格式。三种做法的核心都是这一步。

`JsonRejection` 被标记为 `#[non_exhaustive]`,意思是 axum 保留未来添加新变体的权利。所以你 `match` 它时必须带一个 `_ =>` 兜底分支,否则新版本会编译失败。这个约定在[第 11 章](11-customize-path-rejection.md)会更明显。

### thiserror:定义错误类型的省事办法

Rust 里自定义错误类型通常要实现 `std::error::Error` 和 `Display`,手写很啰嗦。`thiserror` 用派生宏把这些生成掉:

```rust
#[derive(Debug, Error)]
pub enum ApiError {
    #[error(transparent)]
    JsonExtractorRejection(#[from] JsonRejection),
}
```

两个属性各做一件事:

- **`#[error(transparent)]`** 生成 `Display` 实现,内容直接转发给内部错误——`ApiError` 打印出来就是 `JsonRejection` 打印出来的样子
- **`#[from]`** 生成 `impl From<JsonRejection> for ApiError`,这样 `?` 运算符能自动做转换

第二个是本章用它的真正原因:后面两种做法都要求"提供一个 `From<原拒绝类型>` 的转换",`#[from]` 一行搞定。

> **本章不深入**:`thiserror` 还有 `#[error("...{field}...")]` 格式化消息、`#[source]` 错误链、和 `anyhow` 的分工等更多内容。本章只用到 `transparent` 和 `from` 两个属性。完整的错误处理体系见[第 14 章](14-error-handling.md),`anyhow` 的用法见[第 15 章](15-anyhow-error-response.md)。

### MatchedPath:拿到路由模式而非实际路径

做法一会用到这个提取器,先说明白它和 `Uri` 的区别。

请求 `GET /users/42` 时:

- `Uri` 给你 `/users/42` —— 实际访问的路径
- `MatchedPath` 给你 `/users/{id}` —— 匹配上的**路由模式**

写日志和监控指标时后者更有用:`/users/{id}` 只有一个,`/users/42`、`/users/1337` 有无数个。用实际路径做指标标签会产生"高基数"问题,把监控系统撑爆。[第 47 章](47-prometheus-metrics.md) prometheus-metrics 会正经讲这件事。

`MatchedPath` 是 `FromRequestParts` 类别的提取器,只读元信息。

### 多文件项目的模块声明

本章第一次出现多个源文件。Rust 里 `src/main.rs` 之外的每个文件都是一个**模块**,必须在 `main.rs` 里声明才会被编译:

```rust
mod custom_extractor;
mod derive_from_request;
mod with_rejection;
```

`mod foo;` 告诉编译器"去找 `src/foo.rs`(或 `src/foo/mod.rs`),把它当作模块 `foo`"。**没有这行声明,文件就算存在也不会参与编译**——这是从别的语言转过来最容易踩的坑,因为不像 Python/JS 那样按路径自动发现。

模块里的项默认私有,要被 `main.rs` 用到得加 `pub`。本章三个模块里的 `handler` 都是 `pub async fn`。

## 项目搭建

```bash
cargo new customize-extractor-error
cd customize-extractor-error
```

依赖比前面几章多:

```bash
cargo add axum --features macros
cargo add axum-extra --features with-rejection
cargo add serde --features derive
cargo add serde_json
cargo add thiserror
cargo add tokio --features full
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

三个新东西:

- **`axum` 的 `macros` feature** —— 提供 `#[derive(FromRequest)]`,做法二要用
- **`axum-extra` 的 `with-rejection` feature** —— 提供 `WithRejection`,做法三要用。`axum-extra` 是 axum 官方的扩展 crate,放一些不适合进核心库的实用工具
- **`thiserror`** —— 见前置知识

本章要建三个模块文件,先创建空文件:

```bash
touch src/custom_extractor.rs src/derive_from_request.rs src/with_rejection.rs
```

## 分步构建

本章按**做法**切分步骤,不按文件切分。步骤 1 搭好骨架并确认默认行为,步骤 2、3、4 各实现一种做法,步骤 5 横向对比。

### 步骤 1:骨架与默认行为

**要做什么**:先写 `main.rs`,只挂一条用**内置** `Json` 的路由,确认默认拒绝长什么样。这是后面三种做法的对比基准。

```rust
// src/main.rs
use axum::{routing::post, Router};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=trace", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // Build our application with some routes
    let app = Router::new().route("/baseline", post(baseline));

    // Run our application
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

// 临时的对比基准,步骤 5 会删掉
async fn baseline(axum::Json(value): axum::Json<serde_json::Value>) -> axum::Json<serde_json::Value> {
    value.into()
}
```

日志默认级别这里是 `trace` 而不是前几章的 `debug`,官方示例如此,没有特别含义。

`serde_json::Value` 是"任意 JSON 值"的类型——对象、数组、字符串、数字都能装。用它是因为本章关心的是**提取失败时**发生什么,成功路径上具体是什么结构无所谓。

**跑起来看**:

```bash
cargo run
```

正常请求:

```bash
curl -i -X POST http://127.0.0.1:3000/baseline \
  -H 'Content-Type: application/json' \
  -d '{"foo":"bar"}'
```

```
HTTP/1.1 200 OK
content-type: application/json
content-length: 13

{"foo":"bar"}
```

现在制造三种失败,记住它们的样子:

**缺 `Content-Type`**:

```bash
curl -i -X POST http://127.0.0.1:3000/baseline -d '{"foo":"bar"}'
```

```
HTTP/1.1 415 Unsupported Media Type
content-type: text/plain; charset=utf-8

Expected request with `Content-Type: application/json`
```

**JSON 语法错误**:

```bash
curl -i -X POST http://127.0.0.1:3000/baseline \
  -H 'Content-Type: application/json' \
  -d '{"foo":'
```

```
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8

Failed to parse the request body as JSON: EOF while parsing a value at line 1 column 7
```

**三种失败都是 `text/plain`**,而成功时是 `application/json`。这就是要解决的不一致。注意状态码是对的(415、400),错误描述也挺清楚——**我们要保留这两样,只改包装格式**。

### 步骤 2:做法一,手写 FromRequest

**要做什么**:实现 `src/custom_extractor.rs`,手写一个 `Json` 提取器包住 `axum::Json`,失败时返回 JSON,并且**额外带上路由模式**。

```rust
// src/custom_extractor.rs
//! Manual implementation of `FromRequest` that wraps another extractor
//!
//! + Powerful API: Implementing `FromRequest` grants access to `RequestParts`
//!   and `async/await`. This means that you can create more powerful rejections
//! - Boilerplate: Requires creating a new extractor for every custom rejection
//! - Complexity: Manually implementing `FromRequest` results on more complex code
use axum::{
    extract::{rejection::JsonRejection, FromRequest, MatchedPath, Request},
    http::StatusCode,
    response::IntoResponse,
    RequestPartsExt,
};
use serde_json::{json, Value};

pub async fn handler(Json(value): Json<Value>) -> impl IntoResponse {
    Json(dbg!(value));
}

// We define our own `Json` extractor that customizes the error from `axum::Json`
pub struct Json<T>(pub T);

impl<S, T> FromRequest<S> for Json<T>
where
    axum::Json<T>: FromRequest<S, Rejection = JsonRejection>,
    S: Send + Sync,
{
    type Rejection = (StatusCode, axum::Json<Value>);

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let (mut parts, body) = req.into_parts();

        // We can use other extractors to provide better rejection messages.
        // For example, here we are using `axum::extract::MatchedPath` to
        // provide a better error message.
        //
        // Have to run that first since `Json` extraction consumes the request.
        let path = parts
            .extract::<MatchedPath>()
            .await
            .map(|path| path.as_str().to_owned())
            .ok();

        let req = Request::from_parts(parts, body);

        match axum::Json::<T>::from_request(req, state).await {
            Ok(value) => Ok(Self(value.0)),
            // convert the error from `axum::Json` into whatever we want
            Err(rejection) => {
                let payload = json!({
                    "message": rejection.body_text(),
                    "origin": "custom_extractor",
                    "path": path,
                });

                Err((rejection.status(), axum::Json(payload)))
            }
        }
    }
}
```

文件开头那段 `//!` 是**模块级文档注释**,写的正是这种做法的优缺点。官方示例把评价直接写进代码,这个习惯值得学。

几个关键点:

**trait bound 写成 `axum::Json<T>: FromRequest<S, Rejection = JsonRejection>`**,而不是 `T: DeserializeOwned`。这样约束更精确:不关心 `T` 具体要满足什么,只要求"`axum::Json<T>` 是个提取器且失败时给 `JsonRejection`"。内置提取器的约束将来变了,这里也不用改。

**`Rejection` 类型是 `(StatusCode, axum::Json<Value>)`** —— [第 08 章](08-parse-body-based-on-content-type.md#rejection-该选什么类型)那张表里的第三行。状态码从 `rejection.status()` 来,body 是自己拼的 JSON。

**拆请求是为了先跑 `MatchedPath`。** 这段是本做法最有价值的部分:

```rust
let (mut parts, body) = req.into_parts();
let path = parts.extract::<MatchedPath>().await.map(...).ok();
let req = Request::from_parts(parts, body);
```

[第 09 章](09-consume-body-in-extractor-or-middleware.md#步骤-2在中间件里也读一遍-body)的"拆开—处理—重组"手法,这里换了个用途:不是为了读 body,而是**为了在 body 被消耗之前先从 `parts` 里提取别的信息**。

注释说得很直白:`Have to run that first since Json extraction consumes the request`。一旦调用 `axum::Json::from_request(req, ...)`,`req` 就没了,拿不到 `MatchedPath` 了。所以顺序必须是:先取元信息 → 再重组 → 再交给 `Json`。

`RequestPartsExt` 是给 `Parts` 加 `.extract::<T>()` 方法的扩展 trait,必须显式导入才能用。这是[第 08 章](08-parse-body-based-on-content-type.md#requestextextract-便利方法)那个 `RequestExt` 的姊妹——一个作用于完整 `Request`,一个作用于拆出来的 `Parts`。

`.ok()` 把 `Result` 变成 `Option`:拿不到 `MatchedPath` 就是 `None`,不影响主流程。错误信息里少一个字段总比整个提取失败强。

**`json!` 宏**来自 `serde_json`,用近似 JSON 字面量的语法构造 `Value`。`path` 是 `Option<String>`,序列化后是字符串或 `null`。

**handler 里的 `dbg!`** 见[第 03 章](03-form.md#步骤-2用-form-提取器接收提交)。

> **更好的写法**:`Json(dbg!(value));` 末尾这个**分号**让整个函数返回 `()` 而不是那个 `Json`,所以成功时响应 body 是空的——对比步骤 3 的做法二,那里没有分号,能正常回显数据。这几乎肯定是笔误而非有意为之。不过去掉分号还不够:`custom_extractor::Json<T>` 没有实现 `IntoResponse`,得照做法二那样补一个 impl 才行。练习题 3 会让你修。

现在在 `main.rs` 里挂上:

```rust
// src/main.rs
mod custom_extractor;                                        // 新增

use axum::{routing::post, Router};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    let app = Router::new()
        .route("/baseline", post(baseline))
        .route("/custom-extractor", post(custom_extractor::handler));   // 新增

    // ...(listener 和 serve 不变)
}

// ...(baseline 不变)
```

**跑起来看**:

```bash
cargo run
```

```bash
curl -i -X POST http://127.0.0.1:3000/custom-extractor -d '{"foo":"bar"}'
```

```
HTTP/1.1 415 Unsupported Media Type
content-type: application/json
content-length: 118

{"message":"Expected request with `Content-Type: application/json`","origin":"custom_extractor","path":"/custom-extractor"}
```

**`content-type` 变成 `application/json` 了**,状态码仍然是 `415`(来自 `rejection.status()`),错误描述也保留了(来自 `rejection.body_text()`),另外多了 `origin` 和 `path` 两个字段。

`path` 字段就是 `MatchedPath` 的产出。换一条路由试试对比效果——把 `/custom-extractor` 改成带参数的路由就能看出它给的是模式而非实际路径,这留给练习题。

语法错误也变成 JSON 了:

```bash
curl -s -X POST http://127.0.0.1:3000/custom-extractor \
  -H 'Content-Type: application/json' -d '{"foo":'
```

```json
{"message":"Failed to parse the request body as JSON: EOF while parsing a value at line 1 column 7","origin":"custom_extractor","path":"/custom-extractor"}
```

成功路径:

```bash
curl -i -X POST http://127.0.0.1:3000/custom-extractor \
  -H 'Content-Type: application/json' -d '{"foo":"bar"}'
```

```
HTTP/1.1 200 OK
content-length: 0
```

body 是空的——就是上面说的那个分号。服务端终端能看到 `dbg!` 的输出。

### 步骤 3:做法二,派生 FromRequest

**要做什么**:实现 `src/derive_from_request.rs`,用 `#[derive(FromRequest)]` 代替手写实现。

```rust
// src/derive_from_request.rs
//! Uses `axum::extract::FromRequest` to wrap another extractor and customize the
//! rejection
//!
//! + Easy learning curve: Deriving `FromRequest` generates a `FromRequest`
//!   implementation for your type using another extractor. You only need
//!   to provide a `From` impl between the original rejection type and the
//!   target rejection. Crates like [`thiserror`] can provide such conversion
//!   using derive macros.
//! - Boilerplate: Requires deriving `FromRequest` for every custom rejection
//! - There are some known limitations: [FromRequest#known-limitations]
//!
//! [`thiserror`]: https://crates.io/crates/thiserror
//! [FromRequest#known-limitations]: https://docs.rs/axum-macros/*/axum_macros/derive.FromRequest.html#known-limitations
use axum::{
    extract::rejection::JsonRejection, extract::FromRequest, http::StatusCode,
    response::IntoResponse,
};
use serde::Serialize;
use serde_json::{json, Value};

pub async fn handler(Json(value): Json<Value>) -> impl IntoResponse {
    Json(dbg!(value))
}

// create an extractor that internally uses `axum::Json` but has a custom rejection
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(ApiError))]
pub struct Json<T>(T);

// We implement `IntoResponse` for our extractor so it can be used as a response
impl<T: Serialize> IntoResponse for Json<T> {
    fn into_response(self) -> axum::response::Response {
        let Self(value) = self;
        axum::Json(value).into_response()
    }
}

// We create our own rejection type
#[derive(Debug)]
pub struct ApiError {
    status: StatusCode,
    message: String,
}

// We implement `From<JsonRejection> for ApiError`
impl From<JsonRejection> for ApiError {
    fn from(rejection: JsonRejection) -> Self {
        Self {
            status: rejection.status(),
            message: rejection.body_text(),
        }
    }
}

// We implement `IntoResponse` so `ApiError` can be used as a response
impl IntoResponse for ApiError {
    fn into_response(self) -> axum::response::Response {
        let payload = json!({
            "message": self.message,
            "origin": "derive_from_request"
        });

        (self.status, axum::Json(payload)).into_response()
    }
}
```

对比步骤 2,那 30 行 `impl FromRequest` 消失了,换成两行属性:

```rust
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(ApiError))]
pub struct Json<T>(T);
```

`via(axum::Json)` 说"用 `axum::Json` 做实际的提取工作",`rejection(ApiError)` 说"失败时把拒绝转成 `ApiError`"。派生宏生成的实现大致等价于:调用 `axum::Json::from_request`,成功就包进自己,失败就 `.into()` 转成 `ApiError`。

**所以你必须提供 `From<JsonRejection> for ApiError`** —— 这就是文件里那个手写的 `impl From`。它做的事和步骤 2 一样:取 `status()` 和 `body_text()`。

`ApiError` 还要实现 `IntoResponse`(因为它是拒绝类型),在那里拼出最终的 JSON。

**这个模块还多做了一件事:给 `Json<T>` 自己实现了 `IntoResponse`。** 于是同一个类型既能当提取器又能当返回值,handler 写成 `Json(dbg!(value))`(注意这次没有分号)就能把数据原样返回。步骤 2 的做法没做这件事,所以那个 handler 只能返回空 body。

挂上路由:

```rust
// src/main.rs
mod custom_extractor;
mod derive_from_request;                                     // 新增

// ...

    let app = Router::new()
        .route("/baseline", post(baseline))
        .route("/custom-extractor", post(custom_extractor::handler))
        .route("/derive-from-request", post(derive_from_request::handler));   // 新增
```

**跑起来看**:

```bash
cargo run
```

```bash
curl -i -X POST http://127.0.0.1:3000/derive-from-request -d '{"foo":"bar"}'
```

```
HTTP/1.1 415 Unsupported Media Type
content-type: application/json
content-length: 91

{"message":"Expected request with `Content-Type: application/json`","origin":"derive_from_request"}
```

和做法一几乎一样,少了 `path` 字段——**因为派生宏没有给你插手提取过程的机会**,拿不到 `MatchedPath`。这是它最主要的限制。

成功路径这次有 body 了:

```bash
curl -i -X POST http://127.0.0.1:3000/derive-from-request \
  -H 'Content-Type: application/json' -d '{"foo":"bar"}'
```

```
HTTP/1.1 200 OK
content-type: application/json
content-length: 13

{"foo":"bar"}
```

这就是给 `Json<T>` 实现 `IntoResponse` 的效果。

### 步骤 4:做法三,WithRejection

**要做什么**:实现 `src/with_rejection.rs`,不定义新提取器,直接用 `axum-extra` 的包装器。

```rust
// src/with_rejection.rs
//! Uses `axum_extra::extract::WithRejection` to transform one rejection into
//! another
//!
//! + Easy learning curve: `WithRejection` acts as a wrapper for another
//!   already existing extractor. You only need to provide a `From` impl
//!   between the original rejection type and the target rejection. Crates like
//!   `thiserror` can provide such conversion using derive macros. See
//!   [`thiserror`]
//! - Verbose types: types become much larger, which makes them difficult to
//!   read. Current limitations on type aliasing make it impossible to destructure
//!   a type alias. See [#1116]
//!
//! [`thiserror`]: https://crates.io/crates/thiserror
//! [#1116]: https://github.com/tokio-rs/axum/issues/1116#issuecomment-1186197684

use axum::{extract::rejection::JsonRejection, response::IntoResponse, Json};
use axum_extra::extract::WithRejection;
use serde_json::{json, Value};
use thiserror::Error;

pub async fn handler(
    // `WithRejection` will extract `Json<Value>` from the request. If it fails,
    // `JsonRejection` will be transform into `ApiError` and returned as response
    // to the client.
    //
    // The second constructor argument is not meaningful and can be safely ignored
    WithRejection(Json(value), _): WithRejection<Json<Value>, ApiError>,
) -> impl IntoResponse {
    Json(dbg!(value))
}

// We derive `thiserror::Error`
#[derive(Debug, Error)]
pub enum ApiError {
    // The `#[from]` attribute generates `From<JsonRejection> for ApiError`
    // implementation. See `thiserror` docs for more information
    #[error(transparent)]
    JsonExtractorRejection(#[from] JsonRejection),
}

// We implement `IntoResponse` so ApiError can be used as a response
impl IntoResponse for ApiError {
    fn into_response(self) -> axum::response::Response {
        let (status, message) = match self {
            ApiError::JsonExtractorRejection(json_rejection) => {
                (json_rejection.status(), json_rejection.body_text())
            }
        };

        let payload = json!({
            "message": message,
            "origin": "with_rejection"
        });

        (status, Json(payload)).into_response()
    }
}
```

这次**完全没有定义新的提取器类型**。`WithRejection<E, R>` 是 `axum-extra` 提供的泛型包装:用 `E` 做提取,失败时把拒绝转成 `R`。

参数签名是本做法最扎眼的地方:

```rust
WithRejection(Json(value), _): WithRejection<Json<Value>, ApiError>
```

嵌套了两层解构。`WithRejection` 是个二元元组结构体,第一个字段是提取结果 `Json<Value>`(再解构一层拿到 `value`),第二个字段是个类型标记(`PhantomData`),没有实际值,所以用 `_` 忽略——注释也说了 "not meaningful and can be safely ignored"。

**类型冗长是这个做法的主要代价。** 提取器一多,签名会迅速变得难读:

```rust
async fn handler(
    WithRejection(Path(id), _): WithRejection<Path<u32>, ApiError>,
    WithRejection(Json(body), _): WithRejection<Json<Payload>, ApiError>,
) -> impl IntoResponse
```

模块文档里提到的 issue #1116 就是关于这个:Rust 目前不能对类型别名做解构,所以没法用 `type MyJson<T> = WithRejection<Json<T>, ApiError>;` 简化。

**错误类型这次用了 `thiserror`。** `#[from]` 自动生成了 `From<JsonRejection> for ApiError`,省掉了做法二里手写的那个 `impl From`。用 enum 而不是 struct 是为了将来扩展——再加一个提取器的拒绝类型,加个变体和 `#[from]` 就行。

挂上路由,这次把三条路由和 `baseline` 一起整理成官方示例的最终形态:

```rust
// src/main.rs
mod custom_extractor;
mod derive_from_request;
mod with_rejection;                                          // 新增

use axum::{routing::post, Router};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    // 改动:删掉 baseline,保留三种做法
    let app = Router::new()
        .route("/with-rejection", post(with_rejection::handler))
        .route("/custom-extractor", post(custom_extractor::handler))
        .route("/derive-from-request", post(derive_from_request::handler));

    // ...(listener 和 serve 不变)
}
```

把步骤 1 里那个临时的 `baseline` 函数删掉。

**跑起来看**:

```bash
cargo run
```

```bash
curl -i -X POST http://127.0.0.1:3000/with-rejection -d '{"foo":"bar"}'
```

```
HTTP/1.1 415 Unsupported Media Type
content-type: application/json
content-length: 86

{"message":"Expected request with `Content-Type: application/json`","origin":"with_rejection"}
```

三条路由一起验证,`origin` 字段能区分是哪种做法在工作:

```bash
for p in with-rejection custom-extractor derive-from-request; do
  echo "--- /$p ---"
  curl -s -X POST "http://127.0.0.1:3000/$p" -d '{"foo":"bar"}'
  echo
done
```

```
--- /with-rejection ---
{"message":"Expected request with `Content-Type: application/json`","origin":"with_rejection"}
--- /custom-extractor ---
{"message":"Expected request with `Content-Type: application/json`","origin":"custom_extractor","path":"/custom-extractor"}
--- /derive-from-request ---
{"message":"Expected request with `Content-Type: application/json`","origin":"derive_from_request"}
```

**三种做法产出的响应几乎一致,只有做法一多了 `path` 字段。** 这正是本章要说明的:**目标相同,手段的差别在于写起来的成本和能达到的上限**。

### 步骤 5:横向对比

**要做什么**:不写代码,把三种做法摆在一起看清取舍。

| | 做法一 手写 `FromRequest` | 做法二 `#[derive(FromRequest)]` | 做法三 `WithRejection` |
|---|---|---|---|
| 代码量 | 最多(约 30 行 impl) | 中等(2 行属性 + `From` impl) | 最少(无新类型) |
| 需要定义新提取器类型 | 是 | 是 | **否** |
| 能否在提取过程中插手 | **能**(可跑其它提取器) | 否 | 否 |
| 错误里能否带路由/请求上下文 | **能** | 否 | 否 |
| handler 签名 | 干净 | 干净 | **冗长** |
| 同一类型能否兼作返回值 | 需自己实现 | 需自己实现 | 不适用 |
| 需要的 feature | 无 | `axum/macros` | `axum-extra/with-rejection` |
| 学习曲线 | 陡 | 平缓 | 平缓 |

**怎么选:**

**默认选做法二。** 大多数项目只需要"把默认拒绝换成自己的 JSON 格式",派生宏成本最低,handler 签名也干净。定义一个 `Json<T>` 放在 `extractors` 模块里,全项目共用。

**需要错误上下文时选做法一。** 想在错误里带路由模式、请求 ID、用户信息,只有它能在提取过程中跑其它提取器。代价是每种提取器都要写一遍 impl。

**做法三适合临时或局部场景。** 不想为一次性需求定义新类型时它最快。但签名冗长,大面积使用会伤害可读性,而且[类型别名不能解构](https://github.com/tokio-rs/axum/issues/1116)这个限制目前无解。

三者不互斥。真实项目常见的组合是:定义一个派生版 `Json<T>` 作为全局默认,少数需要额外上下文的接口单独手写提取器。

**一个共同的前提值得强调**:三种做法都建立在 `rejection.status()` 和 `rejection.body_text()` 之上——**没有一种是在重新判断"什么错该返回什么码"**。axum 已经把这部分做对了,定制工作只是换个包装。自己重写状态码映射既费力又容易出错。

## 完整代码

`src/main.rs`:

```rust
mod custom_extractor;
mod derive_from_request;
mod with_rejection;

use axum::{routing::post, Router};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=trace", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // Build our application with some routes
    let app = Router::new()
        .route("/with-rejection", post(with_rejection::handler))
        .route("/custom-extractor", post(custom_extractor::handler))
        .route("/derive-from-request", post(derive_from_request::handler));

    // Run our application
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

`src/custom_extractor.rs`:

```rust
//! Manual implementation of `FromRequest` that wraps another extractor
//!
//! + Powerful API: Implementing `FromRequest` grants access to `RequestParts`
//!   and `async/await`. This means that you can create more powerful rejections
//! - Boilerplate: Requires creating a new extractor for every custom rejection
//! - Complexity: Manually implementing `FromRequest` results on more complex code
use axum::{
    extract::{rejection::JsonRejection, FromRequest, MatchedPath, Request},
    http::StatusCode,
    response::IntoResponse,
    RequestPartsExt,
};
use serde_json::{json, Value};

pub async fn handler(Json(value): Json<Value>) -> impl IntoResponse {
    Json(dbg!(value));
}

// We define our own `Json` extractor that customizes the error from `axum::Json`
pub struct Json<T>(pub T);

impl<S, T> FromRequest<S> for Json<T>
where
    axum::Json<T>: FromRequest<S, Rejection = JsonRejection>,
    S: Send + Sync,
{
    type Rejection = (StatusCode, axum::Json<Value>);

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let (mut parts, body) = req.into_parts();

        // We can use other extractors to provide better rejection messages.
        // For example, here we are using `axum::extract::MatchedPath` to
        // provide a better error message.
        //
        // Have to run that first since `Json` extraction consumes the request.
        let path = parts
            .extract::<MatchedPath>()
            .await
            .map(|path| path.as_str().to_owned())
            .ok();

        let req = Request::from_parts(parts, body);

        match axum::Json::<T>::from_request(req, state).await {
            Ok(value) => Ok(Self(value.0)),
            // convert the error from `axum::Json` into whatever we want
            Err(rejection) => {
                let payload = json!({
                    "message": rejection.body_text(),
                    "origin": "custom_extractor",
                    "path": path,
                });

                Err((rejection.status(), axum::Json(payload)))
            }
        }
    }
}
```

`src/derive_from_request.rs`:

```rust
//! Uses `axum::extract::FromRequest` to wrap another extractor and customize the
//! rejection
//!
//! + Easy learning curve: Deriving `FromRequest` generates a `FromRequest`
//!   implementation for your type using another extractor. You only need
//!   to provide a `From` impl between the original rejection type and the
//!   target rejection. Crates like [`thiserror`] can provide such conversion
//!   using derive macros.
//! - Boilerplate: Requires deriving `FromRequest` for every custom rejection
//! - There are some known limitations: [FromRequest#known-limitations]
//!
//! [`thiserror`]: https://crates.io/crates/thiserror
//! [FromRequest#known-limitations]: https://docs.rs/axum-macros/*/axum_macros/derive.FromRequest.html#known-limitations
use axum::{
    extract::rejection::JsonRejection, extract::FromRequest, http::StatusCode,
    response::IntoResponse,
};
use serde::Serialize;
use serde_json::{json, Value};

pub async fn handler(Json(value): Json<Value>) -> impl IntoResponse {
    Json(dbg!(value))
}

// create an extractor that internally uses `axum::Json` but has a custom rejection
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(ApiError))]
pub struct Json<T>(T);

// We implement `IntoResponse` for our extractor so it can be used as a response
impl<T: Serialize> IntoResponse for Json<T> {
    fn into_response(self) -> axum::response::Response {
        let Self(value) = self;
        axum::Json(value).into_response()
    }
}

// We create our own rejection type
#[derive(Debug)]
pub struct ApiError {
    status: StatusCode,
    message: String,
}

// We implement `From<JsonRejection> for ApiError`
impl From<JsonRejection> for ApiError {
    fn from(rejection: JsonRejection) -> Self {
        Self {
            status: rejection.status(),
            message: rejection.body_text(),
        }
    }
}

// We implement `IntoResponse` so `ApiError` can be used as a response
impl IntoResponse for ApiError {
    fn into_response(self) -> axum::response::Response {
        let payload = json!({
            "message": self.message,
            "origin": "derive_from_request"
        });

        (self.status, axum::Json(payload)).into_response()
    }
}
```

`src/with_rejection.rs`:

```rust
//! Uses `axum_extra::extract::WithRejection` to transform one rejection into
//! another
//!
//! + Easy learning curve: `WithRejection` acts as a wrapper for another
//!   already existing extractor. You only need to provide a `From` impl
//!   between the original rejection type and the target rejection. Crates like
//!   `thiserror` can provide such conversion using derive macros. See
//!   [`thiserror`]
//! - Verbose types: types become much larger, which makes them difficult to
//!   read. Current limitations on type aliasing make it impossible to destructure
//!   a type alias. See [#1116]
//!
//! [`thiserror`]: https://crates.io/crates/thiserror
//! [#1116]: https://github.com/tokio-rs/axum/issues/1116#issuecomment-1186197684

use axum::{extract::rejection::JsonRejection, response::IntoResponse, Json};
use axum_extra::extract::WithRejection;
use serde_json::{json, Value};
use thiserror::Error;

pub async fn handler(
    // `WithRejection` will extract `Json<Value>` from the request. If it fails,
    // `JsonRejection` will be transform into `ApiError` and returned as response
    // to the client.
    //
    // The second constructor argument is not meaningful and can be safely ignored
    WithRejection(Json(value), _): WithRejection<Json<Value>, ApiError>,
) -> impl IntoResponse {
    Json(dbg!(value))
}

// We derive `thiserror::Error`
#[derive(Debug, Error)]
pub enum ApiError {
    // The `#[from]` attribute generates `From<JsonRejection> for ApiError`
    // implementation. See `thiserror` docs for more information
    #[error(transparent)]
    JsonExtractorRejection(#[from] JsonRejection),
}

// We implement `IntoResponse` so ApiError can be used as a response
impl IntoResponse for ApiError {
    fn into_response(self) -> axum::response::Response {
        let (status, message) = match self {
            ApiError::JsonExtractorRejection(json_rejection) => {
                (json_rejection.status(), json_rejection.body_text())
            }
        };

        let payload = json!({
            "message": message,
            "origin": "with_rejection"
        });

        (status, Json(payload)).into_response()
    }
}
```

## 动手练习

**1. 让 `MatchedPath` 的价值显现出来。** 把 `/custom-extractor` 改成带路径参数的路由,比如 `/custom-extractor/{id}`。

提示:路由改成 `.route("/custom-extractor/{id}", post(custom_extractor::handler))`,handler 不用改。然后请求 `/custom-extractor/42` 并故意让 JSON 提取失败,看 `path` 字段返回的是 `/custom-extractor/{id}` 还是 `/custom-extractor/42`。这个区别对做监控指标至关重要,理由见前置知识。

**2. 给做法三加第二种拒绝类型。** 让 `/with-rejection` 的路由带路径参数,handler 同时提取 `Path<u32>` 和 `Json<Value>`,两者的拒绝都转成 `ApiError`。

提示:`ApiError` 加一个变体 `PathExtractorRejection(#[from] PathRejection)`,`into_response` 的 `match` 加一条分支。handler 签名会变成两个 `WithRejection`——写完你就能直观感受到"类型冗长"这个代价到底有多痛。

**3. 修掉做法一的空 body 问题。** 让 `/custom-extractor` 成功时也返回 JSON。

提示:去掉 `Json(dbg!(value));` 末尾的分号是不够的——`custom_extractor::Json<T>` 没有实现 `IntoResponse`。参照做法二给它补一个 impl。

## 小结与下一章预告

定制提取器拒绝有三条路:手写 `FromRequest` 最灵活,是唯一能在提取过程中插手、拿到 `MatchedPath` 这类上下文的做法;`#[derive(FromRequest)]` 用 `via` + `rejection` 两个属性替代手写实现,只需提供一个 `From` 转换,日常首选;`WithRejection` 不定义新类型,最轻量,但 handler 签名会变得冗长。

三者的共同基础是 `rejection.status()` 和 `rejection.body_text()` —— **定制工作只是换包装,不要自己重写状态码映射**。

本章还第一次接触了多文件项目:`src/` 下的每个文件都是模块,必须在 `main.rs` 里用 `mod` 声明才会参与编译,模块里的项要加 `pub` 才能被外部使用。

本章定制的是 `Json` 的拒绝。[第 11 章 customize-path-rejection](11-customize-path-rejection.md) 换成 `Path` 提取器,而且做得更细:不是笼统地把拒绝转成 JSON,而是**区分具体的失败原因**——参数个数不对、某个参数类型转换失败、路径里有非法 UTF-8——分别给出不同的状态码和定位信息。你会看到 `#[non_exhaustive]` 枚举该怎么匹配,以及为什么有些错误该返回 `400` 而另一些该返回 `500`。
