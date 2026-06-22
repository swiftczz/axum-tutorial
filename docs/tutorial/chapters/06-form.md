# 06. form

对应示例：`examples/form`

前面几章用 `curl` 手动发请求。本章让浏览器参与进来：`GET /` 返回一个 HTML 表单，用户填写提交后浏览器发送 `POST /`，axum 用 `Form<T>` 把表单解析成 Rust 结构体。

## Cargo.toml

````toml
[package]
name = "example-form"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

[dev-dependencies]
http-body-util = "0.1.3"
mime = "0.3.17"
tower = "0.5.2"
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

````rust
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

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app()).await;
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
````

## 运行

````bash
cd examples
cargo run -p example-form
````

浏览器访问 `http://127.0.0.1:3000/`,看到一个表单。填写并提交,或用 curl 模拟提交:

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'content-type: application/x-www-form-urlencoded' \
  -d 'name=foo&email=bar@axum'
````

预期响应:

````text
email='bar@axum'
name='foo'
````

运行测试:

````bash
cargo test -p example-form
````

## 解读

### 浏览器表单请求长什么样

HTML 关键部分:

````html
<form action="/" method="post">
    <input type="text" name="name">
    <input type="text" name="email">
    <input type="submit" value="Subscribe!">
</form>
````

提交时浏览器发的请求是:

```text
方法：POST
路径：/
Header：content-type: application/x-www-form-urlencoded
Body：name=foo&email=bar%40example.com
```

后端只要定义字段名匹配的结构体就能接住这些值。

### `get(show_form).post(accept_form)`

````rust
fn app() -> Router {
    Router::new().route("/", get(show_form).post(accept_form))
}
````

同一个路径 `/` 上同时挂 GET 和 POST。`get(...).post(...)` 返回一个 MethodRouter,把方法与 handler 配对。常见模式:`GET /` 展示表单,`POST /` 接收表单。

### `Form(input): Form<Input>`

````rust
async fn accept_form(Form(input): Form<Input>) -> Html<String> {
````

和 `Json(payload): Json<CreateUser>` 写法一样,是个 extractor。它告诉 axum:读请求 body,按 `application/x-www-form-urlencoded` 解析,转成 `Input`,绑定到 `input`。body 不是合法表单或字段类型不匹配,axum 会在进入 handler 前返回错误。

`Form<T>` 内部用 serde 解析,所以 `Input` 需要 `#[derive(Deserialize)]`。字段名要和 HTML 的 `name="..."` 对上:

| HTML input | Rust 字段 |
| --- | --- |
| `name="name"` | `name: String` |
| `name="email"` | `email: String` |

`#[allow(dead_code)]` 是因为这个 example 开了较严格的 lint,避免字段被认为没使用。`Debug` 是为了让 `dbg!(&input)` 能打印结构体。

### `show_form` 返回 `Html<&'static str>`

`r#"..."#` 是 Rust 的 raw string,适合写多行 HTML,里面可以直接包含双引号。

### 测试不启动端口

测试用 `oneshot` 直接调 Router,关键两行模拟浏览器提交:

````rust
.header(http::header::CONTENT_TYPE, mime::APPLICATION_WWW_FORM_URLENCODED.as_ref())
.body(Body::from("name=foo&email=bar@axum"))
````

告诉服务端 body 是表单格式,发送两个字段。这比启动端口再开浏览器更适合自动测试。

## 手写任务

跑通后做三个小改动:

1. 给表单增加一个 `age` 输入框。
2. 给 `Input` 加 `age: String` 字段。
3. 在 `accept_form` 响应里也输出 age。

关键认识:HTML input 的 `name` 属性必须和 Rust struct 字段名对上。

## 小结

- HTML 表单提交是一次 HTTP POST 请求,`content-type: application/x-www-form-urlencoded`。
- `Form<T>` 解析表单 body,内部用 serde,所以 struct 要 `#[derive(Deserialize)]`。
- 表单字段名要和 struct 字段名匹配。
- `GET /` 展示表单,`POST /` 接收表单,是常见后端模式。
- `Form<T>` 和 `Json<T>` 是同类 extractor,只是解析格式不同。

## 源码对照

- `examples/form/Cargo.toml`
- `examples/form/src/main.rs`
