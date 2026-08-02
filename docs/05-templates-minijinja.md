# 第 05 章 templates-minijinja:运行期模板、模板继承与共享状态

## 本章你将做出什么

一个三页的小网站:首页、内容页、关于页,共享同一套布局和导航菜单。模板引擎换成 **minijinja**——运行期方案,和[第 04 章](04-templates.md)的 askama 形成对照。

本章的重点是两件事:**模板继承**(布局复用)和 **`State` 提取器**(共享应用状态)。后者是 axum 里和提取器同等重要的机制,本章第一次用到它。

学完本章你将掌握:

- 运行期模板引擎的取舍,minijinja 与 askama 的具体差别
- 模板继承:`{% extends %}`、`{% block %}`、`super()` 如何组合出布局复用
- `State` 提取器:为什么需要共享状态、`Arc` 的作用、`with_state` 做了什么
- `include_str!` 如何把模板文件嵌入二进制

## 前置知识

### 运行期模板引擎的取舍

[第 04 章](04-templates.md#askama编译期模板引擎)对比过两派方案,这里把 minijinja 这一侧说清楚。

**minijinja 在运行时加载和解析模板。** 你在程序启动时把模板字符串注册进一个 `Environment`,渲染时按名字取出来用。这带来几个具体差别:

**模板错误在运行时才暴露。** askama 里模板引用了不存在的字段会编译失败;minijinja 里这只会在渲染那一刻报错(或者按配置静默输出空值)。所以 minijinja 项目更依赖测试覆盖。

**模板可以动态来源。** 既然是运行时注册,模板就不必来自源码目录——可以从数据库读、从远程配置拉、由用户上传。askama 做不到这些。

**支持热重载。** 开发时改模板不用重新编译,配合文件监听可以做到保存即生效。askama 改模板必须 `cargo build`。

**语法更完整。** minijinja 相当完整地实现了 Jinja2 语法,askama 只支持一个子集。本章要用的模板继承,两者都支持,但 minijinja 的行为更贴近 Jinja2 原版。

选哪个取决于项目:模板固定、追求编译期保障用 askama;模板需要灵活加载、或者团队熟悉 Jinja2 用 minijinja。

有意思的是本章示例做了个折中:用 `include_str!` 在编译期把模板文件读成字符串常量,再在运行时注册给 minijinja。这样模板文件仍然是源码的一部分(文件不存在会编译失败、部署时不用额外拷贝模板目录),但解析和渲染仍是运行时行为。**它拿到了"模板文件必须存在"这个编译期保障,但没拿到"模板内容正确"的保障。** 后面会详细讲这个取舍。

### 模板继承:布局复用

三个页面共享导航菜单和 `<html>` 骨架,内容各不相同。如果每个模板都写一遍完整的 HTML,改导航就要改三个文件——这是典型的重复。

Jinja2 系模板引擎用**继承**解决它。先写一个**布局模板**(layout),把公共部分固定下来,把会变化的地方开出**占位块**(block):

```html
<!doctype html>
<html>
  <head><title>{% block title %}Website Name{% endblock %}</title></head>
  <body>
    <nav>...</nav>
    {% block body %}{% endblock %}
  </body>
</html>
```

`{% block title %}` 到 `{% endblock %}` 之间的内容是**默认值**——子模板不覆盖就用它。`{% block body %}{% endblock %}` 里是空的,意思是子模板必须自己填。

然后每个页面模板声明继承,只填自己关心的块:

```html
{% extends "layout" %}
{% block body %}
<h1>{{ title }}</h1>
{% endblock %}
```

渲染子模板时,引擎实际渲染的是布局模板,只是把里面的 block 替换成子模板提供的版本。所以子模板里 `{% block %}` 之外的内容会被完全忽略——写了也不会输出。

`super()` 用来在覆盖块的同时保留父模板的内容:

```html
{% block title %}{{ super() }} | {{ title }} {% endblock %}
```

这会输出"父模板的 title 块内容 + ` | ` + 本页标题",也就是 `Website Name | Home`。这是网页标题的常见做法——站名固定,页名变化。

### 为什么需要共享状态

到目前为止 handler 都是自给自足的:参数从请求里提取,返回值凭空构造,不依赖任何外部东西。

但真实服务几乎总要访问某些**启动时创建一次、之后被所有请求共享**的资源:

- 数据库连接池
- Redis 客户端
- 配置对象
- 缓存
- **本章的模板引擎 `Environment`**

这些东西的共同点是初始化有成本(建连接、解析模板),不能每个请求重做一遍。它们必须在 `main` 里创建一次,然后让所有 handler 都能拿到。

问题是 handler 是普通函数,axum 调用它时不会传额外参数。怎么把 `main` 里的东西送进去?

Rust 的常规答案是全局变量(`static` 加 `OnceLock` 或 `lazy_static`),但那样会让 handler 隐式依赖全局状态,测试时无法替换,类型上也看不出依赖关系。

**axum 的答案是把状态挂到 `Router` 上,再用 `State<S>` 提取器取出。**

### State 与 with_state

分两步。第一步,构建 `Router` 时用 `with_state` 把状态交给它:

```rust
let app = Router::new()
    .route("/", get(handler))
    .with_state(app_state);
```

第二步,handler 用 `State` 提取器声明"我需要这个状态":

```rust
async fn handler(State(state): State<Arc<AppState>>) -> ... {
    // 用 state
}
```

`State<S>` 属于 `FromRequestParts` 类别——不碰 body,可以和其他提取器任意搭配。

**类型层面发生了什么。** [第 02 章](02-routes-and-handlers-close-together.md#methodrouters-里的-s-是什么)讲过 `Router<S>` 的泛型参数 `S` 就是状态类型,当时 `S = ()` 表示无状态。现在:

- `Router::new()` 起初是 `Router<S>`,`S` 待定
- 挂上要求 `State<Arc<AppState>>` 的 handler 后,编译器推断出 `S = Arc<AppState>`,类型变成 `Router<Arc<AppState>>`
- `with_state(app_state)` 把实际的状态值填进去,返回 `Router<()>`

**只有 `Router<()>` 能交给 `axum::serve`。** 这是个漂亮的类型级保证:如果你的 handler 需要状态但忘了调用 `with_state`,`Router` 的类型停留在 `Router<Arc<AppState>>`,`serve` 那一行编译不过。**"忘记提供状态"这个错误在编译期就被挡住了**,不会变成运行时 panic。

顺带说个实用提示:这类错误的编译信息有时不太好读,因为 `Router` 的泛型和 `Handler` 的 trait bound 缠在一起。看到一长串 `Handler<...>` 不满足的报错,先检查是不是漏了 `with_state`。

**为什么要 `Arc`。** `State<S>` 要求 `S: Clone`——axum 为每个请求克隆一份状态传给 handler。如果 `S` 是 `AppState` 本身,每个请求都会深拷贝整个结构(本章是把模板引擎整体复制一遍),开销无法接受。

`Arc<T>`(原子引用计数)的 `Clone` 只是把计数器加一,几乎零成本,克隆出来的所有副本指向同一份数据。所以 `Arc<AppState>` 既满足 `Clone` 又避免了拷贝。

`Arc` 提供的是**共享只读**访问。本章的模板引擎注册完就不再修改,只读正合适。

> **本章不深入**:需要**共享可变**状态时(比如所有请求都能写的计数器或缓存),要在 `Arc` 里再套一层锁,写成 `Arc<RwLock<T>>` 或 `Arc<Mutex<T>>`。本章状态只读,用不到锁。锁的选择、持锁跨 `await` 的坑、`Clone` 语义的更多细节见[第 22 章](22-key-value-store.md)。

### include_str! 把文件内容嵌进二进制

`include_str!("../templates/layout.jinja")` 是标准库宏,在**编译期**把指定文件的内容读成一个 `&'static str` 常量嵌入二进制。

路径相对于**当前源文件**(不是项目根目录),所以从 `src/main.rs` 指向 `templates/` 要写 `../templates/`。这和 askama 的 `path = "hello.html"`(相对于项目根的 `templates/` 目录)不同,容易记混。

它带来两个好处:

- **文件不存在会编译失败**,不会等到运行时才发现模板丢了
- **部署只需要一个二进制文件**,不用把 `templates/` 目录一起拷过去

代价是失去了热重载能力——模板内容已经固化在二进制里,改文件必须重新编译。想要热重载得改成运行时 `std::fs::read_to_string`,那样又会把"文件必须存在"的检查推迟到运行时。这就是前置知识开头说的那个折中。

## 项目搭建

```bash
cargo new templates-minijinja
cd templates-minijinja
```

依赖比前两章少——本章示例用 `println!` 而不是 `tracing`:

```bash
cargo add axum
cargo add tokio --features full
cargo add minijinja
```

`minijinja` 不需要额外 feature。它默认就支持模板继承。

建模板目录,四个文件在步骤 1 和步骤 2 创建:

```bash
mkdir templates
```

> 和[第 04 章](04-templates.md#项目搭建)一样,本章渲染 HTML 页面但不涉及 CSS/JS 等静态资源的托管。导航菜单是裸 `<ul>` 列表,样式欠佳是正常的。静态资源见[第 38 章](38-static-file-server.md)。

## 分步构建

### 步骤 1:布局模板 + 首页,认识 State

**要做什么**:写出布局模板和首页模板,在 `main` 里初始化模板引擎并通过 `State` 交给 handler。这一步会打通全链路,只有一个页面。

先写布局模板:

```html
<!-- templates/layout.jinja -->
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
```

导航菜单里已经写好了三个链接,后两个要到步骤 2 才能访问。`title` 块有默认值 `Website Name`,`body` 块是空的。

再写首页模板:

```html
<!-- templates/home.jinja -->
{% extends "layout" %}
{% block title %}{{ super() }} | {{ title }} {% endblock %}
{% block body %}
<h1>{{ title }}</h1>
<p>{{ welcome_text }}</p>
{% endblock %}
```

注意 `{% extends "layout" %}` 里的 `"layout"` 是**注册时用的名字**,不是文件名——下面 `add_template("layout", ...)` 的第一个参数。minijinja 不知道文件系统的存在,它只认注册过的名字。

`{{ super() }} | {{ title }}` 会渲染成 `Website Name | Home`。

现在写 Rust 代码:

```rust
// src/main.rs
use axum::extract::State;
use axum::http::StatusCode;
use axum::{response::Html, routing::get, Router};
use minijinja::{context, Environment};
use std::sync::Arc;

// 应用状态:装着模板引擎
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

    // pass env to handlers via state
    let app_state = Arc::new(AppState { env });

    // define routes
    let app = Router::new()
        .route("/", get(handler_home))
        .with_state(app_state);

    // run it
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
```

拆开看几个关键点。

**`Environment<'static>` 的生命周期参数。** minijinja 的 `Environment` 可以借用模板字符串而不复制,所以带生命周期参数。这里是 `'static`,因为 `include_str!` 产出的正是 `&'static str`——模板内容嵌在二进制里,活得和程序一样久。这让 `AppState` 不带生命周期参数,可以干净地放进 `Arc`。

**`add_template(名字, 内容)`** 注册模板。第一个参数是模板在 `Environment` 里的键,`{% extends %}` 和 `get_template` 都用它。第二个参数是模板内容字符串。

`.unwrap()` 处理注册失败——minijinja 在注册时就会解析模板,语法错误在这里暴露。这算是运行期方案里比较早的一道检查:**语法错误在启动时就崩,不会等到某个请求进来才发现。** 但"引用了不存在的变量"这类错误仍然要等到渲染时。

**`Arc::new(AppState { env })`** 把状态包进 `Arc`,理由见前置知识。

**`.with_state(app_state)`** 把状态交给 `Router`,让类型从 `Router<Arc<AppState>>` 变回 `Router<()>`。

**`State(state): State<Arc<AppState>>`** 是解构写法,和 `Form`、`Path` 一致。`state` 的类型是 `Arc<AppState>`,通过 `Deref` 可以直接写 `state.env`,不用 `(*state).env`。

**返回类型 `Result<Html<String>, StatusCode>`。** 这是本章新出现的返回类型。`Result<T, E>` 只要 `T` 和 `E` 都实现 `IntoResponse` 就自身实现 `IntoResponse`:`Ok(v)` 走 `T` 的实现,`Err(e)` 走 `E` 的实现。所以这个签名的含义是"成功返回 HTML,失败返回一个状态码"。

有意思的是**示例里这个 `Result` 从来不返回 `Err`**——两个 `.unwrap()` 会直接 panic。签名预留了错误通道但没用上,`Ok(...)` 包装因此显得多余。

> **更好的写法**:签名既然写了 `Result`,就该让错误真的走 `Err` 通道:`get_template()` 和 `render()` 都返回 `Result`,把两个 `.unwrap()` 换成 `.map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?` 即可,一行不多。示例保留 `unwrap` 是为了让这一章的注意力全在模板上,但这确实不是可以照搬的错误处理——模板渲染失败会让整个请求 panic,而不是返回一个规规矩矩的 500。练习题 1 会让你改。

> **本章不深入**:把这类 `.unwrap()` 换成真正的错误传播,需要设计自己的错误类型并为它实现 `IntoResponse`,再用 `?` 让错误自动向上冒泡。本章练习题 1 会让你先试一次最简单的形式。完整的错误处理模式见[第 14 章](14-error-handling.md),用 `anyhow` 简化的做法见[第 15 章](15-anyhow-error-response.md)。

**`context!` 宏** 构造传给模板的变量集合,语法是 `键 => 值`,用逗号分隔。它生成一个 minijinja 的 `Value` 对象。和 askama 的差别很明显:askama 里模板变量就是 struct 字段,编译期绑定;minijinja 里是运行时构造的键值对,**键名写错不会编译失败**。

**跑起来看**:

```bash
cargo run
```

```
listening on 127.0.0.1:3000
```

```bash
curl http://127.0.0.1:3000/
```

```html
<!doctype html>
<html>
  <head><title>Website Name | Home </title></head>
  <body>
    <nav>
        <ul>
            <li><a href="/">Home</a></li>
            <li><a href="/content">Content</a></li>
            <li><a href="/about">About</a></li>
        </ul>
    </nav>
    
<h1>Home</h1>
<p>Hello World!</p>

  </body>
</html>
```

**模板继承生效了。** 输出是布局模板的完整骨架,`title` 块被 `Website Name | Home` 替换(`super()` 带出了父模板的默认值),`body` 块被首页内容填充。

浏览器访问 `http://127.0.0.1:3000/` 能看到导航菜单和标题。点 Content 或 About 会得到 404,因为还没注册那两条路由。

**验证运行期方案的代价。** 把 `home.jinja` 里的 `{{ welcome_text }}` 改成 `{{ welcom_text }}`(少个 e),然后重新 `cargo run`:

**编译成功,启动成功**,访问 `/` 得到:

```html
<h1>Home</h1>
<p></p>
```

段落是空的。minijinja 默认把未定义的变量渲染成空字符串,**既不编译报错也不运行报错,只是静默输出空白**。这和[第 04 章](04-templates.md#步骤-2换成-askama-模板)askama 直接编译失败形成鲜明对比——这就是运行期方案最主要的代价。

(minijinja 支持配置 `undefined_behavior` 让未定义变量报错,但默认是宽容模式。)

改回来。

**再验证一下类型级保证。** 临时把 `.with_state(app_state)` 那一行注释掉,`cargo build`:

编译失败。错误信息里会提到 `Router<Arc<AppState>>` 和 `axum::serve` 的类型不匹配。**忘记提供状态是编译错误,不是运行时崩溃**——前置知识里说的那个保证。恢复它。

### 步骤 2:补上另外两个页面

**要做什么**:加上 `/content` 和 `/about`,让导航菜单的三个链接都能用。顺便看看模板里的循环。

先写两个模板:

```html
<!-- templates/content.jinja -->
{% extends "layout" %}
{% block title %}{{ super() }} | {{ title }} {% endblock %}
{% block body %}
<h1>{{ title }}</h1>
{% for data_entry in entries %}
<ul>
    <li>{{ data_entry }}</li>
</ul>
{% endfor %}
{% endblock %}
```

```html
<!-- templates/about.jinja -->
{% extends "layout" %}
{% block title %}{{ super() }} | {{ title }} {% endblock %}
{% block body %}
<h1>{{ title }}</h1>
<p>{{ about_text }}</p>
{% endblock %}
```

`{% for x in items %}...{% endfor %}` 是循环语法。

> **更好的写法**:这个模板把 `<ul>` 写在了循环**里面**,三个条目会生成三个各含一个 `<li>` 的独立列表。正确做法是 `<ul>` 在循环外、`<li>` 在循环内,生成一个含三项的列表。这不只是美观问题——屏幕阅读器会把它读成"三个列表,每个一项",而不是"一个列表,三项"。练习题 3 会让你改。

然后改 Rust 代码,三处:

```rust
// src/main.rs
// ...(use 和 AppState 不变)

#[tokio::main]
async fn main() {
    let mut env = Environment::new();
    env.add_template("layout", include_str!("../templates/layout.jinja"))
        .unwrap();
    env.add_template("home", include_str!("../templates/home.jinja"))
        .unwrap();
    // 新增:注册另外两个模板
    env.add_template("content", include_str!("../templates/content.jinja"))
        .unwrap();
    env.add_template("about", include_str!("../templates/about.jinja"))
        .unwrap();

    let app_state = Arc::new(AppState { env });

    let app = Router::new()
        .route("/", get(handler_home))
        .route("/content", get(handler_content))   // 新增
        .route("/about", get(handler_about))       // 新增
        .with_state(app_state);

    // ...(listener 和 serve 不变)
}

// ...(handler_home 不变)

// 新增
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

// 新增
async fn handler_about(State(state): State<Arc<AppState>>) -> Result<Html<String>, StatusCode> {
    let template = state.env.get_template("about").unwrap();

    let rendered = template.render(context!{
        title => "About",
        about_text => "Simple demonstration layout for an axum project with minijinja as templating engine.",
    }).unwrap();

    Ok(Html(rendered))
}
```

三个 handler 的结构完全一样:从 `State` 拿引擎、按名字取模板、用 `context!` 传变量、渲染、包成 `Html`。这种重复在真实项目里会想办法收敛(比如抽一个 `render(state, name, ctx)` 辅助函数),练习题里可以试。

`entries => some_example_entries` 传的是 `Vec<&str>`。minijinja 能把常见的 Rust 类型自动转成模板可用的值——`Vec`、`HashMap`、数字、字符串、`Option` 等都支持。

**跑起来看**:

```bash
cargo run
```

```bash
curl http://127.0.0.1:3000/content
```

```html
<!doctype html>
<html>
  <head><title>Website Name | Content </title></head>
  <body>
    <nav>
        <ul>
            <li><a href="/">Home</a></li>
            <li><a href="/content">Content</a></li>
            <li><a href="/about">About</a></li>
        </ul>
    </nav>
    
<h1>Content</h1>

<ul>
    <li>Data 1</li>
</ul>

<ul>
    <li>Data 2</li>
</ul>

<ul>
    <li>Data 3</li>
</ul>

  </body>
</html>
```

循环生效了,三个条目各自一个 `<ul>`(前面说的那个瑕疵)。

```bash
curl http://127.0.0.1:3000/about
```

```html
<!doctype html>
<html>
  <head><title>Website Name | About </title></head>
  <body>
    <nav>
        <ul>
            <li><a href="/">Home</a></li>
            <li><a href="/content">Content</a></li>
            <li><a href="/about">About</a></li>
        </ul>
    </nav>
    
<h1>About</h1>
<p>Simple demonstration layout for an axum project with minijinja as templating engine.</p>

  </body>
</html>
```

用浏览器访问 `http://127.0.0.1:3000/`,现在三个导航链接都能点了,来回切换时你会看到布局固定、内容变化、浏览器标签页的标题也跟着变。

**验证一下继承的威力。** 打开 `templates/layout.jinja`,在 `<nav>` 里加一个链接:

```html
<li><a href="/">Home 2</a></li>
```

`cargo run` 后访问任意页面——三个页面的导航都变了。**改一个文件,三个页面同步更新**,这就是布局复用的价值。改回来。

顺便确认转义也在工作(和 askama 一样,minijinja 的 `{{ }}` 默认转义):把 `about_text` 临时改成含 `<script>` 的字符串,渲染出来会是 `&lt;script&gt;`。

## 完整代码

`templates/layout.jinja`:

```html
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
```

`templates/home.jinja`:

```html
{% extends "layout" %}
{% block title %}{{ super() }} | {{ title }} {% endblock %}
{% block body %}
<h1>{{ title }}</h1>
<p>{{ welcome_text }}</p>
{% endblock %}
```

`templates/content.jinja`:

```html
{% extends "layout" %}
{% block title %}{{ super() }} | {{ title }} {% endblock %}
{% block body %}
<h1>{{ title }}</h1>
{% for data_entry in entries %}
<ul>
    <li>{{ data_entry }}</li>
</ul>
{% endfor %}
{% endblock %}
```

`templates/about.jinja`:

```html
{% extends "layout" %}
{% block title %}{{ super() }} | {{ title }} {% endblock %}
{% block body %}
<h1>{{ title }}</h1>
<p>{{ about_text }}</p>
{% endblock %}
```

`src/main.rs`:

```rust
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
```

## 两种模板方案的对照

学完两章,可以把 askama 和 minijinja 摆在一起看:

| | askama(第 04 章) | minijinja(本章) |
|---|---|---|
| 解析时机 | 编译期 | 运行期 |
| 模板变量来源 | struct 字段 | `context!` 键值对 |
| 变量名写错 | **编译失败** | 静默输出空值(可配置为报错) |
| 语法错误 | 编译失败 | 注册时 panic(启动即崩) |
| 模板来源 | 必须是源码目录下的文件 | 任意字符串,可从数据库/网络加载 |
| 热重载 | 不支持 | 支持(本章因用 `include_str!` 而放弃了) |
| 与 axum 集成 | 自己写 `IntoResponse`(第 04 章步骤 3) | 引擎放 `State`,handler 里 `get_template` |
| 状态需求 | 无(模板已编译进代码) | 需要 `State` 共享 `Environment` |

最后一行是个容易忽略的差别:**askama 不需要应用状态,minijinja 需要。** 因为 askama 把模板变成了代码,不存在"运行时要拿到引擎"的问题;minijinja 的引擎是个运行时对象,必须共享。这也解释了为什么 `State` 这个机制在本章而不是第 04 章出现。

实际选择建议:模板固定、追求编译期保障、不需要热重载 → askama;需要动态加载模板、团队熟悉 Jinja2、开发期希望改模板即生效 → minijinja。

## 动手练习

**1. 消掉三个 handler 的重复。** 抽一个辅助函数,让三个 handler 各自只剩两三行。

提示:签名可以是 `fn render(state: &AppState, name: &str, ctx: minijinja::value::Value) -> Result<Html<String>, StatusCode>`。`context!` 宏返回的就是 `Value`。顺手把 `.unwrap()` 换成 `.map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?`——这样那个一直没用上的 `Err` 分支终于有了意义,也是第 14 章错误处理的预习。

**2. 改成运行时读取模板文件,实现改模板不重编译。** 把 `include_str!` 换成 `std::fs::read_to_string`。

提示:`Environment<'static>` 要求模板字符串是 `'static`,而 `read_to_string` 返回 `String`。minijinja 的 `add_template_owned` 方法接受拥有所有权的字符串,正是为这种场景准备的。改完后修改 `templates/home.jinja` 并重启程序(不用 `cargo build`)验证效果。想想这个改动放弃了什么:文件路径写错会怎样?什么时候才会发现?

**3. 修掉 content 页的 HTML 瑕疵。** 让三个条目在同一个 `<ul>` 里。

提示:把 `<ul>` 和 `</ul>` 移到 `{% for %}` 循环外面。

## 小结与下一章预告

minijinja 在运行期加载和解析模板,换来了模板来源灵活、可热重载,代价是变量名写错不会编译失败——本章亲手验证了它会静默输出空值。模板继承用 `{% extends %}` + `{% block %}` 实现布局复用,`super()` 用来在覆盖块时保留父模板内容,改一处布局三个页面同步生效。

`State<S>` 提取器解决"启动时创建一次、所有 handler 共享"的资源该怎么传递的问题。状态用 `with_state` 挂到 `Router` 上,类型从 `Router<S>` 变回 `Router<()>`——只有后者能交给 `serve`,所以忘记提供状态是编译错误。状态必须 `Clone`,所以通常包一层 `Arc` 让克隆变成引用计数加一。共享**可变**状态还需要在 `Arc` 里套锁,那是第 22 章的内容。

入门阶段到这里结束。你已经能写出一个有路由、能收表单、能渲染页面、有共享状态的服务。但这六章里有不少地方是"先凑合"的:满地的 `.unwrap()`、默认的纯文本错误响应、看不懂的 404 页面。

接下来的「请求与响应」阶段会把这些逐个补上。[第 06 章 global-404-handler](06-global-404-handler.md) 从最简单的一个开始:如何让所有未匹配的路径返回你自己设计的响应,而不是 axum 那个空 body 的 404。
