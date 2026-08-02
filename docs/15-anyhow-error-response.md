# 第 15 章 anyhow-error-response:用 anyhow 装下所有错误

## 本章你将做出什么

一个只有 30 行的服务,但它演示了一个能用在整个项目里的模式:**任何返回 `Result<_, anyhow::Error>` 的函数,都能在 handler 里直接用 `?`**。

[第 14 章](14-error-handling.md)那套做法很完整,代价是每加一种错误来源就要加一个 enum 变体和一个 `From` 实现。这一章走另一条路——十几行代码换来"什么错误都能往上抛"。

学完本章你将掌握:

- `anyhow::Error` 是什么,它和 `thiserror` 的分工
- 一个泛型 `From` 实现如何让 `?` 接住任意错误类型
- 为什么必须自己定义 `AppError` 而不能直接用 `anyhow::Error`(孤儿规则)
- 这个模式的适用边界,以及它和第 14 章做法的取舍

## 前置知识

### anyhow 与 thiserror 的分工

Rust 错误处理生态里这两个 crate 经常一起出现,它们解决的是**相反**的问题。

**`thiserror` 用于定义具体的错误类型。** [第 12 章](12-validator.md#步骤-2声明校验规则包进提取器)和[第 14 章](14-error-handling.md)用过:你明确列出所有可能的失败,每种一个 enum 变体。调用方可以 `match` 它,针对不同错误做不同处理。

**`anyhow` 用于把任意错误装进一个统一的类型。** `anyhow::Error` 可以容纳任何实现了 `std::error::Error + Send + Sync + 'static` 的错误值。调用方**不能**再区分具体是什么错——它只知道"出错了",以及一串描述信息。

选择的标准是**调用方需不需要区分错误类型**:

| | `thiserror` | `anyhow` |
|---|---|---|
| 典型场景 | 写库 | 写应用 |
| 调用方能否 `match` 具体错误 | 能 | 不能 |
| 加一种错误来源的成本 | 加变体 + `From` | 零 |
| 错误链与上下文 | 手动维护 | 内置 |

**写库必须用 `thiserror`**:库的使用者需要根据不同错误做不同处理,你不能剥夺这个能力。

**写应用通常可以用 `anyhow`**:大多数错误的最终归宿都是"记一条日志,返回 500"。既然处理方式相同,区分类型就没有价值。

本章示例是应用,所以用 `anyhow`。第 14 章那种"客户端错误和服务端错误要区别对待"的场景,则需要能区分类型——两章的取舍不同,结尾会对比。

### anyhow 的两个实用能力

**`bail!` 宏**用一句话构造并返回错误:

```rust
anyhow::bail!("it failed!");
// 等价于 return Err(anyhow::anyhow!("it failed!"));
```

它接受格式化字符串,所以可以写 `bail!("user {id} not found")`。

**`context` 方法**在错误向上传播时附加说明:

```rust
use anyhow::Context;

let config = std::fs::read_to_string(path)
    .context("failed to read config file")?;
```

这解决了[第 14 章](14-error-handling.md#代价与边界)结尾提到的问题:`?` 自动转换会丢失"在做什么的时候出的错"。`anyhow` 会把上下文串成一条链:

```
failed to create user

Caused by:
    0: failed to read config file
    1: No such file or directory (os error 2)
```

`{:?}`(Debug 格式)会打印完整的链,`{}`(Display 格式)只打印最外层那句。这个区别在写日志时很重要——本章末尾会提到。

### 为什么不能直接用 anyhow::Error

最省事的写法应该是让 handler 直接返回 `Result<(), anyhow::Error>`。但这不行:

```rust
async fn handler() -> Result<(), anyhow::Error> {   // 编译不过
```

原因是 `anyhow::Error` **没有实现 `IntoResponse`**。而 axum 要求 `Result<T, E>` 的 `E` 必须实现它。

那就给它实现一个?也不行——**孤儿规则**(orphan rule)禁止这么做:`IntoResponse` 定义在 axum 里,`anyhow::Error` 定义在 anyhow 里,两个都不是你的 crate。

[第 04 章](04-templates.md#步骤-3为自己的类型实现-intoresponse)遇到过一模一样的情况(想给 askama 的 `Template` 实现 `IntoResponse`),解法也一样:**定义一个自己的 newtype 包住它**。

```rust
struct AppError(anyhow::Error);
```

这个类型属于你的 crate,想给它实现什么 trait 都行。

## 项目搭建

```bash
cargo new anyhow-error-response
cd anyhow-error-response
```

依赖很少,本章示例连日志都没用:

```bash
cargo add axum
cargo add tokio --features full
cargo add anyhow
```

## 分步构建

### 步骤 1:先撞一次编译错误

**要做什么**:故意用 `anyhow::Error` 当 handler 的错误类型,看编译器怎么说。这一步是为了让下一步的 `AppError` 有存在理由。

```rust
// src/main.rs
use axum::{routing::get, Router};

#[tokio::main]
async fn main() {
    let app = app();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

fn app() -> Router {
    Router::new().route("/", get(handler))
}

// 这一步故意写错,下一步修
async fn handler() -> Result<(), anyhow::Error> {
    try_thing()?;
    Ok(())
}

fn try_thing() -> Result<(), anyhow::Error> {
    anyhow::bail!("it failed!")
}
```

`try_thing` 模拟一个会失败的内部函数,`bail!` 直接返回错误。

**跑起来看**:

```bash
cargo run
```

编译失败:

```
error[E0277]: the trait bound `fn() -> ... {handler}: Handler<_, _>` is not satisfied
   |
   |     Router::new().route("/", get(handler))
   |                              --- ^^^^^^^ the trait `Handler<_, _>` is not implemented
   |
   = note: Consider using `#[axum::debug_handler]` to improve the error message
```

**这段报错不好读**,它只说 `handler` 不满足 `Handler` trait,没直说是为什么。[第 01 章](01-hello-world.md#handler为什么普通函数能当处理器)讲过:`Handler` 的实现要求"参数能提取、返回值能转成响应"。这里参数没问题,所以问题在返回值——`Result<(), anyhow::Error>` 里的 `anyhow::Error` 没有实现 `IntoResponse`。

编译器的提示值得记住:**`#[axum::debug_handler]` 能让这类错误变得可读**。给 handler 加上这个属性宏(需要 `axum` 的 `macros` feature),它会生成更精确的诊断,直接指出是哪个参数或返回值不满足要求。调试 handler 签名问题时这是第一手段。

### 步骤 2:包一个 newtype 并实现 IntoResponse

**要做什么**:定义 `AppError` 包住 `anyhow::Error`,给它实现 `IntoResponse`。

```rust
// src/main.rs
use axum::{
    http::StatusCode,                                    // 新增
    response::{IntoResponse, Response},                  // 新增
    routing::get,
    Router,
};

// ...(main 和 app 不变)

// 改动:错误类型换成 AppError
async fn handler() -> Result<(), AppError> {
    try_thing()?;
    Ok(())
}

fn try_thing() -> Result<(), anyhow::Error> {
    anyhow::bail!("it failed!")
}

// 新增:Make our own error that wraps `anyhow::Error`.
struct AppError(anyhow::Error);

// 新增:Tell axum how to convert `AppError` into a response.
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            format!("Something went wrong: {}", self.0),
        )
            .into_response()
    }
}
```

现在还编译不过,但错误变了——`try_thing()?` 那一行报错,因为 `?` 不知道怎么把 `anyhow::Error` 变成 `AppError`。下一步补上。

先看 `into_response` 做了什么:固定返回 `500` 加一句带错误描述的文本。这是[第 06 章](06-global-404-handler.md#元组作为返回类型)讲的元组返回。

**注意 `{}` 用的是 `Display` 格式**,只打印最外层的错误消息("it failed!"),不含 `Caused by` 链。这里是有意的——[第 14 章](14-error-handling.md#4xx-与-5xx-决定了两件事)讲过,返回给客户端的 5xx 信息应该尽量少。

> **更好的写法**:示例把实际的错误描述拼进了响应体,这和[第 14 章](14-error-handling.md#4xx-与-5xx-决定了两件事)"5xx 只回一句 Something went wrong"的原则相冲突——错误文本常常泄露文件路径、数据库表名、内部服务地址。正确做法是响应固定文本、详情记进日志。示例这么写纯粹是为了让你在 curl 里能看到效果,别照搬。练习题 1 会让你改。

### 步骤 3:一个泛型 From 让 ? 接住一切

**要做什么**:补上让 `?` 工作的转换。

```rust
// src/main.rs
// ...(前面不变)

// 新增:This enables using `?` on functions that return `Result<_, anyhow::Error>` to turn them into
// `Result<_, AppError>`. That way you don't need to do that manually.
impl<E> From<E> for AppError
where
    E: Into<anyhow::Error>,
{
    fn from(err: E) -> Self {
        Self(err.into())
    }
}
```

**这五行是本章的精华。**

它不是 `impl From<anyhow::Error> for AppError`,而是对**所有能转成 `anyhow::Error` 的类型 `E`** 都实现了 `From`。

而 `anyhow` 为所有 `E: std::error::Error + Send + Sync + 'static` 提供了 `impl From<E> for anyhow::Error`。两者叠加的结果是:

**任何标准错误类型都能通过 `?` 直接变成 `AppError`。**

```rust
async fn handler() -> Result<String, AppError> {
    let s = std::fs::read_to_string("config.toml")?;   // io::Error 直接过
    let n: u32 = s.trim().parse()?;                    // ParseIntError 直接过
    let json: serde_json::Value = serde_json::from_str(&s)?;  // serde_json::Error 直接过
    Ok(format!("{n} {json}"))
}
```

三种完全不同的错误类型,一个 `From` 实现全接住,**不需要为它们各写一个变体**。对比[第 14 章](14-error-handling.md#步骤-2区分两类错误)每种错误都要写 enum 变体加 `From` impl,省下的样板相当可观。

代价是**丢失了类型信息**:`AppError` 内部装的到底是 IO 错误还是解析错误,调用方无从得知,只能拿到描述文本。这正是前置知识说的 `anyhow` 特性——适用于"反正都是 500"的场景。

**跑起来看**:

```bash
cargo run
```

```
listening on 127.0.0.1:3000
```

```bash
curl -i http://127.0.0.1:3000/
```

```
HTTP/1.1 500 Internal Server Error
content-type: text/plain; charset=utf-8
content-length: 32

Something went wrong: it failed!
```

`500`,而且带上了 `bail!` 里那句消息。

**验证"接住一切"这个说法。** 临时把 `handler` 改成用一个真实的 IO 错误:

```rust
async fn handler() -> Result<(), AppError> {
    std::fs::read_to_string("/nonexistent/path")?;   // io::Error
    Ok(())
}
```

不用改 `AppError`、不用加任何 `From`,直接编译通过:

```bash
curl http://127.0.0.1:3000/
```

```
Something went wrong: No such file or directory (os error 2)
```

再试试加上下文:

```rust
use anyhow::Context;

async fn handler() -> Result<(), AppError> {
    std::fs::read_to_string("/nonexistent/path")
        .context("failed to load config")?;
    Ok(())
}
```

```bash
curl http://127.0.0.1:3000/
```

```
Something went wrong: failed to load config
```

**`Display` 只显示最外层。** 底层的 "No such file or directory" 还在错误链里,但没有输出——因为 `into_response` 里用的是 `{}`。改成 `{:?}` 就能看到完整的链:

```
Something went wrong: failed to load config

Caused by:
    No such file or directory (os error 2)
```

这个区别很关键:**`{}` 给客户端,`{:?}` 给日志。** 本章示例把 `{}` 的结果给了客户端,已经算是泄露了;要是用 `{:?}`,整条错误链连同文件路径都会暴露出去。

改回官方示例的 `try_thing()?` 版本。

## 完整代码

`src/main.rs`:

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::get,
    Router,
};

#[tokio::main]
async fn main() {
    let app = app();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Result<(), AppError> {
    try_thing()?;
    Ok(())
}

fn try_thing() -> Result<(), anyhow::Error> {
    anyhow::bail!("it failed!")
}

// Make our own error that wraps `anyhow::Error`.
struct AppError(anyhow::Error);

// Tell axum how to convert `AppError` into a response.
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            format!("Something went wrong: {}", self.0),
        )
            .into_response()
    }
}

fn app() -> Router {
    Router::new().route("/", get(handler))
}

// This enables using `?` on functions that return `Result<_, anyhow::Error>` to turn them into
// `Result<_, AppError>`. That way you don't need to do that manually.
impl<E> From<E> for AppError
where
    E: Into<anyhow::Error>,
{
    fn from(err: E) -> Self {
        Self(err.into())
    }
}
```

> 官方示例的 `main.rs` 末尾还有一个 `#[cfg(test)] mod tests`,断言 `GET /` 返回 `500` 且 body 是 `Something went wrong: it failed!`。测试写法见第 21 章。

## 代价与边界

**所有错误都变成 500,没有例外。** 这是本章模式最大的问题。用户提交了不合法的数据(该 `400`)、请求了不存在的资源(该 `404`)、没有权限(该 `403`),在这套代码里统统是 `500`。而 `500` 的语义是"服务端有 bug",会污染你的错误率监控——本来是正常的客户端错误,却触发了告警。

**改进方向**是让 `AppError` 携带状态码:

```rust
struct AppError {
    status: StatusCode,
    inner: anyhow::Error,
}
```

默认 `500`,需要时显式指定。练习题 2 会做这件事。

**示例把错误消息返回给了客户端。** `format!("Something went wrong: {}", self.0)` 里的 `self.0` 是实际的错误描述。[第 14 章](14-error-handling.md#4xx-与-5xx-决定了两件事)讲过为什么不该这样——错误信息常常泄露文件路径、数据库表名、内部服务地址。生产环境应该改成固定文本,把详细信息记进日志。

**没有记日志。** 本章示例完全没有日志——错误返回给客户端之后就消失了。第 14 章那套"扩展 + 中间件"的做法在这里同样适用,而且更有必要,因为 `anyhow` 的错误链只有记录下来才有价值。

**丢失类型信息是双刃剑。** 省掉了 enum 变体,但也失去了"针对特定错误做特定处理"的能力。如果某天需要"数据库连接失败时重试、其它错误直接返回",用 `anyhow` 就得靠 `downcast_ref` 去猜类型,不如一开始就用 enum。

**两章做法怎么选:**

| | 第 14 章(enum) | 第 15 章(anyhow) |
|---|---|---|
| 加一种错误来源 | 加变体 + `From` | 零成本 |
| 能否按错误类型分流 | 能 | 不能(除非 downcast) |
| 状态码控制 | 每种错误精确指定 | 默认全 500 |
| 上下文链 | 手动 | 内置 `context` |
| 适合 | 需要区分 4xx/5xx 的 API | 内部服务、原型、错误都是 500 的场景 |

**实际项目常见的是混合用法**:定义一个 enum,其中一个变体装 `anyhow::Error` 作为兜底:

```rust
enum AppError {
    NotFound,
    Unauthorized,
    BadRequest(String),
    Internal(anyhow::Error),   // 兜底,全部 500
}
```

明确需要区分的错误各占一个变体,其余一律进 `Internal`。这样既有精确的状态码控制,又不必为每个第三方库错误单开变体。

## 动手练习

**1. 把错误细节从响应里挪进日志。**

提示:`cargo add tracing tracing-subscriber -F env-filter`,在 `main` 里初始化订阅器([第 03 章](03-form.md#步骤-1显示表单页面)的样板)。`into_response` 里用 `tracing::error!(err = ?self.0, "handler failed")` 记录完整错误链(注意用 `?` 前缀走 `Debug`),响应改成固定的 `"Something went wrong"`。做完对比 curl 输出和终端日志的差异。

顺带思考:这么写违反了[第 14 章](14-error-handling.md#为什么不能在-into_response-里记日志)"不要在 `into_response` 里记日志"的原则吗?如果要遵守那条原则,该怎么改?

**2. 让 `AppError` 支持自定义状态码。**

提示:改成 `struct AppError { status: StatusCode, inner: anyhow::Error }`。泛型 `From` 实现里默认给 `StatusCode::INTERNAL_SERVER_ERROR`。再加一个构造方法 `fn with_status(status: StatusCode, err: impl Into<anyhow::Error>) -> Self`,让 handler 能显式指定。验证:加一条路由返回 `404`,确认状态码正确。

**3. 验证 `?` 到底能接住多少种错误。**

提示:写一个 handler 依次调用 `std::fs::read_to_string`、`str::parse::<u32>`、`serde_json::from_str`,三种错误类型全用 `?`。确认不需要为它们添加任何 `From` 实现。然后想想:如果换成第 14 章那套 enum 做法,你需要写多少代码?

## 小结与下一章预告

`anyhow` 和 `thiserror` 解决相反的问题:前者把任意错误装进一个统一类型(写应用),后者定义可被 `match` 的具体错误(写库)。选择标准是**调用方需不需要区分错误类型**。

`anyhow::Error` 不能直接当 handler 的错误类型——它没实现 `IntoResponse`,而孤儿规则不允许你给它补上。解法和[第 04 章](04-templates.md#步骤-3为自己的类型实现-intoresponse)一样:定义一个自己的 newtype 包住它。

核心技巧是那个泛型 `From` 实现:对所有 `E: Into<anyhow::Error>` 实现转换,于是**任何标准错误类型都能被 `?` 直接接住**,不用为每种错误写变体。代价是丢失类型信息、所有错误都变成同一个状态码。

实际项目常用混合方案:enum 里明确列出需要区分的错误,再留一个装 `anyhow::Error` 的兜底变体。

前面两章讲的是错误怎么组织。接下来转向另一个工程化话题:**依赖怎么注入**。[第 16 章 dependency-injection](16-dependency-injection.md) 讨论 handler 如何拿到数据库、缓存这类外部依赖,并且让它们在测试时可替换。你会看到 Rust 里两种做法——trait 对象(`Arc<dyn Trait>`)和泛型(`T: Trait`)——各自的写法、开销和适用场景,以及官方为什么推荐前者。
