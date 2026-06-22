# 08. stream-to-file

对应示例：`examples/stream-to-file`

上一章 multipart-form 只读取并打印上传文件的信息（`field.bytes().await` 把整个文件读进内存）。本章真正把上传内容**流式写入磁盘**——边读边写，内存占用恒定，适合大文件。同时处理一个安全问题：校验文件名，防止路径穿越。

代码约 100 行，分 3 步搭。

## Cargo.toml

````toml
[package]
name = "example-stream-to-file"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["multipart"] }
futures-util = { version = "0.3", default-features = false, features = ["sink", "std"] }
tokio = { version = "1.0", features = ["full"] }
tokio-util = { version = "0.7", features = ["io"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

- `tokio-util`：提供 `StreamReader`，把字节 stream 适配成 `AsyncRead`。
- `futures-util`：stream 相关工具（注意是 `futures-util` 不是 `futures`）。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：直接把请求 body 流式写盘

先写最简单的入口：`POST /file/{file_name}` 把整个请求体原样写成一个文件。核心问题是**流式写入**——不把整个文件一次性读进内存。

````rust
use axum::{
    body::Bytes,
    extract::{Path, Request},
    http::StatusCode,
    routing::post,
    BoxError, Router,
};
use futures_util::{Stream, TryStreamExt};
use std::{io, pin::pin};
use tokio::{fs::File, io::BufWriter};
use tokio_util::io::StreamReader;

const UPLOADS_DIRECTORY: &str = "uploads";

#[tokio::main]
async fn main() {
    tokio::fs::create_dir(UPLOADS_DIRECTORY)
        .await
        .expect("failed to create `uploads` directory");

    let app = Router::new().route("/file/{file_name}", post(save_request_body));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await;
}

async fn save_request_body(
    Path(file_name): Path<String>,
    request: Request,
) -> Result<(), (StatusCode, String)> {
    stream_to_file(&file_name, request.into_body().into_data_stream()).await
}

async fn stream_to_file<S, E>(path: &str, stream: S) -> Result<(), (StatusCode, String)>
where
    S: Stream<Item = Result<Bytes, E>>,
    E: Into<BoxError>,
{
    async {
        let body_with_io_error = stream.map_err(io::Error::other);
        let mut body_reader = pin!(StreamReader::new(body_with_io_error));

        let path = std::path::Path::new(UPLOADS_DIRECTORY).join(path);
        let mut file = BufWriter::new(File::create(path).await?);

        tokio::io::copy(&mut body_reader, &mut file).await?;

        Ok::<_, io::Error>(())
    }
    .await
    .map_err(|err| (StatusCode::INTERNAL_SERVER_ERROR, err.to_string()))
}
````

跑起来验证：

````bash
cargo run -p example-stream-to-file
curl -i -X POST http://127.0.0.1:3000/file/hello.txt --data-binary 'hello from axum'
cat uploads/hello.txt  # 预期: hello from axum
````

> **新面孔：`tokio::io::copy` + `BufWriter`**
>
> `tokio::io::copy` 是异步版"从 reader 拷贝到 writer"：循环读一块、写一块，直到读完。这是流式写入的核心——请求体被切成一块块 `Bytes`，每来一块写一块，不需要把整个文件同时放进内存。上传 2GB 文件也只占恒定内存。
>
> `BufWriter` 让零散的小写入先攒进缓冲再批量落盘，减少 syscall 次数。

> **新面孔：`StreamReader` 和 `pin!`**
>
> `tokio::io::copy` 要求 reader 实现 `AsyncRead`，但请求体是"不断产出 `Bytes` 的 `Stream`"。`tokio_util::io::StreamReader` 把字节 stream 适配成 `AsyncRead`。
>
> `tokio::io::copy` 还要求 reader 是 `Unpin`（可以被安全移动）。但 `StreamReader` 内部是状态机、可能自引用，不满足 `Unpin`。`std::pin::pin!` 把它"钉"在栈上、地址固定，编译器才允许调用。第一遍先记住"需要 `Unpin` 但类型不满足时，用 `pin!` 固定"。

---

## 第二步：加 multipart 表单上传，复用 `stream_to_file`

浏览器上传文件用 multipart 表单（ch07 讲过）。现在加 `GET /` 返回表单、`POST /` 接收 multipart，**把每个 field 直接交给同一个 `stream_to_file` 写盘**。

````rust
use axum::{
    extract::Multipart,
    response::{Html, Redirect},
    routing::get,
};

async fn show_form() -> Html<&'static str> {
    Html(r#"
        <!doctype html>
        <html>
            <head><title>Upload something!</title></head>
            <body>
                <form action="/" method="post" enctype="multipart/form-data">
                    <div><label>Upload file: <input type="file" name="file" multiple></label></div>
                    <div><input type="submit" value="Upload files"></div>
                </form>
            </body>
        </html>
    "#)
}

async fn accept_form(mut multipart: Multipart) -> Result<Redirect, (StatusCode, String)> {
    while let Ok(Some(field)) = multipart.next_field().await {
        let file_name = if let Some(file_name) = field.file_name() {
            file_name.to_owned()
        } else {
            continue;
        };
        stream_to_file(&file_name, field).await?;
    }
    Ok(Redirect::to("/"))
}
````

路由更新：同一路径 `/` 挂 GET 和 POST。

````rust
let app = Router::new()
    .route("/", get(show_form).post(accept_form))
    .route("/file/{file_name}", post(save_request_body));
````

> **新面孔：`field` 本身就是 `Stream`**
>
> ch07 用 `field.bytes().await` 把整个文件读进内存再打印。但 multipart 的 `field` 同时实现了 `Stream<Item = Result<Bytes, _>>`——正是 `stream_to_file` 接受的输入类型。所以这里**直接把 `field` 传给 `stream_to_file`**，不经过 `bytes()`，省掉整份内存拷贝。这就是 ch07 末尾预告的"流式版本"。

---

## 第三步：路径校验，防路径穿越

文件名来自用户，不可信。`POST /file/../Cargo.toml` 会拼成 `uploads/../Cargo.toml`，覆盖项目根的 `Cargo.toml`——**任意文件被覆盖**。这就是路径穿越攻击。

````rust
fn path_is_valid(path: &str) -> bool {
    let path = std::path::Path::new(path);
    let mut components = path.components().peekable();

    if let Some(first) = components.peek() {
        if !matches!(first, std::path::Component::Normal(_)) {
            return false;
        }
    }

    components.count() == 1
}
````

在 `stream_to_file` 最前面加校验：

````rust
async fn stream_to_file<S, E>(path: &str, stream: S) -> Result<(), (StatusCode, String)>
where
    S: Stream<Item = Result<Bytes, E>>,
    E: Into<BoxError>,
{
    if !path_is_valid(path) {
        return Err((StatusCode::BAD_REQUEST, "Invalid path".to_owned()));
    }
    // ... 后面的写盘逻辑
}
````

校验放在 `stream_to_file` 里，**对两条入口都生效**。

> **新面孔：`Path::components` 与路径穿越防御**
>
> | 输入 | 是否允许 | 原因 |
> | --- | --- | --- |
> | `hello.txt` | 允许 | 一个普通文件名 |
> | `images/a.png` | 不允许 | 多个组件 |
> | `../a.txt` | 不允许 | 第一段是 `ParentDir` |
> | `/tmp/a.txt` | 不允许 | 第一段是 `RootDir` |
>
> 规则："第一个组件必须是 Normal，且总共只有一个组件"。
>
> 安全要点：**不能依赖 URL 规范化防御路径穿越**。测试 `curl /file/../bad.txt` 时必须加 `--path-as-is`——不加的话 curl 直接把 `..` 抹掉，根本走不到校验。

---

## 完整代码

````rust
use axum::{
    body::Bytes,
    extract::{Multipart, Path, Request},
    http::StatusCode,
    response::{Html, Redirect},
    routing::{get, post},
    BoxError, Router,
};
use futures_util::{Stream, TryStreamExt};
use std::{io, pin::pin};
use tokio::{fs::File, io::BufWriter};
use tokio_util::io::StreamReader;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

const UPLOADS_DIRECTORY: &str = "uploads";

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    tokio::fs::create_dir(UPLOADS_DIRECTORY)
        .await
        .expect("failed to create `uploads` directory");

    let app = Router::new()
        .route("/", get(show_form).post(accept_form))
        .route("/file/{file_name}", post(save_request_body));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await;
}

async fn save_request_body(
    Path(file_name): Path<String>,
    request: Request,
) -> Result<(), (StatusCode, String)> {
    stream_to_file(&file_name, request.into_body().into_data_stream()).await
}

async fn show_form() -> Html<&'static str> {
    Html(
        r#"
        <!doctype html>
        <html>
            <head>
                <title>Upload something!</title>
            </head>
            <body>
                <form action="/" method="post" enctype="multipart/form-data">
                    <div>
                        <label>
                            Upload file:
                            <input type="file" name="file" multiple>
                        </label>
                    </div>

                    <div>
                        <input type="submit" value="Upload files">
                    </div>
                </form>
            </body>
        </html>
        "#,
    )
}

async fn accept_form(mut multipart: Multipart) -> Result<Redirect, (StatusCode, String)> {
    while let Ok(Some(field)) = multipart.next_field().await {
        let file_name = if let Some(file_name) = field.file_name() {
            file_name.to_owned()
        } else {
            continue;
        };

        stream_to_file(&file_name, field).await?;
    }

    Ok(Redirect::to("/"))
}

async fn stream_to_file<S, E>(path: &str, stream: S) -> Result<(), (StatusCode, String)>
where
    S: Stream<Item = Result<Bytes, E>>,
    E: Into<BoxError>,
{
    if !path_is_valid(path) {
        return Err((StatusCode::BAD_REQUEST, "Invalid path".to_owned()));
    }

    async {
        let body_with_io_error = stream.map_err(io::Error::other);
        let mut body_reader = pin!(StreamReader::new(body_with_io_error));

        let path = std::path::Path::new(UPLOADS_DIRECTORY).join(path);
        let mut file = BufWriter::new(File::create(path).await?);

        tokio::io::copy(&mut body_reader, &mut file).await?;

        Ok::<_, io::Error>(())
    }
    .await
    .map_err(|err| (StatusCode::INTERNAL_SERVER_ERROR, err.to_string()))
}

fn path_is_valid(path: &str) -> bool {
    let path = std::path::Path::new(path);
    let mut components = path.components().peekable();

    if let Some(first) = components.peek() {
        if !matches!(first, std::path::Component::Normal(_)) {
            return false;
        }
    }

    components.count() == 1
}
````

## 运行

````bash
cd examples
cargo run -p example-stream-to-file
````

浏览器访问 `http://127.0.0.1:3000/`，选文件提交，文件写入 `examples/uploads/`。

用 curl 直接写：

````bash
curl -i -X POST http://127.0.0.1:3000/file/hello.txt --data-binary 'hello from axum'
cat uploads/hello.txt  # 预期: hello from axum
````

测试路径校验（必须加 `--path-as-is`）：

````bash
curl -i --path-as-is -X POST http://127.0.0.1:3000/file/../bad.txt --data-binary 'bad'
# 预期 400 Bad Request
````

### 常见卡点

**`create_dir` 第二次运行 panic**：目录已存在时会报错。真实项目用 `create_dir_all`（幂等）。

**本章没有 body limit**：攻击者可上传超大文件填满磁盘。用 `DefaultBodyLimit::max(...)` 限制。

## 手写任务

1. 把 `UPLOADS_DIRECTORY` 改成 `tmp_uploads`，观察保存目录变化。
2. 用 curl 写入 `note.txt`，确认内容正确。
3. 用 `curl --path-as-is` 测 `/file/../bad.txt`，确认返回 400。
4. 把 `create_dir` 改成 `create_dir_all`，连续运行两次，确认第二次不再 panic。

## 小结

这章分 3 步把上传内容流式写入磁盘：

1. **请求 body 写盘**：`tokio::io::copy` 从 `StreamReader`（stream 适配成的 `AsyncRead`）一块块拷到 `BufWriter<File>`，内存占用恒定；`pin!` 解决 `Unpin` 约束。
2. **multipart 表单上传**：`field` 本身就是 `Stream`，直接传给 `stream_to_file`，无需 `bytes()` 中转。
3. **路径校验**：`path_is_valid` 只允许"一个 Normal 组件"，挡住 `..` 和绝对路径；校验放在 `stream_to_file` 里对两条入口同时生效。

## 源码对照

- `examples/stream-to-file/Cargo.toml`
- `examples/stream-to-file/src/main.rs`
