# 27. templates

对应示例：`examples/templates`

本章目标：使用 Askama 模板引擎在 Axum 中渲染 HTML，理解模板文件、模板数据结构和 `IntoResponse` 包装器如何配合。

前面的接口大多返回 JSON 或纯文本。  
这一章开始学习服务端渲染 HTML。

## 这个小项目在做什么

应用只有一个页面：

```text
GET /greet/{name}
```

访问：

```text
/greet/Foo
```

返回：

````html
<h1>Hello, Foo!</h1>
````

这个 HTML 不是在 Rust 代码里手动拼字符串，而是来自模板文件：

```text
templates/hello.html
```

请求主线是：

```text
客户端 GET /greet/Foo
-> Path<String> 提取 Foo
-> 创建 HelloTemplate { name: "Foo" }
-> Askama 渲染 templates/hello.html
-> HtmlTemplate 把渲染结果转成 HTML 响应
-> 客户端收到 <h1>Hello, Foo!</h1>
```

## 先理解为什么需要模板

如果页面很简单，你可以直接写：

````rust
Html(format!("<h1>Hello, {name}!</h1>"))
````

但页面变复杂后，手写字符串会很痛苦：

```text
HTML 结构难读
变量插入容易乱
条件渲染和循环不好写
容易忘记转义
```

模板引擎把 HTML 放回 `.html` 文件，把动态数据留在 Rust 结构体里：

```text
Rust 负责准备数据
模板负责展示数据
```

这就是服务端渲染页面的基本分工。

## 文件和依赖

这个 example 有三个主要文件：

1. `examples/templates/Cargo.toml`：声明 Askama、Axum、Tokio、tracing。
2. `examples/templates/src/main.rs`：实现路由、模板结构体、HTML 响应包装器和测试。
3. `examples/templates/templates/hello.html`：Askama 模板文件。

关键依赖：

- `askama`：模板引擎，负责把模板文件渲染成 HTML 字符串。
- `axum`：提供 Router、Path extractor、Html response。
- `tokio`：异步运行时。
- `tracing` / `tracing-subscriber`：输出启动日志。
- `tower`、`http-body-util`：测试中调用 Router 并读取响应 body。

Askama 依赖配置：

````toml
askama = { version = "0.15", default-features = false, features = ["std", "derive"] }
````

这里启用了：

- `derive`：允许使用 `#[derive(Template)]`。
- `std`：使用标准库能力。

## 第一步：模板文件

模板文件内容：

````html
<h1>Hello, {{ name }}!</h1>
````

位置：

```text
examples/templates/templates/hello.html
```

`{{ name }}` 是模板变量。  
它会从 Rust 结构体字段里读取同名字段：

````rust
struct HelloTemplate {
    name: String,
}
````

如果 `name = "Foo"`，最终渲染结果就是：

````html
<h1>Hello, Foo!</h1>
````

## 第二步：定义模板结构体

源码：

````rust
#[derive(Template)]
#[template(path = "hello.html")]
struct HelloTemplate {
    name: String,
}
````

这段代码做了三件事：

1. `#[derive(Template)]`：让 Askama 为这个结构体生成模板渲染代码。
2. `#[template(path = "hello.html")]`：声明这个结构体对应哪个模板文件。
3. `name: String`：提供模板里 `{{ name }}` 需要的数据。

可以把它理解成：

```text
HelloTemplate 不是普通数据结构
它是 hello.html 的数据模型
```

模板需要什么变量，结构体就要提供什么字段。

## 第三步：注册路由

源码：

````rust
fn app() -> Router {
    Router::new().route("/greet/{name}", get(greet))
}
````

路由里有路径参数：

```text
{name}
```

所以请求：

```text
/greet/Foo
```

会把 `Foo` 提取出来交给 handler。

把 Router 放进 `app()` 函数，和第 25 章一样，是为了方便测试直接调用。

## 第四步：handler 准备模板数据

源码：

````rust
async fn greet(extract::Path(name): extract::Path<String>) -> impl IntoResponse {
    let template = HelloTemplate { name };
    HtmlTemplate(template)
}
````

这段 handler 做的事很少：

1. 用 `Path<String>` 从 URL 提取 `name`。
2. 创建 `HelloTemplate { name }`。
3. 用 `HtmlTemplate(template)` 包起来返回。

handler 没有直接调用：

````rust
template.render()
````

因为渲染和错误处理被放进了 `HtmlTemplate` 这个包装器里。

## 第五步：为什么需要 HtmlTemplate 包装器

源码：

````rust
struct HtmlTemplate<T>(T);
````

这是一个元组结构体。  
它只有一个字段，用来包住任意模板：

````rust
HtmlTemplate(template)
````

为什么不直接返回 `HelloTemplate`？

因为 Axum handler 需要返回能转换成 HTTP 响应的类型。  
Askama 的模板结构体本身知道怎么渲染 HTML，但 Axum 不知道它应该变成什么 HTTP 响应。

所以这里给 `HtmlTemplate<T>` 实现 `IntoResponse`：

```text
模板 -> HTML 字符串 -> Axum Html -> HTTP 响应
```

## 第六步：实现 IntoResponse

源码：

````rust
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

这里的泛型约束是：

````rust
T: Template
````

意思是：

```text
只要 T 是 Askama 模板，就可以被 HtmlTemplate 包起来返回
```

成功时：

````rust
Ok(html) => Html(html).into_response()
````

把渲染后的 HTML 字符串放进 Axum 的 `Html` response。

失败时：

````rust
Err(err) => (
    StatusCode::INTERNAL_SERVER_ERROR,
    format!("Failed to render template. Error: {err}"),
).into_response()
````

返回 500 和错误信息。

真实项目里通常不把完整模板错误直接返回给用户。  
这里为了 example 清楚展示错误，所以直接输出了。

## 第七步：Askama 渲染发生在什么时候

`#[derive(Template)]` 会在编译期读取模板文件并生成渲染代码。  
真正执行渲染是在请求返回响应时：

````rust
self.0.render()
````

也就是说：

```text
编译期：检查模板结构
运行期：把本次请求的数据填进模板
```

这也是 Askama 的一个优势：很多模板错误可以更早暴露。

## 第八步：测试如何验证 HTML

源码里的测试：

````rust
let response = app()
    .oneshot(Request::get("/greet/Foo").body(Body::empty()).unwrap())
    .await
    .unwrap();
assert_eq!(response.status(), StatusCode::OK);
let body = response.into_body();
let bytes = body.collect().await.unwrap().to_bytes();
let html = String::from_utf8(bytes.to_vec()).unwrap();

assert_eq!(html, "<h1>Hello, Foo!</h1>");
````

这段没有启动真实端口。  
它直接调用：

````rust
app().oneshot(...)
````

把请求喂给 Router，然后读取响应 body。

最后检查：

````rust
assert_eq!(html, "<h1>Hello, Foo!</h1>");
````

这验证了三件事：

1. 路由 `/greet/Foo` 能匹配。
2. `Path<String>` 能提取出 `Foo`。
3. 模板能正确渲染成 HTML。

## 函数职责速查

- `main`：初始化日志，创建 Router，绑定端口并启动服务。
- `app`：注册 `/greet/{name}` 路由。
- `greet`：提取路径参数，创建模板数据并返回 `HtmlTemplate`。
- `HtmlTemplate<T>::into_response`：渲染模板，成功返回 HTML，失败返回 500。
- `test_main`：直接调用 Router，验证模板渲染结果。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-templates
//! ```

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
    // 初始化日志。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 创建应用。
    let app = app();

    // 绑定端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动服务。
    axum::serve(listener, app).await.unwrap();
}

// 单独封装 Router，方便 main 和测试复用。
fn app() -> Router {
    Router::new().route("/greet/{name}", get(greet))
}

// 页面 handler。
async fn greet(extract::Path(name): extract::Path<String>) -> impl IntoResponse {
    // 准备模板需要的数据。
    let template = HelloTemplate { name };

    // 用 HtmlTemplate 包装，让它能变成 Axum 响应。
    HtmlTemplate(template)
}

// Askama 模板结构体。
// path = "hello.html" 对应 templates/hello.html。
#[derive(Template)]
#[template(path = "hello.html")]
struct HelloTemplate {
    // 模板里可以使用 {{ name }}。
    name: String,
}

// 一个通用包装器：把 Askama 模板变成 HTML 响应。
struct HtmlTemplate<T>(T);

impl<T> IntoResponse for HtmlTemplate<T>
where
    T: Template,
{
    fn into_response(self) -> Response {
        // render 会把模板和数据合成最终 HTML 字符串。
        match self.0.render() {
            // 渲染成功时，返回 text/html 响应。
            Ok(html) => Html(html).into_response(),
            // 渲染失败时，返回 500。
            Err(err) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                format!("Failed to render template. Error: {err}"),
            )
                .into_response(),
        }
    }
}
````

模板文件：

````html
<h1>Hello, {{ name }}!</h1>
````

## 运行和验证

运行：

````bash
cargo run -p example-templates
````

请求页面：

````bash
curl http://127.0.0.1:3000/greet/Foo
````

预期返回：

````html
<h1>Hello, Foo!</h1>
````

也可以在浏览器打开：

```text
http://127.0.0.1:3000/greet/Foo
```

运行测试：

````bash
cargo test -p example-templates
````

测试会验证 `/greet/Foo` 返回的 HTML 是否等于：

````html
<h1>Hello, Foo!</h1>
````

## 常见卡点

### 1. 模板文件为什么找不到？

Askama 默认会在 crate 的 `templates` 目录下找模板。  
这个 example 的模板路径是：

```text
examples/templates/templates/hello.html
```

不要把它放到 `src/templates` 里。

### 2. 为什么结构体字段名要和模板变量一致？

模板里写的是：

````html
{{ name }}
````

所以结构体里必须有：

````rust
name: String
````

字段名对不上，模板就无法正确编译或渲染。

### 3. 为什么要写 HtmlTemplate<T>？

因为 Axum 需要返回 HTTP 响应。  
Askama 模板只知道如何渲染字符串，不知道如何设置响应类型和错误状态码。

`HtmlTemplate<T>` 就是中间适配层。

### 4. 模板渲染失败应该返回什么？

这个 example 返回 500 和错误文本。  
真实项目里更常见的是：

```text
日志记录详细错误
用户看到通用错误页面
```

避免把内部路径、模板细节直接暴露给用户。

### 5. 什么时候用服务端模板，什么时候用前端框架？

简单后台、文档页、管理页、传统表单页面，用服务端模板很合适。  
交互复杂、状态很多的产品前端，可能更适合 React、Vue 等前端框架。

## 手写任务

建议按下面顺序自己敲一遍：

1. 新建 `templates/hello.html`，写 `<h1>Hello, {{ name }}!</h1>`。
2. 定义 `HelloTemplate { name: String }`。
3. 给它加 `#[derive(Template)]` 和 `#[template(path = "hello.html")]`。
4. 写 `HtmlTemplate<T>` 包装器。
5. 实现 `IntoResponse`，成功返回 `Html(html)`，失败返回 500。
6. 写 `/greet/{name}` 路由，从 Path 提取 name。
7. 用 curl 验证 HTML。

加深练习：

1. 把模板改成包含 `<title>` 和 `<body>` 的完整 HTML。
2. 给模板结构体增加 `items: Vec<String>`，练习 Askama 循环。
3. 新增一个 `/` 首页模板。

## 本章真正要记住什么

服务端模板的核心分工是：

```text
handler 准备数据
模板文件负责 HTML 结构
IntoResponse 包装器负责把渲染结果变成 HTTP 响应
```

在 Axum + Askama 里，最常见的结构是：

````rust
#[derive(Template)]
#[template(path = "...")]
struct PageTemplate { ... }

HtmlTemplate(PageTemplate { ... })
````

这比在 handler 里手写 HTML 字符串更清晰，也更适合页面逐渐变复杂的项目。

## 源码对照

本章手写版对应源码：

- `examples/templates/src/main.rs`
- `examples/templates/templates/hello.html`
- `examples/templates/Cargo.toml`
