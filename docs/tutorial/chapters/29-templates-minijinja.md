# 29. templates-minijinja

对应示例：`examples/templates-minijinja`

上一章用 Askama 渲染简单模板,这章换成 MiniJinja,并加入更接近真实网站的结构:多个页面共享同一个 layout。理解模板环境、共享布局、模板继承和 `State<Arc<AppState>>`。

## Cargo.toml

````toml
[package]
name = "example-templates-minijinja"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
minijinja = "2.3.1"
tokio = { version = "1.0", features = ["full"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

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

    env.add_template("layout", include_str!("../templates/layout.jinja"))
        .unwrap();
    env.add_template("home", include_str!("../templates/home.jinja"))
        .unwrap();
    env.add_template("content", include_str!("../templates/content.jinja"))
        .unwrap();
    env.add_template("about", include_str!("../templates/about.jinja"))
        .unwrap();

    let app_state = Arc::new(AppState { env });

    let app = Router::new()
        .route("/", get(handler_home))
        .route("/content", get(handler_content))
        .route("/about", get(handler_about))
        .with_state(app_state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
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

## templates/layout.jinja

````jinja
<!doctype html>
<html>
  <head><title>{% block title %}Website Name{% endblock %}</title></head>
  <body>
    <nav>
        <ul>
            <li><a href="/">Home</a></li>
            <li><a href="/content">Content</a></li>
            <li><a href="/about">About</a></li>
        </ul>
    </nav>
    {% block body %}{% endblock %}
  </body>
</html>
````

## templates/home.jinja(继承 layout)

````jinja
{% extends "layout" %}
{% block title %}{{ super() }} | {{ title }} {% endblock %}
{% block body %}
<h1>{{ title }}</h1>
<p>{{ welcome_text }}</p>
{% endblock %}
````

> `content.jinja` 和 `about.jinja` 见源码对照。

## 运行

````bash
cd examples
cargo run -p example-templates-minijinja
````

访问三个页面:

````bash
curl http://127.0.0.1:3000/
curl http://127.0.0.1:3000/content
curl http://127.0.0.1:3000/about
````

浏览器打开 `http://127.0.0.1:3000/` 点击导航链接。

## 解读

### Askama vs MiniJinja

```text
Askama:  给 Rust struct derive Template,struct 和模板文件一一对应(编译期)
MiniJinja: 程序启动时创建 Environment,注册多个模板,handler 按名字取模板渲染(运行期)
```

MiniJinja 更强调**模板环境**和**模板继承**,适合多个页面共享 layout/导航/页脚的场景。

### Environment + add_template

````rust
let mut env = Environment::new();
env.add_template("home", include_str!("../templates/home.jinja")).unwrap();
````

`Environment::new()` 创建空模板环境。`add_template(name, content)` 给模板起名并注册内容。`include_str!` 在**编译期**把文件内容嵌入二进制,运行时不需读磁盘(优点:部署简单;缺点:改模板需重新编译)。

模板名和文件名不必一致——`{% extends "layout" %}` 用的是注册名 `"layout"`,不是文件路径 `layout.jinja`。

### `State<Arc<AppState>>` 共享环境

````rust
struct AppState {
    env: Environment<'static>,
}

let app_state = Arc::new(AppState { env });
let app = Router::new()...with_state(app_state);
````

模板环境是应用级资源,启动时创建一次,所有 handler 共享。axum 并发处理请求,多个请求同时读同一环境,用 `Arc` 让多个 handler 共享同一份只读状态。handler 用 `State(state): State<Arc<AppState>>` 取出。

### 模板继承

`layout.jinja` 定义完整 HTML 结构,留两个 block:

````jinja
{% block title %}Website Name{% endblock %}
...
{% block body %}{% endblock %}
````

`home.jinja` 第一行 `{% extends "layout" %}` 继承 layout,然后覆盖 block:

````jinja
{% block title %}{{ super() }} | {{ title }} {% endblock %}
````

`super()` 保留父模板原内容,title 渲染成 `Website Name | Home`。所有页面共享 `<!doctype html>`/导航/body 外壳,只改 title 和 body。

### handler 三步渲染

````rust
async fn handler_home(State(state): State<Arc<AppState>>) -> Result<Html<String>, StatusCode> {
    let template = state.env.get_template("home").unwrap();
    let rendered = template.render(context! {
        title => "Home",
        welcome_text => "Hello World!",
    }).unwrap();
    Ok(Html(rendered))
}
````

1. 从 env 按名字取模板。
2. `context! { ... }` 传入模板变量(键名要和模板变量一致,否则模板拿不到数据)。
3. 渲染成字符串,返回 `Html(rendered)`。

多页面模式一致,只是 context 内容不同。content 页传 `entries: Vec<&str>`,模板用 `{% for data_entry in entries %}` 循环渲染。

### 错误处理边界

handler 返回 `Result<Html<String>, StatusCode>`,但内部用多个 `unwrap()`(模板注册在启动时,example 为突出主流程)。真实项目应把模板找不到/渲染失败转成 500 响应并记日志,也可像第 28 章封装统一模板响应类型。

## 常见问题

**为什么用 State?** handler 需要访问模板环境,环境是启动时创建的共享资源,用 `State<Arc<AppState>>` 注入。

**为什么用 Arc?** axum 并发处理请求,多请求同时读同一环境,`Arc` 让 handler 共享同一份只读状态。

**模板名为什么是 `layout` 不是 `layout.jinja`?** 注册时写的是什么名字,`{% extends %}` 就用什么。注册成 `"layout.jinja"` 模板里也要写 `{% extends "layout.jinja" %}`。

**`unwrap` 会不会崩溃?** 会,模板不存在或渲染失败会 panic。example 为突出主流程用 `unwrap`,真实项目应转成 500 响应并记日志。

## 手写任务

按下面顺序敲:

1. 写 `layout.jinja`,包含导航和 `title`、`body` 两个 block。
2. 写 `home.jinja`,用 `{% extends "layout" %}` 继承。
3. Rust 里创建 `Environment::new()`。
4. `add_template` 注册 layout 和 home。
5. 定义 `AppState { env }`,用 `Arc` 包起来。
6. `.with_state(app_state)` 挂到 Router。
7. handler 用 `State<Arc<AppState>>` 取状态。
8. `get_template`、`context!`、`render` 渲染页面。

加深练习:

1. 给 `content.jinja` 增加空列表分支。
2. 新增 `/contact` 页面共用同一 layout。
3. 把重复渲染逻辑封装成 `render_template(state, name, context)`。

## 小结

- MiniJinja 核心结构:`Environment` 保存模板 → `State` 让 handler 拿到 → `context!` 传数据 → `render` 生成 HTML → `Html` 返回响应。
- 模板环境是应用级共享资源,启动时创建一次,用 `State<Arc<AppState>>` 注入 handler。
- `include_str!` 编译期嵌入模板,部署简单但改模板需重新编译。
- 模板继承:`{% extends "layout" %}` + `{% block ... %}` 让多页面共享 layout/导航/页脚。
- `context!` 的键名要和模板变量一致,否则模板拿不到数据。

## 源码对照

- `examples/templates-minijinja/Cargo.toml`
- `examples/templates-minijinja/src/main.rs`
- `examples/templates-minijinja/templates/layout.jinja`
- `examples/templates-minijinja/templates/home.jinja`
- `examples/templates-minijinja/templates/content.jinja`
- `examples/templates-minijinja/templates/about.jinja`
