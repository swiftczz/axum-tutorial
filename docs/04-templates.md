# 第 04 章 templates:模板引擎与自定义 IntoResponse

## 本章你将做出什么

一个带动态路径的服务:访问 `/greet/张三` 返回 `<h1>Hello, 张三!</h1>`。HTML 不再写在 Rust 源码里,而是放在独立的模板文件 `templates/hello.html` 中。

这一章有三件新事:用 `Path` 提取器接收路径参数、用 **askama** 模板引擎渲染 HTML、**亲手为自己的类型实现 `IntoResponse`**。最后一件是本章最有价值的部分——[第 01 章](01-hello-world.md#intoresponse返回值如何变成-http-响应)你只是用了 `IntoResponse`,这一章你会实现它。

学完本章你将掌握:

- 路径参数的路由语法 `/greet/{name}`,以及 `Path<T>` 提取器
- 模板引擎解决什么问题,askama 的编译期检查和自动转义
- 如何为自己的类型实现 `IntoResponse`,以及 `impl IntoResponse` 作为返回类型的含义
- 渲染失败时如何转成 `500` 响应

## 前置知识

### 动态路径与路径参数

前三章的路由都是固定路径:`/`、`/foo`。但真实服务的 URL 里常常有一段是变量:

```
/users/42
/users/1337
/posts/hello-world
```

你不可能为每个用户 ID 注册一条路由。这类 URL 的正确理解是"一个**模式**加上其中的**参数**":模式是 `/users/{id}`,参数 `id` 的值随请求变化。

axum 的路由语法用花括号声明参数:

```rust
.route("/greet/{name}", get(greet))
```

`{name}` 匹配路径里的一段(**不跨 `/`**),匹配到的内容以 `name` 为键存起来,handler 用 `Path` 提取器取出。

所以 `/greet/张三` 匹配成功,`name = "张三"`;`/greet/a/b` 匹配失败(有两段),返回 `404`;`/greet/` 也匹配失败(那一段是空的)。

想匹配剩余全部路径(包括 `/`)要用通配语法 `{*rest}`,[第 38 章](38-static-file-server.md)讲静态文件服务时会用到。

> axum 0.7 及更早版本用 `:name` 这种冒号语法,0.8 改成了 `{name}`。网上很多教程还是老写法,照抄会报 panic。本教程一律用 0.8 的花括号语法。

`Path<T>` 属于[第 03 章](03-form.md#提取器axum-的核心机制)讲的 `FromRequestParts` 类别——它只看路径,不碰 body,所以可以和 `Form`、`Json` 同时出现在参数列表里,而且不必放最后。

单个参数时 `T` 直接写目标类型即可:`Path<String>`、`Path<u32>`。多个参数用元组 `Path<(u32, String)>`,或者一个实现了 `Deserialize` 的 struct。类型转换失败会返回 `400`——比如路由是 `/users/{id}`、提取器是 `Path<u32>`,访问 `/users/abc` 就转不出来。定制这类错误响应是[第 11 章](11-customize-path-rejection.md)的内容。

### 模板引擎解决什么问题

[第 03 章](03-form.md#步骤-1显示表单页面)的 HTML 是一整块字符串塞在 Rust 源码里的。那样做有几个实际问题。

**没有语法高亮和补全。** 编辑器把它当普通字符串,写 HTML 得不到任何帮助。

**动态内容只能靠字符串拼接。** 想插入变量就得 `format!`,一旦页面复杂起来,拼接代码会变得难以阅读,而且容易漏掉转义。

**转义漏掉就是安全漏洞。** 假设你写 `format!("<h1>Hello, {}!</h1>", name)`,而用户把名字设成 `<script>fetch('/api/delete-all')</script>`,这段脚本会原样进入页面并在访客浏览器里执行——这就是 **XSS**(跨站脚本攻击)。[第 01 章](01-hello-world.md#步骤-3改成返回-html)提到 `Html` 不做转义,风险就在这里。

**模板引擎**把 HTML 移到独立文件里,用特殊语法标记要插入变量的位置:

```html
<h1>Hello, {{ name }}!</h1>
```

渲染时把 `{{ name }}` 替换成实际值,**并自动做 HTML 转义**——`<` 变成 `&lt;`,`>` 变成 `&gt;`,于是恶意脚本变成了普普通通的可见文本,不会被浏览器当代码执行。

### askama:编译期模板引擎

Rust 生态的模板引擎分两派:

- **编译期**(askama、maud)— 模板在编译时就被解析并生成 Rust 代码。模板语法错误、变量名写错、类型不匹配,全部在 `cargo build` 时报错。代价是改模板必须重新编译。
- **运行期**(minijinja、tera、handlebars)— 模板在运行时加载解析。改模板不用重编,甚至可以热重载;代价是错误只在运行时暴露。

askama 属于编译期派,语法借鉴 Jinja2(Python 生态的模板引擎)。用法是给 struct 加派生宏:

```rust
#[derive(Template)]
#[template(path = "hello.html")]
struct HelloTemplate {
    name: String,
}
```

`path` 相对于**项目根目录下的 `templates/` 目录**(这是 askama 的默认约定,可以配置)。所以 `path = "hello.html"` 指的是 `templates/hello.html`。

派生宏会读取模板文件,生成一个 `render()` 方法。模板里 `{{ name }}` 编译成"读取 `self.name` 并转义输出"的代码——**所以模板里引用了 struct 没有的字段,编译就会失败**。这是编译期方案最实用的好处:模板和数据结构不会失去同步。

`render()` 返回 `Result<String, askama::Error>`,因为渲染仍可能失败(比如模板里调用了会出错的过滤器)。

本章用 askama 0.15。这个版本需要 `default-features = false` 加上 `std` 和 `derive`,对应的 `cargo add` 命令在下面。

> 早期版本有个配套 crate `askama_axum`,提供现成的 axum 集成。它已经不再维护,现在的做法是自己写一个几行的 `IntoResponse` 实现——这反而成了本章最好的教学材料。

## 项目搭建

```bash
cargo new templates
cd templates
```

依赖:

```bash
cargo add axum
cargo add tokio --features full
cargo add askama --no-default-features --features std,derive
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

askama 的 `--no-default-features --features std,derive` 对应官方示例里 `default-features = false, features = ["std", "derive"]`。默认 feature 里包含一些本例不需要的东西(如 `config`、`urlencode`),关掉能减少编译时间和依赖体积。`std` 提供标准库支持,`derive` 提供 `#[derive(Template)]`。

`tracing` 两个 crate 的作用见[第 03 章](03-form.md#步骤-1显示表单页面),本章不再解释。`fn app() -> Router` 的用途同样见第 03 章。

建立模板目录:

```bash
mkdir templates
```

注意这个目录和你的项目同名(项目叫 `templates`,目录也叫 `templates`),路径会是 `templates/templates/hello.html`。看着有点绕,但这是官方示例的命名,照做即可。目录里的文件在步骤 2 创建。

> 本章会渲染 HTML 页面,但**不涉及给页面配 CSS、JS、图片**。托管这类静态资源需要 `tower-http` 的 `ServeDir`,那是[第 38 章 static-file-server](38-static-file-server.md) 的内容。现在页面是裸 HTML,不好看是正常的。

## 分步构建

### 步骤 1:用 Path 提取器接收路径参数

**要做什么**:`GET /greet/{name}` 把路径里的名字回显出来。先不用模板,确认路由和提取器工作正常。

```rust
// src/main.rs
use axum::{extract, response::Html, routing::get, Router};
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
    axum::serve(listener, app).await.unwrap();
}

fn app() -> Router {
    Router::new().route("/greet/{name}", get(greet))
}

async fn greet(extract::Path(name): extract::Path<String>) -> Html<String> {
    Html(format!("<h1>Hello, {name}!</h1>"))
}
```

`main` 和[第 03 章](03-form.md)完全一样,只有 `app()` 和 handler 是新的。

`extract::Path(name): extract::Path<String>` 是[第 03 章](03-form.md#步骤-2用-form-提取器接收提交)讲过的解构写法。这里用 `extract::Path` 而不是 `use axum::extract::Path` 后直接写 `Path`,是官方示例的选择——好处是读代码时一眼看出这是个提取器,而且避免和标准库的 `std::path::Path` 混淆。后者是很实际的考虑:一个文件相关的模块里同时用到两个 `Path` 会很难受。

**跑起来看**:

```bash
cargo run
```

```bash
curl -i http://127.0.0.1:3000/greet/Foo
```

```
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 20

<h1>Hello, Foo!</h1>
```

中文也没问题(路径里的中文会被 URL 编码传输,`Path` 提取器负责解码):

```bash
curl http://127.0.0.1:3000/greet/张三
```

```
<h1>Hello, 张三!</h1>
```

试试路由匹配的边界。多一段路径:

```bash
curl -i http://127.0.0.1:3000/greet/a/b
```

```
HTTP/1.1 404 Not Found
```

`{name}` 不跨 `/`,所以两段路径匹配不上。少一段:

```bash
curl -i http://127.0.0.1:3000/greet/
```

```
HTTP/1.1 404 Not Found
```

**现在看这一步的问题。** 故意传一个带 HTML 标签的名字:

```bash
curl 'http://127.0.0.1:3000/greet/<script>alert(1)</script>'
```

```
<h1>Hello, <script>alert(1)</script>!</h1>
```

**标签原样进入了页面。** 用浏览器访问这个地址,弹窗真的会出现——这就是前置知识里说的 XSS。`format!` 只做字符串拼接,不懂 HTML 语义,更不会转义。

下一步用模板引擎解决它。

### 步骤 2:换成 askama 模板

**要做什么**:把 HTML 挪进模板文件,用 askama 渲染,顺便消掉 XSS 风险。

先建模板文件:

```html
<!-- templates/hello.html -->
<h1>Hello, {{ name }}!</h1>
```

就一行。`{{ name }}` 是 askama 的插值语法,渲染时替换成同名字段的值。

然后改 Rust 代码:

```rust
// src/main.rs
use askama::Template;                                             // 新增
use axum::{extract, response::Html, routing::get, Router};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// ...(main 和 app 不变)

// 改动:改成渲染模板
async fn greet(extract::Path(name): extract::Path<String>) -> Html<String> {
    let template = HelloTemplate { name };
    Html(template.render().unwrap())
}

// 新增:和 templates/hello.html 绑定的数据结构
#[derive(Template)]
#[template(path = "hello.html")]
struct HelloTemplate {
    name: String,
}
```

`#[derive(Template)]` 加 `#[template(path = "hello.html")]` 让 askama 在编译期读取 `templates/hello.html`,生成 `render()` 方法。struct 的字段就是模板可用的变量——模板里的 `{{ name }}` 对应 `HelloTemplate::name`。

`HelloTemplate { name }` 用了 Rust 的字段简写(变量名和字段名相同时可以省略 `name: name`)。

`render().unwrap()` 先凑合一下,步骤 3 会把它换成正经的错误处理。

**验证编译期检查确实存在。** 把模板临时改成 `{{ nam }}`(少个 e),然后 `cargo build`:

```
error: field `nam` not found in struct `HelloTemplate`
```

**编译就失败了**,不用等到运行时。这是编译期模板引擎的核心价值。改回来。

**跑起来看**:

```bash
cargo run
```

```bash
curl -i http://127.0.0.1:3000/greet/Foo
```

```
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 20

<h1>Hello, Foo!</h1>
```

和步骤 1 完全一样。但这次:

```bash
curl 'http://127.0.0.1:3000/greet/<script>alert(1)</script>'
```

```
<h1>Hello, &lt;script&gt;alert(1)&lt;/script&gt;!</h1>
```

**`<` 变成了 `&lt;`,`>` 变成了 `&gt;`。** 浏览器现在会把它显示成可见的文本 `<script>alert(1)</script>`,而不是执行它。askama 对 `{{ }}` 插值默认做 HTML 转义,XSS 风险自动消失。

这个默认值的方向是对的:**安全应该是默认行为,不安全才需要显式声明。** 确实需要输出原始 HTML 时,askama 提供 `{{ content|safe }}` 过滤器——写出 `|safe` 就是在明确表示"我知道风险,这段内容可信"。

### 步骤 3:为自己的类型实现 IntoResponse

**要做什么**:去掉 `render().unwrap()`,把渲染失败正确地转成 `500` 响应。做法是定义一个包装类型并为它实现 `IntoResponse`。

`unwrap()` 的问题很明确:渲染一旦失败,handler 直接 panic。axum 会捕获 panic 让整个服务不崩,但客户端拿到的是一个空的连接错误,日志里只有一段 panic backtrace。正确做法是返回一个 `500 Internal Server Error` 并附上错误信息。

按[第 01 章](01-hello-world.md#intoresponse返回值如何变成-http-响应)讲的,handler 的返回类型只要实现 `IntoResponse` 就行。那就自己实现一个:

```rust
// src/main.rs
use askama::Template;
use axum::{
    extract,
    http::StatusCode,                                    // 新增
    response::{Html, IntoResponse, Response},            // 改动:加上 IntoResponse 和 Response
    routing::get,
    Router,
};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// ...(main 和 app 不变)

// 改动:返回 HtmlTemplate 包装,返回类型换成 impl IntoResponse
async fn greet(extract::Path(name): extract::Path<String>) -> impl IntoResponse {
    let template = HelloTemplate { name };
    HtmlTemplate(template)
}

// ...(HelloTemplate 不变)

// 新增:一个包装任意 askama 模板的类型
struct HtmlTemplate<T>(T);

// 新增:把"渲染模板"这件事翻译成 HTTP 响应
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
```

这段代码值得逐块理解,因为它是"扩展 axum"的基本套路。

**`struct HtmlTemplate<T>(T);`** 是个元组结构体,只是把任意值包一层。它自己不做任何事,存在的意义是**给 `IntoResponse` 一个挂载点**。

为什么不直接给 `HelloTemplate` 实现 `IntoResponse`?可以,但那样每加一个模板都要重写一遍实现。用泛型包装类型,`impl<T: Template>` 一次覆盖所有 askama 模板。

为什么不直接给 `T: Template` 实现?因为 **Rust 的孤儿规则**(orphan rule):`IntoResponse` 定义在 axum 里,`Template` 定义在 askama 里,两个都不是你的 crate,你不能为"所有实现了外部 trait 的外部类型"实现另一个外部 trait。定义一个自己的类型 `HtmlTemplate<T>` 就绕开了这个限制——这个模式在 Rust 里叫 **newtype**,专门用来突破孤儿规则。

**`fn into_response(self) -> Response`** 是 `IntoResponse` 唯一要实现的方法。里面的逻辑是:

- 渲染成功 → `Html(html).into_response()`,借用[第 01 章](01-hello-world.md#intoresponse返回值如何变成-http-响应)讲过的 `Html<T>` 的实现,得到 `200` + `text/html`
- 渲染失败 → `(StatusCode::INTERNAL_SERVER_ERROR, format!(...)).into_response()`,用元组的实现,得到 `500` + 纯文本错误信息

> **更好的写法**:失败分支把 askama 的原始错误 `{err}` 直接拼进了响应体发给客户端。`5xx` 属于服务端错误,响应里不该带内部细节——模板路径、字段名、依赖库的内部结构都可能顺着错误消息泄露出去。生产环境应该响应固定文本(如 `"Internal Server Error"`),把 `err` 用 `tracing::error!` 记进日志。示例这么写是为了让你在浏览器里直接看到渲染失败的原因,省去配日志的步骤。这条原则和配套做法见[第 14 章](14-error-handling.md#4xx-与-5xx-决定了两件事)。

**这里的关键手法是复用而非重写。** 你没有手工构造 `Response`、没有设置任何头部、没有算 `Content-Length`——只是把工作转交给了 axum 已有的 `IntoResponse` 实现。自己实现 `IntoResponse` 时几乎总是这个套路:**决定要返回什么,然后交给已有实现去构造。**

**返回类型 `impl IntoResponse`。** 这是本章另一个值得注意的点。它的含义是"返回某个实现了 `IntoResponse` 的类型,具体是哪个由编译器推断,调用方不需要知道"。

好处是写起来省事,而且以后改返回类型不用改签名。代价是**类型信息被隐藏了**,可能带来两个麻烦:

- 编译器报错时会更难读,因为错误里出现的是一个匿名类型
- 如果函数有多个 `return` 分支返回**不同**类型,`impl IntoResponse` 编译不过——`impl Trait` 要求所有分支是同一个具体类型。这种情况得用 `Response`(即 `.into_response()` 的结果)或者 `Box<dyn IntoResponse>`

本章只有一个返回分支,所以 `impl IntoResponse` 没问题。

> **本章不深入**:handler 有多个返回分支时,常用的返回类型是 `Result<T, E>`——只要 `T` 和 `E` 都实现 `IntoResponse`,`Result` 自身就实现。[第 05 章](05-templates-minijinja.md#步骤-1布局模板-首页认识-state)会先用到它,错误类型该怎么设计、如何用 `?` 传播错误见[第 14 章](14-error-handling.md)。

**跑起来看**:

```bash
cargo run
```

正常路径的行为和步骤 2 完全一致:

```bash
curl -i http://127.0.0.1:3000/greet/Foo
```

```
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 20

<h1>Hello, Foo!</h1>
```

转义依然生效:

```bash
curl 'http://127.0.0.1:3000/greet/<script>alert(1)</script>'
```

```
<h1>Hello, &lt;script&gt;alert(1)&lt;/script&gt;!</h1>
```

`404` 行为也没变:

```bash
curl -i http://127.0.0.1:3000/greet/a/b
```

```
HTTP/1.1 404 Not Found
```

`500` 分支很难自然触发——`hello.html` 这么简单的模板不会渲染失败。想验证那条分支,可以临时在模板里加一个会出错的表达式,或者直接读代码确认逻辑。这条分支的价值在于**它存在**:哪天模板变复杂了,失败会变成一个规规矩矩的 `500` 响应,而不是 panic。

> 官方示例的 `main.rs` 末尾还有一个 `#[cfg(test)] mod tests`,用 `oneshot` 断言 `/greet/Foo` 返回 `<h1>Hello, Foo!</h1>`。测试写法见[第 21 章](21-testing.md)。

## 完整代码

`templates/hello.html`:

```html
<h1>Hello, {{ name }}!</h1>
```

`src/main.rs`:

```rust
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
```


## 动手练习

**1. 加一个带条件和循环的模板。** 新增路由 `GET /list/{count}`,渲染一个 `<ul>` 列表,列出 `1` 到 `count`。`count` 为 0 时显示"列表为空"。

提示:askama 的控制流语法是 `{% if ... %}...{% else %}...{% endif %}` 和 `{% for x in items %}...{% endfor %}`。`Path<u32>` 直接把路径段转成数字,转不出来会返回 `400`——试试 `/list/abc` 看看。struct 里放 `Vec<u32>` 字段,模板里遍历它。

**2. 验证 `|safe` 过滤器的效果和风险。** 把 `hello.html` 改成 `{{ name|safe }}`,重新运行,再访问 `/greet/<script>alert(1)</script>`。

提示:用浏览器访问才能看到区别。做完把它改回去,并想清楚:什么情况下你会真的需要 `|safe`?(提示:内容来自你自己的数据库、由你的编辑器生成,而非用户直接输入。)

## 小结与下一章预告

路由用 `{name}` 声明路径参数,`Path<T>` 提取器把它取出来并转成目标类型。模板引擎把 HTML 从 Rust 源码里搬进独立文件,askama 在编译期解析模板——模板引用了不存在的字段会导致编译失败,而 `{{ }}` 插值默认做 HTML 转义,XSS 风险不需要你操心。

自己实现 `IntoResponse` 是扩展 axum 的基本手法:定一个 newtype 绕开孤儿规则,在 `into_response` 里决定返回什么,然后把构造工作交给 axum 已有的实现。返回类型写 `impl IntoResponse` 很省事,但只适用于单一返回类型的场景。

askama 是编译期方案,改模板要重新编译。[第 05 章 templates-minijinja](05-templates-minijinja.md) 换成运行期方案 minijinja,你会看到两种取舍的具体差别,还会接触**模板继承**——多个页面共享同一套布局。那一章还会第一次用到 `State` 提取器:模板引擎需要在启动时初始化一次、被所有 handler 共享,这正是应用状态的典型场景。
