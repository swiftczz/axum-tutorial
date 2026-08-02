# 第 08 章 parse-body-based-on-content-type:自己实现 FromRequest

## 本章你将做出什么

一个能同时接受 JSON 和表单的接口。同一个 `POST /`,客户端发 `application/json` 也行,发 `application/x-www-form-urlencoded` 也行,handler 拿到的都是同一个 struct。

做法是**自己实现 `FromRequest`**——[第 04 章](04-templates.md#步骤-3为自己的类型实现-intoresponse)你实现过 `IntoResponse`(出方向),这一章实现它的对偶(入方向)。这两个 trait 是扩展 axum 的两把主要工具,学完这章你就都掌握了。

学完本章你将掌握:

- `Json<T>` 的两副面孔:当提取器收 JSON、当返回值发 JSON
- `FromRequest` 的方法签名和 `Rejection` 关联类型的含义
- 如何在自己的提取器里复用 axum 内置的提取器
- `RequestExt::extract` 这个便利方法
- 为什么 `Rejection = Response` 是自定义提取器的实用选择

## 前置知识

### Json:收 JSON 与发 JSON

后面要包装 `Json`,先把它本身讲清楚。前面章节只在[第 01 章](01-hello-world.md#intoresponse返回值如何变成-http-响应)的表格里提过一行,这里补上。

**`Json<T>` 是一个类型,但有两个用途**,取决于它出现在 handler 的什么位置:

```rust
async fn create_user(Json(payload): Json<CreateUser>) -> (StatusCode, Json<User>) {
    let user = User { id: 1337, username: payload.username };
    (StatusCode::CREATED, Json(user))
}
```

**出现在参数位置 → 它是提取器。** 要求 `T: DeserializeOwned`。axum 会:检查 `Content-Type` 是不是 `application/json`(不是就 `415`)→ 读取 body → 用 `serde_json` 反序列化成 `T`(失败就 `400` 或 `422`)。这些行为[第 03 章](03-form.md#步骤-3观察提取失败时的行为)在 `Form` 上验证过,`Json` 完全对应。

**出现在返回值位置 → 它是响应包装。** 要求 `T: Serialize`。axum 会把 `T` 序列化成 JSON 文本,设置 `Content-Type: application/json`,算好 `Content-Length`。

一个最小的 REST 接口就是这两件事的组合:

```rust
use axum::{http::StatusCode, routing::post, Json, Router};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]           // 入参:需要反序列化
struct CreateUser {
    username: String,
}

#[derive(Serialize)]             // 出参:需要序列化
struct User {
    id: u64,
    username: String,
}

async fn create_user(Json(payload): Json<CreateUser>) -> (StatusCode, Json<User>) {
    let user = User {
        id: 1337,
        username: payload.username,
    };
    (StatusCode::CREATED, Json(user))
}

let app = Router::new().route("/users", post(create_user));
```

注意两个 struct 的派生宏不一样:入参只需 `Deserialize`,出参只需 `Serialize`。**把请求类型和响应类型分开定义是常见做法**——它们的字段往往不同(响应多了服务端生成的 `id`,请求里可能有响应不该回显的 `password`)。

返回类型 `(StatusCode, Json<User>)` 是[第 06 章](06-global-404-handler.md#元组作为返回类型)讲的元组组合:`Json<User>` 提供 body 和 `Content-Type`,`StatusCode::CREATED` 把状态码从默认的 `200` 改成 `201`。**`201 Created` 是 REST 里创建成功的标准回答**([第 00 章](00-web-basics.md#状态码一个三位数的分类回答)的状态码分类里提过)。

请求它:

```bash
curl -i -X POST http://127.0.0.1:3000/users \
  -H 'Content-Type: application/json' \
  -d '{"username":"zhangsan"}'
```

```
HTTP/1.1 201 Created
content-type: application/json
content-length: 34

{"id":1337,"username":"zhangsan"}
```

**这段代码不是本章要写的项目**,只是为了让你在看后面那些"包装 `Json`"的代码时,清楚被包装的东西本身是什么。本章的项目从下面的"项目搭建"开始。

`Json` 由 axum 的 `json` feature 提供,它在默认 feature 里,不用额外开启。

### 为什么需要"内容协商"

同一个接口支持多种请求格式,在真实项目里比想象中常见:

- 一个接口既服务浏览器的 HTML 表单提交(urlencoded),又服务前端 JS 或移动端的 API 调用(JSON)
- 迁移期间要同时兼容老客户端的旧格式和新客户端的新格式
- 对外的开放 API 希望对接入方友好,不强制某一种格式

HTTP 里表达"我发的是什么格式"就靠 `Content-Type` 头部,[第 00 章](00-web-basics.md#content-type-与-mime-类型)讲过。服务端根据它选择解析方式,这个过程有个正式名字叫**内容协商**(content negotiation)。

**`Json<T>` 和 `Form<T>` 各自只认一种格式,而且认得很死。** 上一节看到 `Json` 会先检查 `Content-Type`;[第 03 章](03-form.md#步骤-3观察提取失败时的行为)也验证过反过来的情况——给 `Form` 发 JSON 会得到 `415 Unsupported Media Type`。想要"两种都行",得自己写一个提取器。

### FromRequest 与 FromRequestParts 的实现细节

[第 03 章](03-form.md#提取器axum-的核心机制)介绍过这两个 trait 的分工,现在看它们的实际定义。

`FromRequestParts` 只能看请求的"头部部分":

```rust
pub trait FromRequestParts<S>: Sized {
    type Rejection: IntoResponse;

    async fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> Result<Self, Self::Rejection>;
}
```

`FromRequest` 拿到整个请求,包括 body:

```rust
pub trait FromRequest<S>: Sized {
    type Rejection: IntoResponse;

    async fn from_request(
        req: Request,
        state: &S,
    ) -> Result<Self, Self::Rejection>;
}
```

关键差别在参数:

- `from_request_parts` 收到 `&mut Parts`——**借用**,而且只有元信息(方法、URI、头部、扩展)。借用意味着多个提取器可以依次调用它,所以 `FromRequestParts` 类别的提取器可以有任意多个。
- `from_request` 收到 `Request`——**所有权**。拿走了整个请求,别人就没得用了。这就是为什么 `FromRequest` 类别的提取器只能有一个,而且必须放在参数列表最后。

两个 trait 都有 `S` 泛型参数,那是[第 05 章](05-templates-minijinja.md#state-与-with_state)讲的应用状态类型。提取器需要访问状态时用得上(比如从数据库连接池取连接),不需要就在 impl 里加个 `S: Send + Sync` 约束放过去。

`Rejection` 关联类型是**提取失败时返回什么**。它必须实现 `IntoResponse`——因为失败时 axum 直接把它变成响应发给客户端,不调用 handler。[第 03 章](03-form.md#步骤-3观察提取失败时的行为)那些 `415`、`422` 响应,就是内置提取器的 `Rejection` 转成的。

### Rejection 该选什么类型

自定义提取器时,`Rejection` 有几种常见选择:

| 选择 | 适用场景 |
|---|---|
| `StatusCode` | 只需要一个状态码,不带任何说明 |
| `(StatusCode, String)` | 状态码 + 纯文本说明 |
| `(StatusCode, Json<T>)` | 状态码 + 结构化 JSON 错误(API 服务常用) |
| 自定义 error 类型 | 需要统一的错误格式,给它实现 `IntoResponse` |
| **`Response`** | **失败形态多样,每种要返回不同的东西** |

本章用 `Response`。理由是这个提取器有三种失败可能:JSON 解析失败、表单解析失败、`Content-Type` 不认识。前两种要把内置提取器的拒绝原样转发出去(它们的错误信息很有用),第三种要自己造一个 `415`。**三种失败的响应形态不同**,用具体类型套不住,统一擦除成 `Response` 最省事。

代价是丢掉了类型信息——调用方看不出这个提取器可能怎么失败。写库时这算缺点,写应用时通常无所谓。[第 10 章](10-customize-extractor-error.md)会展示更讲究的做法:定义专门的错误类型并实现 `IntoResponse`。

### RequestExt::extract 便利方法

自己实现 `FromRequest` 时经常要**在内部调用别的提取器**。直接调用 trait 方法有点啰嗦:

```rust
let Json(payload) = axum::Json::<T>::from_request(req, state).await?;
```

axum 提供了 `RequestExt` 扩展 trait,让这件事写成方法调用的形式:

```rust
use axum::RequestExt;

let Json(payload) = req.extract().await?;
```

`extract()` 会根据你的解构模式推断要提取什么类型——上面那行左边写了 `Json(payload)`,编译器就知道要提取 `Json<T>`。

它还有几个变体:

- `req.extract::<T, _>()` — 显式指定类型
- `req.extract_parts::<T>()` — 只提取 `FromRequestParts` 类别的,不消耗 body
- `parts.extract::<T>()` — 在 `Parts` 上用,来自 `RequestPartsExt` trait

用 `RequestExt` 要先 `use axum::RequestExt`,这是 Rust 扩展 trait 的常规要求——方法定义在 trait 里,不导入 trait 就看不到方法。

### 组合提取器时的 trait bound 技巧

本章代码里有一处值得提前解释的 trait bound:

```rust
impl<S, T> FromRequest<S> for JsonOrForm<T>
where
    S: Send + Sync,
    Json<T>: FromRequest<()>,
    Form<T>: FromRequest<()>,
    T: 'static,
```

`Json<T>: FromRequest<()>` 的意思是"要求 `Json<T>` 能在**无状态**的情况下被提取"。这里写 `()` 而不是 `S`,是因为 `Json` 和 `Form` 的提取根本不需要应用状态——它们只看 `Content-Type` 和 body。约束成 `()` 比约束成 `S` 更宽松,也更准确地表达了"我不需要状态"。

这个技巧的实用价值在于:它让 `JsonOrForm` 可以用在任何 `Router<S>` 里,不管 `S` 是什么。如果写成 `Json<T>: FromRequest<S>`,调用方的状态类型就会被卷进来,反而更受限。

`T: 'static` 是因为 `Json<T>` 的提取要求内部类型不含短生命周期的借用——反序列化出来的值要能脱离请求独立存在。

## 项目搭建

```bash
cargo new parse-body-based-on-content-type
cd parse-body-based-on-content-type
```

```bash
cargo add axum
cargo add tokio --features full
cargo add serde --features derive
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

和[第 03 章](03-form.md#项目搭建)完全相同。`axum` 的默认 feature 已经包含 `json` 和 `form`,两个内置提取器都能直接用。

## 分步构建

### 步骤 1:先只支持 JSON

**要做什么**:用内置的 `Json` 提取器做一个只收 JSON 的接口,作为对照起点。

日志初始化样板的作用见[第 03 章](03-form.md#步骤-1显示表单页面),照抄即可。

```rust
// src/main.rs
use axum::{routing::post, Json, Router};
use serde::{Deserialize, Serialize};
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

    let app = Router::new().route("/", post(handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

#[derive(Debug, Serialize, Deserialize)]
struct Payload {
    foo: String,
}

async fn handler(Json(payload): Json<Payload>) {
    dbg!(payload);
}
```

日志样板和前几章略有不同:默认过滤器是 `"{}=debug,tower_http=debug"`,多了 `tower_http=debug`。这是官方示例的写法(为了在用到 `tower-http` 中间件时也能看到它的日志),本章其实没装 `tower-http`,多这一段无害。

`Payload` 派生了 `Serialize` 和 `Deserialize` 两个。`Deserialize` 是提取必需的;`Serialize` 本章用不到(handler 没把它返回出去),官方示例加上大概是为了方便读者改成回显。

**`handler` 的返回类型是 `()`。** [第 01 章](01-hello-world.md#intoresponse返回值如何变成-http-响应)那张表里有这一行:`()` 实现了 `IntoResponse`,生成 `200` + 空 body。所以这个 handler 收下数据、打印、什么都不返回。

`dbg!(payload)` 把值打到 stderr,[第 03 章](03-form.md#步骤-2用-form-提取器接收提交)解释过。注意这里是 `dbg!(payload)` 而非 `dbg!(&payload)`——`dbg!` 会返回它接收的值,但这里返回值被丢弃了,所以 `payload` 在此被消耗掉。因为后面不再用它,这样写没问题。

**跑起来看**:

```bash
cargo run
```

发 JSON:

```bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: application/json' \
  -d '{"foo":"json works"}'
```

```
HTTP/1.1 200 OK
content-length: 0
date: Thu, 30 Jul 2026 07:31:20 GMT
```

`200` 且 body 为空,符合 `()` 的行为。服务端终端:

```
[src/main.rs:33:5] payload = Payload {
    foo: "json works",
}
```

现在发表单:

```bash
curl -i -X POST http://127.0.0.1:3000/ -d 'foo=form works'
```

```
HTTP/1.1 415 Unsupported Media Type
content-type: text/plain; charset=utf-8
content-length: 68

Expected request with `Content-Type: application/json`
```

**`415`。** `Json` 提取器只认 JSON,这和[第 03 章](03-form.md#步骤-3观察提取失败时的行为)里 `Form` 拒绝 JSON 是镜像的情形。

下一步造一个两种都收的提取器。

### 步骤 2:实现 JsonOrForm

**要做什么**:定义 `JsonOrForm<T>` 并为它实现 `FromRequest`,根据 `Content-Type` 选择解析方式。

```rust
// src/main.rs
use axum::{
    extract::{FromRequest, Request},                    // 新增
    http::{header::CONTENT_TYPE, StatusCode},           // 新增
    response::{IntoResponse, Response},                 // 新增
    routing::post,
    Form, Json, RequestExt, Router,                     // 改动:加上 Form 和 RequestExt
};
use serde::{Deserialize, Serialize};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// ...(main 和 Payload 不变)

// 改动:参数类型换成自己的提取器
async fn handler(JsonOrForm(payload): JsonOrForm<Payload>) {
    dbg!(payload);
}

// 新增:一个包装类型,给 FromRequest 一个挂载点
struct JsonOrForm<T>(T);

// 新增:根据 Content-Type 分派到 Json 或 Form
impl<S, T> FromRequest<S> for JsonOrForm<T>
where
    S: Send + Sync,
    Json<T>: FromRequest<()>,
    Form<T>: FromRequest<()>,
    T: 'static,
{
    type Rejection = Response;

    async fn from_request(req: Request, _state: &S) -> Result<Self, Self::Rejection> {
        let content_type_header = req.headers().get(CONTENT_TYPE);
        let content_type = content_type_header.and_then(|value| value.to_str().ok());

        if let Some(content_type) = content_type {
            if content_type.starts_with("application/json") {
                let Json(payload) = req.extract().await.map_err(IntoResponse::into_response)?;
                return Ok(Self(payload));
            }

            if content_type.starts_with("application/x-www-form-urlencoded") {
                let Form(payload) = req.extract().await.map_err(IntoResponse::into_response)?;
                return Ok(Self(payload));
            }
        }

        Err(StatusCode::UNSUPPORTED_MEDIA_TYPE.into_response())
    }
}
```

逐块拆解这个实现。

**`struct JsonOrForm<T>(T);`** 是个 newtype 包装。和[第 04 章](04-templates.md#步骤-3为自己的类型实现-intoresponse)的 `HtmlTemplate<T>` 同理:你不能给别人的类型实现 `FromRequest`,所以定义自己的类型作为挂载点。

**读取 `Content-Type`:**

```rust
let content_type_header = req.headers().get(CONTENT_TYPE);
let content_type = content_type_header.and_then(|value| value.to_str().ok());
```

`req.headers()` 返回 `&HeaderMap`,`get(CONTENT_TYPE)` 返回 `Option<&HeaderValue>`——头部可能不存在。

`HeaderValue` 内部是字节而不是字符串,因为 HTTP 头部理论上可以含非 UTF-8 字节。`to_str()` 返回 `Result<&str, _>`,`.ok()` 把它变成 `Option`,配合 `and_then` 就得到 `Option<&str>`:头部不存在或者不是合法字符串,都归为 `None`。

**用 `starts_with` 而不是 `==` 比较:**

```rust
if content_type.starts_with("application/json") {
```

因为 `Content-Type` 常常带参数。浏览器可能发 `application/json; charset=utf-8`,精确相等就匹配不上了。用 `starts_with` 能容忍这些后缀。

> **更好的写法**:`starts_with` 有两个真实的判断错误。一是**误判**:`application/jsonx` 也会被当成 JSON。二是**漏判**:[RFC 9110](https://www.rfc-editor.org/rfc/rfc9110#name-media-type) 规定媒体类型**大小写不敏感**,而 `starts_with` 是大小写敏感的——客户端发 `Content-Type: Application/JSON` 完全合法,却会落到最后的 `415` 分支。第二个问题更隐蔽,因为主流客户端都发小写,平时撞不上,一旦遇到某个发大写的客户端就会变成难查的"偶发 415"。
>
> 正确做法是用 `mime` crate 解析后比较 `type_()` 和 `subtype()`,axum 内置的 `Json`、`Form` 提取器正是这么做的:
>
> ```rust
> let mime: mime::Mime = content_type.parse().ok()?;
> if mime.type_() == mime::APPLICATION && mime.subtype() == mime::JSON { /* ... */ }
> ```
>
> 示例用 `starts_with` 是为了不引入额外依赖、让判断逻辑一眼可读——这一章的重点是 `FromRequest` 的实现结构,不是 MIME 解析。练习题 4 会让你换成 `mime`。

**复用内置提取器:**

```rust
let Json(payload) = req.extract().await.map_err(IntoResponse::into_response)?;
```

这一行做了三件事。

`req.extract()` 是前置知识讲的 `RequestExt` 方法,它把 `req` 的所有权交给内置的 `Json` 提取器。左边解构 `Json(payload)`,编译器据此推断要提取 `Json<T>`。

`.map_err(IntoResponse::into_response)` 把内置提取器的拒绝类型转成 `Response`。`Json` 的拒绝是 `JsonRejection`,它实现了 `IntoResponse`;这里把 `into_response` 当函数值传给 `map_err`(等价于 `|e| e.into_response()`,但更简洁)。

`?` 提前返回错误。因为 `Rejection = Response`,而 `map_err` 已经转成了 `Response`,类型对得上。

**这个"复用"是本章最值钱的手法。** 你没有重新实现 JSON 解析,没有重新写 `Content-Type` 检查,没有重新处理反序列化错误——全部交给了 axum 内置的 `Json`。你只加了一层"选哪个"的判断。自定义提取器几乎总该这么写:**包装已有的提取器,而不是从零解析。**

**两种格式都不匹配:**

```rust
Err(StatusCode::UNSUPPORTED_MEDIA_TYPE.into_response())
```

返回 `415`。`StatusCode` 自己实现了 `IntoResponse`(生成一个只有状态码、body 为空的响应),`.into_response()` 把它转成 `Response` 以匹配 `Rejection` 类型。

注意这个分支覆盖两种情况:`Content-Type` 头部完全不存在(外层 `if let` 不进入),或者存在但不是这两种格式(两个内层 `if` 都不匹配)。都归为 `415` 是合理的——语义都是"你发的格式我不支持"。

**`_state: &S` 参数没用上。** 前置知识说过,`Json` 和 `Form` 的提取不需要应用状态,所以这里用 `_` 前缀标记未使用,避免编译警告。

**跑起来看**:

```bash
cargo run
```

JSON 依然能用:

```bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: application/json' \
  -d '{"foo":"json works"}'
```

```
HTTP/1.1 200 OK
content-length: 0
```

```
[src/main.rs:43:5] payload = Payload {
    foo: "json works",
}
```

表单现在也能用了:

```bash
curl -i -X POST http://127.0.0.1:3000/ -d 'foo=form works'
```

```
HTTP/1.1 200 OK
content-length: 0
```

```
[src/main.rs:43:5] payload = Payload {
    foo: "form works",
}
```

**同一个 handler,同一个 `Payload`,两种请求格式。** 这就是本章的目标。

带 charset 参数的 `Content-Type` 也能识别(验证 `starts_with` 的作用):

```bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: application/json; charset=utf-8' \
  -d '{"foo":"with charset"}'
```

```
HTTP/1.1 200 OK
```

### 步骤 3:观察三种失败路径

**要做什么**:不写新代码,把三条错误分支各触发一次,理解 `Rejection = Response` 的实际效果。

**不支持的格式**——走自己造的 `415`:

```bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: text/plain' \
  -d 'hello'
```

```
HTTP/1.1 415 Unsupported Media Type
content-length: 0
date: Thu, 30 Jul 2026 07:36:02 GMT
```

body 为空,因为 `StatusCode` 单独转成响应时不带 body。

**完全没有 Content-Type**——同样走 `415`:

```bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type:' \
  --data-binary 'whatever'
```

```
HTTP/1.1 415 Unsupported Media Type
```

**JSON 格式错误**——走内置 `Json` 的拒绝:

```bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: application/json' \
  -d '{"foo":'
```

```
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8
content-length: 96

Failed to parse the request body as JSON: EOF while parsing a value at line 1 column 7
```

**注意这是 `400` 而不是 `415`,而且错误信息很详细**——精确到行列位置。这是内置 `Json` 提取器的拒绝被 `map_err` 原样转发出来的结果。

**JSON 合法但字段不对**:

```bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: application/json' \
  -d '{"bar":"wrong field"}'
```

```
HTTP/1.1 422 Unprocessable Entity
content-type: text/plain; charset=utf-8
content-length: 63

Failed to deserialize the JSON body into the target type: missing field `foo` at line 1 column 21
```

`422`,和[第 03 章](03-form.md#步骤-3观察提取失败时的行为)表单缺字段时是同一个状态码。

把五种失败摆在一起看:

| 请求 | 状态码 | 来源 |
|---|---|---|
| `Content-Type: text/plain` | `415` | 你写的 `Err(StatusCode::UNSUPPORTED_MEDIA_TYPE)` |
| 无 `Content-Type` | `415` | 同上 |
| JSON 语法错误 | `400` | 内置 `Json` 的拒绝,原样转发 |
| JSON 缺字段 | `422` | 同上 |
| 表单缺字段 | `422` | 内置 `Form` 的拒绝,原样转发 |

**这张表说明了 `Rejection = Response` 的价值:** 三种来源、三种状态码、三种 body 格式,全都塞进了同一个 `Response` 类型里。要是把 `Rejection` 定成 `(StatusCode, String)`,就得手工把内置拒绝的状态码和文本拆出来重新组装——不但麻烦,还容易丢信息。

## 完整代码

`src/main.rs`:

```rust
use axum::{
    extract::{FromRequest, Request},
    http::{header::CONTENT_TYPE, StatusCode},
    response::{IntoResponse, Response},
    routing::post,
    Form, Json, RequestExt, Router,
};
use serde::{Deserialize, Serialize};
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

    let app = Router::new().route("/", post(handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

#[derive(Debug, Serialize, Deserialize)]
struct Payload {
    foo: String,
}

async fn handler(JsonOrForm(payload): JsonOrForm<Payload>) {
    dbg!(payload);
}

struct JsonOrForm<T>(T);

impl<S, T> FromRequest<S> for JsonOrForm<T>
where
    S: Send + Sync,
    Json<T>: FromRequest<()>,
    Form<T>: FromRequest<()>,
    T: 'static,
{
    type Rejection = Response;

    async fn from_request(req: Request, _state: &S) -> Result<Self, Self::Rejection> {
        let content_type_header = req.headers().get(CONTENT_TYPE);
        let content_type = content_type_header.and_then(|value| value.to_str().ok());

        if let Some(content_type) = content_type {
            if content_type.starts_with("application/json") {
                let Json(payload) = req.extract().await.map_err(IntoResponse::into_response)?;
                return Ok(Self(payload));
            }

            if content_type.starts_with("application/x-www-form-urlencoded") {
                let Form(payload) = req.extract().await.map_err(IntoResponse::into_response)?;
                return Ok(Self(payload));
            }
        }

        Err(StatusCode::UNSUPPORTED_MEDIA_TYPE.into_response())
    }
}
```

## 代价与边界

手写 `FromRequest` 换来的是灵活性,也把几件事交到你手里——官方内置提取器已经替你挡住的坑,现在要自己看着。

**错误格式不统一。** 步骤 4 那张表已经写明:`415` 是空 body,`400`/`422` 是纯文本。正经 API 通常希望所有错误都是同一种 JSON 结构。统一错误格式是下面几章的主题:[第 10 章](10-customize-extractor-error.md) 展示三种把提取器拒绝改成统一 JSON 的办法,[第 14 章](14-error-handling.md) 讲整个应用层面的错误处理设计。

**`starts_with` 不是 MIME 解析。** 步骤 3 的「更好的写法」说过:它会误判 `application/jsonx`,也会漏判合法的大写 `Application/JSON`。演示 `FromRequest` 结构时够用,上线前应换成 `mime` 或等价解析(练习题 4)。

**什么时候值得自己写。** 同一条路由要同时吃 JSON 和表单、或者要在提取阶段掺进业务上下文时,手写提取器是对的路。只是想换错误 JSON 外壳,别从这一章起步——直接看[第 10 章](10-customize-extractor-error.md) 的派生宏做法,代码量小一个数量级。

## 动手练习

**1. 让 `415` 响应带上说明信息。**

提示:把 `Err(StatusCode::UNSUPPORTED_MEDIA_TYPE.into_response())` 改成带 body 的形式,比如 `(StatusCode::UNSUPPORTED_MEDIA_TYPE, "expected application/json or application/x-www-form-urlencoded").into_response()`。更进一步:把客户端实际发来的 `Content-Type` 也写进错误信息里,方便对方排查。

**2. 加上对 `multipart/form-data` 的支持,或者明确拒绝它。**

提示:`multipart` 的处理方式和前两者不同(它是流式的,不能一次性反序列化成 struct),所以简单加个分支行不通。先想清楚为什么,再决定是返回一个更具体的错误信息,还是查阅[第 13 章](13-multipart-form.md)后再来实现。

**3. 把 handler 改成回显收到的数据。**

提示:返回类型从 `()` 改成 `Json<Payload>`,把 `payload` 返回出去。这时 `Payload` 上那个一直没用到的 `Serialize` 派生就派上用场了。验证两种输入格式的响应是否都是 JSON——想想为什么响应格式没有跟着请求格式变(提示:内容协商的出方向靠的是 `Accept` 头部,不是 `Content-Type`)。

**4. 用 `mime` crate 正确解析 `Content-Type`,修掉大小写敏感的问题。**

提示:`cargo add mime`,把两处 `starts_with` 换成解析后比较 `mime.type_()` 和 `mime.subtype()`(常量是 `mime::APPLICATION`、`mime::JSON`、`mime::WWW_FORM_URLENCODED`)。改之前先复现问题:`curl -X POST http://127.0.0.1:3000/ -H 'Content-Type: Application/JSON' -d '{"foo":"bar"}'` 应该返回 `415`;改完之后应该正常解析。

## 小结与下一章预告

`FromRequest` 是提取器的入方向 trait,和 `IntoResponse` 构成一对。实现它需要定一个 newtype 作挂载点、指定 `Rejection` 关联类型、在 `from_request` 里返回 `Result<Self, Rejection>`。`from_request` 拿到 `Request` 的所有权,所以这类提取器每个 handler 只能有一个且必须放最后;`FromRequestParts` 只借用 `&mut Parts`,可以有多个。

自定义提取器的正确姿势是**包装内置提取器而不是从零解析**:本章只写了"选 JSON 还是选表单"的判断,解析、校验、错误信息全部复用了 axum 的 `Json` 和 `Form`。`RequestExt::extract` 让这种复用写起来很顺。`Rejection = Response` 适合失败形态多样的场合,代价是客户端拿到的错误格式不统一。

本章的提取器消耗了 body,而 body 只能读一次。这个限制会带来一个实际难题:**如果中间件也想看 body 怎么办?** [第 09 章 consume-body-in-extractor-or-middleware](09-consume-body-in-extractor-or-middleware.md) 解决它——你会第一次写 axum 中间件,并学会"把请求拆开、缓冲 body、再装回去"这个关键技巧。
