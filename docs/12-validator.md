# 第 12 章 validator:类型对了不等于数据对了

## 本章你将做出什么

一个把校验规则写进 struct 属性的服务。访问 `/?name=LT` 返回问候语,访问 `/?name=` 或 `/?name=X` 返回:

```
Input validation error: [name: Can not be empty]
```

关键在于:`name` 在这三种情况下**都是合法的 `String`**,反序列化全都成功。挡住后两个的是**业务校验**,这一层是提取器管不了的。

学完本章你将掌握:

- 类型校验和业务校验的分界,以及为什么框架只能负责前者
- `validator` crate:用属性声明规则,`#[derive(Validate)]` 生成校验代码
- 把校验嵌进提取器的做法——让非法数据在进入 handler 之前就被拦下
- `thiserror` 的 `#[from]` 如何让 `?` 在两种错误之间自动转换
- 一个容易忽略的细节:`Form` 提取器在 `GET` 请求下读的是查询字符串,不是 body

## 前置知识

### 类型对了不等于数据对了

[第 03 章](03-form.md#步骤-3观察提取失败时的行为)展示过 `Form` 提取器能挡住什么:

- 缺字段 → `422`
- `Content-Type` 不对 → `415`
- 类型转不过去(比如 `age=abc` 但字段是 `u32`)→ `422`

那一章末尾留了一句话:`Form` 只保证**类型**正确,不保证**业务**合法。现在展开说。

反序列化的职责是"把字节变成 Rust 值"。它能回答的问题只有一个:**这段输入能不能装进这个类型?** 至于装进去之后的值合不合理,它无从判断——因为"合理"是业务定义的,不是类型定义的:

| 输入 | `String` 类型校验 | 业务是否合法 |
|---|---|---|
| `name=` | 通过(空字符串是合法 `String`) | 通常不合法 |
| `name=X` | 通过 | 取决于业务(本章要求至少 2 个字符) |
| `email=随便乱写` | 通过 | 不是邮箱格式 |
| `age=999`(字段是 `u32`) | 通过 | 人的年龄不可能 |
| `password=1` | 通过 | 太短 |

**这些值全都能通过提取器,然后带着问题进入你的 handler。** 于是每个 handler 开头就得写一堆:

```rust
if input.name.is_empty() {
    return Err(...);
}
if input.name.len() < 2 {
    return Err(...);
}
if !input.email.contains('@') {
    return Err(...);
}
```

这样写的问题很实际:校验逻辑散落在各个 handler 里、同一个字段在不同接口里的规则可能写得不一致、规则改了要满项目搜索、而且这些代码把真正的业务逻辑埋没了。

**更好的做法是把规则声明在数据结构上**——规则和字段写在一起,一处定义处处生效。这就是 `validator` 做的事。

### validator crate

用法是给 struct 派生 `Validate`,给字段加 `#[validate(...)]` 属性:

```rust
#[derive(Debug, Deserialize, Validate)]
pub struct NameInput {
    #[validate(length(min = 2, message = "Can not be empty"))]
    pub name: String,
}
```

然后调用 `value.validate()`,返回 `Result<(), ValidationErrors>`。

常用的规则:

| 属性 | 作用 |
|---|---|
| `length(min = 2, max = 20)` | 字符串或集合的长度范围 |
| `range(min = 1, max = 150)` | 数值范围 |
| `email` | 邮箱格式 |
| `url` | URL 格式 |
| `contains(pattern = "@")` | 包含某子串 |
| `regex(path = ...)` | 匹配正则 |
| `must_match(other = "password")` | 与另一字段相等(确认密码) |
| `custom(function = ...)` | 调用自定义函数 |
| `nested` | 递归校验嵌套结构体 |

`message = "..."` 给出自定义错误消息,不写则用默认消息。

> **更好的写法**:本章示例的规则名不副实——`length(min = 2, message = "Can not be empty")` 的消息说"不能为空",规则却要求至少 2 个字符,于是 `name=X` 也会报"不能为空",而它并不为空。**校验消息是直接给最终用户看的**,说不清问题会让人反复试错。要么把消息改成"至少 2 个字符",要么拆成两条规则分别对应"为空"和"太短"。练习题 1 会让你修。

`ValidationErrors` 是个结构化的错误集合,按字段名组织,一次校验能报出所有不合规的字段——比"发现第一个错就返回"对用户友好得多。它的 `Display` 输出形如:

```
name: Can not be empty
```

多个字段出错时用换行分隔,所以本章代码里会看到一个 `.replace('\n', ", ")` 把它压成一行。

### Form 在 GET 请求下读查询字符串

本章示例有个容易看漏的地方:路由注册的是 `get(handler)`,而 handler 用的是 `Form` 提取器。

[第 03 章](03-form.md#html-表单是怎么发数据的)说 `Form` 解析 `application/x-www-form-urlencoded` 的请求体,而 `GET` 请求通常没有 body。这不矛盾——**axum 的 `Form` 提取器对方法是敏感的**:

- `GET` / `HEAD` 请求 → 从**查询字符串**(URL 里 `?` 后面的部分)反序列化
- 其它方法 → 从**请求体**反序列化

设计上说得通:HTML 表单用 `method="get"` 提交时,浏览器正是把数据拼进查询字符串的([第 03 章](03-form.md#html-表单是怎么发数据的)讲过)。所以同一个 `Form<T>` 能同时服务两种提交方式,handler 不用区分。

这也解释了为什么本章可以用 `curl '127.0.0.1:3000?name=LT'` 这么简单的命令测试——不用构造请求体。

> 如果你只想读查询字符串、不关心表单语义,用 `Query<T>` 更直白。`Form` 的这个行为主要是为了兼容 HTML 表单的两种 method。

### 校验放在哪一层

拿到 `validate()` 之后,还有个选择题:在哪里调用它?

**选项一:在 handler 里调。**

```rust
async fn handler(Form(input): Form<NameInput>) -> Result<Html<String>, ServerError> {
    input.validate()?;
    // 业务逻辑
}
```

简单直接,但每个 handler 都要记得写这一行。**忘了写就没有校验,而且编译器不会提醒你**——这是最危险的地方。

**选项二:包进提取器。**

```rust
async fn handler(ValidatedForm(input): ValidatedForm<NameInput>) -> Html<String> {
    // input 一定是校验通过的
}
```

把"提取 + 校验"打包成一个提取器。handler 签名里写了 `ValidatedForm`,校验就一定发生过;写了 `Form`,就一定没有。**要不要校验变成了类型层面的选择,不再依赖人的记性。**

本章用选项二。这也延续了[第 03 章](03-form.md#步骤-3观察提取失败时的行为)那个思路:**不合法的请求在进入你的代码之前就被挡住**,函数体里可以放心假设数据是好的。区别只是这次"合法"的定义从类型扩展到了业务规则。

## 项目搭建

```bash
cargo new validator
cd validator
```

```bash
cargo add axum
cargo add tokio --features full
cargo add serde --features derive
cargo add validator --features derive
cargo add thiserror
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`validator` 的 `derive` feature 提供 `#[derive(Validate)]`,和 `serde` 的 `derive` 是同样的道理。

`thiserror` 见[第 10 章](10-customize-extractor-error.md#thiserror定义错误类型的省事办法),本章会用到它的 `#[from]` 属性。

## 分步构建

### 步骤 1:没有校验的版本

**要做什么**:先用内置 `Form` 写出功能,亲眼确认空名字和单字符名字都能畅通无阻。

```rust
// src/main.rs
use axum::{extract::Form, response::Html, routing::get, Router};
use serde::Deserialize;
use tokio::net::TcpListener;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // build our application with a route
    let app = app();

    // run it
    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

fn app() -> Router {
    Router::new().route("/", get(handler))
}

#[derive(Debug, Deserialize)]
pub struct NameInput {
    pub name: String,
}

async fn handler(Form(input): Form<NameInput>) -> Html<String> {
    Html(format!("<h1>Hello, {}!</h1>", input.name))
}
```

`fn app() -> Router` 的用途见[第 03 章](03-form.md#步骤-1显示表单页面)。

这里 `use tokio::net::TcpListener;` 单独导入了类型,`main` 里就能写 `TcpListener::bind(...)` 而不是全路径。官方示例如此,和前面章节的写法略有不同,无实质差别。

**跑起来看**:

```bash
cargo run
```

正常输入:

```bash
curl '127.0.0.1:3000?name=LT'
```

```
<h1>Hello, LT!</h1>
```

**注意 URL 里带了 `?name=LT`,而这是个 `GET` 请求** —— `Form` 提取器从查询字符串里读到了数据,这就是前置知识讲的行为。

现在试试有问题的输入:

```bash
curl '127.0.0.1:3000?name='
```

```
<h1>Hello, !</h1>
```

```bash
curl '127.0.0.1:3000?name=X'
```

```
<h1>Hello, X!</h1>
```

**两个都通过了。** 空字符串是合法的 `String`,单字符也是,反序列化没有任何理由拒绝它们。页面渲染出了一个没有名字的问候语——这就是"类型对了但数据不对"。

缺字段仍然会被挡住,因为那是类型层面的问题:

```bash
curl -i '127.0.0.1:3000'
```

```
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8

Failed to deserialize form: missing field `name`
```

**对比这三种情况,分界线就很清楚了**:缺字段 = 类型问题,提取器管;字段在但值不合理 = 业务问题,提取器不管。

### 步骤 2:声明校验规则,包进提取器

**要做什么**:加上 `validator` 的规则,写一个 `ValidatedForm` 提取器把校验嵌进提取过程。

```rust
// src/main.rs
use axum::{
    extract::{rejection::FormRejection, Form, FromRequest, Request},   // 改动:加上 FromRequest、Request、FormRejection
    http::StatusCode,                                                  // 新增
    response::{Html, IntoResponse, Response},                          // 改动:加上 IntoResponse、Response
    routing::get,
    Router,
};
use serde::{de::DeserializeOwned, Deserialize};                        // 改动:加上 DeserializeOwned
use thiserror::Error;                                                  // 新增
use tokio::net::TcpListener;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};
use validator::Validate;                                               // 新增

// ...(main 和 app 不变)

// 改动:加上 Validate 派生和校验规则
#[derive(Debug, Deserialize, Validate)]
pub struct NameInput {
    #[validate(length(min = 2, message = "Can not be empty"))]
    pub name: String,
}

// 改动:提取器换成 ValidatedForm
async fn handler(ValidatedForm(input): ValidatedForm<NameInput>) -> Html<String> {
    Html(format!("<h1>Hello, {}!</h1>", input.name))
}

// 新增:提取 + 校验的组合提取器
#[derive(Debug, Clone, Copy, Default)]
pub struct ValidatedForm<T>(pub T);

impl<T, S> FromRequest<S> for ValidatedForm<T>
where
    T: DeserializeOwned + Validate,
    S: Send + Sync,
    Form<T>: FromRequest<S, Rejection = FormRejection>,
{
    type Rejection = ServerError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Form(value) = Form::<T>::from_request(req, state).await?;
        value.validate()?;
        Ok(ValidatedForm(value))
    }
}

// 新增:两种失败合并成一个错误类型
#[derive(Debug, Error)]
pub enum ServerError {
    #[error(transparent)]
    ValidationError(#[from] validator::ValidationErrors),

    #[error(transparent)]
    AxumFormRejection(#[from] FormRejection),
}

impl IntoResponse for ServerError {
    fn into_response(self) -> Response {
        match self {
            ServerError::ValidationError(_) => {
                let message = format!("Input validation error: [{self}]").replace('\n', ", ");
                (StatusCode::BAD_REQUEST, message)
            }
            ServerError::AxumFormRejection(_) => (StatusCode::BAD_REQUEST, self.to_string()),
        }
        .into_response()
    }
}
```

自定义提取器的套路[第 08 章](08-parse-body-based-on-content-type.md#步骤-2实现-jsonorform)和[第 10 章](10-customize-extractor-error.md#步骤-2做法一手写-fromrequest)讲过,这里只看新东西。

**`from_request` 的三行是本章的核心:**

```rust
let Form(value) = Form::<T>::from_request(req, state).await?;
value.validate()?;
Ok(ValidatedForm(value))
```

第一行委托给内置 `Form` 做反序列化,第二行做业务校验,第三行包装返回。**两个 `?` 分别处理两类失败,却都转成了同一个 `ServerError`** —— 这是靠 `#[from]` 实现的。

`?` 运算符在返回错误时会自动调用 `From::from` 做转换。`Form::from_request` 失败时给的是 `FormRejection`,`validate()` 失败时给的是 `ValidationErrors`,两者类型完全不同,但因为 `ServerError` 的两个变体都标了 `#[from]`,thiserror 生成了两个 `From` 实现,`?` 就能把它们都转成 `ServerError`。

**没有这两个 `#[from]`,你得写成 `.map_err(ServerError::AxumFormRejection)?` 这样的形式。** 一两处还好,错误类型多了会很啰嗦。这是 `thiserror` 最实用的地方。

**trait bound 里的 `T: DeserializeOwned + Validate`** 说明了对目标类型的全部要求:能反序列化(给 `Form` 用)、能校验(给 `validate()` 用)。`Form<T>: FromRequest<S, Rejection = FormRejection>` 这一行的写法[第 10 章](10-customize-extractor-error.md#步骤-2做法一手写-fromrequest)解释过——约束"`Form<T>` 是个提取器且失败时给 `FormRejection`",比直接约束 `T` 更精确。

**`IntoResponse` 里两个分支都返回 `400`。** 区别只在消息格式:校验失败包装成 `Input validation error: [...]`,反序列化失败直接用原始描述。

`format!("...{self}...")` 用的是 `ServerError` 的 `Display`,而 `#[error(transparent)]` 让它直接转发给内部错误——所以打印出来是 `ValidationErrors` 的内容。`.replace('\n', ", ")` 把多字段错误的换行压成逗号,让响应保持单行。

> **更好的写法**:把结构化的错误压成一行字符串是**信息降级**——`ValidationErrors` 本身是"字段名 → 错误列表"的映射,而且实现了 `Serialize`,直接 `Json(errors)` 返回就能保留完整结构,前端可以把错误标注到对应的输入框上。压成 `[name: Can not be empty, email: invalid]` 之后,前端只能整段展示,或者去解析这个自定义格式。示例这么写是为了让 curl 输出好看,写 API 时应该返回 JSON。练习题 2 会让你改。

`match self` 的两个分支都写了 `(_)` 忽略内部值,因为它们只用 `self` 的 `Display`,不需要拆出内部错误。

**跑起来看**:

```bash
cargo run
```

合法输入照常工作:

```bash
curl '127.0.0.1:3000?name=LT'
```

```
<h1>Hello, LT!</h1>
```

空值现在被挡住了:

```bash
curl -i '127.0.0.1:3000?name='
```

```
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8
content-length: 50

Input validation error: [name: Can not be empty]
```

单字符也一样:

```bash
curl '127.0.0.1:3000?name=X'
```

```
Input validation error: [name: Can not be empty]
```

**注意这里消息说 "Can not be empty" 但 `X` 并不为空** —— 前置知识提到的那个瑕疵,规则是 `min = 2` 而消息写的是"不能为空"。练习题 1 会修它。

缺字段走的是另一条分支:

```bash
curl '127.0.0.1:3000'
```

```
Failed to deserialize form: missing field `name`
```

**同样是 `400`,但消息格式不同** —— 一个来自 `ValidationError` 分支,一个来自 `AxumFormRejection` 分支。两类失败合并进了同一个错误类型,但在响应里仍然可以区分对待。

**验证一下"校验一定发生"这个保证。** 把 handler 的提取器临时改回 `Form`:

```rust
async fn handler(Form(input): Form<NameInput>) -> Html<String> {
```

重新运行,`curl '127.0.0.1:3000?name='` 又能通过了。**校验规则还在 struct 上,但没人调用它。**

这说明规则本身不会自动生效——`#[validate(...)]` 只是生成了 `validate()` 方法,得有人调。把提取器换成 `ValidatedForm` 就是"让类型系统替你记得调用它"。改回 `ValidatedForm`。

> 官方示例的 `main.rs` 末尾还有一个 `#[cfg(test)] mod tests`,四个测试分别覆盖缺字段、空值、单字符、合法值四种输入。测试写法见[第 21 章](21-testing.md)。

## 完整代码

`src/main.rs`:

```rust
use axum::{
    extract::{rejection::FormRejection, Form, FromRequest, Request},
    http::StatusCode,
    response::{Html, IntoResponse, Response},
    routing::get,
    Router,
};
use serde::{de::DeserializeOwned, Deserialize};
use thiserror::Error;
use tokio::net::TcpListener;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};
use validator::Validate;

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // build our application with a route
    let app = app();

    // run it
    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

fn app() -> Router {
    Router::new().route("/", get(handler))
}

#[derive(Debug, Deserialize, Validate)]
pub struct NameInput {
    #[validate(length(min = 2, message = "Can not be empty"))]
    pub name: String,
}

async fn handler(ValidatedForm(input): ValidatedForm<NameInput>) -> Html<String> {
    Html(format!("<h1>Hello, {}!</h1>", input.name))
}

#[derive(Debug, Clone, Copy, Default)]
pub struct ValidatedForm<T>(pub T);

impl<T, S> FromRequest<S> for ValidatedForm<T>
where
    T: DeserializeOwned + Validate,
    S: Send + Sync,
    Form<T>: FromRequest<S, Rejection = FormRejection>,
{
    type Rejection = ServerError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Form(value) = Form::<T>::from_request(req, state).await?;
        value.validate()?;
        Ok(ValidatedForm(value))
    }
}

#[derive(Debug, Error)]
pub enum ServerError {
    #[error(transparent)]
    ValidationError(#[from] validator::ValidationErrors),

    #[error(transparent)]
    AxumFormRejection(#[from] FormRejection),
}

impl IntoResponse for ServerError {
    fn into_response(self) -> Response {
        match self {
            ServerError::ValidationError(_) => {
                let message = format!("Input validation error: [{self}]").replace('\n', ", ");
                (StatusCode::BAD_REQUEST, message)
            }
            ServerError::AxumFormRejection(_) => (StatusCode::BAD_REQUEST, self.to_string()),
        }
        .into_response()
    }
}
```


## 这个模式的边界

**校验器不认识业务上下文。** `validator` 能检查格式、长度、范围,但不能检查"这个用户名是否已被注册"、"这个订单是否属于当前用户"、"库存是否充足"——这些需要查数据库或读取应用状态,属于 handler 里的业务逻辑,不该塞进提取器。

分界线大致是:**只看输入本身就能判断的,归校验器;需要外部信息才能判断的,归业务层。**

**`custom` 规则可以调函数,但仍然是同步的、无状态的。** `validator` 的自定义校验函数签名是 `fn(&T) -> Result<(), ValidationError>`,拿不到数据库连接、不能 `await`。想在校验里查库,得自己在提取器里做——但那样提取器就依赖 `State` 了,复杂度会上一个台阶。

**错误格式是纯文本,不适合 API。** 本章的 `IntoResponse` 输出的是一行文本。真实 API 通常需要结构化的字段级错误:

```json
{"errors": {"name": ["Can not be empty"], "email": ["Invalid email format"]}}
```

`ValidationErrors` 本身实现了 `Serialize`,可以直接 `Json(errors)` 返回,信息比压成一行完整得多。练习题 2 会做这件事。

**状态码的选择有争议。** 本章两种失败都返回 `400`。也有观点认为语义上应该区分:JSON/表单语法错误用 `400`(请求格式就不对),语义校验失败用 `422 Unprocessable Entity`(格式对但内容不合要求)。[第 03 章](03-form.md#步骤-3观察提取失败时的行为)见过 axum 内置提取器就是这么分的——反序列化失败给 `422`。两种做法都有人用,团队内统一即可。

## 动手练习

**1. 修掉那条名不副实的规则。** 让消息和规则对得上。

提示:两种改法。要么把消息改成 `"must be at least 2 characters"`;要么保留"不能为空"的语义,改用两条规则——`#[validate(length(min = 1, message = "Can not be empty"))]` 加一条 `#[validate(length(min = 2, message = "Too short"))]`(同一字段可以叠多个 `#[validate]` 属性)。做完分别用 `name=`、`name=X`、`name=LT` 验证三种输入的消息是否各不相同。

**2. 把错误改成结构化 JSON。**

提示:`ValidationErrors` 实现了 `Serialize`,所以 `ServerError::ValidationError(errors)` 分支可以写成 `(StatusCode::BAD_REQUEST, Json(errors)).into_response()`——注意这次要在 `match` 里把内部值绑出来,不能再用 `(_)`。需要 `cargo add serde_json`(或直接用 `axum::Json`)。对比一下 `curl -i '127.0.0.1:3000?name='` 前后的响应,看看 JSON 版本多保留了哪些信息。

**3. 加一个多字段的结构体,观察一次报多个错。**

提示:给 `NameInput` 加 `#[validate(email)] pub email: String` 和 `#[validate(range(min = 1, max = 150))] pub age: u32`,然后请求 `'127.0.0.1:3000?name=&email=bad&age=999'`。三个字段同时不合规时,`ValidationErrors` 会一次全部报出来——这就是前置知识说的"比发现第一个错就返回更友好"。

## 小结与下一章预告

反序列化只能回答"这段输入能不能装进这个类型",装进去之后值合不合理是业务问题。空字符串、单字符名字、格式错误的邮箱全都是合法的 `String`,提取器没有理由拒绝它们。

`validator` 把校验规则声明在 struct 字段上,一处定义处处生效,`validate()` 一次性报出所有不合规的字段。把"提取 + 校验"打包成 `ValidatedForm` 这样的提取器,校验就从"每个 handler 记得调用"变成了"写在类型里,忘不掉"。

`thiserror` 的 `#[from]` 让 `?` 能把两种不同的失败(反序列化拒绝、校验失败)自动转成同一个错误类型,省掉满地的 `.map_err(...)`。

顺带认识了 `Form` 提取器的一个细节:`GET`/`HEAD` 请求下它读查询字符串,其它方法读请求体——为了兼容 HTML 表单的两种提交方式。

本章的表单只有文本字段。[第 13 章 multipart-form](13-multipart-form.md) 处理带文件上传的表单:`multipart/form-data` 格式长什么样、`Multipart` 提取器如何逐个取出字段、以及一个绕不开的话题——**上传大小限制**。你会看到 axum 默认的 2MB 上限([第 09 章](09-consume-body-in-extractor-or-middleware.md#缓冲整个-body-的代价)提过)怎么关掉、以及关掉之后必须用什么来兜底。
