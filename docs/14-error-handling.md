# 第 14 章 error-handling:一个应用该怎么组织错误

## 本章你将做出什么

一个 `POST /users` 接口,它有两种失败方式,而且**处理方式截然不同**:

- 客户端发的 JSON 不合法 → 返回 `400`,把具体原因告诉客户端,**不记日志**
- 内部依赖的库出错 → 返回 `500`,只告诉客户端 "Something went wrong",**详细原因记进日志**

前面章节的错误处理都是零散的:每章各写各的错误类型,`.unwrap()` 散落各处。这一章把它们收拢成一套可以复制到真实项目里的做法。

学完本章你将掌握:

- 4xx 和 5xx 的根本区别,以及它决定了"暴露什么、记录什么"
- 用一个 enum 收拢全应用的错误,让 `?` 贯穿整条调用链
- 一个反直觉但重要的技巧:**不要在 `into_response` 里写日志**,改用响应扩展 + 中间件
- 为什么错误日志要带上请求上下文,以及 `TraceLayer` 的 span 如何提供它

## 前置知识

### 4xx 与 5xx 决定了两件事

[第 00 章](00-web-basics.md#状态码一个三位数的分类回答)讲过分界:`4xx` 是"你的请求有问题",`5xx` 是"我的代码有问题"。这个区别不只是语义,它直接决定了两个实际操作:

| | `4xx` 客户端错误 | `5xx` 服务端错误 |
|---|---|---|
| **告诉客户端多少** | 尽量详细 | 尽量少 |
| **要不要记日志** | 通常不记 | 必须记 |

**为什么 4xx 要详细但不记日志。** 客户端错误是**对方能修复的**——告诉他"缺少 `name` 字段",他改了就好。信息越具体,对接效率越高。但这类错误是正常流量的一部分:扫描器、写错的前端代码、过期的客户端每天都在制造它们。全记下来只会淹没真正需要注意的问题。

**为什么 5xx 要记日志但不详细。** 服务端错误是**只有你能修复的**,客户端知道再多也没用。而且详细错误信息往往泄露内部结构——数据库表名、文件路径、依赖的第三方服务、甚至连接字符串。这些是攻击者摸清系统的素材。

所以标准做法是:**对外说 "Something went wrong",对内记完整的错误链。** 本章的代码就是这个原则的直接实现。

### 用一个 enum 收拢应用错误

一个 handler 可能因为很多原因失败:请求格式不对、数据库查询失败、调用外部服务超时、业务规则不满足。如果每种失败用不同的错误类型,handler 的返回类型会变得没法写。

常见做法是定义一个应用级的错误 enum:

```rust
enum AppError {
    JsonRejection(JsonRejection),
    TimeError(time_library::Error),
    // 真实项目里还会有 DatabaseError、NotFound、Forbidden……
}
```

然后为每种来源错误实现 `From`:

```rust
impl From<time_library::Error> for AppError {
    fn from(error: time_library::Error) -> Self {
        Self::TimeError(error)
    }
}
```

**有了 `From`,`?` 就能自动转换。** [第 12 章](12-validator.md#步骤-2声明校验规则包进提取器)用 `thiserror` 的 `#[from]` 自动生成过这些 impl,本章的官方示例选择手写——效果一样,只是更直白地展示了发生了什么。

最后给 `AppError` 实现 `IntoResponse`,handler 就能写成:

```rust
async fn users_create(...) -> Result<AppJson<User>, AppError> {
    let created_at = Timestamp::now()?;   // 失败自动转成 AppError 并返回
    // ...
}
```

`Result<T, E>` 实现 `IntoResponse` 的前提是 `T` 和 `E` 都实现——[第 05 章](05-templates-minijinja.md#步骤-1布局模板-首页认识-state)提过一次。

### 为什么不能在 into_response 里记日志

这是本章最值得记住的一点,而且违反直觉。

看起来最自然的写法是在错误转成响应的地方顺手记一条日志:

```rust
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        match self {
            AppError::TimeError(err) => {
                tracing::error!(?err, "出错了");   // ← 看起来很合理
                (StatusCode::INTERNAL_SERVER_ERROR, "Something went wrong").into_response()
            }
            // ...
        }
    }
}
```

官方示例的注释明确否定了这种写法,理由是:**这给一个纯转换函数引入了副作用。**

`IntoResponse` 的职责是"把值变成响应",它应该是个纯函数——同样的输入总是产出同样的输出,不改变外部世界。往里面塞日志会带来几个实际问题:

**难以测试。** 测试 `into_response` 时会产生日志输出,想验证"这个错误有没有被记录"却又得去捕获日志系统。

**日志缺少请求上下文。** `into_response` 被调用时,你手上只有一个错误值,不知道它来自哪个请求、哪个路径、哪个用户。日志里只有一句"出错了",排查时毫无用处。

**记录时机不受控。** 有些错误可能被上层中间件捕获并转换,最终根本没返回给客户端,但日志已经记下了。

**本章的做法是把"转换"和"记录"分开:**

1. `into_response` 只负责转换,顺便**把错误对象塞进响应的扩展**里
2. 一个中间件在响应返回时检查扩展,发现错误就记日志

这样日志发生在中间件里,而中间件运行在 `TraceLayer` 建立的 span 之内,自动带上请求上下文。

### 响应扩展(Extensions)

`Request` 和 `Response` 都有一个 `extensions` 容器,可以按类型存放任意值:

```rust
response.extensions_mut().insert(Arc::new(err));       // 存
response.extensions().get::<Arc<AppError>>()           // 取
```

它是个按 `TypeId` 索引的 map,每种类型只能存一个值。用途是**在处理链的不同环节之间传递带外数据**——不走 HTTP 头部(不该暴露给客户端),也不走函数参数(中间件和 handler 之间没有直接调用关系)。

前面章节其实已经间接用过它:`MatchedPath` 就是 axum 自动放进请求扩展的,[第 10 章](10-customize-extractor-error.md#matchedpath拿到路由模式而非实际路径)用 `parts.extract::<MatchedPath>()` 取出来过。

**为什么要包一层 `Arc`。** 扩展要求存入的值满足 `Clone + Send + Sync + 'static`。`AppError` 内部装着第三方库的错误类型,不一定实现 `Clone`。`Arc<T>` 对任意 `T` 都是 `Clone` 的(克隆只增加引用计数),所以包一层就绕开了这个约束。这个技巧[第 05 章](05-templates-minijinja.md#state-与-with_state)讲 `State` 时见过,思路一致。

### 日志需要请求上下文

一条 `ERROR failed to get time` 的日志几乎没有价值——你不知道它属于哪个请求。有价值的是这样:

```
ERROR request{method=POST uri=/users matched_path="/users"}: error from time_library err=failed to get time
```

前面那段 `request{...}` 是 **span**(跨度)。tracing 的核心概念之一:span 表示一段有始有终的上下文,在 span 内部记录的所有事件都会自动带上 span 的字段。

`TraceLayer` 会为每个请求创建一个 span。只要日志记录发生在这个 span 内部,就自动获得请求的方法、路径等信息,不用手动传递。

这也解释了本章代码里那个容易看漏的细节——**两个 layer 的顺序不能反**:

```rust
.layer(from_fn(log_app_errors))   // 内层
.layer(TraceLayer::new_for_http() /* ... */)   // 外层
```

axum 的 `.layer()` 是**由内向外**包装的:后调用的 `.layer()` 成为更外层。所以上面这段里 `TraceLayer` 在外、`log_app_errors` 在内。请求先经过 `TraceLayer`(span 建立),再进入 `log_app_errors`,于是中间件里的 `tracing::error!` 落在 span 内部。

顺序反过来的话,`log_app_errors` 在外层先执行,那时 span 还没建立,日志就没有请求上下文了。

> **本章不深入**:`.layer()` 的由内向外包装规则、`TraceLayer` 的各种回调(`on_request`/`on_response`/`on_failure`)、span 与 event 的关系、日志级别设计,都是[第 17 章](17-tracing-aka-logging.md)的主题。本章只需理解"`TraceLayer` 要放在外层,这样错误日志才有请求上下文",配置照抄即可。

## 项目搭建

```bash
cargo new error-handling
cd error-handling
```

```bash
cargo add axum --features macros
cargo add serde --features derive
cargo add tokio --features full
cargo add tower-http --features trace
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`axum` 的 `macros` feature 提供 `#[derive(FromRequest)]`,[第 10 章](10-customize-extractor-error.md#步骤-3做法二派生-fromrequest)用过。

## 分步构建

### 步骤 1:能失败的 handler 与最简单的错误类型

**要做什么**:搭出骨架——一个会随机失败的第三方库、一个应用错误类型、一个用 `?` 传播错误的 handler。

```rust
// src/main.rs
use axum::{http::StatusCode, response::{IntoResponse, Response}, routing::post, Json, Router};
use serde::{Deserialize, Serialize};
use time_library::Timestamp;
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

    let app = Router::new().route("/users", post(users_create));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

#[derive(Deserialize)]
struct UserParams {
    name: String,
}

#[derive(Serialize, Clone)]
struct User {
    id: u64,
    name: String,
    created_at: Timestamp,
}

async fn users_create(Json(params): Json<UserParams>) -> Result<Json<User>, AppError> {
    // 失败时 `?` 自动把 time_library::Error 转成 AppError
    let created_at = Timestamp::now()?;

    Ok(Json(User {
        id: 1,
        name: params.name,
        created_at,
    }))
}

#[derive(Debug)]
enum AppError {
    TimeError(time_library::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (StatusCode::INTERNAL_SERVER_ERROR, "Something went wrong").into_response()
    }
}

impl From<time_library::Error> for AppError {
    fn from(error: time_library::Error) -> Self {
        Self::TimeError(error)
    }
}

// Imagine this is some third party library that we're using. It sometimes returns errors which we
// want to log.
mod time_library {
    use std::sync::atomic::{AtomicU64, Ordering};

    use serde::Serialize;

    #[derive(Serialize, Clone)]
    pub struct Timestamp(u64);

    impl Timestamp {
        pub fn now() -> Result<Self, Error> {
            static COUNTER: AtomicU64 = AtomicU64::new(0);

            // Fail on every third call just to simulate errors
            if COUNTER.fetch_add(1, Ordering::SeqCst).is_multiple_of(3) {
                Err(Error::FailedToGetTime)
            } else {
                Ok(Self(1337))
            }
        }
    }

    #[derive(Debug, Clone)]
    pub enum Error {
        FailedToGetTime,
    }

    impl std::fmt::Display for Error {
        fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
            write!(f, "failed to get time")
        }
    }
}
```

**`mod time_library` 模拟一个不可靠的第三方库。** 它用一个静态计数器,每调用三次失败一次——这样你不用真的接一个会出错的服务就能演练错误路径。这个模块是示例的脚手架,不是要学的内容。

`is_multiple_of` 是 Rust 1.87 引入的整数方法,等价于 `% 3 == 0`。

**`Timestamp::now()?`** 是本步骤的核心。`now()` 返回 `Result<Timestamp, time_library::Error>`,而函数的错误类型是 `AppError`。`?` 之所以能工作,全靠下面那个 `impl From<time_library::Error> for AppError`——[第 12 章](12-validator.md#步骤-2声明校验规则包进提取器)讲过这个机制。

> **更好的写法**:这里的两个 `impl From` 是手写的,用 `thiserror` 的 `#[from]` 属性能自动生成,错误来源一多能省下可观的样板([第 12 章](12-validator.md#步骤-2声明校验规则包进提取器)就是那么写的)。示例选择手写是为了直白展示 `?` 背后到底发生了什么——这在教学上是对的,但你自己的项目建议用 `thiserror`。练习题 3 会让你换。

**跑起来看**:

```bash
cargo run
```

连发四次请求。第 1 次和第 4 次会失败(计数器 0 和 3 都是 3 的倍数):

```bash
for i in 1 2 3 4; do
  echo "--- 第 $i 次 ---"
  curl -s -i -X POST http://127.0.0.1:3000/users \
    -H 'Content-Type: application/json' -d '{"name":"zhangsan"}' | head -1
done
```

```
--- 第 1 次 ---
HTTP/1.1 500 Internal Server Error
--- 第 2 次 ---
HTTP/1.1 200 OK
--- 第 3 次 ---
HTTP/1.1 200 OK
--- 第 4 次 ---
HTTP/1.1 500 Internal Server Error
```

失败时客户端只看到:

```
Something went wrong
```

**符合前置知识说的"对外说得少"。但现在的问题是:对内也什么都没说。** 服务端终端一条错误日志都没有——`FailedToGetTime` 这个信息被彻底丢弃了。线上出了 500,你无从查起。

### 步骤 2:区分两类错误

**要做什么**:加上第二种错误(JSON 不合法),让两类错误得到不同的对待。

用[第 10 章](10-customize-extractor-error.md#步骤-3做法二派生-fromrequest)的派生做法包一个自己的 JSON 提取器:

```rust
// src/main.rs
use axum::{
    extract::{rejection::JsonRejection, FromRequest},     // 新增
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::post,
    Router,                                               // 改动:去掉 Json
};
use serde::{Deserialize, Serialize};
use time_library::Timestamp;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// ...(main 不变)

// 改动:提取器和返回类型都换成 AppJson
async fn users_create(
    // Make sure to use our own JSON extractor so we get input errors formatted in a way that
    // matches our application
    AppJson(params): AppJson<UserParams>,
) -> Result<AppJson<User>, AppError> {
    let created_at = Timestamp::now()?;

    Ok(AppJson(User {
        id: 1,
        name: params.name,
        created_at,
    }))
}

// 新增:自己的 JSON 提取器,拒绝时走 AppError
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(AppError))]
struct AppJson<T>(T);

impl<T> IntoResponse for AppJson<T>
where
    axum::Json<T>: IntoResponse,
{
    fn into_response(self) -> Response {
        axum::Json(self.0).into_response()
    }
}

// 改动:两种错误
#[derive(Debug)]
enum AppError {
    // The request body contained invalid JSON
    JsonRejection(JsonRejection),
    // Some error from a third party library we're using
    TimeError(time_library::Error),
}

// 改动:两类错误区别对待
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        // How we want errors responses to be serialized
        #[derive(Serialize)]
        struct ErrorResponse {
            message: String,
        }

        let (status, message) = match &self {
            AppError::JsonRejection(rejection) => {
                // This error is caused by bad user input so don't log it
                (rejection.status(), rejection.body_text())
            }
            AppError::TimeError(_err) => {
                // Don't expose any details about the error to the client
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "Something went wrong".to_owned(),
                )
            }
        };

        (status, AppJson(ErrorResponse { message })).into_response()
    }
}

// 新增
impl From<JsonRejection> for AppError {
    fn from(rejection: JsonRejection) -> Self {
        Self::JsonRejection(rejection)
    }
}

// ...(From<time_library::Error> 和 mod time_library 不变)
```

**`AppJson<T>` 一个类型干两件事**:当提取器(靠 `#[derive(FromRequest)]`)、当响应包装(靠手写的 `IntoResponse`)。这正是[第 08 章](08-parse-body-based-on-content-type.md#json收-json-与发-json)讲的 `axum::Json` 的两副面孔,只是换成了自己的类型,好处是**错误响应也走同一套 JSON 序列化**,接口格式统一。

**`match &self` 的两个分支体现了前置知识那张表**:

- `JsonRejection` → 用 `rejection.status()` 和 `rejection.body_text()`,把具体原因**原样告诉客户端**。注释写着 "don't log it"。
- `TimeError` → 固定 `500` + "Something went wrong",**什么细节都不给**。注释写着 "Don't expose any details"。

`ErrorResponse` 这个 struct 定义在函数体**内部**——Rust 允许在任何作用域定义类型。这里这么写是因为它只在这一处使用,不需要污染模块命名空间。

**跑起来看**:

```bash
cargo run
```

发一个不合法的 JSON:

```bash
curl -i -X POST http://127.0.0.1:3000/users \
  -H 'Content-Type: application/json' -d '{"name":'
```

```
HTTP/1.1 400 Bad Request
content-type: application/json
content-length: 91

{"message":"Failed to parse the request body as JSON: EOF while parsing a value at line 1 column 8"}
```

**`400`,JSON 格式,原因具体。** 客户端拿到这个信息就知道该怎么改。

漏字段:

```bash
curl -s -X POST http://127.0.0.1:3000/users \
  -H 'Content-Type: application/json' -d '{}'
```

```json
{"message":"Failed to deserialize the JSON body into the target type: missing field `name` at line 1 column 2"}
```

内部错误(多试几次触发):

```bash
for i in 1 2 3 4; do
  curl -s -X POST http://127.0.0.1:3000/users \
    -H 'Content-Type: application/json' -d '{"name":"zhangsan"}'
  echo
done
```

```
{"message":"Something went wrong"}
{"id":1,"name":"zhangsan","created_at":1337}
{"id":1,"name":"zhangsan","created_at":1337}
{"message":"Something went wrong"}
```

**成功和失败都是 JSON,格式统一了。** 而且两类错误的详略程度截然不同——这就是本章要达到的效果。

**但日志问题还在。** 服务端终端依然没有任何 `TimeError` 的记录。

### 步骤 3:用响应扩展把错误传给中间件

**要做什么**:让内部错误被记录,同时保持 `into_response` 无副作用。

```rust
// src/main.rs
use std::sync::Arc;                                       // 新增

use axum::{
    extract::{rejection::JsonRejection, FromRequest, Request},   // 改动:加上 Request
    http::StatusCode,
    middleware::{from_fn, Next},                                 // 新增
    response::{IntoResponse, Response},
    routing::post,
    Router,
};
// ...(其余 use 不变)

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    // 改动:挂上日志中间件
    let app = Router::new()
        .route("/users", post(users_create))
        .layer(from_fn(log_app_errors));

    // ...(listener 和 serve 不变)
}

// ...(users_create、AppJson 不变)

// 改动:内部错误额外塞进响应扩展
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        #[derive(Serialize)]
        struct ErrorResponse {
            message: String,
        }

        let (status, message, err) = match &self {
            AppError::JsonRejection(rejection) => {
                // This error is caused by bad user input so don't log it
                (rejection.status(), rejection.body_text(), None)
            }
            AppError::TimeError(_err) => {
                // While we could simply log the error here we would introduce
                // a side-effect to our conversion, instead add the AppError to
                // the Response as an Extension
                // Don't expose any details about the error to the client
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "Something went wrong".to_owned(),
                    Some(self),
                )
            }
        };

        let mut response = (status, AppJson(ErrorResponse { message })).into_response();
        if let Some(err) = err {
            // Insert our error into the response, our logging middleware will use this.
            // By wrapping the error in an Arc we can use it as an Extension regardless of any inner types not deriving Clone.
            response.extensions_mut().insert(Arc::new(err));
        }
        response
    }
}

// ...(两个 From impl 不变)

// 新增:Our middleware is responsible for logging error details internally
async fn log_app_errors(request: Request, next: Next) -> Response {
    let response = next.run(request).await;
    // If the response contains an AppError Extension, log it.
    if let Some(err) = response.extensions().get::<Arc<AppError>>() {
        tracing::error!(?err, "an unexpected error occurred inside a handler");
    }
    response
}
```

三处变化配合完成一件事。

**`match` 现在返回三元组**,多出来的 `err` 是 `Option<AppError>`:客户端错误给 `None`(不需要记录),内部错误给 `Some(self)`(需要记录)。

注意 `Some(self)` 移动了 `self` 的所有权。这能通过借用检查,是因为 `match &self` 借用的是引用,而这个分支里 `_err` 没有实际使用——所以可以在分支里把 `self` 整个移出去。

**`response.extensions_mut().insert(Arc::new(err))`** 把错误挂到响应上。前置知识讲过为什么要 `Arc`。

**`log_app_errors` 中间件**用的是[第 09 章](09-consume-body-in-extractor-or-middleware.md#中间件在请求前后插入逻辑)的 `from_fn`。这次它只在**响应路上**做事:先 `next.run(request).await` 拿到响应,再检查扩展。请求路上什么都不做。

这正是[第 09 章](09-consume-body-in-extractor-or-middleware.md)说的"洋葱结构"——`next.run` 之后的代码在响应返回时执行。

**`tracing::error!(?err, "...")`** 里的 `?err` 按 `Debug` 记录,[第 09 章](09-consume-body-in-extractor-or-middleware.md#步骤-1一个缓冲-body-的提取器)见过这个语法。

**跑起来看**:

```bash
cargo run
```

```bash
for i in 1 2 3 4; do
  curl -s -o /dev/null -X POST http://127.0.0.1:3000/users \
    -H 'Content-Type: application/json' -d '{"name":"zhangsan"}'
done
```

服务端终端现在有了错误日志:

```
2026-07-30T10:03:12.442181Z ERROR error_handling: an unexpected error occurred inside a handler err=TimeError(FailedToGetTime)
2026-07-30T10:03:12.451093Z ERROR error_handling: an unexpected error occurred inside a handler err=TimeError(FailedToGetTime)
```

**内部错误被记录了,而客户端仍然只看到 "Something went wrong"。** 对内详细、对外简略,两边都满足。

再验证客户端错误确实**不**记日志:

```bash
curl -s -o /dev/null -X POST http://127.0.0.1:3000/users \
  -H 'Content-Type: application/json' -d '{"name":'
```

终端没有任何新增日志。符合预期——`JsonRejection` 分支给的是 `None`。

**但这条日志还不够好用**:

```
ERROR error_handling: an unexpected error occurred inside a handler err=TimeError(FailedToGetTime)
```

它不告诉你是哪个请求触发的。线上一分钟几百条这样的日志,你没法把它和具体的调用对应起来。下一步解决。

### 步骤 4:给日志加上请求上下文

**要做什么**:加上 `TraceLayer` 并自定义 span,让错误日志自动带上方法、URI 和匹配的路由。顺便把示例里剩下的 `State` 部分补齐,抵达官方示例的最终形态。

```rust
// src/main.rs
use std::{
    collections::HashMap,
    sync::{
        atomic::{AtomicU64, Ordering},
        Arc, Mutex,
    },
};

use axum::{
    extract::{rejection::JsonRejection, FromRequest, MatchedPath, Request, State},  // 改动:加上 MatchedPath、State
    http::StatusCode,
    middleware::{from_fn, Next},
    response::{IntoResponse, Response},
    routing::post,
    Router,
};
use serde::{Deserialize, Serialize};
use time_library::Timestamp;
use tower_http::trace::TraceLayer;                          // 新增
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    let state = AppState::default();                        // 新增

    // 改动:加上 TraceLayer(外层)和状态
    let app = Router::new()
        // A dummy route that accepts some JSON but sometimes fails
        .route("/users", post(users_create))
        // Layers wraps from the inside out, each new .layer() call becomes the outer one.
        // Putting TraceLayer outermost means its `request` span is entered before
        // log_app_errors runs, so our tracing::error! events inherit that span and get
        // prefixed with `request{method=… uri=… matched_path=…}:`.
        .layer(from_fn(log_app_errors)) // <- Inner layer
        .layer(
            TraceLayer::new_for_http()
                // Create our own span for the request and include the matched path. The matched
                // path is useful for figuring out which handler the request was routed to.
                .make_span_with(|req: &Request| {
                    let method = req.method();
                    let uri = req.uri();

                    // axum automatically adds this extension.
                    let matched_path = req
                        .extensions()
                        .get::<MatchedPath>()
                        .map(|matched_path| matched_path.as_str());

                    tracing::debug_span!("request", %method, %uri, matched_path)
                })
                // By default `TraceLayer` will log 5xx responses but we're doing our specific
                // logging of errors so disable that
                .on_failure(()),
        ) // <- Outer layer
        .with_state(state);                                 // 新增

    // ...(listener 和 serve 不变)
}

// 新增:存放用户的应用状态
#[derive(Default, Clone)]
struct AppState {
    next_id: Arc<AtomicU64>,
    users: Arc<Mutex<HashMap<u64, User>>>,
}

// ...(UserParams、User 不变)

// 改动:用上状态
async fn users_create(
    State(state): State<AppState>,
    // Make sure to use our own JSON extractor so we get input errors formatted in a way that
    // matches our application
    AppJson(params): AppJson<UserParams>,
) -> Result<AppJson<User>, AppError> {
    let id = state.next_id.fetch_add(1, Ordering::SeqCst);

    // We have implemented `From<time_library::Error> for AppError` which allows us to use `?` to
    // automatically convert the error
    let created_at = Timestamp::now()?;

    let user = User {
        id,
        name: params.name,
        created_at,
    };

    state.users.lock().unwrap().insert(id, user.clone());

    Ok(AppJson(user))
}

// ...(其余不变)
```

**两个 layer 的顺序是重点,前置知识讲过**:`from_fn(log_app_errors)` 先调用所以在内层,`TraceLayer` 后调用所以在外层。官方示例的注释专门解释了这一点,还用 `// <- Inner layer` / `// <- Outer layer` 标了出来。

**`make_span_with` 自定义 span 的字段**:方法、URI、匹配到的路由模式。`%method` 和 `%uri` 是 tracing 的 `Display` 前缀。`MatchedPath` 从请求扩展里取——[第 10 章](10-customize-extractor-error.md#matchedpath拿到路由模式而非实际路径)讲过它和 `Uri` 的区别,以及为什么做监控时前者更有用。

**`.on_failure(())`** 关掉 `TraceLayer` 自带的 5xx 日志。不关的话每个内部错误会被记两次——一次来自 `TraceLayer`,一次来自我们的中间件。传 `()` 表示"什么都不做",这是 `tower-http` 的一个约定。

**`AppState` 部分是示例的脚手架**,不是本章重点:`AtomicU64` 生成自增 ID,`Arc<Mutex<HashMap>>` 存用户。

> **本章不深入**:`Arc<Mutex<HashMap<...>>>` 是**共享可变状态**——[第 05 章](05-templates-minijinja.md#state-与-with_state)讲的 `Arc` 只提供共享只读访问,套一层 `Mutex` 才能写。本章拿它当存储用,不展开。锁的选择(`Mutex` vs `RwLock`)、`.lock().unwrap()` 为什么会 panic、持锁跨 `await` 的陷阱,见[第 22 章](22-key-value-store.md)。

**跑起来看**:

```bash
cargo run
```

```bash
for i in 1 2 3 4; do
  curl -s -o /dev/null -X POST http://127.0.0.1:3000/users \
    -H 'Content-Type: application/json' -d '{"name":"zhangsan"}'
done
```

日志现在长这样:

```
2026-07-30T10:15:03.221847Z DEBUG request{method=POST uri=/users matched_path="/users"}: tower_http::trace::on_request: started processing request
2026-07-30T10:15:03.222103Z ERROR request{method=POST uri=/users matched_path="/users"}: error_handling: an unexpected error occurred inside a handler err=TimeError(FailedToGetTime)
2026-07-30T10:15:03.222291Z DEBUG request{method=POST uri=/users matched_path="/users"}: tower_http::trace::on_response: finished processing request latency=0 ms status=500
```

**`ERROR` 那一行前面多了 `request{method=POST uri=/users matched_path="/users"}`。** 这就是 span 的作用:错误日志自动继承了请求上下文,不用手动传递任何东西。而且能和前后的 `on_request`/`on_response` 日志串成一条完整的请求轨迹。

成功的请求只有两条 DEBUG,没有 ERROR:

```
DEBUG request{method=POST uri=/users matched_path="/users"}: tower_http::trace::on_request: started processing request
DEBUG request{method=POST uri=/users matched_path="/users"}: tower_http::trace::on_response: finished processing request latency=0 ms status=200
```

**验证一下 layer 顺序确实重要。** 把两个 `.layer()` 调换位置:

```rust
        .layer(
            TraceLayer::new_for_http()
                // ...
        )
        .layer(from_fn(log_app_errors))   // 现在它变成外层了
```

重新运行并触发一次内部错误,`ERROR` 那一行变成:

```
2026-07-30T10:18:44.117203Z ERROR error_handling: an unexpected error occurred inside a handler err=TimeError(FailedToGetTime)
```

**`request{...}` 前缀消失了。** 因为 `log_app_errors` 现在跑在 `TraceLayer` 外面,记录日志时那个 span 已经退出了。代码逻辑一个字没改,日志的可用性却大幅下降——而且不会有任何报错提示你写反了。

改回来。

## 完整代码

`src/main.rs`:

```rust
use std::{
    collections::HashMap,
    sync::{
        atomic::{AtomicU64, Ordering},
        Arc, Mutex,
    },
};

use axum::{
    extract::{rejection::JsonRejection, FromRequest, MatchedPath, Request, State},
    http::StatusCode,
    middleware::{from_fn, Next},
    response::{IntoResponse, Response},
    routing::post,
    Router,
};
use serde::{Deserialize, Serialize};
use time_library::Timestamp;
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

    let state = AppState::default();

    let app = Router::new()
        // A dummy route that accepts some JSON but sometimes fails
        .route("/users", post(users_create))
        // Layers wraps from the inside out, each new .layer() call becomes the outer one.
        // Putting TraceLayer outermost means its `request` span is entered before
        // log_app_errors runs, so our tracing::error! events inherit that span and get
        // prefixed with `request{method=… uri=… matched_path=…}:`.
        .layer(from_fn(log_app_errors)) // <- Inner layer
        .layer(
            TraceLayer::new_for_http()
                // Create our own span for the request and include the matched path. The matched
                // path is useful for figuring out which handler the request was routed to.
                .make_span_with(|req: &Request| {
                    let method = req.method();
                    let uri = req.uri();

                    // axum automatically adds this extension.
                    let matched_path = req
                        .extensions()
                        .get::<MatchedPath>()
                        .map(|matched_path| matched_path.as_str());

                    tracing::debug_span!("request", %method, %uri, matched_path)
                })
                // By default `TraceLayer` will log 5xx responses but we're doing our specific
                // logging of errors so disable that
                .on_failure(()),
        ) // <- Outer layer
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

#[derive(Default, Clone)]
struct AppState {
    next_id: Arc<AtomicU64>,
    users: Arc<Mutex<HashMap<u64, User>>>,
}

#[derive(Deserialize)]
struct UserParams {
    name: String,
}

#[derive(Serialize, Clone)]
struct User {
    id: u64,
    name: String,
    created_at: Timestamp,
}

async fn users_create(
    State(state): State<AppState>,
    // Make sure to use our own JSON extractor so we get input errors formatted in a way that
    // matches our application
    AppJson(params): AppJson<UserParams>,
) -> Result<AppJson<User>, AppError> {
    let id = state.next_id.fetch_add(1, Ordering::SeqCst);

    // We have implemented `From<time_library::Error> for AppError` which allows us to use `?` to
    // automatically convert the error
    let created_at = Timestamp::now()?;

    let user = User {
        id,
        name: params.name,
        created_at,
    };

    state.users.lock().unwrap().insert(id, user.clone());

    Ok(AppJson(user))
}

// Create our own JSON extractor by wrapping `axum::Json`. This makes it easy to override the
// rejection and provide our own which formats errors to match our application.
//
// `axum::Json` responds with plain text if the input is invalid.
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(AppError))]
struct AppJson<T>(T);

impl<T> IntoResponse for AppJson<T>
where
    axum::Json<T>: IntoResponse,
{
    fn into_response(self) -> Response {
        axum::Json(self.0).into_response()
    }
}

// The kinds of errors we can hit in our application.
#[derive(Debug)]
enum AppError {
    // The request body contained invalid JSON
    JsonRejection(JsonRejection),
    // Some error from a third party library we're using
    TimeError(time_library::Error),
}

// Tell axum how `AppError` should be converted into a response.
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        // How we want errors responses to be serialized
        #[derive(Serialize)]
        struct ErrorResponse {
            message: String,
        }

        let (status, message, err) = match &self {
            AppError::JsonRejection(rejection) => {
                // This error is caused by bad user input so don't log it
                (rejection.status(), rejection.body_text(), None)
            }
            AppError::TimeError(_err) => {
                // While we could simply log the error here we would introduce
                // a side-effect to our conversion, instead add the AppError to
                // the Response as an Extension
                // Don't expose any details about the error to the client
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "Something went wrong".to_owned(),
                    Some(self),
                )
            }
        };

        let mut response = (status, AppJson(ErrorResponse { message })).into_response();
        if let Some(err) = err {
            // Insert our error into the response, our logging middleware will use this.
            // By wrapping the error in an Arc we can use it as an Extension regardless of any inner types not deriving Clone.
            response.extensions_mut().insert(Arc::new(err));
        }
        response
    }
}

impl From<JsonRejection> for AppError {
    fn from(rejection: JsonRejection) -> Self {
        Self::JsonRejection(rejection)
    }
}

impl From<time_library::Error> for AppError {
    fn from(error: time_library::Error) -> Self {
        Self::TimeError(error)
    }
}

// Our middleware is responsible for logging error details internally
async fn log_app_errors(request: Request, next: Next) -> Response {
    let response = next.run(request).await;
    // If the response contains an AppError Extension, log it.
    if let Some(err) = response.extensions().get::<Arc<AppError>>() {
        tracing::error!(?err, "an unexpected error occurred inside a handler");
    }
    response
}

// Imagine this is some third party library that we're using. It sometimes returns errors which we
// want to log.
mod time_library {
    use std::sync::atomic::{AtomicU64, Ordering};

    use serde::Serialize;

    #[derive(Serialize, Clone)]
    pub struct Timestamp(u64);

    impl Timestamp {
        pub fn now() -> Result<Self, Error> {
            static COUNTER: AtomicU64 = AtomicU64::new(0);

            // Fail on every third call just to simulate errors
            if COUNTER.fetch_add(1, Ordering::SeqCst).is_multiple_of(3) {
                Err(Error::FailedToGetTime)
            } else {
                Ok(Self(1337))
            }
        }
    }

    #[derive(Debug, Clone)]
    pub enum Error {
        FailedToGetTime,
    }

    impl std::fmt::Display for Error {
        fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
            write!(f, "failed to get time")
        }
    }
}
```

## 代价与边界

**这套做法适合中小型应用。** 单个 `AppError` enum 收拢全部错误,在几十个错误来源以内很好用。项目再大就会出现问题:enum 变得臃肿、模块之间被迫共享一个错误类型、修改它会触发全项目重编译。那时的做法通常是按模块分层定义错误,再逐层 `From` 转换。

**`?` 的自动转换会丢失上下文。** `Timestamp::now()?` 只知道"时间库出错了",不知道"在为用户 zhangsan 创建记录时时间库出错了"。真实项目通常需要在传播过程中附加上下文,这正是 `anyhow` 擅长的事——[第 15 章](15-anyhow-error-response.md)会讲。

**响应扩展的做法有个隐含前提:错误必须真的变成响应。** 如果某个错误在中途被吞掉(比如被另一个中间件捕获后替换成别的响应),扩展也会跟着丢失,日志就不会记录。本章的链路很短所以没问题,链路复杂时要留意。

**`.on_failure(())` 关掉了框架自带的 5xx 日志,这意味着"记录 5xx"的责任完全落在你自己的中间件上。** 如果某个 500 响应不是由 `AppError` 产生的(比如某处直接返回了 `StatusCode::INTERNAL_SERVER_ERROR`,或者 handler panic 了),它就不会被任何人记录。这是一个真实的观测盲区。稳妥的做法是在中间件里额外检查 `response.status().is_server_error()`,兜住所有 5xx。

**`state.users.lock().unwrap()` 会 panic。** `Mutex` 中毒(持锁线程 panic)后所有 `lock()` 都返回 `Err`。示例直接 `unwrap`,细节见[第 22 章](22-key-value-store.md)。

## 动手练习

**1. 补上 5xx 的观测盲区。** 让中间件兜住所有 5xx 响应,不只是 `AppError` 产生的那些。

提示:在 `log_app_errors` 里,先按现有逻辑检查扩展;如果没有扩展但 `response.status().is_server_error()` 为真,记一条不同措辞的日志(比如 "5xx response without AppError")。验证:加一条 `GET /boom` 路由直接返回 `StatusCode::INTERNAL_SERVER_ERROR`,确认它也被记录了。

**2. 加一种"资源不存在"的错误。** 新增 `GET /users/{id}`,查不到时返回 `404`。

提示:`AppError` 加一个 `UserNotFound(u64)` 变体。想清楚它属于哪一类——用户请求了不存在的资源,这是客户端的问题还是服务端的?按前置知识那张表决定 `into_response` 里给不给 `Some(self)`。

**3. 用 `thiserror` 替换手写的 `From` 实现。**

提示:`cargo add thiserror`,给 `AppError` 派生 `Error`,两个变体加 `#[from]`,删掉两个手写的 `impl From`。做完对比一下代码量,以及[第 12 章](12-validator.md#步骤-2声明校验规则包进提取器)用过的写法。注意 `thiserror` 要求每个变体有 `#[error(...)]` 消息。

## 小结与下一章预告

`4xx` 和 `5xx` 的区别决定了两件事:客户端错误要**详细告知但不记日志**(对方能修,而且这类流量很多),服务端错误要**记录完整细节但只回一句 "Something went wrong"**(只有你能修,详细信息还会泄露内部结构)。

用一个 enum 收拢应用错误,为每种来源实现 `From`,`?` 就能贯穿整条调用链自动转换。

**不要在 `into_response` 里写日志** —— 那会给纯转换函数引入副作用,而且那时拿不到请求上下文。正确做法是把错误对象塞进响应扩展(包一层 `Arc` 绕开 `Clone` 约束),由中间件在响应路上取出来记录。中间件必须放在 `TraceLayer` 的**内层**,这样日志才落在请求 span 里、自动带上方法和路径——顺序写反不会报错,只会让日志悄悄失去上下文。

本章的错误类型是手写的 enum,每种来源都要加变体、加 `From`。项目里错误来源一多,这套样板会变得繁琐,而且 `?` 传播时丢失了"在做什么的时候出的错"这层上下文。[第 15 章 anyhow-error-response](15-anyhow-error-response.md) 换一条思路:用 `anyhow::Error` 装下任意错误,配合一个薄薄的包装类型接上 `IntoResponse`,十几行代码就能让 `?` 在整个应用里畅通无阻。你会看到它和本章做法各自适合什么场景。
