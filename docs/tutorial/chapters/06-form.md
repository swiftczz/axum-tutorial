# 06. form

对应示例：`examples/form`

前面用 `curl` 手动发请求。这章让浏览器参与进来：`GET /` 返回一个 HTML 表单，用户填写提交后浏览器发送 `POST /`，axum 用 `Form<T>` 把表单解析成 Rust 结构体。代码不长（约 70 行），分 3 步搭。

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
tower = { version = "0.5.2", features = ["util"] }
````

本章相比前面章节新增：`http-body-util`（body 工具，提供 `BodyExt::collect` 把流式 body 收集成 bytes）和 `mime`（MIME 类型常量，比手写字符串 `"application/x-www-form-urlencoded"` 类型安全）。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：返回 HTML 表单页面

先写一个只返回表单页面的 `GET /`，浏览器能打开看到表单。

````rust
use axum::{response::Html, routing::get, Router};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(show_form));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await;
}

async fn show_form() -> Html<&'static str> {
    Html(r#"
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
    "#)
}
````

跑起来，浏览器打开 `http://127.0.0.1:3000/`，能看到表单。但点"Subscribe!"会失败——因为没有 `POST /` 的 handler。

表单 HTML 的关键部分：

- `action="/"`：提交到 `/` 这个路径。
- `method="post"`：用 POST 方法提交。
- `name="name"` 和 `name="email"`：提交时的字段名。

`r#"..."#` 是 Rust 的 raw string，适合写多行 HTML，里面可以直接包含双引号。

---

## 第二步：定义输入结构体，接收表单

现在加 `POST /`。浏览器提交表单时，body 格式是 `name=foo&email=bar%40axum`（`application/x-www-form-urlencoded`）。axum 的 `Form<T>` extractor 负责解析它。

先定义结构体：

````rust
use serde::Deserialize;

#[derive(Deserialize, Debug)]
#[allow(dead_code)]
struct Input {
    name: String,
    email: String,
}
````

字段名 `name` 和 `email` 要和 HTML 里的 `name="..."` 一一对应：

| HTML input | Rust 字段 |
| --- | --- |
| `name="name"` | `name: String` |
| `name="email"` | `email: String` |

> **新面孔：`#[derive(Deserialize)]`**
>
> 这是 serde 的 derive 宏：编译器自动帮你实现 `Deserialize` trait（把表单数据反序列化成 Rust 结构体）。整个教程你会反复看到 `#[derive(...)]`——`Serialize`、`Clone`、`Debug` 都是这个模式。

`#[allow(dead_code)]` 是因为这个 example 开了较严格的 lint，避免字段被认为没使用。`Debug` 让 `dbg!(&input)` 能打印结构体。

---

## 第三步：写 POST handler，把路由合在一起

现在写 `accept_form` handler，用 `Form<Input>` 接收表单数据：

````rust
use axum::extract::Form;

async fn accept_form(Form(input): Form<Input>) -> Html<String> {
    dbg!(&input);
    Html(format!("email='{}'\nname='{}'\n", input.email, input.name))
}
````

`Form(input): Form<Input>` 和前面学过的 `Json(payload): Json<CreateUser>` 是同类 extractor。它告诉 axum：从请求 body 读表单数据，解析成 `Input`，绑定到 `input`。

最后把两个 handler 挂到同一路径（`GET /` 展示表单，`POST /` 接收表单）：

````rust
fn app() -> Router {
    Router::new().route("/", get(show_form).post(accept_form))
}
````

---

## 完整代码

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

    let app = app();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
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

浏览器访问 `http://127.0.0.1:3000/`，填写表单点提交。也可以用 curl 模拟：

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'content-type: application/x-www-form-urlencoded' \
  -d 'name=foo&email=bar@axum'
````

预期响应：

````text
email='bar@axum'
name='foo'
````

运行测试：

````bash
cargo test -p example-form
````

## 常见问题

**表单提交和 JSON 有什么区别？** 表单用 `application/x-www-form-urlencoded` 格式（`name=foo&email=bar`），JSON 用 `application/json`（`{"name":"foo","email":"bar"}`）。`Form<T>` 解析前者，`Json<T>` 解析后者。

**字段名对不上会怎样？** 如果 HTML 写 `name="username"` 但 Rust 结构体写 `name: String`，提交时 serde 会报错（找不到 `name` 字段），axum 返回 400。

## 手写任务

跑通后做两个小改动：

1. 给表单加一个 `age` 输入框，给 `Input` 加 `age: String` 字段，在响应里也输出。
2. 把响应格式从文本改成 HTML，让结果更好看。

## 小结

这章分 3 步搭了一个表单提交服务：

1. **GET 返回表单**：`Html(r#"..."#)` 返回包含 `<form>` 的 HTML。
2. **定义结构体**：`#[derive(Deserialize)]` + 字段名和 HTML `name` 对应。
3. **POST 接收表单**：`Form(input): Form<Input>` 提取请求 body 里的表单数据。

`GET /` 展示表单、`POST /` 接收表单是常见的后端模式。

## 源码对照

- `examples/form/Cargo.toml`
- `examples/form/src/main.rs`
