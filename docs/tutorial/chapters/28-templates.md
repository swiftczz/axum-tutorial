# 28. templates

对应示例：`examples/templates`

第 29 章用 MiniJinja（运行时模板引擎）。这章用 **Askama**——**编译期**把模板编译成 Rust 代码，类型安全、性能好（零运行时开销）。语法是 Jinja2 风格，但模板生成的是 Rust 函数。

分 3 步：先 derive `Template` 定义模板 struct，再写 `HtmlTemplate` wrapper 实现 `IntoResponse`，最后看完整 handler。

相比前面章节新引入：**`askama` crate、`#[derive(Template)]` + `#[template(path = "...")]`、编译期模板检查、`IntoResponse` wrapper for `Template`**。

## Cargo.toml

````toml
[package]
name = "example-templates"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
askama = "0.13"
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：derive `Template` 定义模板 struct

Askama 用 derive 宏把模板文件和 Rust struct 关联——编译期读取模板文件、生成对应的 Rust 渲染代码。struct 字段就是模板变量。

````rust
use askama::Template;

#[derive(Template)]
#[template(path = "hello.html")]
struct HelloTemplate {
    name: String,
}
````

`templates/hello.html`：

````html
<h1>Hello, {{ name }}!</h1>
````

> **新面孔：`askama::Template` derive**
>
> derive `Template` 让 struct 变成可渲染模板。`#[template(path = "hello.html")]` 指定模板文件路径（相对 `templates/` 目录，目录可在 Cargo.toml 配置）。
>
> 编译期 askama 读取模板文件、生成 `render(&self) -> String` 方法。模板里写错变量名（如 `{{ nam }}`）会**编译失败**——类型安全。

> **新面孔：编译期模板检查**
>
> 这是 askama 比 MiniJinja/Tera 强的地方：模板里用 `{{ name }}` 但 struct 没有 `name` 字段，**编译失败**而不是运行时 panic。
>
> 好处：重构 struct 字段时编译器自动检查所有模板，不会漏改。

---

## 第二步：`HtmlTemplate` wrapper 实现 `IntoResponse`

Askama 的 `Template` 提供 `.render()` 方法返回 `String`，但不是 `IntoResponse`。写个泛型 wrapper 把它转成 axum 响应。

````rust
use axum::{
    http::StatusCode,
    response::{Html, IntoResponse, Response},
};

struct HtmlTemplate<T>(T);

impl<T> IntoResponse for HtmlTemplate<T>
where
    T: Template,
{
    fn into_response(self) -> Response {
        // Render 模板，成功返回 Html 响应，失败返回 500
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

> **新面孔：泛型 wrapper 实现 `IntoResponse`**
>
> `HtmlTemplate<T>(T)` 包住任意 `T: Template`。`impl<T> IntoResponse for HtmlTemplate<T>` 让任何模板类型都能当响应返回。
>
> 模式：定义 wrapper struct + 给 wrapper 实现 `IntoResponse`，避免每个模板 struct 单独实现。可复用——任何项目都用这个 wrapper。

> **新面孔：模板渲染错误处理**
>
> `.render()` 返回 `Result<String, askama::Error>`。成功包成 `Html(html)`（自动设 content-type: text/html）；失败返回 500 + 错误消息。
>
> 编译期检查能拦掉大部分错误（变量名错、语法错），运行时错误极少（IO 失败等）。

---

## 第三步：完整 handler + 测试

最后写 handler 把 `Path` 参数传进模板，渲染返回。

````rust
use axum::{
    extract,
    response::Html,
    routing::get,
    Router,
};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(/* tracing 初始化 */)
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app = app();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
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

# #[derive(Template)]
# #[template(path = "hello.html")]
# struct HelloTemplate { name: String }
#
# struct HtmlTemplate<T>(T);
# impl<T: Template> IntoResponse for HtmlTemplate<T> { /* ... */ }
````

验证：

````bash
cd examples
cargo run -p example-templates

curl http://127.0.0.1:3000/greet/Foo
# <h1>Hello, Foo!</h1>
````

> **新面孔：handler 返回 wrapper**
>
> `async fn greet(...) -> impl IntoResponse` 返回 `HtmlTemplate<HelloTemplate>`。axum 看到 `IntoResponse` 自动调 `into_response()` 把模板渲染成 HTML 响应。
>
> 不用 `.into_response()` 显式调用——`impl IntoResponse` 让 axum 自动处理。

---

## 完整代码

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

    // build our application with some routes
    let app = app();

    // run it
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
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

#[cfg(test)]
mod tests;
````

## 运行

````bash
cd examples
cargo run -p example-templates

curl http://127.0.0.1:3000/greet/Foo
# <h1>Hello, Foo!</h1>

curl http://127.0.0.1:3000/greet/Alice
# <h1>Hello, Alice!</h1>
````

## 解读

### Askama vs MiniJinja（ch29）

| 维度 | Askama（这章） | MiniJinja（ch29） |
| --- | --- | --- |
| 编译时机 | 编译期 | 运行时 |
| 类型安全 | 是（变量名错编译失败） | 否（运行时 panic） |
| 性能 | 快（生成 Rust 代码） | 慢（运行时解释） |
| 灵活性 | 低（模板改要重编译） | 高（运行时可加载） |
| 模板继承 | 支持 | 支持 |

**Askama 适合**：模板稳定、追求性能和类型安全。
**MiniJinja 适合**：模板频繁改、需要热重载、运行时动态加载。

### Cargo.toml 配置模板目录

默认 Askama 从 `templates/` 找模板文件。改目录在 Cargo.toml：

````toml
[package.metadata.askama]
template_dirs = ["$CARGO_MANIFEST_DIR/src/templates"]
````

## 常见问题

**改了模板文件不生效？** Askama 是编译期，改完模板必须 `cargo build` 重新编译。开发体验差，但生产稳定。

**模板编译报错 "field not found"？** 模板用了 `{{ x }}` 但 struct 没有 `x` 字段。这是 askama 的类型安全特性，加上字段就好。

**怎么测模板？** `cargo test`——模板渲染是纯函数，单元测试直接 `template.render().unwrap()` 检查输出。

## 手写任务

1. 加 `{% if user.is_admin %}...{% endif %}` 条件渲染。
2. 加 `{% for item in items %}` 循环（struct 字段用 `Vec<String>`）。
3. 写 layout 模板，子模板用 `{% extends "layout" %}` + `{% block content %}`（和 ch29 同样的继承机制）。
4. 把 ch29 MiniJinja 版改成 Askama，对比代码。

## 小结

这章用 3 步讲了 Askama 模板：

1. **derive Template**：`#[derive(Template)] #[template(path = "...")]` 把 struct 变模板，编译期生成渲染代码。
2. **HtmlTemplate wrapper**：泛型 wrapper + `IntoResponse` impl，让任何模板类型都能当响应。
3. **handler 集成**：handler 返回 `HtmlTemplate(template)`，axum 自动渲染。

核心：Askama 是**编译期**模板引擎，类型安全 + 高性能；MiniJinja（ch29）是运行时，灵活 + 热重载。新项目模板稳定时优先 Askama。

## 源码对照

- `examples/templates/Cargo.toml`
- `examples/templates/src/main.rs`
- `examples/templates/templates/hello.html`
