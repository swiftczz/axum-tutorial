# 27. templates

对应示例：`examples/templates`

前面接口大多返回 JSON 或纯文本,这章开始服务端渲染 HTML。用 Askama 模板引擎在 axum 中渲染 HTML,理解模板文件、模板数据结构和 `IntoResponse` 包装器如何配合。

## Cargo.toml

````toml
[package]
name = "example-templates"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
askama = { version = "0.15", default-features = false, features = ["std", "derive"] }
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

[dev-dependencies]
http-body-util = "0.1.0"
tower = { version = "0.5.2", features = ["util"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## templates/hello.html

````html
<h1>Hello, {{ name }}!</h1>
````

`{{ name }}` 是模板变量,从 Rust 结构体同名字段读取。

## src/main.rs

````rust
use askama::Template;
use axum::{
    extract,
    http::StatusCode,
    response::{Html, IntoResponse, Response},
    routing::get,
    Router,
};
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

    axum::serve(listener, app).await.unwrap();
}

fn app() -> Router {
    Router::new().route("/greet/{name}", get(greet))
}

async fn greet(extract::Path(name): extract::Path<String>) -> impl IntoResponse {
    let template = HelloTemplate { name };
    HtmlTemplate(template)
}

#[derive(Template)]
#[template(path = "hello.html")]
struct HelloTemplate {
    name: String,
}

struct HtmlTemplate<T>(T);

impl<T> IntoResponse for HtmlTemplate<T>
where
    T: Template,
{
    fn into_response(self) -> Response {
        match self.0.render() {
            Ok(html) => Html(html).into_response(),
            Err(err) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                format!("Failed to render template. Error: {err}"),
            )
                .into_response(),
        }
    }
}
````

## 运行

````bash
cd examples
cargo run -p example-templates
````

请求页面:

````bash
curl http://127.0.0.1:3000/greet/Foo
# 预期: <h1>Hello, Foo!</h1>
````

浏览器打开 `http://127.0.0.1:3000/greet/Foo` 也可以。运行测试:

````bash
cargo test -p example-templates
````

## 解读

### 为什么需要模板

页面很简单时可以直接 `Html(format!("<h1>Hello, {name}!</h1>"))`。但页面变复杂后手写字符串很痛苦:HTML 结构难读、变量插入容易乱、条件渲染和循环不好写、容易忘转义。模板引擎把 HTML 放回 `.html` 文件,动态数据留在 Rust 结构体里:**Rust 负责准备数据,模板负责展示数据**。

### 模板结构体三件套

````rust
#[derive(Template)]
#[template(path = "hello.html")]
struct HelloTemplate {
    name: String,
}
````

1. `#[derive(Template)]`:让 Askama 为这个结构体生成渲染代码。
2. `#[template(path = "hello.html")]`:声明对应哪个模板文件。
3. `name: String`:提供模板里 `{{ name }}` 需要的数据。

字段名必须和模板变量一致——模板里写 `{{ name }}`,结构体里必须有 `name` 字段,否则编译/渲染失败。`HelloTemplate` 不是普通数据结构,它是 `hello.html` 的数据模型。

### handler 只准备数据

````rust
async fn greet(extract::Path(name): extract::Path<String>) -> impl IntoResponse {
    let template = HelloTemplate { name };
    HtmlTemplate(template)
}
````

`Path<String>` 从 URL 提取 name,创建 `HelloTemplate`,用 `HtmlTemplate` 包起来返回。handler 不直接调 `template.render()`——渲染和错误处理放进了 `HtmlTemplate` 包装器。

### `HtmlTemplate<T>` 包装器

````rust
struct HtmlTemplate<T>(T);

impl<T> IntoResponse for HtmlTemplate<T>
where
    T: Template,
{
    fn into_response(self) -> Response {
        match self.0.render() {
            Ok(html) => Html(html).into_response(),
            Err(err) => (StatusCode::INTERNAL_SERVER_ERROR, format!("Failed to render template. Error: {err}")).into_response(),
        }
    }
}
````

为什么不直接返回 `HelloTemplate`?axum handler 要返回能转换成 HTTP 响应的类型,Askama 模板知道怎么渲染 HTML 但不知道怎么变成 HTTP 响应。`HtmlTemplate<T>` 就是中间适配层:模板 → HTML 字符串 → axum `Html` → HTTP 响应。

泛型约束 `T: Template` 让任意 Askama 模板都能被 `HtmlTemplate` 包起来返回。成功返回 HTML,失败返回 500(真实项目通常不把完整模板错误返回用户,这里为 example 清晰直接输出)。

### Askama 编译期 vs 运行期

`#[derive(Template)]` 在**编译期**读模板文件并生成渲染代码(检查模板结构),真正渲染在**运行期**:`self.0.render()` 把本次请求的数据填进模板。这是 Askama 的优势——很多模板错误可以更早暴露。

## 常见问题

**模板文件找不到?** Askama 默认在 crate 的 `templates` 目录找,这个 example 是 `examples/templates/templates/hello.html`,不要放 `src/templates`。

**字段名要和模板变量一致?** 是的,模板写 `{{ name }}`,结构体必须有 `name` 字段。

**渲染失败返回什么?** example 返回 500 和错误文本。真实项目更常见是日志记详细错误,用户看通用错误页面,避免暴露内部路径和模板细节。

**服务端模板 vs 前端框架?** 简单后台/文档页/管理页/传统表单用服务端模板合适;交互复杂、状态多的产品前端更适合 React/Vue。

## 手写任务

按下面顺序敲:

1. 新建 `templates/hello.html`,写 `<h1>Hello, {{ name }}!</h1>`。
2. 定义 `HelloTemplate { name: String }`。
3. 加 `#[derive(Template)]` 和 `#[template(path = "hello.html")]`。
4. 写 `HtmlTemplate<T>` 包装器。
5. 实现 `IntoResponse`,成功返回 `Html(html)`,失败返回 500。
6. 写 `/greet/{name}` 路由,从 Path 提取 name。
7. curl 验证 HTML。

加深练习:

1. 模板改成含 `<title>` 和 `<body>` 的完整 HTML。
2. 给模板结构体加 `items: Vec<String>`,练习 Askama 循环。
3. 新增 `/` 首页模板。

## 小结

- 服务端模板分工:handler 准备数据,模板文件负责 HTML 结构,`IntoResponse` 包装器把渲染结果变成 HTTP 响应。
- Askama 模板三件套:`#[derive(Template)]` + `#[template(path = "...")]` + 与模板变量同名的字段。
- Askama 在编译期生成渲染代码,运行期填数据,模板错误能更早暴露。
- `HtmlTemplate<T>` 适配 Askama 模板到 axum 响应;成功返回 HTML,失败返回 500。
- 简单后台/管理页适合服务端模板,复杂产品前端用 React/Vue。

## 源码对照

- `examples/templates/Cargo.toml`
- `examples/templates/src/main.rs`
- `examples/templates/templates/hello.html`
