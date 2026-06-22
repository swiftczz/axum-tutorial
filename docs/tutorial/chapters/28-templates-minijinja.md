# 28. templates-minijinja

对应示例：`examples/templates-minijinja`

本章目标：使用 MiniJinja 在 Axum 中渲染多页面 HTML，理解模板环境、共享布局、模板继承和 `State<Arc<AppState>>`。

上一章用 Askama 渲染了一个简单模板。  
这一章换成 MiniJinja，并加入更接近真实网站的结构：多个页面共享同一个 layout。

## 这个小项目在做什么

应用有三个页面：

```text
GET /        -> Home
GET /content -> Content
GET /about   -> About
```

它们都共享同一个基础布局：

```text
layout.jinja
```

每个页面只填自己的标题和正文内容：

```text
home.jinja
content.jinja
about.jinja
```

请求主线是：

```text
程序启动
-> 创建 MiniJinja Environment
-> 把 layout/home/content/about 模板加入 Environment
-> 用 AppState 保存 Environment
-> Router 通过 with_state 共享 AppState
-> handler 从 State 里取模板
-> render(context! { ... })
-> 返回 Html<String>
```

## Askama 和 MiniJinja 的主要区别

上一章 Askama 的写法是：

```text
给 Rust struct derive Template
struct 和模板文件一一对应
```

这一章 MiniJinja 的写法是：

```text
程序启动时创建 Environment
把多个模板注册进去
handler 按名字取模板并渲染
```

可以先这样理解：

```text
Askama 更偏编译期模板结构
MiniJinja 更偏运行时模板环境
```

MiniJinja 很适合多个模板共享一个环境，并使用继承、block、循环等 Jinja 风格语法。

## 文件和依赖

这个 example 有六个主要文件：

1. `examples/templates-minijinja/Cargo.toml`：声明 Axum、MiniJinja、Tokio。
2. `examples/templates-minijinja/src/main.rs`：创建模板环境、共享 state、注册路由。
3. `examples/templates-minijinja/templates/layout.jinja`：基础 HTML 布局。
4. `examples/templates-minijinja/templates/home.jinja`：首页模板。
5. `examples/templates-minijinja/templates/content.jinja`：内容页模板。
6. `examples/templates-minijinja/templates/about.jinja`：关于页模板。

关键依赖：

- `axum`：提供 Router、State extractor、Html response。
- `minijinja`：模板环境、模板渲染和 `context!` 宏。
- `tokio`：异步运行时。

`Cargo.toml`：

````toml
minijinja = "2.3.1"
````

## 第一步：定义 AppState

源码：

````rust
struct AppState {
    env: Environment<'static>,
}
````

`AppState` 保存应用共享状态。  
这一章只有一个字段：

```text
env
```

它是 MiniJinja 的模板环境。  
所有 handler 都要从这个环境里取模板。

为什么不在每个 handler 里重新创建 `Environment`？

因为模板环境是应用级资源。  
启动时创建一次，然后所有请求共享它更合理。

## 第二步：创建模板环境

源码：

````rust
let mut env = Environment::new();
env.add_template("layout", include_str!("../templates/layout.jinja"))
    .unwrap();
env.add_template("home", include_str!("../templates/home.jinja"))
    .unwrap();
env.add_template("content", include_str!("../templates/content.jinja"))
    .unwrap();
env.add_template("about", include_str!("../templates/about.jinja"))
    .unwrap();
````

`Environment::new()` 创建一个空模板环境。

`add_template` 做两件事：

1. 给模板起一个名字，比如 `"home"`。
2. 把模板内容注册进环境。

`include_str!` 是 Rust 宏。  
它会在编译时把文件内容嵌进二进制里：

````rust
include_str!("../templates/home.jinja")
````

所以运行程序时，不需要再动态读取这个文件。

## 第三步：用 Arc 共享 AppState

源码：

````rust
let app_state = Arc::new(AppState { env });
````

Axum 服务会同时处理多个请求。  
多个 handler 都要读取同一个模板环境。

`Arc` 是原子引用计数指针，可以让多个任务安全共享同一份只读状态。

这里可以先记住：

```text
需要在多个请求 handler 之间共享应用状态时，经常会用 Arc<AppState>
```

## 第四步：把 state 交给 Router

源码：

````rust
let app = Router::new()
    .route("/", get(handler_home))
    .route("/content", get(handler_content))
    .route("/about", get(handler_about))
    .with_state(app_state);
````

`.with_state(app_state)` 把共享状态挂到 Router 上。

之后 handler 就可以用：

````rust
State(state): State<Arc<AppState>>
````

取出这份状态。

这和前面章节的 extractor 思想一致：

```text
Path 提取路径参数
Json 提取请求体 JSON
State 提取应用共享状态
```

## 第五步：基础布局 layout.jinja

模板内容：

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

这个模板定义了完整 HTML 结构。  
其中有两个 block：

```text
title
body
```

block 可以理解成“给子模板填内容的位置”。

所有页面都共享：

- `<!doctype html>`
- `<html>`
- `<head>`
- 导航菜单
- `<body>`

各页面只需要改 title 和 body。

## 第六步：首页模板继承 layout

`home.jinja`：

````jinja
{% extends "layout" %}
{% block title %}{{ super() }} | {{ title }} {% endblock %}
{% block body %}
<h1>{{ title }}</h1>
<p>{{ welcome_text }}</p>
{% endblock %}
````

第一行：

````jinja
{% extends "layout" %}
````

表示这个模板继承 `layout`。

`block title` 覆盖 layout 里的 title block：

````jinja
{% block title %}{{ super() }} | {{ title }} {% endblock %}
````

`super()` 表示保留父模板原来的内容。  
所以 title 会变成类似：

```text
Website Name | Home
```

`block body` 填页面正文：

````jinja
<h1>{{ title }}</h1>
<p>{{ welcome_text }}</p>
````

这些变量来自 handler 传入的 context。

## 第七步：首页 handler

源码：

````rust
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

这段逻辑分三步：

1. 从 state 里的环境取出 `"home"` 模板。
2. 用 `context!` 传入模板变量。
3. 渲染成字符串，并返回 `Html(rendered)`。

`context!` 里的键名要和模板变量一致：

```text
title
welcome_text
```

否则模板里拿不到对应数据。

## 第八步：content 页面使用循环

handler 源码：

````rust
let some_example_entries = vec!["Data 1", "Data 2", "Data 3"];

let rendered = template
    .render(context! {
        title => "Content",
        entries => some_example_entries,
    })
    .unwrap();
````

模板：

````jinja
{% for data_entry in entries %}
<ul>
    <li>{{ data_entry }}</li>
</ul>
{% endfor %}
````

`entries` 是 Rust 传给模板的列表。  
模板用 `for` 循环逐个渲染。

最终页面会出现：

```text
Data 1
Data 2
Data 3
```

这就是模板里常见的列表渲染。

## 第九步：about 页面传普通文本

handler 源码：

````rust
let rendered = template.render(context!{
    title => "About",
    about_text => "Simple demonstration layout for an axum project with minijinja as templating engine.",
}).unwrap();
````

模板：

````jinja
<h1>{{ title }}</h1>
<p>{{ about_text }}</p>
````

这一页和首页很像，只是变量名不同。

这说明多页面模板的基本模式是一样的：

```text
get_template(...)
render(context! { ... })
Html(rendered)
```

## 第十步：这个示例的错误处理边界

handler 返回类型是：

````rust
Result<Html<String>, StatusCode>
````

但源码内部使用了多个：

````rust
.unwrap()
````

例如：

````rust
state.env.get_template("home").unwrap()
template.render(...).unwrap()
````

这在 example 里可以接受，因为模板都在启动时注册好了，代码重点是展示 MiniJinja 用法。

真实项目里更稳妥的写法是：

```text
模板找不到 -> 记录日志，返回 500
模板渲染失败 -> 记录日志，返回 500
```

也可以像上一章一样封装一个统一的模板响应类型，集中处理渲染错误。

## 函数职责速查

- `main`：创建 MiniJinja 环境，注册模板，创建共享状态，注册路由并启动服务。
- `handler_home`：渲染首页模板。
- `handler_content`：渲染内容页模板，并传入列表数据。
- `handler_about`：渲染关于页模板。
- `AppState`：保存应用共享的模板环境。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//!
//! Run with
//!
//! ```not_rust
//! cargo run -p example-templates-minijinja
//! ```
//! 这个 demo 使用 MiniJinja 模板引擎。
//! 三个页面共享同一个 layout 和导航菜单。

use axum::extract::State;
use axum::http::StatusCode;
use axum::{response::Html, routing::get, Router};
use minijinja::{context, Environment};
use std::sync::Arc;

// 应用共享状态。
// env 保存所有已经注册的 MiniJinja 模板。
struct AppState {
    env: Environment<'static>,
}

#[tokio::main]
async fn main() {
    // 创建模板环境。
    let mut env = Environment::new();

    // 注册基础布局模板。
    env.add_template("layout", include_str!("../templates/layout.jinja"))
        .unwrap();

    // 注册三个页面模板。
    env.add_template("home", include_str!("../templates/home.jinja"))
        .unwrap();
    env.add_template("content", include_str!("../templates/content.jinja"))
        .unwrap();
    env.add_template("about", include_str!("../templates/about.jinja"))
        .unwrap();

    // 用 Arc 包住共享状态，让多个请求 handler 可以共享同一个模板环境。
    let app_state = Arc::new(AppState { env });

    // 注册路由，并把共享状态挂到 Router 上。
    let app = Router::new()
        .route("/", get(handler_home))
        .route("/content", get(handler_content))
        .route("/about", get(handler_about))
        .with_state(app_state);

    // 启动服务。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

// 首页 handler。
async fn handler_home(State(state): State<Arc<AppState>>) -> Result<Html<String>, StatusCode> {
    // 从模板环境中按名字取出 home 模板。
    let template = state.env.get_template("home").unwrap();

    // 传入模板变量并渲染。
    let rendered = template
        .render(context! {
            title => "Home",
            welcome_text => "Hello World!",
        })
        .unwrap();

    // 返回 text/html 响应。
    Ok(Html(rendered))
}

// 内容页 handler。
async fn handler_content(State(state): State<Arc<AppState>>) -> Result<Html<String>, StatusCode> {
    let template = state.env.get_template("content").unwrap();

    // 准备一个列表，交给模板里的 for 循环渲染。
    let some_example_entries = vec!["Data 1", "Data 2", "Data 3"];

    let rendered = template
        .render(context! {
            title => "Content",
            entries => some_example_entries,
        })
        .unwrap();

    Ok(Html(rendered))
}

// 关于页 handler。
async fn handler_about(State(state): State<Arc<AppState>>) -> Result<Html<String>, StatusCode> {
    let template = state.env.get_template("about").unwrap();

    let rendered = template.render(context!{
        title => "About",
        about_text => "Simple demonstration layout for an axum project with minijinja as templating engine.",
    }).unwrap();

    Ok(Html(rendered))
}
````

基础布局模板：

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

首页模板：

````jinja
{% extends "layout" %}
{% block title %}{{ super() }} | {{ title }} {% endblock %}
{% block body %}
<h1>{{ title }}</h1>
<p>{{ welcome_text }}</p>
{% endblock %}
````

## 运行和验证

运行：

````bash
cargo run -p example-templates-minijinja
````

访问首页：

````bash
curl http://127.0.0.1:3000/
````

访问内容页：

````bash
curl http://127.0.0.1:3000/content
````

访问关于页：

````bash
curl http://127.0.0.1:3000/about
````

也可以用浏览器打开：

```text
http://127.0.0.1:3000/
```

然后点击页面里的导航链接。

## 常见卡点

### 1. 为什么要用 State？

因为 handler 需要访问模板环境。  
模板环境是应用启动时创建的共享资源，所以用 `State<Arc<AppState>>` 注入到 handler。

### 2. 为什么要用 Arc？

Axum 会并发处理请求。  
多个请求可能同时读取同一个模板环境，`Arc` 可以让这些 handler 共享同一份状态。

### 3. include_str! 和读取文件有什么区别？

`include_str!` 在编译时把模板内容嵌入程序。  
运行时不需要再从磁盘读取模板文件。

优点是部署简单。  
缺点是改模板后需要重新编译程序。

### 4. 为什么模板名是 layout，不是 layout.jinja？

因为注册模板时写的是：

````rust
env.add_template("layout", include_str!("../templates/layout.jinja"))
````

MiniJinja 里 `{% extends "layout" %}` 使用的是注册名，不是文件路径。

如果你注册成 `"layout.jinja"`，模板里也要写：

````jinja
{% extends "layout.jinja" %}
````

### 5. unwrap 会不会导致服务崩溃？

会。  
如果模板不存在或渲染失败，`unwrap()` 会 panic。

example 为了突出 MiniJinja 主流程用了 `unwrap()`。  
真实项目应该把错误转成 500 响应，并记录日志。

## 手写任务

建议按下面顺序自己敲一遍：

1. 写 `layout.jinja`，包含导航和 `title`、`body` 两个 block。
2. 写 `home.jinja`，用 `{% extends "layout" %}` 继承 layout。
3. 在 Rust 里创建 `Environment::new()`。
4. 用 `add_template` 注册 layout 和 home。
5. 定义 `AppState { env }`，并用 `Arc` 包起来。
6. 用 `.with_state(app_state)` 把状态挂到 Router。
7. 在 handler 里用 `State<Arc<AppState>>` 取状态。
8. 用 `get_template`、`context!`、`render` 渲染页面。

加深练习：

1. 给 `content.jinja` 增加一个空列表分支。
2. 新增 `/contact` 页面，共用同一个 layout。
3. 把重复的渲染逻辑封装成 `render_template(state, name, context)`。

## 本章真正要记住什么

MiniJinja 的核心结构是：

```text
Environment 保存模板
State 让 handler 拿到 Environment
context! 传入页面数据
render 生成 HTML 字符串
Html 返回 text/html 响应
```

相比上一章 Askama，MiniJinja 更强调模板环境和模板继承。  
当你有多个页面共享 layout、导航、页脚时，这种结构会更自然。

## 源码对照

本章手写版对应源码：

- `examples/templates-minijinja/src/main.rs`
- `examples/templates-minijinja/templates/layout.jinja`
- `examples/templates-minijinja/templates/home.jinja`
- `examples/templates-minijinja/templates/content.jinja`
- `examples/templates-minijinja/templates/about.jinja`
- `examples/templates-minijinja/Cargo.toml`
