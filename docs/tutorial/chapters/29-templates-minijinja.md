# 29. templates-minijinja

对应示例：`examples/templates-minijinja`

前面章节 handler 都手写 HTML 字符串（`format!("<h1>...</h1>")`）。真实项目用**模板引擎**分离 HTML 和业务代码。这章用 **MiniJinja**——Jinja2 语法的 Rust 实现，轻量（不像 Askama 要编译期生成代码）。

实现一个三页面小 demo（home / content / about），共享同一 layout（导航栏），理解模板继承、context、`include_str!` 嵌入模板。

分 3 步：先建 Environment 加载模板，再写 home handler 渲染，最后扩展到多页面共享 layout。

相比前面章节新引入：**`minijinja` crate、`Environment`（模板注册表）、`context!` 宏、`include_str!` 编译期嵌入模板、模板继承（`extends`）**。

## Cargo.toml

````toml
[package]
name = "example-templates-minijinja"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
minijinja = "2"
tokio = { version = "1.0", features = ["full"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：建 `Environment` 加载模板

MiniJinja 用 `Environment` 管理所有模板（注册一次，多次渲染）。这步建 Environment，加载 home 模板，写第一个 handler 渲染它。

````rust
use axum::extract::State;
use axum::http::StatusCode;
use axum::{response::Html, routing::get, Router};
use minijinja::{context, Environment};
use std::sync::Arc;

struct AppState {
    env: Environment<'static>,
}

#[tokio::main]
async fn main() {
    let mut env = Environment::new();
    // 编译期把模板文件嵌入二进制
    env.add_template("home", include_str!("../templates/home.jinja"))
        .unwrap();

    let app_state = Arc::new(AppState { env });

    let app = Router::new()
        .route("/", get(handler_home))
        .with_state(app_state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler_home(State(state): State<Arc<AppState>>) -> Result<Html<String>, StatusCode> {
    let template = state.env.get_template("home").unwrap();
    let rendered = template
        .render(context! {
            title => "Home",
            welcome_text => "Hello World!",
        })
        .unwrap();
    Ok(Html(rendered))
}
````

`templates/home.jinja`：

````jinja
<!DOCTYPE html>
<html>
<head><title>{{ title }}</title></head>
<body>
<h1>{{ title }}</h1>
<p>{{ welcome_text }}</p>
</body>
</html>
````

验证：

````bash
cd examples
cargo run -p example-templates-minijinja

curl http://127.0.0.1:3000/
# <html>...<h1>Home</h1><p>Hello World!</p>...</html>
````

> **新面孔：`minijinja::Environment`**
>
> 模板注册表。启动时 `env.add_template("name", source)` 把模板源代码注册进去（编译一次），handler 用 `env.get_template("name")` 取出已编译的模板。
>
> Environment 设计成"一次注册多次渲染"——编译开销只付一次。

> **新面孔：`include_str!`**
>
> Rust 标准库宏，编译期把文件内容读成 `&'static str`。`include_str!("../templates/home.jinja")` 把模板文件嵌入二进制——部署时不需要带 templates 目录。
>
> 注意路径相对 `.rs` 文件，不是 Cargo.toml。

> **新面孔：`context!` 宏**
>
> MiniJinja 的便捷宏，构造模板变量字典。语法 `context! { key => value, ... }`，模板里用 `{{ key }}` 取值。

> **新面孔：`State<Arc<AppState>>` 共享模板**
>
> `Environment` 不是 `Sync + Clone` 友好的，用 `Arc<AppState>` 包一层共享。每个 handler 拿到 `Arc` 引用，零成本 clone。
>
> 和 ch33 的 `Arc<Pool>` 同构——把"启动时建一次、运行时只读"的资源塞进 State。

---

## 第二步：共享 layout（模板继承）

加 `layout.jinja` 作为公共布局（导航栏 + 占位符），其他模板 `extends layout` 填内容。这样所有页面共享同样的头部/底部，改 layout 一处生效。

`templates/layout.jinja`：

````jinja
<!DOCTYPE html>
<html>
<head><title>{{ title }}</title></head>
<body>
<nav>
  <a href="/">Home</a> |
  <a href="/content">Content</a> |
  <a href="/about">About</a>
</nav>
<hr>
{% block content %}{% endblock %}
</body>
</html>
````

`templates/home.jinja`（改成继承 layout）：

````jinja
{% extends "layout" %}
{% block content %}
<h1>{{ title }}</h1>
<p>{{ welcome_text }}</p>
{% endblock %}
````

main 里注册所有模板：

````rust
# let mut env = Environment::new();
env.add_template("layout", include_str!("../templates/layout.jinja")).unwrap();
env.add_template("home", include_str!("../templates/home.jinja")).unwrap();
env.add_template("content", include_str!("../templates/content.jinja")).unwrap();
env.add_template("about", include_str!("../templates/about.jinja")).unwrap();
````

> **新面孔：Jinja 模板继承（`extends` + `block`）**
>
> Jinja 家族（Jinja2、MiniJinja、Tera）的招牌特性：base 模板定义 `{% block name %}...{% endblock %}`，child 模板 `{% extends "base" %}` 后用同名 block 覆盖。
>
> 渲染 child 时会先渲染 base，遇到 block 用 child 的内容填进去。这章所有页面共享 layout 的导航栏，只覆盖 content block。

---

## 第三步：多 handler + 复杂 context

加 content 和 about 两个 handler。content handler 演示**列表渲染**（`{% for %}` 循环）。

````rust
async fn handler_content(State(state): State<Arc<AppState>>) -> Result<Html<String>, StatusCode> {
    let template = state.env.get_template("content").unwrap();

    let some_example_entries = vec!["Data 1", "Data 2", "Data 3"];

    let rendered = template
        .render(context! {
            title => "Content",
            entries => some_example_entries,
        })
        .unwrap();

    Ok(Html(rendered))
}

async fn handler_about(State(state): State<Arc<AppState>>) -> Result<Html<String>, StatusCode> {
    let template = state.env.get_template("about").unwrap();

    let rendered = template.render(context! {
        title => "About",
        about_text => "Simple demonstration layout for an axum project with minijinja as templating engine.",
    }).unwrap();

    Ok(Html(rendered))
}

# #[tokio::main]
# async fn main() {
#     // ...
    let app = Router::new()
        .route("/", get(handler_home))
        .route("/content", get(handler_content))
        .route("/about", get(handler_about))
        .with_state(app_state);
#     // ...
# }
````

`templates/content.jinja`：

````jinja
{% extends "layout" %}
{% block content %}
<h1>{{ title }}</h1>
<ul>
{% for entry in entries %}
<li>{{ entry }}</li>
{% endfor %}
</ul>
{% endblock %}
````

> **新面孔：Jinja 循环 `{% for %}`**
>
> `context! { entries => vec![...] }` 传 Vec 进模板，`{% for entry in entries %}` 遍历。Rust 的 Vec/HashMap 都能直接传，MiniJinja 自动桥接。

> **新面孔：handler 返回 `Result<Html<String>, StatusCode>`**
>
> 模板渲染可能失败（变量缺失、语法错），用 `Result` 处理。`StatusCode::INTERNAL_SERVER_ERROR` 作为错误类型——模板渲染失败时直接返回 500（`StatusCode` 实现了 `IntoResponse`）。

验证：

````bash
curl http://127.0.0.1:3000/         # Home 页
curl http://127.0.0.1:3000/content  # Content 页（带列表）
curl http://127.0.0.1:3000/about    # About 页
````

三个页面顶部都有同样的导航栏（来自 layout），下面是各自的内容（来自 child template 的 content block）。

---

## 完整代码

````rust
use axum::extract::State;
use axum::http::StatusCode;
use axum::{response::Html, routing::get, Router};
use minijinja::{context, Environment};
use std::sync::Arc;

struct AppState {
    env: Environment<'static>,
}

#[tokio::main]
async fn main() {
    // init template engine and add templates
    let mut env = Environment::new();
    env.add_template("layout", include_str!("../templates/layout.jinja"))
        .unwrap();
    env.add_template("home", include_str!("../templates/home.jinja"))
        .unwrap();
    env.add_template("content", include_str!("../templates/content.jinja"))
        .unwrap();
    env.add_template("about", include_str!("../templates/about.jinja"))
        .unwrap();

    // pass env to handlers via state
    let app_state = Arc::new(AppState { env });

    // define routes
    let app = Router::new()
        .route("/", get(handler_home))
        .route("/content", get(handler_content))
        .route("/about", get(handler_about))
        .with_state(app_state);

    // run it
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}

async fn handler_home(State(state): State<Arc<AppState>>) -> Result<Html<String>, StatusCode> {
    let template = state.env.get_template("home").unwrap();

    let rendered = template
        .render(context! {
            title => "Home",
            welcome_text => "Hello World!",
        })
        .unwrap();

    Ok(Html(rendered))
}

async fn handler_content(State(state): State<Arc<AppState>>) -> Result<Html<String>, StatusCode> {
    let template = state.env.get_template("content").unwrap();

    let some_example_entries = vec!["Data 1", "Data 2", "Data 3"];

    let rendered = template
        .render(context! {
            title => "Content",
            entries => some_example_entries,
        })
        .unwrap();

    Ok(Html(rendered))
}

async fn handler_about(State(state): State<Arc<AppState>>) -> Result<Html<String>, StatusCode> {
    let template = state.env.get_template("about").unwrap();

    let rendered = template.render(context!{
        title => "About",
        about_text => "Simple demonstration layout for an axum project with minijinja as templating engine.",
    }).unwrap();

    Ok(Html(rendered))
}
````

## 运行

````bash
cd examples
cargo run -p example-templates-minijinja
````

浏览器访问 `http://127.0.0.1:3000/`，点导航栏切换三个页面，观察 layout（导航）保持不变、content 区域变化。

## 解读

### Rust 模板引擎对比

| 引擎 | 特点 | 类比 |
| --- | --- | --- |
| **MiniJinja**（这章） | 运行时编译，Jinja2 语法 | Python Flask 默认 |
| **Askama**（ch28） | 编译期生成 Rust 代码，类型安全 | Rust 类型系统深度融合 |
| **Tera** | 运行时，Jinja2 语法 | MiniJinja 的老大哥 |
| **Maud** | 用 Rust 宏写 HTML，不是模板 | HTML-as-code |

MiniJinja 比 Askama 灵活（模板可在运行时加载/修改），Askama 性能更好（编译期优化）。这章选 MiniJinja 因为它简单。

## 常见问题

**MiniJinja 性能怎么样？** 每次渲染都解释执行模板，比 Askama 慢。但对绝大多数 web 应用，模板渲染不是瓶颈（数据库/网络才是）。

**模板能热重载吗？** `Environment::new()` 是编译期加载的（`include_str!`）。运行时加载用 `env.set_source(Source::from_path("templates/"))`，文件改动重启才生效。

**XSS 怎么防？** MiniJinja 默认对 `{{ var }}` 自动 HTML escape。`{{ var|safe }}` 标记为安全不 escape（慎用）。

## 手写任务

1. 加 `/user/{id}` 路由用模板显示用户信息（用 `Path` extractor + template）。
2. 加 footer 模板，所有页面共享（在 layout 里 `{% include "footer" %}`）。
3. 模板里用条件 `{% if user %}...{% else %}...{% endif %}`，根据登录状态显示不同内容。
4. 把 MiniJinja 换成 Askama，对比代码差异。

## 小结

这章用 3 步讲了 MiniJinja 模板引擎：

1. **Environment 加载模板**：启动时 `env.add_template()`，`include_str!` 嵌入文件，`Arc<AppState>` 共享。
2. **layout 继承**：`{% extends %}` + `{% block %}` 让所有页面共享导航/页脚。
3. **多 handler + 循环**：`context! { entries => vec }` 传列表，`{% for %}` 遍历。

核心：模板引擎分离 HTML 和业务代码；MiniJinja 用 Jinja2 语法（继承、循环、条件）；`Environment` 是模板注册表，启动注册一次运行时多次渲染。

## 源码对照

- `examples/templates-minijinja/Cargo.toml`
- `examples/templates-minijinja/src/main.rs`
- `examples/templates-minijinja/templates/layout.jinja`
- `examples/templates-minijinja/templates/home.jinja`
- `examples/templates-minijinja/templates/content.jinja`
- `examples/templates-minijinja/templates/about.jinja`
