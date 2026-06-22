# 08. stream-to-file

对应示例：`examples/stream-to-file`

上一章只读取并打印上传文件信息。本章把上传内容**流式写入磁盘**——边读边写,不把整个文件一次性放进内存。同时处理一个安全问题:校验文件名,防止路径穿越。

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

- `tokio-util`:提供 `StreamReader`,把 stream 转成 `AsyncRead`。
- `futures-util`:stream 相关工具。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

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

浏览器访问 `http://127.0.0.1:3000/`,选文件提交,文件写入 `examples/uploads/`。

用 curl 直接把 body 写成文件:

````bash
curl -i -X POST http://127.0.0.1:3000/file/hello.txt \
  --data-binary 'hello from axum'

cat uploads/hello.txt
# 预期: hello from axum
````

测试路径校验——**注意必须加 `--path-as-is`**,否则 curl 会把 `..` 规范化掉:

````bash
curl -i --path-as-is -X POST http://127.0.0.1:3000/file/../bad.txt \
  --data-binary 'bad'
````

预期 `400 Bad Request`。测试 multipart 路径:

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -F 'file=@/dev/null;filename=../evil.txt'
````

同样应返回 `400 Bad Request`。

### 常见卡点

**`create_dir` 第二次运行报 `File exists` panic**:`create_dir` 在目录已存在时一定返回 `AlreadyExists`,所以这个 example 第二次运行会 panic。真实项目用 `create_dir_all`,它对"目录已存在"幂等。

**本章没有 body limit**:攻击者可上传超大文件填满磁盘。真实项目用 `DefaultBodyLimit::max(...)` 或 `DefaultBodyLimit::disable()` 配合自己的限额逻辑。

## 解读

### 为什么流式写入

最直接但危险的做法是把整个文件读进内存再写盘。上传 2GB 文件就会占 2GB 内存。流式写入把请求体切成一块块 `Bytes`,每来一块写一块,内存占用恒定。

### 两个入口共用一个 `stream_to_file`

```text
POST /                    → accept_form → 逐个 multipart field 写盘
POST /file/{file_name}    → save_request_body → 整个请求 body 写盘
```

两条路最终都调用 `stream_to_file`,因为 multipart field 和请求 body 都能作为字节流使用。`stream_to_file` 不关心 stream 来自哪里,只要给它 `Stream<Item = Result<Bytes, E>>` 就能写盘。

### `stream_to_file` 的三步

````rust
let body_with_io_error = stream.map_err(io::Error::other);
let mut body_reader = pin!(StreamReader::new(body_with_io_error));
let mut file = BufWriter::new(File::create(path).await?);
tokio::io::copy(&mut body_reader, &mut file).await?;
````

1. `map_err(io::Error::other)`:把 stream 错误转成 `io::Error`。
2. `StreamReader::new(...)`:把字节 stream 适配成 `AsyncRead`(因为 `tokio::io::copy` 要 reader)。
3. `tokio::io::copy`:从 reader 读数据,写入 file。`BufWriter` 做缓冲,让小块写入更高效。

### 为什么要 `pin!`

`tokio::io::copy` 要求 reader 是 `AsyncRead + Unpin`。`Unpin` 表示"这个值可以被安全移动"。但 `StreamReader` 内部是状态机,可能自引用(buffer 里存着指针),被移动会导致指针指向错误地址,所以它不是 `Unpin`。`pin!` 把它"钉"在栈上,地址不变,编译器才允许调用 `tokio::io::copy`。

直觉:很多异步类型内部是状态机,状态机可能有自引用,自引用的东西不能随便移动,`pin!` 就是告诉编译器位置已固定。`async fn`、`Stream`、`Future` 都会反复遇到 `pin`,第一遍先记住"需要 Unpin 但类型不满足时,用 `pin!` 固定"。

### 路径校验防穿越

文件名来自用户,不可信。两个入口对应两个攻击面:

**攻击面 1:URL 路径参数**。`POST /file/{file_name}` 如果 `file_name` 是 `../Cargo.toml`,会拼成 `uploads/../Cargo.toml`,即项目根的 `Cargo.toml`,覆盖任意文件。

**攻击面 2:multipart field 文件名**。攻击者用 curl 伪造 `filename="../evil.txt"`,绕过浏览器限制。

`path_is_valid` 在 `stream_to_file` 里,对两个入口都生效。规则是"路径必须只有一个普通组件":

| 输入 | 是否允许 | 原因 |
| --- | --- | --- |
| `hello.txt` | 允许 | 一个普通文件名 |
| `images/a.png` | 不允许 | 多个路径组件 |
| `../a.txt` | 不允许 | 向上级跳转 |
| `/tmp/a.txt` | 不允许 | 绝对路径 |

安全要点:**不能依赖 URL 规范化防御路径穿越**。浏览器、curl、反向代理的规范化行为各不相同,服务端必须自己校验。这也是为什么 `curl /file/../bad.txt` 没加 `--path-as-is` 时根本走不到校验——curl 把它简化成 `/bad.txt`,直接 404。

## 手写任务

跑通后做四个小改动:

1. 把 `UPLOADS_DIRECTORY` 改成 `tmp_uploads`,观察保存目录变化。
2. 用 curl 写入 `note.txt`,确认内容正确。
3. 用 `curl --path-as-is` 测 `/file/../bad.txt`,确认返回 400。
4. 把 `create_dir` 改成 `create_dir_all`,连续运行两次,确认第二次不再 panic。

## 小结

- 请求体是一段不断产出 `Bytes` 的 stream,流式写入适合大文件,内存占用恒定。
- `StreamReader` 把字节 stream 转成 `AsyncRead`,`tokio::io::copy` 把 reader 内容复制到 writer。
- 类型不满足 `Unpin` 时,用 `pin!` 把它固定在栈上,才能传给要求 `Unpin` 的 API。
- 文件名来自用户时必须校验,防路径穿越;不能依赖 URL 规范化。
- `create_dir` 在目录已存在时会报错,真实项目用幂等的 `create_dir_all`。

## 源码对照

- `examples/stream-to-file/Cargo.toml`
- `examples/stream-to-file/src/main.rs`
