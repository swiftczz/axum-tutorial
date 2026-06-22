# 06. form

对应示例：`examples/form`

本章目标：手写 HTML 表单提交，理解 `Form<T>` 如何把浏览器提交的表单解析成 Rust 结构体。

## 这个小项目在做什么

前面几章主要用 `curl` 手动发请求。这一章开始让浏览器参与进来：

- `GET /` 返回一个 HTML 页面。
- 页面里有一个 `<form>` 表单。
- 用户填写 `name` 和 `email`。
- 点击提交后，浏览器发送 `POST /`。
- Axum 用 `Form<Input>` 把表单 body 解析成 Rust 结构体。
- 服务端返回提交结果。

请求主线是：

```text
浏览器访问 GET /
-> show_form 返回 HTML 表单
-> 用户填写并提交
-> 浏览器发送 POST /
-> Form<Input> 解析 name 和 email
-> accept_form 返回 HTML 响应
```

这一章的重点不是前端页面，而是理解浏览器表单会怎样变成后端 handler 的输入。

## 先看懂 HTML 表单请求

HTML 里这段最关键：

````html
<form action="/" method="post">
    <input type="text" name="name">
    <input type="text" name="email">
    <input type="submit" value="Subscribe!">
</form>
````

它表达的是：

- `action="/"`：提交到 `/` 这个路径。
- `method="post"`：用 POST 方法提交。
- `name="name"`：这个输入框提交时的字段名叫 `name`。
- `name="email"`：这个输入框提交时的字段名叫 `email`。

浏览器提交时，请求大致长这样：

```text
方法：POST
路径：/
Header：content-type: application/x-www-form-urlencoded
Body：name=foo&email=bar%40example.com
```

所以后端只要定义一个字段名匹配的结构体，就可以接住这些值。

## 文件和依赖

这个 example 有两个主要文件：

1. `examples/form/Cargo.toml`：声明 Axum、Serde、Tokio、tracing 和测试依赖。
2. `examples/form/src/main.rs`：实现表单页面、表单提交 handler 和测试。

关键依赖：

- `axum`：提供 `Router`、`Html`、`Form<T>`。
- `serde`：让表单字段能反序列化成 Rust struct。
- `tokio`：提供异步运行时和异步测试支持。
- `tracing` / `tracing-subscriber`：初始化日志。
- `tower`、`http-body-util`、`mime`：测试里直接调用 Router 并检查响应。

## 第一步：把 Router 提成 app 函数

源码：

````rust
fn app() -> Router {
    Router::new().route("/", get(show_form).post(accept_form))
}
````

这里的写法比前几章紧凑一点：

```text
GET /  -> show_form
POST / -> accept_form
```

它们路径相同，方法不同。  
`get(show_form).post(accept_form)` 表示同一个路径 `/` 上同时挂 GET 和 POST。

为什么要提成 `app()`？

```text
main 用 app() 启动服务。
测试也用 app() 直接发请求。
```

这样运行代码和测试代码共用同一套路由。

## 第二步：写返回表单页面的 handler

源码：

````rust
async fn show_form() -> Html<&'static str> {
    Html(r#"..."#)
}
````

这个 handler 处理 `GET /`。

返回类型是 `Html<&'static str>`，表示响应体是一段 HTML。  
`r#"..."#` 是 Rust 的 raw string，适合写多行 HTML，因为里面可以直接包含双引号。

新手先记住：

```text
GET / 通常用来展示表单。
POST / 通常用来接收表单。
```

## 第三步：定义表单输入结构体

源码：

````rust
#[derive(Deserialize, Debug)]
#[allow(dead_code)]
struct Input {
    name: String,
    email: String,
}
````

这里的字段名必须和 HTML 里的 `name="..."` 对上：

| HTML input | Rust 字段 |
| --- | --- |
| `name="name"` | `name: String` |
| `name="email"` | `email: String` |

为什么要 `Deserialize`？

因为方向是：

```text
表单请求 body -> Rust struct
```

这个方向叫反序列化。

`Debug` 是为了 `dbg!(&input)` 能把结构体打印出来。  
`#[allow(dead_code)]` 是因为示例项目开启了较严格的 lint，避免字段在某些检查里被认为没有使用。

## 第四步：写接收表单的 handler

源码：

````rust
async fn accept_form(Form(input): Form<Input>) -> Html<String> {
    dbg!(&input);
    Html(format!("email='{}'\nname='{}'\n", input.email, input.name))
}
````

`Form(input): Form<Input>` 和前面学过的 `Json(payload): Json<CreateUser>` 很像。

它的意思是：

```text
请 Axum 读取请求 body。
按照表单格式解析它。
解析成 Input。
把解析结果绑定到 input 变量。
```

如果请求 body 不是合法表单，或者字段类型不匹配，Axum 会在进入 handler 前返回错误。

返回值是 `Html<String>`：

- `format!(...)` 生成一个新的字符串，所以类型是 `String`。
- `Html(...)` 告诉 Axum 按 HTML 响应返回。

## 第五步：理解测试

这个 example 有两个测试：

- `test_get`：直接请求 `GET /`，确认 HTML 里有提交按钮。
- `test_post`：直接构造 `POST /` 表单请求，确认响应体正确。

POST 测试里最关键的是这两行：

````rust
.header(http::header::CONTENT_TYPE, mime::APPLICATION_WWW_FORM_URLENCODED.as_ref())
.body(Body::from("name=foo&email=bar@axum"))
````

它们模拟了浏览器提交表单时做的事：

```text
告诉服务端 body 是表单格式
发送 name 和 email 两个字段
```

这比手动启动端口再打开浏览器更适合自动测试。

## 函数职责速查

- `main`：初始化日志，创建 app，绑定端口并启动服务。
- `app`：构造 Router，注册 `GET /` 和 `POST /`。
- `show_form`：返回 HTML 表单页面。
- `Input`：描述后端期望接收的表单字段。
- `accept_form`：用 `Form<Input>` 接收表单，返回提交结果。
- `test_get`：验证表单页面能正常返回。
- `test_post`：验证表单提交能被解析成 `Input`。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-form
//! ```

// 引入 Form 提取器、Html 响应、GET 路由函数和 Router。
use axum::{extract::Form, response::Html, routing::get, Router};
// serde 负责把表单字段反序列化成 Rust struct。
use serde::Deserialize;
// tracing_subscriber 用来初始化日志。
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 初始化日志系统。
    tracing_subscriber::registry()
        .with(
            // 优先读取环境变量里的日志级别；没有时默认当前 crate 为 debug。
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        // 把日志格式化输出到终端。
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 构造应用路由。
    let app = app();

    // 绑定本机 3000 端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 输出监听地址。
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动 HTTP 服务。
    axum::serve(listener, app).await;
}

// 构造 Router：GET / 显示表单，POST / 接收表单。
fn app() -> Router {
    Router::new().route("/", get(show_form).post(accept_form))
}

// GET / 的 handler，返回一个 HTML 表单页面。
async fn show_form() -> Html<&'static str> {
    // raw string 适合写多行 HTML。
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

// 表单输入类型：字段名要和 HTML input 的 name 属性对应。
#[derive(Deserialize, Debug)]
#[allow(dead_code)]
struct Input {
    name: String,
    email: String,
}

// POST / 的 handler，用 Form<Input> 解析浏览器提交的表单。
async fn accept_form(Form(input): Form<Input>) -> Html<String> {
    // 把解析后的表单内容打印到终端，方便观察。
    dbg!(&input);

    // 返回提交结果。
    Html(format!("email='{}'\nname='{}'\n", input.email, input.name))
}
````

## 运行和验证

运行前先确认 `examples/form/Cargo.toml` 里的 `axum = { path = "../../axum" }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-form
````

浏览器访问：

````text
http://127.0.0.1:3000/
````

你应该看到一个包含 name、email 和提交按钮的表单。

也可以用 curl 模拟表单提交：

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'content-type: application/x-www-form-urlencoded' \
  -d 'name=foo&email=bar@axum'
````

预期响应体：

````text
email='bar@axum'
name='foo'
````

运行测试：

````bash
cd examples
cargo test -p example-form
````

常见卡点：

- HTML 里的 `name="email"` 必须和 Rust 结构体里的 `email` 字段对应。
- 表单提交不是 JSON，`content-type` 应该是 `application/x-www-form-urlencoded`。
- 浏览器里直接访问 `/` 是 GET；点击提交按钮才会发 POST。

## 手写任务

完成本章代码后，试着做三个小改动：

1. 给表单增加一个 `age` 输入框。
2. 给 `Input` 增加 `age: String` 字段。
3. 在 `accept_form` 的响应里也输出 age。

你要确认一件事：

```text
HTML input 的 name 属性
必须和 Rust struct 的字段名对上。
```

## 本章真正要记住什么

- HTML 表单提交会变成一次 HTTP POST 请求。
- `Form<T>` 用来解析 `application/x-www-form-urlencoded` 表单。
- 表单字段名要和 Rust struct 字段名匹配。
- `GET /` 展示表单，`POST /` 接收表单，是很常见的后端模式。
- 把 Router 提到 `app()` 里，可以让 main 和测试复用同一套路由。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/form/Cargo.toml`
- `examples/form/src/main.rs`
