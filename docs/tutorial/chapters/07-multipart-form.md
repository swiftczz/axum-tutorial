# 07. multipart-form

对应示例：`examples/multipart-form`

本章目标：手写 multipart 文件上传表单，理解 `Multipart` 和请求体大小限制。

## 这个小项目在做什么

上一章的表单只提交短文本，body 类似：

```text
name=foo&email=bar@axum
```

这一章提交的是文件。文件可能很大，也可能一次上传多个，所以不能再用普通 `Form<T>` 处理。

这个 example 演示：

- `GET /` 返回一个文件上传表单。
- 表单使用 `enctype="multipart/form-data"`。
- 用户选择一个或多个文件后提交。
- `POST /` 用 `Multipart` 逐个读取上传字段。
- 服务端打印每个文件的字段名、文件名、content type 和字节数。
- Router 上加 body limit，避免无限制接收大请求。

请求主线是：

```text
浏览器访问 GET /
-> show_form 返回上传表单
-> 用户选择文件并提交
-> 浏览器发送 multipart/form-data 请求
-> accept_form 逐个读取 multipart field
-> 打印文件信息
```

这一章的重点是理解：文件上传不是普通 JSON，也不是上一章的普通表单，它有自己的 multipart 格式。

## 先理解普通表单和 multipart 表单的区别

普通表单适合短文本：

```text
content-type: application/x-www-form-urlencoded
body: name=foo&email=bar@axum
```

文件上传表单使用 multipart：

```text
content-type: multipart/form-data; boundary=...
body: 分成多个 part，每个 part 有自己的 header 和内容
```

可以先这样记：

| 场景 | HTML 写法 | Axum 提取器 |
| --- | --- | --- |
| 提交短文本 | 普通 `<form method="post">` | `Form<T>` |
| 上传文件 | `<form method="post" enctype="multipart/form-data">` | `Multipart` |

`Multipart` 不是一次把所有字段解析成一个 struct，而是让你逐个读取字段。这样更适合文件上传。

## 文件和依赖

这个 example 有两个文件：

1. `examples/multipart-form/Cargo.toml`：声明 Axum multipart feature、Tokio、tower-http 和 tracing。
2. `examples/multipart-form/src/main.rs`：实现上传表单、multipart 处理和 body limit。

关键依赖：

- `axum`：提供 `Router`、`Html`、`Multipart`、`DefaultBodyLimit`。
- `tokio`：提供异步运行时。
- `tower-http`：提供请求体大小限制和 HTTP trace 日志。
- `tracing` / `tracing-subscriber`：初始化日志。

这里的 `Cargo.toml` 有一个细节：

````toml
axum = { path = "../../axum", features = ["multipart"] }
````

`Multipart` 需要启用 Axum 的 `multipart` feature。

## 第一步：创建 Router 并设置 body limit

核心代码：

````rust
let app = Router::new()
    .route("/", get(show_form).post(accept_form))
    .layer(DefaultBodyLimit::disable())
    .layer(RequestBodyLimitLayer::new(
        250 * 1024 * 1024, /* 250mb */
    ))
    .layer(tower_http::trace::TraceLayer::new_for_http());
````

先看路由：

```text
GET /  -> show_form
POST / -> accept_form
```

再看三个 layer：

- `DefaultBodyLimit::disable()`：关闭 Axum 默认 body limit。
- `RequestBodyLimitLayer::new(250 * 1024 * 1024)`：设置新的请求体上限，约 250MB。
- `TraceLayer::new_for_http()`：给 HTTP 请求加 trace 日志。

为什么要关掉默认限制再设置新的限制？

因为文件上传通常比普通 JSON 或普通表单大。  
但完全不限制也危险，所以示例改成一个更大的上限，而不是无限制接收。

## 第二步：写上传表单页面

源码：

````rust
async fn show_form() -> Html<&'static str> {
    Html(r#"... multipart form ..."#)
}
````

HTML 里最关键的是：

````html
<form action="/" method="post" enctype="multipart/form-data">
    <input type="file" name="file" multiple>
</form>
````

逐个看：

- `action="/"`：提交到 `/`。
- `method="post"`：用 POST 上传。
- `enctype="multipart/form-data"`：告诉浏览器按 multipart 格式编码请求体。
- `type="file"`：这是文件选择框。
- `name="file"`：上传字段名叫 `file`。
- `multiple`：允许一次选择多个文件。

没有 `enctype="multipart/form-data"`，浏览器不会按文件上传格式发送，后端的 `Multipart` 就接不到正确内容。

## 第三步：用 Multipart 接收上传内容

源码：

````rust
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
````

`Multipart` 是一个 extractor。它告诉 Axum：

```text
这个 handler 要读取 multipart/form-data 请求体。
```

为什么参数是 `mut multipart`？

因为读取 multipart field 是一个逐步消耗请求体的过程。每调用一次 `next_field()`，就从请求体里取出下一个 part。

循环逻辑是：

```text
读取下一个 field
-> 取 field name
-> 取上传文件名
-> 取 content type
-> 读取这个 field 的全部字节
-> 打印字节数
```

注意：示例里用了很多 `.unwrap()`。这是为了让官方 example 保持短小。真实项目里应该处理错误，例如文件名不存在、content type 不合法、读取失败等情况。

## 第四步：理解 field 和文件

multipart 请求里的每个 part 都可以看成一个 field。  
文件上传时，一个 field 通常包含：

- 字段名：来自 HTML 的 `name="file"`。
- 文件名：用户选择的本地文件名。
- content type：浏览器根据文件推断出的类型。
- 文件内容：真正的字节数据。

本章示例只打印这些信息，没有把文件保存到磁盘。  
下一章 `stream-to-file` 会继续处理“如何把上传内容写入文件”。

## 函数职责速查

- `main`：初始化日志，注册上传路由，设置 body limit 和 trace layer，启动服务。
- `show_form`：返回文件上传 HTML 页面。
- `accept_form`：用 `Multipart` 逐个读取上传字段，并打印文件信息。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-multipart-form
//! ```

// 引入默认 body limit 控制、Multipart 提取器、Html 响应、GET 路由和 Router。
use axum::{
    extract::{DefaultBodyLimit, Multipart},
    response::Html,
    routing::get,
    Router,
};
// tower-http 提供请求体大小限制 layer。
use tower_http::limit::RequestBodyLimitLayer;
// tracing_subscriber 用来初始化日志。
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 初始化日志系统。
    tracing_subscriber::registry()
        .with(
            // 没有环境变量时，默认当前 crate 和 tower_http 都输出 debug 日志。
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        // 把日志格式化输出到终端。
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 创建 Router：GET / 展示上传表单，POST / 接收 multipart 上传。
    let app = Router::new()
        .route("/", get(show_form).post(accept_form))
        // 关闭 Axum 默认 body limit。
        .layer(DefaultBodyLimit::disable())
        // 设置新的请求体上限，这里是 250MB。
        .layer(RequestBodyLimitLayer::new(
            250 * 1024 * 1024, /* 250mb */
        ))
        // 给 HTTP 请求加 trace 日志。
        .layer(tower_http::trace::TraceLayer::new_for_http());

    // 绑定本机 3000 端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 输出监听地址。
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动 HTTP 服务。
    axum::serve(listener, app).await;
}

// GET / 的 handler，返回文件上传表单。
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

// POST / 的 handler，接收 multipart/form-data 请求体。
async fn accept_form(mut multipart: Multipart) {
    // 不断读取下一个 field，直到没有更多 field。
    while let Some(field) = multipart.next_field().await.unwrap() {
        // 读取字段名，例如 HTML 里的 name="file"。
        let name = field.name().unwrap().to_string();
        // 读取上传文件名。
        let file_name = field.file_name().unwrap().to_string();
        // 读取 content type。
        let content_type = field.content_type().unwrap().to_string();
        // 读取这个 field 的全部字节内容。
        let data = field.bytes().await.unwrap();

        // 打印上传文件信息。
        println!(
            "Length of `{name}` (`{file_name}`: `{content_type}`) is {} bytes",
            data.len()
        );
    }
}
````

## 运行和验证

运行前先确认 `examples/multipart-form/Cargo.toml` 里的 `axum = { path = "../../axum", features = ["multipart"] }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-multipart-form
````

浏览器访问：

````text
http://127.0.0.1:3000/
````

选择一个文件并提交。终端应该看到类似输出：

````text
Length of `file` (`hello.txt`: `text/plain`) is 123 bytes
````

也可以用 curl 上传文件：

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -F 'file=@Cargo.toml'
````

常见卡点：

- HTML 表单必须写 `enctype="multipart/form-data"`。
- `axum` 依赖必须启用 `multipart` feature。
- 上传文件可能超过默认 body limit，所以示例显式调整了 limit。
- 这个 example 只打印文件信息，不会把文件保存到磁盘。

## 手写任务

完成本章代码后，试着做三个小观察：

1. 删除 HTML 里的 `multiple`，观察浏览器文件选择行为有什么变化。
2. 把 body limit 从 250MB 改小，例如 `1024`，再尝试上传超过 1KB 的文件。
3. 在 `println!` 里额外打印字段名 `name`，确认它来自 HTML 的 `name="file"`。

## 本章真正要记住什么

- 文件上传通常使用 `multipart/form-data`。
- `Multipart` 适合逐个读取上传字段和文件。
- 文件上传一定要考虑请求体大小限制。
- `field.bytes().await` 会把当前 field 的内容读进内存；大文件保存到磁盘时，后续要学习流式写入。
- 本章只读取并打印文件信息，下一章会处理写文件。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/multipart-form/Cargo.toml`
- `examples/multipart-form/src/main.rs`
