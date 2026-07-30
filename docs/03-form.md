# 第 03 章 form:提取器与 HTML 表单

## 本章你将做出什么

一个能收表单的服务:浏览器访问 `/` 看到一个填名字和邮箱的表单,点提交后服务端把收到的内容回显出来。

这是本教程第一个**读取客户端数据**的服务。前两章的 handler 都没有参数,只能返回固定内容;这一章你会认识 axum 最重要的机制——**提取器**(extractor),它让"解析请求体、按字段反序列化、失败时返回 400"这一整套动作缩成一个函数参数。

学完本章你将掌握:

- 提取器是什么,`Form<T>` 如何把请求体变成你的 struct
- `serde` 的 `Deserialize` 在其中的角色,以及字段名如何与表单控件对应
- 提取失败时 axum 的默认行为,以及为什么这是个好的默认值
- 日志初始化样板的作用(细节留到第 17 章)

## 前置知识

### HTML 表单是怎么发数据的

HTML 里的 `<form>` 是浏览器内置的数据提交机制,不需要 JavaScript。关键属性两个:

```html
<form action="/" method="post">
```

- `action` — 提交到哪个路径
- `method` — 用什么 HTTP 方法,通常是 `get` 或 `post`

表单里每个输入控件靠 `name` 属性标识:

```html
<input type="text" name="email">
```

用户点提交后,浏览器把所有控件的 `name=值` 拼成一个字符串。**拼好的数据放在哪里,取决于 `method`**:

- `method="get"` — 数据拼在 URL 后面,叫**查询字符串**(query string):`/?name=张三&email=a@b.com`。会出现在地址栏和浏览器历史里,长度也有限制,所以只适合搜索、筛选这类无副作用的场景。
- `method="post"` — 数据放在**请求体**(body)里,URL 保持干净。提交注册、下单这类会产生副作用的操作用它。

本章用 `post`。

### application/x-www-form-urlencoded

`method="post"` 的表单默认用这个格式发数据,[第 00 章](00-web-basics.md#content-type-与-mime-类型)的 MIME 列表里见过它。格式很简单:

```
name=%E5%BC%A0%E4%B8%89&email=a%40b.com
```

规则是 `键=值` 用 `&` 连接,键和值都做 **URL 编码**(百分号编码):空格变成 `+` 或 `%20`,`@` 变成 `%40`,中文按 UTF-8 逐字节转成 `%XX`。这么做是因为 `&`、`=` 本身有语法含义,不编码就会歧义。

完整的请求报文长这样:

```
POST / HTTP/1.1
Host: 127.0.0.1:3000
Content-Type: application/x-www-form-urlencoded
Content-Length: 38

name=%E5%BC%A0%E4%B8%89&email=a%40b.com
```

`Content-Type` 声明格式,`Content-Length` 声明 body 字节数——都是[第 00 章](00-web-basics.md)你手写过的头部。

带文件上传的表单要换成 `multipart/form-data` 格式,那是第 13 章的内容。

### serde 与 Deserialize

**serde** 是 Rust 生态的序列化框架,几乎所有涉及数据格式转换的 crate 都建立在它上面。它的核心是两个 trait:

- `Serialize` — 把 Rust 值变成某种格式(struct → JSON)
- `Deserialize` — 把某种格式变成 Rust 值(JSON → struct)

serde 的设计精妙之处在于**数据结构和数据格式解耦**。你的 struct 只需实现一次 `Deserialize`,就能从 JSON、TOML、YAML、urlencoded 等任何格式反序列化——具体格式由另外的 crate(`serde_json`、`serde_urlencoded` 等)提供。

手写 `Deserialize` 的实现很繁琐,所以 serde 提供了派生宏:

```rust
#[derive(Deserialize)]
struct Input {
    name: String,
    email: String,
}
```

这需要开启 serde 的 `derive` feature,所以本章的 `cargo add` 命令带 `--features derive`。

派生出的实现默认**按字段名匹配**:表单里 `name=张三` 对应 struct 里的 `name` 字段。字段名不一致时可以用 `#[serde(rename = "...")]` 调整,类型不匹配或字段缺失则反序列化失败。

### 提取器:axum 的核心机制

[第 01 章](01-hello-world.md#handler为什么普通函数能当处理器)讲 `Handler` trait 时提到一句:handler 的参数需要实现 `FromRequest`。那就是提取器。

概念很简单:**提取器是一种能从 HTTP 请求里取出特定信息的类型。** 你在 handler 参数位置写上它,axum 在调用 handler 之前先把它构造出来。

axum 内置了一批:

| 提取器 | 从哪取 | 取出什么 |
|---|---|---|
| `Path<T>` | URL 路径中的参数 | `/users/{id}` 里的 `id`(第 04 章) |
| `Query<T>` | 查询字符串 | `?page=2&size=10` |
| `Form<T>` | 请求体(urlencoded) | 表单字段(**本章**) |
| `Json<T>` | 请求体(JSON) | JSON 对象 |
| `HeaderMap` | 请求头 | 全部头部 |
| `State<S>` | 应用状态 | 共享的配置、连接池(第 05 章) |
| `Request` | 整个请求 | 原始请求对象 |

提取器分两类,这个区别后面章节会反复用到:

- **`FromRequestParts`** — 只读请求的头部信息(方法、路径、头部),不碰 body。可以有任意多个。
- **`FromRequest`** — 要消耗 body。因为 body 是一次性的流,**只能有一个,而且必须是最后一个参数**。

`Form<T>`、`Json<T>` 属于后者;`Path<T>`、`Query<T>`、`HeaderMap` 属于前者。所以下面这样写是合法的:

```rust
async fn handler(Path(id): Path<u32>, headers: HeaderMap, Form(input): Form<Input>) {}
```

把 `Form` 放到中间就编译不过。axum 会给出一段解释性的错误信息告诉你原因。第 09 章会深入讲 body 只能消费一次这件事带来的各种后果。

## 项目搭建

```bash
cargo new form
cd form
```

依赖比前两章多了三个:

```bash
cargo add axum
cargo add tokio --features full
cargo add serde --features derive
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`serde` 的 `derive` feature 提供 `#[derive(Deserialize)]`。

`tracing` 和 `tracing-subscriber` 是日志相关的,官方示例从这一章开始用它们替代 `println!`。步骤 1 会说明这段样板干什么。

`axum` 的默认 feature 已经包含 `form`,所以 `Form` 提取器直接可用,不用额外开。

## 分步构建

### 步骤 1:显示表单页面

**要做什么**:`GET /` 返回一个 HTML 表单页面。先只做显示,不处理提交。

```rust
// src/main.rs
use axum::{response::Html, routing::get, Router};
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

    // build our application with some routes
    let app = app();

    // run it
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

fn app() -> Router {
    Router::new().route("/", get(show_form))
}

async fn show_form() -> Html<&'static str> {
    Html(
        r#"
        <!doctype html>
        <html>
            <head></head>
            <body>
                <form action="/" method="post">
                    <label for="name">
                        Enter your name:
                        <input type="text" name="name">
                    </label>

                    <label>
                        Enter your email:
                        <input type="text" name="email">
                    </label>

                    <input type="submit" value="Subscribe!">
                </form>
            </body>
        </html>
        "#,
    )
}
```

三处新东西。

**日志初始化样板。** `main` 开头那八行是注册一个**日志订阅器**(subscriber)。`tracing` crate 的设计是记录和输出分离:代码里用 `tracing::debug!` 之类的宏记录事件,但这些事件要有订阅器接收才会真正输出。没注册订阅器,所有日志宏都静默丢弃。

`EnvFilter::try_from_default_env()` 读取 `RUST_LOG` 环境变量决定输出哪些级别;读不到就退回 `unwrap_or_else` 里的默认值 `form=debug`,意思是"本 crate 的 debug 及以上级别都输出"。`env!("CARGO_CRATE_NAME")` 是编译期取得当前 crate 名。`fmt::layer()` 负责把日志格式化打到终端。

> **本章不深入**:这段样板在后面绝大多数章节都会原样出现,照抄即可,下面"跑起来看"会让你亲眼看到它的效果。`Layer` 的叠加机制、日志级别的设计、结构化字段(span 与 event)见[第 17 章](17-tracing-aka-logging.md)。

**`fn app() -> Router`。** 路由构建从 `main` 里抽了出来。目前看不出好处,但这个抽取是为**测试**服务的——测试里可以直接 `app()` 拿到一个 `Router`,不启动真实服务器就能发请求。

> **本章不深入**:官方示例大量使用 `fn app() -> Router` 这个模式,后面章节看到它都是同一个用途。不启动服务器测试 handler 的具体写法(`tower::ServiceExt::oneshot`)见[第 21 章](21-testing.md)。

**`r#"..."#` 原始字符串。** [第 00 章](00-web-basics.md#步骤-3按方法和路径分发)用过,写含大量引号的 HTML 正合适。

注意 HTML 里 `<form action="/" method="post">`:提交目标是 `/`,方法是 `POST`。而路由现在只注册了 `get(show_form)`——所以提交一定会失败。下一步补上。

两个 `<input>` 的 `name` 分别是 `name` 和 `email`,记住这两个名字,步骤 2 的 struct 字段要和它们对应。

**跑起来看**:

```bash
cargo run
```

这次的启动日志和前两章不同:

```
2026-07-30T06:45:12.331847Z DEBUG form: listening on 127.0.0.1:3000
```

带上了时间戳、级别 `DEBUG` 和来源 crate `form`——这是 `tracing_subscriber` 格式化的结果,比 `println!` 信息量大。

试试环境变量的效果,`Ctrl-C` 后:

```bash
RUST_LOG=form=info cargo run
```

启动日志消失了,因为 `debug` 级别低于 `info`,被过滤掉了。改回 `cargo run` 就又出现。这就是 `EnvFilter` 的作用。

浏览器访问 `http://127.0.0.1:3000/`,能看到两个输入框和一个 Subscribe 按钮。填点内容点提交:

浏览器显示 `Method Not Allowed`。这正是[第 01 章](01-hello-world.md#跑起来看)见过的 `405`——路径 `/` 存在但没有 `POST` 分支。用 curl 确认:

```bash
curl -i -X POST http://127.0.0.1:3000/
```

```
HTTP/1.1 405 Method Not Allowed
allow: GET
content-length: 0
date: Thu, 30 Jul 2026 06:45:30 GMT
```

### 步骤 2:用 Form 提取器接收提交

**要做什么**:加上 `POST /` 的处理,把收到的表单字段回显。

改三处:

```rust
// src/main.rs
use axum::{extract::Form, response::Html, routing::get, Router};   // 改动:引入 Form
use serde::Deserialize;                                            // 新增
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// ...(main 完全不变)

fn app() -> Router {
    // 改动:同一路径挂上 POST 分支
    Router::new().route("/", get(show_form).post(accept_form))
}

// ...(show_form 不变)

// 新增:表单数据对应的结构体
#[derive(Deserialize, Debug)]
#[allow(dead_code)]
struct Input {
    name: String,
    email: String,
}

// 新增:接收表单
async fn accept_form(Form(input): Form<Input>) -> Html<String> {
    dbg!(&input);
    Html(format!("email='{}'\nname='{}'\n", input.email, input.name))
}
```

`get(show_form).post(accept_form)` 是[第 02 章](02-routes-and-handlers-close-together.md#router-与-methodrouter-的分工)讲过的 `MethodRouter` 链式写法:同一个路径 `/`,`GET` 显示表单,`POST` 处理提交。

现在看 `accept_form` 的签名,这是本章的重点:

```rust
async fn accept_form(Form(input): Form<Input>) -> Html<String>
```

**参数位置写的是解构模式,不是变量名。** `Form(input): Form<Input>` 的意思是"参数类型是 `Form<Input>`,把它解构出内部的 `Input` 绑定到 `input`"。因为 `Form` 是个元组结构体 `struct Form<T>(pub T);`,`Form(input)` 就把里面的值取出来了。

不解构也行:

```rust
async fn accept_form(form: Form<Input>) -> Html<String> {
    let input = form.0;   // 手动取
    // ...
}
```

但解构写法更常见,少一行代码,也让签名直接读出"我要一个 `Input`"。

**这个签名背后 axum 做了多少事。** 收到 `POST /` 时:

1. 检查 `Content-Type` 是不是 `application/x-www-form-urlencoded`,不是就拒绝
2. 把整个 body 读进内存
3. 按 urlencoded 格式解析,做 URL 解码
4. 交给 serde 按 `Input` 的字段反序列化,类型不对或字段缺失就失败
5. 成功则包成 `Form(input)` 传给你;失败则**不调用你的 handler**,直接返回错误响应

第 00 章你手写过第 2 步(而且写得不健壮,只 `read` 一次),剩下四步全没做。现在它们浓缩成了参数类型里的一个 `Form<Input>`。

**`#[derive(Deserialize, Debug)]`** 里,`Deserialize` 是反序列化必需的,`Debug` 是为了下面 `dbg!` 能打印。

**`#[allow(dead_code)]`** 抑制"字段从未被读取"的警告。

> **更好的写法**:这个属性在这里其实是多余的——`name` 和 `email` 都在下面的 `format!` 里读到了,不加也不会有警告。它多半是示例早期版本的遗留(那时 handler 可能只 `dbg!` 不使用字段)。你自己写代码时不必跟着加:`#[allow(...)]` 会顺带屏蔽掉真正的死代码警告,能不加就不加。

**`dbg!(&input)`** 是标准库宏,把值以 `Debug` 格式打到 **stderr**,同时带上文件名和行号。注意它走的是 stderr 而不是 `tracing`,所以不受 `RUST_LOG` 控制,任何情况下都会输出。

> **更好的写法**:`dbg!` 是**调试用**的,不该出现在提交的代码里——clippy 的 `dbg_macro` lint 会警告,axum 仓库自己也开了这条 lint(所以示例其实是被豁免的)。示例用它是因为在教程语境下"一行代码就能看到结构化数据"最直观。你的项目里应该用 `tracing::debug!(?input)`,它受 `RUST_LOG` 控制、能带 span 上下文、生产环境可以按级别关掉。后面第 08、10 章的示例也有同样的 `dbg!`,不再重复说明。

**返回类型是 `Html<String>` 而不是 `Html<&'static str>`。** 因为内容是运行时 `format!` 出来的,不可能是 `'static` 的字符串字面量。[第 01 章](01-hello-world.md#intoresponse返回值如何变成-http-响应)那张表里 `Html<T>` 对 `T` 是泛型的,`String` 和 `&str` 都行。

**跑起来看**:

```bash
cargo run
```

浏览器访问 `http://127.0.0.1:3000/`,填上名字和邮箱,点 Subscribe。页面跳转到 `/` 并显示:

```
email='a@b.com' name='张三'
```

回到服务端终端,能看到 `dbg!` 的输出:

```
[src/main.rs:69:5] &input = Input {
    name: "张三",
    email: "a@b.com",
}
```

**中文没有乱码,`@` 也正确还原了。** 浏览器发出去的是 `name=%E5%BC%A0%E4%B8%89&email=a%40b.com`,`Form` 提取器替你完成了 URL 解码和 UTF-8 解析。

用 curl 复现一次,看清底层发生了什么:

```bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'name=foo&email=bar@axum'
```

```
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 29

email='bar@axum'
name='foo'
```

其实 `-d` 参数会自动带上 `Content-Type: application/x-www-form-urlencoded` 并把方法改成 `POST`,所以可以简写:

```bash
curl -d 'name=foo&email=bar@axum' http://127.0.0.1:3000/
```

### 步骤 3:观察提取失败时的行为

**要做什么**:不写新代码,故意发几个错误的请求,看 axum 怎么应对。这一步是为了理解提取器的**拒绝**(rejection)机制,它是第 10、11 章的基础。

**缺字段**。只发 `name`,不发 `email`:

```bash
curl -i -d 'name=foo' http://127.0.0.1:3000/
```

```
HTTP/1.1 422 Unprocessable Entity
content-type: text/plain; charset=utf-8
content-length: 68

Failed to deserialize form body: missing field `email`
```

`422 Unprocessable Entity` 的语义是"请求格式我看懂了,但内容不符合要求"——比笼统的 `400 Bad Request` 更精确。错误信息也直接指明了缺哪个字段。

**Content-Type 不对**。发 JSON 但声明成 JSON 格式:

```bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: application/json' \
  -d '{"name":"foo","email":"bar"}'
```

```
HTTP/1.1 415 Unsupported Media Type
content-type: text/plain; charset=utf-8
content-length: 72

Form requests must have `Content-Type: application/x-www-form-urlencoded`
```

`415 Unsupported Media Type` 表示"格式我不支持"。`Form` 提取器在读 body 之前先检查 `Content-Type`,不匹配直接拒绝——省掉了无用的解析工作。

**关键在于:你的 handler 一次都没被调用。** 三次失败请求,`dbg!` 一行输出都没有。提取失败时 axum 直接返回拒绝响应,handler 根本不会执行。

这个设计的价值在于**类型即校验**。`accept_form` 的函数体可以放心假设 `input.name` 和 `input.email` 都是有效的 `String`,不需要写任何 `if let Some(...)` 或 `match` 来处理缺失情况。**不合法的请求在进入你的代码之前就被挡住了**,而挡住它的依据就是你声明的类型。

不过默认的拒绝响应是纯文本,对人类友好但对程序不友好——写 API 时前端通常希望拿到结构化的 JSON 错误。想自定义它有几种办法:

- 第 10 章 customize-extractor-error:包一层自己的提取器,返回 JSON 错误
- 第 11 章 customize-path-rejection:针对 `Path` 提取器定制拒绝响应
- 第 12 章 validator:在反序列化成功之后再加一层业务校验(邮箱格式、长度限制等)

注意 `Form` 只保证**类型**正确,不保证**业务**合法。上面 `email=bar@axum` 能通过,`email=随便乱写` 也能通过——因为它们都是合法的 `String`。真正的邮箱格式校验是第 12 章的事。

## 完整代码

`src/main.rs`:

```rust
use axum::{extract::Form, response::Html, routing::get, Router};
use serde::Deserialize;
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

    // build our application with some routes
    let app = app();

    // run it
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

fn app() -> Router {
    Router::new().route("/", get(show_form).post(accept_form))
}

async fn show_form() -> Html<&'static str> {
    Html(
        r#"
        <!doctype html>
        <html>
            <head></head>
            <body>
                <form action="/" method="post">
                    <label for="name">
                        Enter your name:
                        <input type="text" name="name">
                    </label>

                    <label>
                        Enter your email:
                        <input type="text" name="email">
                    </label>

                    <input type="submit" value="Subscribe!">
                </form>
            </body>
        </html>
        "#,
    )
}

#[derive(Deserialize, Debug)]
#[allow(dead_code)]
struct Input {
    name: String,
    email: String,
}

async fn accept_form(Form(input): Form<Input>) -> Html<String> {
    dbg!(&input);
    Html(format!("email='{}'\nname='{}'\n", input.email, input.name))
}
```

依赖:

- `axum` 0.8.x
- `tokio` 1.x,feature `full`
- `serde` 1.x,feature `derive`
- `tracing` 0.1.x
- `tracing-subscriber` 0.3.x,feature `env-filter`

> 官方示例的 `main.rs` 末尾还有一个 `#[cfg(test)] mod tests`,用 `tower::ServiceExt::oneshot` 在不启动服务器的情况下测试这两个 handler。测试写法见第 21 章,那里会完整讲解并补上对应的 `cargo add --dev` 依赖。

## 动手练习

**1. 加一个可选字段。** 给 `Input` 加上 `phone: Option<String>`,表单里加对应的输入框。

提示:serde 对 `Option<T>` 的处理是"字段缺失或为空则为 `None`"。不用改 handler 逻辑就能工作。验证:`curl -d 'name=foo&email=bar' http://127.0.0.1:3000/` 和 `curl -d 'name=foo&email=bar&phone=123' ...` 都应该成功,对比 `dbg!` 的输出。想想为什么这次缺字段不再是 `422` 了。

**2. 把 `Form` 换成 `Query`,方法换回 `GET`。** 让表单用 `method="get"` 提交。

提示:`axum::extract::Query` 用法和 `Form` 几乎一样,`Input` 一个字都不用改——这正是 serde"数据结构与格式解耦"的体现。路由改成只有 `get`,但同一个路径要同时显示表单和处理提交,需要想办法区分(比如把表单页挪到 `/`,提交目标设为 `/subscribe`)。提交后观察浏览器地址栏的变化,和 `POST` 对比。

## 小结与下一章预告

提取器是 axum 的核心抽象:在 handler 参数位置写一个类型,axum 就在调用你的代码之前把对应信息从请求里取出来。`Form<T>` 负责 urlencoded 请求体——检查 `Content-Type`、读 body、URL 解码、交给 serde 按字段反序列化,任何一步失败就返回 `415` 或 `422` 而**不调用你的 handler**。于是函数体里可以放心假设数据是有效的,校验工作由类型声明完成。

提取器分两类:`FromRequestParts` 只读头部信息、可以有多个;`FromRequest` 要消耗 body、只能有一个且必须放最后。

本章的路由是固定路径 `/`。真实的 Web 服务大量使用**动态路径**——`/users/42`、`/posts/hello-world`,路径里的一段是变量。[第 04 章 templates](04-templates.md) 会用 `Path` 提取器接收这类参数,同时引入模板引擎:把 HTML 从 Rust 源码里搬到独立的模板文件,并且自动做转义,解决[第 01 章](01-hello-world.md#步骤-3改成返回-html)提到的 XSS 风险。你还会亲手为自己的类型实现 `IntoResponse`。
