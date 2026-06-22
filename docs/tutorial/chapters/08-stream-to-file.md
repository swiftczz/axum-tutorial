# 08. stream-to-file

对应示例：`examples/stream-to-file`

本章目标：学会把请求体或上传文件流式写入磁盘，并理解为什么要校验文件路径。

## 这个小项目在做什么

上一章 `multipart-form` 读取上传文件后，只打印了文件大小。  
这一章继续往前走：把上传内容保存成文件。

这个 example 提供两种写文件方式：

- `GET /`：返回文件上传表单。
- `POST /`：接收 multipart 文件上传，把每个上传文件写入 `uploads/` 目录。
- `POST /file/{file_name}`：不走 HTML 表单，直接把整个请求 body 写入指定文件。

请求主线有两条：

```text
浏览器 multipart 上传
-> accept_form 逐个读取 field
-> stream_to_file 把 field 流式写入 uploads/文件名
-> Redirect::to("/") 回到首页
```

```text
curl POST /file/foo.txt
-> save_request_body 取路径参数 foo.txt
-> 把整个 request body 转成数据流
-> stream_to_file 写入 uploads/foo.txt
```

这一章的重点是：不要先把大文件完整读进内存，再一次性写文件；更稳的方式是边读边写。

## 先理解“流式写入”

如果上传一个 2GB 文件，最直接但危险的做法是：

```text
把 2GB 全部读进内存
再写入磁盘
```

这样很容易占满内存。

流式写入的思路是：

```text
请求体分成一块一块 Bytes
-> 每来一块就写一块到文件
-> 不需要把整个文件同时放进内存
```

在这个 example 里，关键函数是：

````rust
async fn stream_to_file<S, E>(path: &str, stream: S) -> Result<(), (StatusCode, String)>
where
    S: Stream<Item = Result<Bytes, E>>,
    E: Into<BoxError>,
```

它不关心 stream 来自哪里。只要你给它一个 `Stream<Item = Result<Bytes, E>>`，它就能把这些字节写进文件。

## 文件和依赖

这个 example 有两个文件：

1. `examples/stream-to-file/Cargo.toml`：声明 Axum multipart feature、futures、tokio-util 等依赖。
2. `examples/stream-to-file/src/main.rs`：实现表单上传、直接 body 写文件、流式 copy 和路径校验。

关键依赖：

- `axum`：提供 `Router`、`Multipart`、`Path`、`Request`、`Redirect`。
- `futures-util`：提供 stream 相关工具，例如 `map_err`。
- `tokio`：提供异步文件 I/O。
- `tokio-util`：提供 `StreamReader`，把 stream 转成 `AsyncRead`。
- `tracing` / `tracing-subscriber`：初始化日志。

## 第一步：创建 uploads 目录

源码：

````rust
const UPLOADS_DIRECTORY: &str = "uploads";

tokio::fs::create_dir(UPLOADS_DIRECTORY)
    .await
    .expect("failed to create `uploads` directory");
````

上传文件不能随便写在当前目录里。示例先创建一个单独的 `uploads/` 目录，避免覆盖项目文件。

这里用的是 `tokio::fs::create_dir`，因为这是异步程序，文件系统操作也可以用 Tokio 的异步 API。

注意：`create_dir` 如果目录已经存在**一定**会返回 `AlreadyExists` 错误。所以这个 example 第二次运行时会 panic 在 `expect(...)` 上。真实项目通常用 `create_dir_all`，它对"目录已存在"是幂等的。

## 第二步：注册两个 POST 入口

源码：

````rust
let app = Router::new()
    .route("/", get(show_form).post(accept_form))
    .route("/file/{file_name}", post(save_request_body));
````

这说明应用有三个入口：

| 请求 | handler | 作用 |
| --- | --- | --- |
| `GET /` | `show_form` | 展示上传页面 |
| `POST /` | `accept_form` | 接收 multipart 表单上传 |
| `POST /file/{file_name}` | `save_request_body` | 直接把整个 body 保存成文件 |

`{file_name}` 是路径参数。  
比如请求 `POST /file/hello.txt`，`Path(file_name)` 就会拿到 `"hello.txt"`。

## 第三步：直接保存整个请求 body

源码：

````rust
async fn save_request_body(
    Path(file_name): Path<String>,
    request: Request,
) -> Result<(), (StatusCode, String)> {
    stream_to_file(&file_name, request.into_body().into_data_stream()).await
}
````

这个 handler 接收两个输入：

- `Path(file_name)`：从 URL 里提取文件名。
- `request: Request`：拿到完整请求对象。

为什么这里不用 `Bytes` 一次性接收 body？

因为本章要演示流式写入。  
`request.into_body().into_data_stream()` 会把请求体变成字节流，然后交给 `stream_to_file`。

请求例子：

```text
POST /file/hello.txt
Body: hello from axum
```

最终会写入：

```text
uploads/hello.txt
```

## 第四步：处理 multipart 表单上传

源码：

````rust
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

这段逻辑是：

```text
逐个读取 multipart field
-> 如果 field 没有文件名，就跳过
-> 如果有文件名，就把这个 field 流式写入文件
-> 全部处理完后重定向回首页
```

这里能直接把 `field` 传给 `stream_to_file`，是因为 multipart 的 field 本身也能作为字节流使用。

返回类型是：

````rust
Result<Redirect, (StatusCode, String)>
````

含义是：

- 成功：返回 `Redirect::to("/")`。
- 失败：返回状态码和错误文本。

这是前面 `impl IntoResponse` 的进一步用法：成功和失败都能变成 HTTP 响应。

## 第五步：把 Stream 写入文件

核心源码：

````rust
let body_with_io_error = stream.map_err(io::Error::other);
let mut body_reader = pin!(StreamReader::new(body_with_io_error));

let path = std::path::Path::new(UPLOADS_DIRECTORY).join(path);
let mut file = BufWriter::new(File::create(path).await?);

tokio::io::copy(&mut body_reader, &mut file).await?;
````

分成三步看：

1. `stream.map_err(io::Error::other)`：把 stream 的错误转换成 `io::Error`。
2. `StreamReader::new(...)`：把字节 stream 适配成 `AsyncRead`。
3. `tokio::io::copy(...)`：从 reader 读数据，写入 file。

为什么要 `BufWriter`？

文件写入时，频繁小块写入可能效率不好。  
`BufWriter` 会做缓冲，让写入更高效。

为什么要 `pin!`？

这和 Rust 异步的一个核心概念有关。`tokio::io::copy` 要求传入的 reader 满足 `AsyncRead + Unpin`。`Unpin` 是一个 trait，表示“这个类型的值可以被安全地移动”。

但 `StreamReader` 内部维护了自引用状态（一个状态机里同时存着 buffer 和指针），不是 `Unpin`。如果它在被 `poll` 的过程中被移动到别的内存地址，那些指针就会指向错误的地方。所以必须先用 `pin!` 把它“钉”在栈上的某个位置，保证地址不变，编译器才允许你调用 `tokio::io::copy`。

可以这样建立直觉：

```text
很多异步类型内部是状态机。
状态机里可能有自引用。
自引用的东西不能随便移动。
pin! 就是告诉编译器：这个东西我已经固定好位置了，你可以放心用。
```

新手第一遍可以先记住：

```text
tokio::io::copy 要求 reader 是 Unpin。
StreamReader 不是 Unpin，所以需要 pin! 把它固定住。
```

后面学到 `async fn`、`Stream`、`Future` 时会反复遇到 `pin`，那时再深入也不迟。

## 第六步：校验文件名，防止路径穿越

源码：

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

为什么要校验？

因为文件名来自用户，不可信。这一章有两个入口接收用户文件名，对应**两个不同的攻击面**：

**攻击面 1：URL 路径参数（`POST /file/{file_name}`）**

如果 `file_name` 是 `../Cargo.toml`，最终路径会拼成 `uploads/../Cargo.toml`，也就是项目根目录的 `Cargo.toml`，攻击者可以覆盖任意文件。

**攻击面 2：multipart field 的文件名（`POST /`）**

浏览器上传文件时，文件名来自 multipart body 里的 `Content-Disposition: form-data; filename="..."`。攻击者可以用 `curl` 伪造这个 filename 为 `../../../etc/passwd`，绕过浏览器限制。

`path_is_valid` 对这两个入口都生效，因为 `save_request_body` 和 `accept_form` 最终都调用 `stream_to_file`，而校验在 `stream_to_file` 里。

示例的规则是：

```text
路径必须只有一个普通组件
```

所以：

| 输入 | 是否允许 | 原因 |
| --- | --- | --- |
| `hello.txt` | 允许 | 一个普通文件名 |
| `images/a.png` | 不允许 | 有多个路径组件 |
| `../a.txt` | 不允许 | 试图向上级目录跳转 |
| `/tmp/a.txt` | 不允许 | 绝对路径 |

这个安全检查虽然短，但非常重要。只要文件名来自用户，就必须先校验。

## 函数职责速查

- `main`：初始化日志，创建 `uploads/` 目录，注册路由并启动服务。
- `save_request_body`：把 `POST /file/{file_name}` 的整个请求体流式写入文件。
- `show_form`：返回 multipart 上传表单。
- `accept_form`：接收 multipart 上传，逐个 field 写入文件。
- `stream_to_file`：把任意字节 stream 保存到 `uploads/` 下的文件。
- `path_is_valid`：校验文件名，阻止路径穿越。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-stream-to-file
//! ```

// 引入 Bytes、提取器、状态码、响应类型、路由函数、错误类型和 Router。
use axum::{
    body::Bytes,
    extract::{Multipart, Path, Request},
    http::StatusCode,
    response::{Html, Redirect},
    routing::{get, post},
    BoxError, Router,
};
// 引入 Stream trait 和 stream 错误转换工具。
use futures_util::{Stream, TryStreamExt};
// 引入标准库 I/O 和 pin 宏。
use std::{io, pin::pin};
// 引入 Tokio 异步文件和带缓冲的异步 writer。
use tokio::{fs::File, io::BufWriter};
// StreamReader 可以把字节 stream 转成 AsyncRead。
use tokio_util::io::StreamReader;
// tracing_subscriber 用来初始化日志。
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 上传文件统一保存到 uploads 目录。
const UPLOADS_DIRECTORY: &str = "uploads";

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 初始化日志系统。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 创建 uploads 目录，把上传文件和当前目录隔离开。
    tokio::fs::create_dir(UPLOADS_DIRECTORY)
        .await
        .expect("failed to create `uploads` directory");

    // 注册三个入口：展示表单、接收 multipart、直接保存请求体。
    let app = Router::new()
        .route("/", get(show_form).post(accept_form))
        .route("/file/{file_name}", post(save_request_body));

    // 绑定本机 3000 端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 输出监听地址。
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动 HTTP 服务。
    axum::serve(listener, app).await;
}

// POST /file/{file_name} 的 handler，把整个请求体保存成文件。
async fn save_request_body(
    // 从路径里提取文件名。
    Path(file_name): Path<String>,
    // 拿到完整请求对象，后面要取出 body。
    request: Request,
) -> Result<(), (StatusCode, String)> {
    // 把请求 body 转成数据流，再交给 stream_to_file。
    stream_to_file(&file_name, request.into_body().into_data_stream()).await
}

// GET / 的 handler，返回 multipart 上传表单。
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

// POST / 的 handler，接收 multipart 上传并把每个文件写入磁盘。
async fn accept_form(mut multipart: Multipart) -> Result<Redirect, (StatusCode, String)> {
    // 逐个读取 multipart field。
    while let Ok(Some(field)) = multipart.next_field().await {
        // 只处理带文件名的 field；没有文件名的字段跳过。
        let file_name = if let Some(file_name) = field.file_name() {
            file_name.to_owned()
        } else {
            continue;
        };

        // 把当前 field 的字节流写入文件。
        stream_to_file(&file_name, field).await?;
    }

    // 全部上传完成后重定向回首页。
    Ok(Redirect::to("/"))
}

// 把一个字节 Stream 保存成文件。
async fn stream_to_file<S, E>(path: &str, stream: S) -> Result<(), (StatusCode, String)>
where
    // stream 每次产出一块 Bytes，或者一个错误。
    S: Stream<Item = Result<Bytes, E>>,
    // stream 的错误要能转换成 BoxError。
    E: Into<BoxError>,
{
    // 先校验文件名，防止路径穿越。
    if !path_is_valid(path) {
        return Err((StatusCode::BAD_REQUEST, "Invalid path".to_owned()));
    }

    async {
        // 把 stream 的错误转换成 io::Error。
        let body_with_io_error = stream.map_err(io::Error::other);
        // 把字节 stream 转成 AsyncRead。
        let mut body_reader = pin!(StreamReader::new(body_with_io_error));

        // 在 uploads 目录下创建目标文件。
        let path = std::path::Path::new(UPLOADS_DIRECTORY).join(path);
        let mut file = BufWriter::new(File::create(path).await?);

        // 从请求体 reader 读取数据，并写入文件。
        tokio::io::copy(&mut body_reader, &mut file).await?;

        Ok::<_, io::Error>(())
    }
    .await
    // 把 I/O 错误转换成 HTTP 500 响应。
    .map_err(|err| (StatusCode::INTERNAL_SERVER_ERROR, err.to_string()))
}

// 校验文件名，防止用户传入 ../ 这类路径穿越内容。
fn path_is_valid(path: &str) -> bool {
    let path = std::path::Path::new(path);
    let mut components = path.components().peekable();

    // 第一个组件必须是普通路径组件，不能是 /、.. 等特殊组件。
    if let Some(first) = components.peek() {
        if !matches!(first, std::path::Component::Normal(_)) {
            return false;
        }
    }

    // 并且总共只能有一个组件，不能是 images/a.png 这种多级路径。
    components.count() == 1
}
````

## 运行和验证

运行前先确认 `examples/stream-to-file/Cargo.toml` 里的 `axum = { path = "../../axum", features = ["multipart"] }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-stream-to-file
````

浏览器访问：

````text
http://127.0.0.1:3000/
````

选择文件并提交后，文件会写入：

````text
examples/uploads/
````

也可以直接用 curl 把请求体写成文件：

````bash
curl -i -X POST http://127.0.0.1:3000/file/hello.txt \
  --data-binary 'hello from axum'
````

然后查看：

````bash
cat uploads/hello.txt
````

预期内容：

````text
hello from axum
````

测试路径校验。这一步有个**容易踩的坑**：`curl` 默认会把 URL 里的 `..` 规范化掉，所以直接写 `/file/../bad.txt` 实际发出去的是 `/bad.txt`，根本不会触发校验（而是 404）。要用 `--path-as-is` 让 `curl` 原样发送路径：

````bash
# 用 --path-as-is 才能让 ../ 真正到达服务端
curl -i --path-as-is -X POST http://127.0.0.1:3000/file/../bad.txt \
  --data-binary 'bad'
````

预期返回 `400 Bad Request`，并且 `uploads/` 外部不会出现新文件。

另一个验证方式是测试 multipart 路径（攻击面 2）：

````bash
# 伪造 multipart filename 为 ../evil.txt
curl -i -X POST http://127.0.0.1:3000/ \
  -F 'file=@/dev/null;filename=../evil.txt'
````

同样应该返回 `400 Bad Request`。

常见卡点：

### 1. 为什么 `curl /file/../bad.txt` 没有触发校验？

`curl` 默认按 RFC 3986 规范化 URL，`/file/../bad.txt` 会被简化成 `/bad.txt`。这个请求匹配不到 `/file/{file_name}` 路由，所以直接返回 404，根本走不到 `path_is_valid`。必须加 `--path-as-is`。

这也说明一个安全要点：**不能依赖 URL 规范化来防御路径穿越**。浏览器、curl、反向代理的行为各不相同，服务端必须自己校验（就像 `path_is_valid` 这样）。

### 2. `create_dir` 报错 "File exists"

`create_dir` 在目录已存在时**一定**返回 `AlreadyExists` 错误（不是"可能"失败）。所以第二次运行这个 example 会 panic。真实项目应该用 `create_dir_all`，它对"目录已存在"是幂等的。

### 3. `field.bytes().await` 和本章 stream 方式的区别

`field.bytes().await` 会把当前 field 的全部内容读进内存，适合小文件；本章用 stream 方式逐块写入，是为了适合大文件。

### 4. 上传大小没有限制

本章没有配置额外 body limit，攻击者可以上传超大文件填满磁盘。真实项目应该用 `DefaultBodyLimit::max(...)` 或 `DefaultBodyLimit::disable()` 配合自己的限额逻辑。

## 手写任务

完成本章代码后，试着做几个小改动：

1. 把 `UPLOADS_DIRECTORY` 改成 `tmp_uploads`，观察文件保存目录变化。
2. 用 `curl` 写入 `note.txt`，确认 `uploads/note.txt` 内容正确。
3. 用 `curl --path-as-is` 测试 `/file/../bad.txt`，确认返回 `400 Bad Request`。
4. 把 `create_dir` 改成 `create_dir_all`，然后连续运行两次，确认第二次不再 panic。

## 本章真正要记住什么

- 请求体可以看成一段不断产出 `Bytes` 的 stream。
- 流式写入适合处理大文件，避免把完整文件一次性放进内存。
- `StreamReader` 可以把字节 stream 适配成 `AsyncRead`。
- `tokio::io::copy` 可以把 reader 的内容复制到 writer。
- 用户传来的文件名必须校验，防止路径穿越。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/stream-to-file/Cargo.toml`
- `examples/stream-to-file/src/main.rs`
