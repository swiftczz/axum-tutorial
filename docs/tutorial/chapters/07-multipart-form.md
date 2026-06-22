# 07. multipart-form

对应示例：`examples/multipart-form`

上一章的表单只提交短文本。本章上传文件——文件可能很大,也可能一次传多个,所以不能用普通 `Form<T>`,改用 `Multipart` 逐个读取字段。



相比前面章节新引入：**`Multipart` extractor、body limit（`DefaultBodyLimit`）、逐个 field 读取**。

## Cargo.toml

````toml
[package]
name = "example-multipart-form"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["multipart"] }
tokio = { version = "1.0", features = ["full"] }
tower-http = { version = "0.6.1", features = ["limit", "trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

`Multipart` 需要启用 axum 的 `multipart` feature;`tower-http` 提供 body limit layer。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 关键概念

> **新面孔：`Multipart` extractor**
>
> 和 `Form<T>`（ch06）类似但更强大：`Form<T>` 一次性把所有字段解析成结构体，`Multipart` 逐个 field 读取，适合文件上传。需要启用 axum 的 `multipart` feature。

> **新面孔：`DefaultBodyLimit` + `RequestBodyLimitLayer`**
>
> axum 默认对请求体设上限（防 DoS）。文件上传通常更大，先 `DefaultBodyLimit::disable()` 关闭默认，再设新上限。


## 完整代码

````rust
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

    let app = Router::new()
        .route("/", get(show_form).post(accept_form))
        .layer(DefaultBodyLimit::disable())
        .layer(RequestBodyLimitLayer::new(
            250 * 1024 * 1024, /* 250mb */
        ))
        .layer(tower_http::trace::TraceLayer::new_for_http());

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app).await;
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
````

## 运行

````bash
cd examples
cargo run -p example-multipart-form
````

浏览器访问 `http://127.0.0.1:3000/`,选文件提交。终端看到:

````text
Length of `file` (`hello.txt`: `text/plain`) is 123 bytes
````

用 curl 上传:

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -F 'file=@Cargo.toml'
````

## 解读

### 普通表单 vs multipart 表单

```text
普通表单:    content-type: application/x-www-form-urlencoded
             body: name=foo&email=bar@axum          → 用 Form<T>

文件上传表单: content-type: multipart/form-data; boundary=...
             body: 分成多个 part,每个 part 有自己的 header 和内容  → 用 Multipart
```

`Multipart` 不一次把所有字段解析成一个 struct,而是让你逐个读取 field——这样更适合文件上传。

### HTML 表单的 `enctype`

````html
<form action="/" method="post" enctype="multipart/form-data">
    <input type="file" name="file" multiple>
</form>
````

`enctype="multipart/form-data"` 告诉浏览器按 multipart 格式编码请求体。没有它,后端的 `Multipart` 接不到正确内容。`multiple` 允许一次选多个文件。

### body limit

````rust
.layer(DefaultBodyLimit::disable())
.layer(RequestBodyLimitLayer::new(250 * 1024 * 1024))
````

axum 默认对请求体设上限(防止 DoS)。文件上传通常比 JSON 大,所以先关掉默认限制,再设一个更大的 250MB 上限。完全不限制是危险的,所以示例改成"更大的上限"而不是"无限制"。

### `Multipart` 逐个读 field

````rust
async fn accept_form(mut multipart: Multipart) {
    while let Some(field) = multipart.next_field().await.unwrap() {
        ...
    }
}
````

`Multipart` 是 extractor,但参数是 `mut multipart`——因为读取 field 是逐步消耗请求体的过程。每调一次 `next_field()` 取下一个 part,直到 `None`。

每个 field 包含:

| 信息 | 来源 |
| --- | --- |
| 字段名 (`field.name()`) | HTML 的 `name="file"` |
| 文件名 (`field.file_name()`) | 用户选的本地文件名 |
| content type (`field.content_type()`) | 浏览器推断的类型 |
| 内容 (`field.bytes().await`) | 真正的字节数据 |

示例里大量 `.unwrap()` 是为了官方 example 保持短小,真实项目要处理文件名缺失、读取失败等错误。

本章只读取并打印文件信息,不存盘。下一章 `stream-to-file` 处理"如何把上传内容写入文件"。

## 手写任务

跑通后做三个小观察:

1. 删除 HTML 里的 `multiple`,观察文件选择行为变化。
2. 把 body limit 从 250MB 改成 `1024`,再上传超过 1KB 的文件,观察被拒绝。
3. 在 `println!` 里额外打印 `name`,确认它来自 HTML 的 `name="file"`。

## 小结

- 文件上传用 `multipart/form-data`,不是普通表单也不是 JSON。
- `Multipart` 逐个读 field,适合大文件和一次上传多个文件。
- 启用 axum 的 `multipart` feature 才能用 `Multipart`。
- 文件上传必须考虑请求体大小限制,通常先 `DefaultBodyLimit::disable()` 再设新上限。
- `field.bytes().await` 把内容读进内存;大文件流式写盘见下一章。

## 源码对照

- `examples/multipart-form/Cargo.toml`
- `examples/multipart-form/src/main.rs`
