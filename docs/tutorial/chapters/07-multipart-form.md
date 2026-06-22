# 07. multipart-form

对应示例：`examples/multipart-form`

第 6 章处理普通表单（`application/x-www-form-urlencoded`）。这章处理 **multipart/form-data**——HTTP 上传文件的标准格式，一个表单同时传多个字段（包括文件），每个文件有元数据（文件名、content-type）。

分 2 步：先写最简 multipart handler（接收上传文件），再说明怎么去掉 `DefaultBodyLimit` 提高大文件上传上限。

相比前面章节新引入：**`Multipart` extractor、`multipart.next_field()`、`field.bytes()`、`DefaultBodyLimit`、`RequestBodyLimitLayer`**。

## Cargo.toml

````toml
[package]
name = "example-multipart-form"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["macros"] }
tokio = { version = "1.0", features = ["full"] }
tower-http = { version = "0.6", features = ["limit", "trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：表单页 + `Multipart` handler

先写一个 HTML 表单页（让用户选文件），POST 提交到 multipart handler。handler 用 `Multipart` extractor 逐个 field 读取。

````rust
use axum::{
    extract::Multipart,
    response::Html,
    routing::get,
    Router,
};
use tower_http::trace::TraceLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(/* tracing 初始化 */)
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app = Router::new()
        .route("/", get(show_form).post(accept_form))
        .layer(TraceLayer::new_for_http());

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
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
````

验证：

````bash
cd examples
cargo run -p example-multipart-form
````

浏览器打开 `http://127.0.0.1:3000/`，选文件上传，服务端日志看到每个文件的元数据和大小。

也可以用 curl：

````bash
curl -X POST http://127.0.0.1:3000/ -F "file=@some.txt" -F "file=@another.png"
````

> **新面孔：`Multipart` extractor**
>
> axum 内置的 multipart 解析器。实现 `FromRequest`，作为 handler 参数自动提取。
>
> `multipart` 是 `multipart::Multipart`（来自 `axum::extract::Multipart`，基于 `multer` crate）。它是个流式 iterator——每个 `next_field().await` 返回下一个 field，避免大文件一次性进内存。

> **新面孔：`multipart.next_field()` 流式遍历**
>
> multipart 请求包含多个 field（字段），每个 field 可能是普通文本字段（`name=alice`）或文件（含 filename + content-type + bytes）。
>
> `while let Some(field) = multipart.next_field().await` 一个个遍历所有 field。`field` 提供：
> - `.name()`：字段名（如 "file"）
> - `.file_name()`：文件名（如 "photo.jpg"，普通字段是 None）
> - `.content_type()`：MIME 类型（如 "image/jpeg"）
> - `.bytes().await`：field 内容（流式收集）

> **新面孔：HTML `enctype="multipart/form-data"`**
>
> 表单上传文件**必须**设 `enctype="multipart/form-data"`，否则浏览器用默认 `application/x-www-form-urlencoded` 编码（不适合二进制）。`<input type="file" multiple>` 允许多选。

> **新面孔：`-F` curl 选项**
>
> curl 的 `-F "file=@path"` 上传文件（`@` 前缀表示文件路径）。多个 `-F` 上传多个 field。等价于浏览器表单的 `multipart/form-data` 提交。

---

## 第二步：去掉默认 body limit 支持大文件

axum 默认 body 大小上限是 **2MB**（防止内存爆炸）。上传大文件（如视频）需要去掉这个限制，换成自定义上限。

````rust
use axum::extract::DefaultBodyLimit;
use tower_http::limit::RequestBodyLimitLayer;

# #[tokio::main]
# async fn main() {
#     // ...
    let app = Router::new()
        .route("/", get(show_form).post(accept_form))
        .layer(DefaultBodyLimit::disable())               // 关掉 axum 默认 2MB 限制
        .layer(RequestBodyLimitLayer::new(                // 设 tower-http 的限制（更大）
            250 * 1024 * 1024, /* 250mb */
        ))
        .layer(TraceLayer::new_for_http());
#     // ...
# }
````

> **新面孔：`DefaultBodyLimit`**
>
> axum 内置的 body 大小限制 layer，默认 2MB。`DefaultBodyLimit::disable()` 完全关掉（让 `Multipart` 自己用流式处理大小），或 `DefaultBodyLimit::max(100 * 1024 * 1024)` 设自定义上限。
>
> 为什么需要关掉？因为 multipart 总 body 大小 = 所有 field 大小之和，单看 body size 限制不到 2MB 没法上传文件。

> **新面孔：`RequestBodyLimitLayer`**
>
> tower-http 的请求大小限制 layer。和 `DefaultBodyLimit` 区别：
>
> - `DefaultBodyLimit`：axum 自带，extractor 解析 body 前检查
> - `RequestBodyLimitLayer`：tower-http，在更底层（HTTP 层）检查
>
> 这章两个都用：`DefaultBodyLimit::disable()` 关掉 axum 限制，`RequestBodyLimitLayer` 设新上限。这样 multipart 流式处理可以接收任意大小，但有 250MB 总上限防爆。

### 为什么 multipart 能处理大文件

```text
普通 Json/Form：
    一次性把整个 body 读进内存 → body 大就 OOM

Multipart：
    next_field() 一个个流式返回 → 每次只在内存里放一个 field
    field.bytes().await 收集完一个再处理下一个
    可以边收边写文件（ch08 stream-to-file 那样）
```

所以大文件上传必须用 multipart，不能用普通 Form。

---

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

浏览器打开 `http://127.0.0.1:3000/`，选一个或多个文件上传。服务端日志看到：

```text
Length of `file` (`photo.jpg`: `image/jpeg`) is 102400 bytes
Length of `file` (`doc.pdf`: `application/pdf`) is 51200 bytes
```

## 解读

### multipart/form-data 格式

```text
POST / HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryXXX

------WebKitFormBoundaryXXX
Content-Disposition: form-data; name="file"; filename="photo.jpg"
Content-Type: image/jpeg

<二进制内容>
------WebKitFormBoundaryXXX
Content-Disposition: form-data; name="file"; filename="doc.pdf"
Content-Type: application/pdf

<二进制内容>
------WebKitFormBoundaryXXX--
```

每个 field 用 boundary 分隔，自带元数据（filename、content-type）。比 form-urlencoded 多了 filename 和 content-type，适合二进制。

### 为什么上传文件必须用 multipart

```text
application/x-www-form-urlencoded:
    key1=value1&key2=value2
    二进制要 base64 编码（膨胀 33%），filename/content-type 没地方放

multipart/form-data:
    每 field 独立块，二进制原样放，自带元数据
    ✓ 文件上传标准
```

## 常见问题

**默认 2MB 限制怎么改？** `DefaultBodyLimit::max(10 * 1024 * 1024)` 或这章 `DefaultBodyLimit::disable()` + `RequestBodyLimitLayer`。

**怎么把上传文件保存到磁盘？** `tokio::fs::write(path, data).await` 保存 `field.bytes()` 的内容。或流式写（参考 ch08 stream-to-file）。

**怎么限制文件类型？** 检查 `field.content_type()`，不在白名单的拒绝（返回错误）。注意只检查 content-type 不安全（可伪造），生产环境要嗅探文件头（magic number）。

## 手写任务

1. 把上传的文件保存到 `./uploads/` 目录（`tokio::fs::write`）。
2. 加文件类型白名单：只接受 image/* 和 application/pdf。
3. 加 progress：每个 chunk 累计 bytes，handler 流式返回 progress（需要 SSE 配合）。
4. 改成流式写文件（不全部进内存）：用 `field.chunk()` 流式读，边读边写文件。

## 小结

这章用 2 步讲了文件上传：

1. **`Multipart` extractor**：流式遍历 `next_field()`，每个 field 带 name/filename/content-type/bytes。
2. **去掉 body limit**：`DefaultBodyLimit::disable()` + `RequestBodyLimitLayer::new(250MB)` 支持大文件上传。

核心：multipart 是文件上传标准，axum 的 `Multipart` extractor 流式处理避免大文件 OOM；body limit 必须显式调整（默认 2MB）。

## 源码对照

- `examples/multipart-form/Cargo.toml`
- `examples/multipart-form/src/main.rs`
