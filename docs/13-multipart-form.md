# 第 13 章 multipart-form:文件上传与请求体限制

## 本章你将做出什么

一个能收文件的服务:浏览器访问 `/` 看到一个文件选择框,选中若干文件提交后,服务端在终端打印每个文件的字段名、文件名、MIME 类型和字节数:

```
Length of `file` (`photo.jpg`: `image/jpeg`) is 284736 bytes
```

学完本章你将掌握:

- `multipart/form-data` 格式长什么样,为什么文件上传必须用它
- `Multipart` 提取器:为什么它是"流式"的,和 `Form`/`Json` 有本质区别
- 请求体大小限制的两层机制:`DefaultBodyLimit` 与 `RequestBodyLimitLayer`
- 处理上传时绕不开的几个安全问题

## 前置知识

### 为什么文件上传不能用 urlencoded

[第 03 章](03-form.md#applicationx-www-form-urlencoded)讲的 `application/x-www-form-urlencoded` 格式是这样的:

```
name=%E5%BC%A0%E4%B8%89&email=a%40b.com
```

键值对用 `&` 连接,内容做百分号编码。这个格式对文本够用,对二进制文件是灾难:

**体积会暴涨。** 百分号编码把每个非安全字节变成三个字符(`%FF`)。一张 1MB 的 JPEG,大部分字节都不是 ASCII 安全字符,编码后可能膨胀到 3MB。

**没地方放元信息。** 上传文件通常要带上原始文件名和 MIME 类型,而 urlencoded 只有扁平的键值对,塞不下这些附加信息。

**必须整体编码解码。** 拿到数据前得先把整个字符串解码完,没法边收边处理。

所以 HTML 表单为文件上传定义了另一种编码方式,`enctype="multipart/form-data"`。

### multipart/form-data 长什么样

它的思路是**用分隔符把请求体切成若干段**,每段是一个字段,自带自己的头部:

```
POST / HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryABC123
Content-Length: 396

------WebKitFormBoundaryABC123
Content-Disposition: form-data; name="username"

zhangsan
------WebKitFormBoundaryABC123
Content-Disposition: form-data; name="file"; filename="photo.jpg"
Content-Type: image/jpeg

<这里是 JPEG 的原始二进制字节,不做任何编码>
------WebKitFormBoundaryABC123--
```

几个要点:

**`boundary` 是分隔符**,由客户端随机生成,声明在 `Content-Type` 头部里。它必须是一个不会出现在任何字段内容里的字符串,所以浏览器生成的都带随机后缀。

**每段以 `--boundary` 开头,最后一段以 `--boundary--` 结尾。**

**每段有自己的头部**,和空行隔开——结构和[第 00 章](00-web-basics.md#http-是一层文本约定)讲的 HTTP 报文一模一样,是个嵌套结构。

- `Content-Disposition` 里的 `name` 是表单字段名,`filename` 是原始文件名(只有文件字段才有)
- `Content-Type` 是这个字段的 MIME 类型(文本字段通常省略)

**二进制内容原样放入,不做编码。** 这就是它比 urlencoded 高效的原因。

### Multipart 提取器是流式的

这是本章和前面几章最重要的区别。

`Form<T>`、`Json<T>` 的工作方式是:**把整个 body 读进内存,一次性反序列化成你的 struct**。[第 09 章](09-consume-body-in-extractor-or-middleware.md#缓冲整个-body-的代价)讲过这么做的代价。

文件上传动辄几十上百 MB,这个模式行不通。所以 `Multipart` 提取器的设计完全不同:

```rust
async fn accept_form(mut multipart: Multipart) {
    while let Some(field) = multipart.next_field().await.unwrap() {
        // 处理这个字段
    }
}
```

**它本身不读任何数据。** 提取 `Multipart` 时,body 还在网络上。每次调用 `next_field().await` 才向前推进,解析出下一个字段的头部。字段的内容也是按需读取的——`field.bytes().await` 读全部,或者用 `field.chunk().await` 一块一块读。

这带来两个直接后果:

**顺序不可跳过。** 你只能从前往后一个个处理字段,不能"先看看有没有 `file` 字段"。因为数据是流,跳过就意味着丢弃。

**必须在 handler 里处理。** `Multipart` 是 `FromRequest` 类别的提取器([第 03 章](03-form.md#提取器axum-的核心机制)讲过的分类),而且它持有 body 的所有权,所以只能有一个、必须放最后。

`Multipart` 需要 axum 的 `multipart` feature,默认不开。

### 两层请求体限制

axum 对请求体大小有**两层**保护,本章两个都会用到,它们的区别是本章的重点之一。

**第一层:`DefaultBodyLimit`,默认 2MB。**

这是 axum 内置的,作用在**提取器**层面。`Bytes`、`String`、`Json`、`Form`、`Multipart` 这些要读 body 的提取器都受它约束,超限返回 `413 Payload Too Large`。

它可以调整或关闭:

```rust
.layer(DefaultBodyLimit::max(10 * 1024 * 1024))  // 改成 10MB
.layer(DefaultBodyLimit::disable())              // 完全关闭
```

**第二层:`RequestBodyLimitLayer`,来自 `tower-http`。**

这是一个真正的中间件,作用在**更外层**——它包住整个请求处理链,不管下游用什么提取器。

[第 09 章](09-consume-body-in-extractor-or-middleware.md#缓冲整个-body-的代价)提过一个漏洞:中间件里直接 `body.collect()` 会绕过 `DefaultBodyLimit`,因为那个限制只对提取器生效。`RequestBodyLimitLayer` 堵的就是这个洞。

**为什么本章要"先关掉第一层,再用第二层"?**

```rust
.layer(DefaultBodyLimit::disable())
.layer(RequestBodyLimitLayer::new(250 * 1024 * 1024))
```

因为文件上传需要远超 2MB 的额度。做法是:关掉提取器层面的默认限制,改由外层中间件统一管控上限。**关键是不能只做第一步** —— 只 `disable()` 不加 `RequestBodyLimitLayer`,服务就完全没有大小限制了,任何人都能用一个超大请求把你的内存吃光。

> **更好的写法**:如果你只是想把上限从 2MB 提到 250MB,一行 `DefaultBodyLimit::max(250 * 1024 * 1024)` 就够了,不必走"关掉 + 外层限制"这条两步路——后者少写一步就变成完全无限制,是个容易踩的坑。示例选两步做法有它的道理:`RequestBodyLimitLayer` 是真正的中间件,连绕过提取器直接读 body 的代码([第 09 章](09-consume-body-in-extractor-or-middleware.md#缓冲整个-body-的代价)那种)也管得住,防护面更广。**两种都对,按需要选**:只调额度用 `max()`,要兜住全链路用两步法。

## 项目搭建

```bash
cargo new multipart-form
cd multipart-form
```

```bash
cargo add axum --features multipart
cargo add tokio --features full
cargo add tower-http --features limit,trace
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`axum` 的 `multipart` feature 提供 `Multipart` 提取器,默认不开启,必须显式加。

`tower-http` 是 tower 生态里的 HTTP 中间件集合,本章用两个 feature:`limit` 提供 `RequestBodyLimitLayer`,`trace` 提供 `TraceLayer`。

> **本章不深入**:`TraceLayer` 是 `tower-http` 提供的请求日志中间件,挂上它就能看到每个请求的方法、路径、状态码和耗时。本章只是照抄官方示例把它加上,不解释配置。它的完整用法、以及 tower 中间件的叠加顺序,见[第 17 章](17-tracing-aka-logging.md)。

## 分步构建

### 步骤 1:显示文件上传表单

**要做什么**:先做出表单页面,确认 `enctype` 的作用。

```rust
// src/main.rs
use axum::{response::Html, routing::get, Router};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // build our application with some routes
    let app = Router::new().route("/", get(show_form));

    // run it with hyper
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn show_form() -> Html<&'static str> {
    Html(
        r#"
        <!doctype html>
        <html>
            <head></head>
            <body>
                <form action="/" method="post" enctype="multipart/form-data">
                    <label>
                        Upload file:
                        <input type="file" name="file" multiple>
                    </label>

                    <input type="submit" value="Upload files">
                </form>
            </body>
        </html>
        "#,
    )
}
```

和[第 03 章](03-form.md#步骤-1显示表单页面)的表单比,HTML 有两处关键差异:

**`enctype="multipart/form-data"`** —— 不加这个属性,浏览器会用默认的 urlencoded 编码,那样文件内容根本传不上来(只会传个文件名)。**这是文件上传最常见的错误之一**,而且浏览器不会报错,只是数据悄悄不对。

**`<input type="file" name="file" multiple>`** —— `type="file"` 让浏览器显示文件选择框;`multiple` 允许一次选多个文件。多选时,**多个文件会用同一个字段名 `file` 分成多段**发送,所以服务端要循环处理。

日志过滤器这次是 `format!("{}=debug,tower_http=debug", ...)`,比前面章节多了 `tower_http=debug` —— 为了让步骤 3 加上的 `TraceLayer` 能输出日志。

**跑起来看**:

```bash
cargo run
```

浏览器访问 `http://127.0.0.1:3000/`,能看到文件选择框和上传按钮。选个文件点提交:

浏览器显示 `Method Not Allowed` —— 意料之中,`POST /` 还没注册。

用 curl 确认:

```bash
curl -i -X POST http://127.0.0.1:3000/
```

```
HTTP/1.1 405 Method Not Allowed
allow: GET
content-length: 0
```

### 步骤 2:接收上传的文件

**要做什么**:加上 `POST /` 处理,逐个取出字段并打印元信息。

```rust
// src/main.rs
use axum::{
    extract::Multipart,                       // 新增
    response::Html,
    routing::get,
    Router,
};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    // 改动:挂上 POST 分支
    let app = Router::new().route("/", get(show_form).post(accept_form));

    // ...(listener 和 serve 不变)
}

// ...(show_form 不变)

// 新增:逐个处理 multipart 字段
async fn accept_form(mut multipart: Multipart) {
    while let Some(field) = multipart.next_field().await.unwrap() {
        let name = field.name().unwrap().to_string();
        let file_name = field.file_name().unwrap().to_string();
        let content_type = field.content_type().unwrap().to_string();
        let data = field.bytes().await.unwrap();

        println!(
            "Length of `{name}` (`{file_name}`: `{content_type}`) is {} bytes",
            data.len()
        );
    }
}
```

`get(show_form).post(accept_form)` 是[第 02 章](02-routes-and-handlers-close-together.md#router-与-methodrouter-的分工)讲的 `MethodRouter` 链式写法。

`accept_form` 的循环是本章的核心:

**`mut multipart: Multipart`** —— 注意 `mut`。`next_field()` 要推进内部的流状态,所以需要可变借用。

**`multipart.next_field().await`** 返回 `Result<Option<Field>>`:`Ok(Some(field))` 是拿到了下一个字段,`Ok(None)` 是字段取完了,`Err` 是解析出错(格式非法、连接断开等)。`while let Some(...)` 循环到取完为止。

**四个 getter 的语义各不相同,这里有坑:**

- `field.name()` —— 表单字段名(`Content-Disposition` 里的 `name`)。返回 `Option`,理论上总是有的。
- `field.file_name()` —— 原始文件名。**只有文件字段才有**,普通文本字段返回 `None`。
- `field.content_type()` —— 该字段的 MIME 类型。**文本字段通常不带**,返回 `None`。
- `field.bytes().await` —— 读取该字段的全部内容。

**示例里四个 `.unwrap()` 全是隐患。** 只要表单里混进一个普通文本字段(比如加个 `<input type="text" name="title">`),`file_name()` 就返回 `None`,`unwrap()` 直接 panic。步骤 3 之后的"代价与边界"会详细说这件事。

**`field.bytes().await` 把这个字段的内容全部读进内存。** 对 250MB 的文件就是 250MB 内存。真实的文件上传应该边读边写磁盘,那是[第 40 章](40-stream-to-file.md) stream-to-file 的内容。

**handler 返回 `()`**,也就是 `200` + 空 body([第 01 章](01-hello-world.md#intoresponse返回值如何变成-http-响应)那张表)。示例只关心终端输出。

**跑起来看**:

```bash
cargo run
```

先造两个测试文件:

```bash
echo 'hello from file A' > /tmp/a.txt
head -c 100000 /dev/urandom > /tmp/b.bin
```

用 curl 上传。`-F` 参数会自动设置 `multipart/form-data` 编码和 boundary:

```bash
curl -i -X POST http://127.0.0.1:3000/ \
  -F 'file=@/tmp/a.txt' \
  -F 'file=@/tmp/b.bin'
```

```
HTTP/1.1 200 OK
content-length: 0
date: Thu, 30 Jul 2026 09:14:22 GMT
```

服务端终端:

```
Length of `file` (`a.txt`: `text/plain`) is 18 bytes
Length of `file` (`b.bin`: `application/octet-stream`) is 100000 bytes
```

**两个文件用了同一个字段名 `file`,分成两段发送**,循环各处理一次——这就是 HTML 里 `multiple` 属性的效果。

`content_type` 是 curl 根据扩展名猜的:`.txt` 猜成 `text/plain`,`.bin` 没有对应类型就用兜底的 `application/octet-stream`。**这个值完全由客户端提供,不可信** —— 攻击者可以把一个 `.exe` 声明成 `image/png`。

也试试浏览器:访问 `http://127.0.0.1:3000/`,选几个文件提交,终端会打印同样格式的信息。

**现在触发那个 panic。** 用 `-F 'title=hello'` 加一个纯文本字段(注意没有 `@`,不是文件):

```bash
curl -i -X POST http://127.0.0.1:3000/ -F 'title=hello'
```

```
curl: (52) Empty reply from server
```

服务端终端:

```
thread 'tokio-runtime-worker' panicked at src/main.rs:44:38:
called `Option::unwrap()` on a `None` value
```

**`file_name()` 返回 `None`,`unwrap()` 炸了。** 客户端收到的是一个空回复,连状态码都没有。axum 会捕获 panic 保证整个服务不倒,但这个请求彻底失败了。练习题 1 会修它。

### 步骤 3:放开大小限制

**要做什么**:先撞一次默认的 2MB 上限,再用两层限制机制放开到 250MB。

**先看默认限制。** 造一个 3MB 的文件:

```bash
head -c 3000000 /dev/urandom > /tmp/big.bin
```

```bash
curl -i -X POST http://127.0.0.1:3000/ -F 'file=@/tmp/big.bin'
```

```
HTTP/1.1 413 Payload Too Large
content-type: text/plain; charset=utf-8
content-length: 30

length limit exceeded
```

**这就是 `DefaultBodyLimit` 的 2MB 默认值在起作用**,前置知识讲过。对文件上传服务来说显然不够用。

现在改:

```rust
// src/main.rs
use axum::{
    extract::{DefaultBodyLimit, Multipart},        // 改动:加上 DefaultBodyLimit
    response::Html,
    routing::get,
    Router,
};
use tower_http::limit::RequestBodyLimitLayer;      // 新增
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    // 改动:关掉默认限制,改用外层中间件管控;顺带加上请求日志
    let app = Router::new()
        .route("/", get(show_form).post(accept_form))
        .layer(DefaultBodyLimit::disable())
        .layer(RequestBodyLimitLayer::new(
            250 * 1024 * 1024, /* 250mb */
        ))
        .layer(tower_http::trace::TraceLayer::new_for_http());

    // ...(listener 和 serve 不变)
}

// ...(其余不变)
```

三个 `.layer()` 的作用前置知识讲过两个:`DefaultBodyLimit::disable()` 关掉提取器层面的 2MB 限制,`RequestBodyLimitLayer::new(250MB)` 在更外层设一个新上限。第三个 `TraceLayer` 是请求日志,见上面的"本章不深入"。

**这两步必须成对出现。** 只写 `disable()` 不写 `RequestBodyLimitLayer`,服务就变成了完全无限制——这是个真实的拒绝服务漏洞。

**跑起来看**:

```bash
cargo run
```

3MB 文件现在能过了:

```bash
curl -i -X POST http://127.0.0.1:3000/ -F 'file=@/tmp/big.bin'
```

```
HTTP/1.1 200 OK
content-length: 0
```

```
Length of `file` (`big.bin`: `application/octet-stream`) is 3000000 bytes
```

而且因为加了 `TraceLayer`,终端多了请求日志:

```
2026-07-30T09:22:31.104223Z DEBUG request{method=POST uri=/ version=HTTP/1.1}: tower_http::trace::on_request: started processing request
2026-07-30T09:22:31.118742Z DEBUG request{method=POST uri=/ version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=14 ms status=200
```

**再撞一次新的上限。** 造一个 260MB 的文件试试(这会花点时间,也会真的占用磁盘):

```bash
head -c 260000000 /dev/zero > /tmp/huge.bin
curl -i -X POST http://127.0.0.1:3000/ -F 'file=@/tmp/huge.bin'
```

```
HTTP/1.1 413 Payload Too Large
content-length: 0
```

`RequestBodyLimitLayer` 拦住了。清理一下:

```bash
rm /tmp/huge.bin /tmp/big.bin /tmp/a.txt /tmp/b.bin
```

**验证一下"只 disable 不加 limit"的后果。** 把 `RequestBodyLimitLayer` 那一行临时注释掉,重新运行,再传那个 260MB 的文件——它会被完整接收,而且**全部进入内存**(因为 `field.bytes()`)。用 `top` 或活动监视器盯着进程内存,你会看到它涨上去。

这就是前置知识强调"不能只做第一步"的原因。改回来。

## 完整代码

`src/main.rs`:

```rust
use axum::{
    extract::{DefaultBodyLimit, Multipart},
    response::Html,
    routing::get,
    Router,
};
use tower_http::limit::RequestBodyLimitLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // build our application with some routes
    let app = Router::new()
        .route("/", get(show_form).post(accept_form))
        .layer(DefaultBodyLimit::disable())
        .layer(RequestBodyLimitLayer::new(
            250 * 1024 * 1024, /* 250mb */
        ))
        .layer(tower_http::trace::TraceLayer::new_for_http());

    // run it with hyper
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn show_form() -> Html<&'static str> {
    Html(
        r#"
        <!doctype html>
        <html>
            <head></head>
            <body>
                <form action="/" method="post" enctype="multipart/form-data">
                    <label>
                        Upload file:
                        <input type="file" name="file" multiple>
                    </label>

                    <input type="submit" value="Upload files">
                </form>
            </body>
        </html>
        "#,
    )
}

async fn accept_form(mut multipart: Multipart) {
    while let Some(field) = multipart.next_field().await.unwrap() {
        let name = field.name().unwrap().to_string();
        let file_name = field.file_name().unwrap().to_string();
        let content_type = field.content_type().unwrap().to_string();
        let data = field.bytes().await.unwrap();

        println!(
            "Length of `{name}` (`{file_name}`: `{content_type}`) is {} bytes",
            data.len()
        );
    }
}
```

## 代价与边界

这个示例是官方示例里隐患最集中的一个,照抄进生产环境会出事。逐条说。

**四个 `.unwrap()` 会 panic。** 步骤 2 已经复现过:表单里有非文件字段时 `file_name()` 返回 `None`,直接 panic。`content_type()` 同理。`next_field()` 的 `unwrap` 更危险——客户端中途断连、发送格式非法的 multipart 都会触发。**正确做法是用 `Option`/`Result` 的组合子处理,或者返回 `Result<_, StatusCode>` 让错误变成响应**,见练习题 1。

**`field.bytes()` 把整个文件读进内存。** 250MB 的上限意味着单个请求最多吃 250MB 内存;10 个并发上传就是 2.5GB。这个数字必须结合你的服务器内存和预期并发量来定,不能照抄。**大文件上传的正确做法是流式写盘**,见[第 40 章](40-stream-to-file.md) stream-to-file。

**`file_name()` 的值完全由客户端控制,直接拿它当路径是严重漏洞。** 攻击者可以传 `filename="../../etc/passwd"` 或 `filename="../../../var/www/shell.php"`,如果你写 `File::create(file_name)`,文件就落到了任意位置。这叫**路径穿越**(path traversal)。防御办法:自己生成文件名(UUID 之类),只把原始文件名当作元数据存起来;或者用 `std::path::Path::file_name()` 剥掉目录部分并严格校验。

**`content_type()` 同样不可信。** 它来自客户端声明的 `Content-Type`,和文件实际内容毫无关系。要判断真实类型得读文件头的**魔数**(magic number),比如 PNG 以 `\x89PNG` 开头。靠客户端声明的类型做安全决策(比如"是图片就直接展示")会被绕过。

**没有限制字段数量。** 攻击者可以发送一个包含十万个微小字段的 multipart 请求,每个字段都合法,但循环十万次也够呛。真实服务应该计数并设上限。

**`250 * 1024 * 1024` 这个数字要按场景改。** 头像上传给 5MB 足矣,视频上传可能需要更大——但那种量级更应该走对象存储的直传方案,而不是过你的服务器。

一句话总结:**这个示例演示的是 `Multipart` API 怎么用,不是文件上传该怎么做。** 真实的上传功能至少还需要:错误处理、流式落盘、文件名净化、类型校验、病毒扫描(视场景)、配额限制。

## 动手练习

**1. 干掉所有 `unwrap`,让非文件字段也能正常处理。**

提示:handler 返回类型改成 `Result<String, (StatusCode, String)>`。`next_field()` 的错误用 `.map_err(...)?` 转成 `400`;`file_name()` 和 `content_type()` 返回 `Option`,用 `.unwrap_or("<none>")` 或 `.map(...)` 给默认值。改完用 `curl -X POST http://127.0.0.1:3000/ -F 'title=hello' -F 'file=@/tmp/a.txt'` 验证——文本字段和文件字段应该都能打印出来,不再 panic。

**2. 统计并返回上传汇总,而不是只打印到终端。**

提示:在循环里累加文件数和总字节数,循环结束后返回 `Html(format!(...))`。这样浏览器提交后能看到结果,不用切到终端看日志。注意 `Multipart` 是流式的,汇总信息只能在循环**之后**才知道。

**3. 加一个字段数量上限,防止十万字段攻击。**

提示:循环里用计数器,超过 20 就 `return Err((StatusCode::BAD_REQUEST, "too many fields".into()))`。做完想想:这个检查能防住"十万个字段"的攻击吗?(提示:能——因为流式处理下,你在第 21 个字段就返回了,剩下的根本不会被解析。这正是流式 API 的一个优势。)

## 小结与下一章预告

`multipart/form-data` 用随机 boundary 把请求体切成若干段,每段有自己的头部和原始二进制内容——这是文件上传唯一实用的编码方式,HTML 表单要写 `enctype="multipart/form-data"` 才会启用,漏写不会报错但文件传不上来。

`Multipart` 提取器是**流式**的,和 `Form`/`Json` 一次性反序列化的模式完全不同:提取时不读任何数据,`next_field().await` 才向前推进。代价是字段只能顺序处理、不能跳过;好处是可以在读完整个请求之前就做出判断和拒绝。

请求体限制有两层:`DefaultBodyLimit` 默认 2MB、作用于提取器;`RequestBodyLimitLayer` 是 `tower-http` 中间件、作用于更外层,连绕过提取器的代码也管得住。放开上传限制的正确姿势是 `disable()` 加 `RequestBodyLimitLayer` **成对使用**,只做前一半就是拒绝服务漏洞。

官方示例那四个 `unwrap` 和"文件名直接可信"的假设都是真实隐患,"代价与边界"一节逐条列了,修复在练习题里。

到这里「请求与响应」阶段结束——你已经能应对各种形态的请求输入,也知道怎么定制拒绝响应。但所有这些章节的错误处理都是零散的:每章各写各的错误类型,`.unwrap()` 还散落在各处。[第 14 章 error-handling](14-error-handling.md) 开始「错误处理与工程化」阶段,系统地讲一个 axum 应用该如何组织错误:什么时候用 `Result`、错误类型怎么设计、如何让 `?` 贯穿整条调用链、以及 handler 之外的错误(中间件、超时、panic)该怎么接住。
