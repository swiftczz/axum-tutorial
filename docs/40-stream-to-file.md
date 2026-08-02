# 第 40 章 stream-to-file:流式写盘

## 本章你将做出什么

一个上传服务,**文件直接从网络流进磁盘,全程不在内存里完整驻留**:

```
GET  /                上传表单
POST /                multipart 表单上传(可多文件)
POST /file/{name}     裸 body 上传,body 就是文件内容
```

[第 13 章](13-multipart-form.md#代价与边界)那个 `field.bytes().await` 把整个文件读进内存的做法,在这一章得到正确解法。

学完本章你将掌握:

- `Stream` 与 `AsyncRead` 的区别,以及 `StreamReader` 如何桥接它们
- **`tokio::io::copy`:一行代码完成流式拷贝**
- `BufWriter` 为什么能显著减少系统调用
- **路径穿越防护的正确写法**(以及为什么"检查是否含 `..`"不够)
- 两种上传方式的取舍:multipart vs 裸 body

## 前置知识

### 缓冲 vs 流式

[第 13 章](13-multipart-form.md#multipart-提取器是流式的)讲过 `Multipart` 提取器本身是流式的,但那一章的处理方式不是:

```rust
let data = field.bytes().await.unwrap();   // ← 整个文件进内存
println!("... is {} bytes", data.len());
```

**250MB 的文件就是 250MB 内存**,10 个并发上传就是 2.5GB。

流式处理的思路是**边收边写**:

```
网络 → 一小块 → 磁盘
     → 下一块 → 磁盘
     → ...
```

**任一时刻内存里只有一小块数据**(几 KB 到几十 KB),文件多大都不影响内存占用。

这不只是省内存的问题——它决定了服务能不能处理大文件。缓冲式的做法有一个**硬上限**:单个文件不能超过可用内存。

### Stream 与 AsyncRead:两套抽象

Rust 的异步 IO 里有两个相关但不同的 trait,理解它们的区别是本章的关键。

**`Stream<Item = T>`**([第 33 章](33-sse.md#stream异步的迭代器)讲过)—— 异步的迭代器,一次产出**一个 `T`**:

```rust
while let Some(chunk) = stream.next().await {
    // chunk 是一个 Bytes
}
```

**`AsyncRead`** —— 异步版的 `std::io::Read`,把数据读进**你提供的缓冲区**:

```rust
let n = reader.read(&mut buf).await?;   // 读了 n 个字节到 buf
```

| | `Stream<Item = Bytes>` | `AsyncRead` |
|---|---|---|
| 数据单位 | 一块一块(库决定大小) | 你提供缓冲区,读多少算多少 |
| 谁分配内存 | 库 | 调用方 |
| 典型来源 | HTTP body、channel | 文件、socket |
| 组合子 | `StreamExt`(`map`/`filter`) | `AsyncReadExt`(`read_to_end`) |

**HTTP 的 body 是 `Stream`,文件是 `AsyncRead`/`AsyncWrite`。** 要把前者写进后者,得先桥接这两套抽象。

### StreamReader:把 Stream 变成 AsyncRead

`tokio_util::io::StreamReader` 就是那座桥:

```rust
let body_reader = StreamReader::new(stream_of_bytes);
// body_reader 现在实现了 AsyncRead
```

它内部维护一个当前块的游标:`read()` 时从当前块拷贝数据,当前块用完就从流里取下一块。

**它要求流的 `Item` 是 `Result<B, io::Error>`**,其中 `B: Buf`(`Bytes` 满足)。所以本章有这么一行转换:

```rust
let body_with_io_error = stream.map_err(io::Error::other);
```

**`io::Error::other`** 把任意错误包装成 `io::Error`。这是 Rust 1.74 稳定的方法,等价于旧写法 `io::Error::new(io::ErrorKind::Other, e)`。

`tokio_util` 还有反方向的 `ReaderStream`(`AsyncRead` → `Stream`),下载文件时会用到。

### tokio::io::copy:一行完成拷贝

有了 `AsyncRead` 和 `AsyncWrite`,拷贝就是一行:

```rust
tokio::io::copy(&mut body_reader, &mut file).await?;
```

它内部是个循环:读一块 → 写一块 → 直到读完。**缓冲区由它自己管理**(默认 8KB),不需要你分配。

**这就是流式的全部** —— 不需要手写循环、不需要管理缓冲区、不需要处理部分写入。

还有个 `copy_buf` 版本,用于源已经是 `AsyncBufRead` 的情况,能少一次内存拷贝。

### BufWriter:减少系统调用

```rust
let mut file = BufWriter::new(File::create(path).await?);
```

**为什么要包一层?** 因为每次 `write()` 都是一次**系统调用**,而系统调用的开销(上下文切换)远大于内存拷贝。

`tokio::io::copy` 默认用 8KB 缓冲,如果直接写文件就是"每 8KB 一次系统调用"。`BufWriter` 默认缓冲 8KB,把多次小写攒成一次大写。

**实际效果取决于块大小。** 如果网络来的块本来就很大(比如 64KB),`BufWriter` 帮助有限;块很小很碎时收益明显。

**注意 `BufWriter` 需要显式 flush 或 drop 才会写完最后一批数据。** 本章依赖 `file` 离开作用域时的 drop,`tokio::fs::File` 的 drop 会尽力 flush——但**这不是有保证的**,「代价与边界」会说。

### 路径穿越:为什么"检查 .." 不够

[第 13 章](13-multipart-form.md#代价与边界)提过 `file_name()` 可被用来路径穿越,[第 38 章](38-static-file-server.md#静态文件服务在做什么)讲过 `ServeDir` 替你防了这个。**这一章要自己防。**

朴素的做法是检查字符串里有没有 `..`:

```rust
if path.contains("..") { return false; }   // ← 不够
```

**它挡不住这些:**

- **绝对路径**:`/etc/passwd`(不含 `..`,但 `Path::join` 遇到绝对路径会**丢弃前面的部分**)
- **多层目录**:`a/b/c.txt`(不含 `..`,但会写到子目录去)
- **Windows 前缀**:`C:\evil.txt`
- **URL 编码**:`%2e%2e`(如果解码发生在检查之后)

**本章的做法是正面校验:要求路径恰好由一个"普通组件"构成。**

```rust
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
```

`Path::components()` 把路径拆成语义化的组件:

| 输入 | 组件 |
|---|---|
| `foo.txt` | `[Normal("foo.txt")]` ✓ |
| `../foo.txt` | `[ParentDir, Normal("foo.txt")]` ✗ 第一个不是 Normal |
| `/etc/passwd` | `[RootDir, Normal("etc"), Normal("passwd")]` ✗ 同上 |
| `a/b.txt` | `[Normal("a"), Normal("b.txt")]` ✗ 数量不是 1 |
| `./foo.txt` | `[CurDir, Normal("foo.txt")]` ✗ 第一个是 CurDir |

**"白名单式校验"比"黑名单式过滤"可靠得多** —— 你不需要枚举所有攻击形式,只需要定义什么是合法的。这是安全编程的一条通用原则。

### 两种上传方式

本章提供了两条上传路由,取舍不同:

| | `POST /` (multipart) | `POST /file/{name}` (裸 body) |
|---|---|---|
| 文件名来源 | 表单里的 `filename` | **URL 路径** |
| 一次能传几个 | **多个** | 一个 |
| 能否带其它字段 | **能** | 不能 |
| 报文开销 | 有 boundary 和每段头部 | **零开销** |
| 浏览器表单支持 | **原生支持** | 需要 JS |
| 适合 | 网页表单 | **API / CLI 客户端** |

**裸 body 上传更简单也更高效** —— `curl --data-binary @file` 直接把文件内容当 body 发过去,没有任何包装。API 场景通常用这种。

## 项目搭建

```bash
cargo new stream-to-file
cd stream-to-file
```

```bash
cargo add axum --features multipart
cargo add futures-util --no-default-features --features sink,std
cargo add tokio --features full
cargo add tokio-util --features io
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`tokio-util` 的 `io` feature 提供 `StreamReader`。

**不需要手动创建 `uploads` 目录** —— 程序启动时会建。

## 分步构建

### 步骤 1:流式写盘的核心函数

**要做什么**:先写出 `stream_to_file`,这是本章的核心。

```rust
// src/main.rs
use axum::{body::Bytes, http::StatusCode, BoxError, Router};
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

    // save files to a separate directory to not override files in the current directory
    tokio::fs::create_dir(UPLOADS_DIRECTORY)
        .await
        .expect("failed to create `uploads` directory");

    let app = Router::new();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

// Save a `Stream` to a file
async fn stream_to_file<S, E>(path: &str, stream: S) -> Result<(), (StatusCode, String)>
where
    S: Stream<Item = Result<Bytes, E>>,
    E: Into<BoxError>,
{
    if !path_is_valid(path) {
        return Err((StatusCode::BAD_REQUEST, "Invalid path".to_owned()));
    }

    async {
        // Convert the stream into an `AsyncRead`.
        let body_with_io_error = stream.map_err(io::Error::other);
        let mut body_reader = pin!(StreamReader::new(body_with_io_error));

        // Create the file. `File` implements `AsyncWrite`.
        let path = std::path::Path::new(UPLOADS_DIRECTORY).join(path);
        let mut file = BufWriter::new(File::create(path).await?);

        // Copy the body into the file.
        tokio::io::copy(&mut body_reader, &mut file).await?;

        Ok::<_, io::Error>(())
    }
    .await
    .map_err(|err| (StatusCode::INTERNAL_SERVER_ERROR, err.to_string()))
}

// to prevent directory traversal attacks we ensure the path consists of exactly one normal
// component
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
```

**函数签名是泛型的:**

```rust
async fn stream_to_file<S, E>(path: &str, stream: S) -> Result<(), (StatusCode, String)>
where
    S: Stream<Item = Result<Bytes, E>>,
    E: Into<BoxError>,
```

这样它能同时接受两种来源:**HTTP body 的流**和 **multipart field 的流**——两者的错误类型不同,靠 `E: Into<BoxError>` 统一。[第 35 章](35-testing-websockets.md#单元测试把-socket-抽象掉)那个泛型 handler 是同样的思路。

**四步核心逻辑:**

```rust
let body_with_io_error = stream.map_err(io::Error::other);        // ① 统一错误类型
let mut body_reader = pin!(StreamReader::new(body_with_io_error)); // ② Stream → AsyncRead
let mut file = BufWriter::new(File::create(path).await?);          // ③ 打开文件
tokio::io::copy(&mut body_reader, &mut file).await?;               // ④ 拷贝
```

前置知识逐个讲过。

**`pin!` 宏**把值固定在栈上并得到 `Pin<&mut T>`。`StreamReader` 的 `AsyncRead` 实现要求 `Self: Unpin` 或者被 pin 住——因为它内部持有一个可能自引用的流。`std::pin::pin!` 是 Rust 1.68 稳定的宏,比 `Box::pin` 少一次堆分配。

**那个包裹的 `async { ... }.await` 块**是为了**统一错误转换**。块内所有 `?` 产生的都是 `io::Error`,块外一次性 `map_err` 成 HTTP 响应。不这么写的话每个 `?` 都要单独 `map_err`。

`Ok::<_, io::Error>(())` 的 turbofish 是必需的——否则编译器推断不出块的错误类型。

**`path_is_valid` 在最前面调用**,在任何文件操作之前。前置知识讲过它的原理。

**跑起来看**:

这一步还没有路由,只验证能编译:

```bash
cargo build
```

会有几个"未使用"的警告,正常。

**注意 `create_dir` 用了 `expect`** —— 目录已存在时会失败。第二次 `cargo run` 会 panic:

```
thread 'main' panicked at src/main.rs:...:
failed to create `uploads` directory: File exists (os error 17)
```

> **更好的写法**:`tokio::fs::create_dir` 在目录已存在时返回错误,导致**第二次启动就 panic**。应该用 `create_dir_all`(幂等,已存在时不报错),或者显式忽略 `AlreadyExists` 错误。示例这么写大概是想确保是个干净的目录,但代价是每次重启前都要手动 `rm -rf uploads`,开发体验很差。练习题会让你修。

先手动清理一下以便继续:

```bash
rm -rf uploads
```

### 步骤 2:裸 body 上传

**要做什么**:加上 `POST /file/{name}`,body 直接就是文件内容。

```rust
// src/main.rs
use axum::{
    body::Bytes,
    extract::{Path, Request},                    // 新增
    http::StatusCode,
    routing::post,                               // 新增
    BoxError, Router,
};
// ...(其余 use 不变)

#[tokio::main]
async fn main() {
    // ...(日志和建目录不变)

    // 改动:加上路由
    let app = Router::new().route("/file/{file_name}", post(save_request_body));

    // ...(listener 和 serve 不变)
}

// 新增:Handler that streams the request body to a file.
//
// POST'ing to `/file/foo.txt` will create a file called `foo.txt`.
async fn save_request_body(
    Path(file_name): Path<String>,
    request: Request,
) -> Result<(), (StatusCode, String)> {
    stream_to_file(&file_name, request.into_body().into_data_stream()).await
}

// ...(stream_to_file、path_is_valid 不变)
```

**`request: Request` 是整个请求**,而不是某个提取器。[第 09 章](09-consume-body-in-extractor-or-middleware.md#步骤-2在中间件里也读一遍-body)讲过 `Request` 拿到的是所有权,所以**它必须是最后一个参数**——`Path` 是 `FromRequestParts` 排在前面。

**`request.into_body().into_data_stream()`** 两步:取出 body,再把它转成 `Stream<Item = Result<Bytes, axum::Error>>`。

**这就是"流式"的入口** —— body 还没有被读取,只是拿到了一个能逐块读取的流。

**跑起来看**:

```bash
cargo run
```

造一个大文件测试:

```bash
head -c 50000000 /dev/urandom > /tmp/big.bin      # 50MB
ls -lh /tmp/big.bin
```

上传:

```bash
curl -X POST --data-binary @/tmp/big.bin http://127.0.0.1:3000/file/big.bin
ls -lh uploads/
```

```
-rw-r--r--  1 user  staff    48M  ... big.bin
```

**50MB 文件上传成功。**

**观察内存占用。** 在另一个终端盯着进程:

```bash
# macOS
ps -o rss= -p $(pgrep -f stream-to-file) 

# Linux
grep VmRSS /proc/$(pgrep -f stream-to-file)/status
```

上传过程中再看一次,**内存基本不涨** —— 几十 MB 的常驻,和文件大小无关。

**对比[第 13 章](13-multipart-form.md#代价与边界)那个 `field.bytes()` 的做法**:那边上传 50MB 就会让进程内存涨 50MB。**这就是流式的价值。**

**验证路径穿越防护:**

```bash
curl -i -s -X POST --data-binary 'evil' 'http://127.0.0.1:3000/file/..%2F..%2Fevil.txt' | head -1
```

```
HTTP/1.1 400 Bad Request
```

试试绝对路径和子目录:

```bash
for name in '..%2Fevil.txt' '%2Fetc%2Fpasswd' 'sub%2Fdir.txt' '.%2Fhidden'; do
  code=$(curl -s -o /dev/null -w '%{http_code}' -X POST --data-binary 'x' "http://127.0.0.1:3000/file/$name")
  printf "%-20s → %s\n" "$name" "$code"
done
```

```
..%2Fevil.txt        → 400
%2Fetc%2Fpasswd      → 400
sub%2Fdir.txt        → 400
.%2Fhidden           → 400
```

**四种都被拒了。** 对照前置知识那张表:`ParentDir`、`RootDir`、组件数不为 1、`CurDir`——`path_is_valid` 全部覆盖。

正常文件名照常工作:

```bash
curl -s -o /dev/null -w '%{http_code}\n' -X POST --data-binary 'hello' http://127.0.0.1:3000/file/ok.txt
cat uploads/ok.txt
```

```
200
hello
```

### 步骤 3:multipart 表单上传

**要做什么**:加上表单页面和 multipart 处理。

```rust
// src/main.rs
use axum::{
    body::Bytes,
    extract::{Multipart, Path, Request},         // 改动
    http::StatusCode,
    response::{Html, Redirect},                  // 新增
    routing::{get, post},                        // 改动
    BoxError, Router,
};
// ...(其余 use 不变)

#[tokio::main]
async fn main() {
    // ...(日志和建目录不变)

    // 改动:加上表单路由
    let app = Router::new()
        .route("/", get(show_form).post(accept_form))
        .route("/file/{file_name}", post(save_request_body));

    // ...
}

// ...(save_request_body 不变)

// 新增:Handler that returns HTML for a multipart form.
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

// 新增:Handler that accepts a multipart form upload and streams each field to a file.
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

// ...(stream_to_file、path_is_valid 不变)
```

**表单和[第 13 章](13-multipart-form.md#步骤-1显示文件上传表单)一样**:`enctype="multipart/form-data"` 加 `<input type="file" multiple>`。

**`stream_to_file(&file_name, field)`** —— 关键在这里:**`Field` 本身实现了 `Stream<Item = Result<Bytes, MultipartError>>`**,所以能直接传给泛型的 `stream_to_file`。

对比[第 13 章](13-multipart-form.md#步骤-2接收上传的文件)那边:

```rust
let data = field.bytes().await.unwrap();   // 第 13 章:整个读进内存
stream_to_file(&file_name, field).await?;  // 本章:流式写盘
```

**同一个 `Field`,两种用法,内存表现天差地别。**

**`if let Some(file_name) = ... else { continue }`** 处理非文件字段(没有 `filename` 的普通表单项),直接跳过。**这比[第 13 章](13-multipart-form.md#步骤-2接收上传的文件)那个 `field.file_name().unwrap()` 好** —— 那边遇到文本字段会 panic。

**`Redirect::to("/")`** 上传完跳回表单页,这是 POST-Redirect-GET 模式,防止刷新页面时重复提交。

**跑起来看**:

```bash
rm -rf uploads && cargo run
```

浏览器访问 `http://127.0.0.1:3000/`,选几个文件上传,页面会跳回表单。

```bash
ls -lh uploads/
```

上传的文件都在。

**用 curl 测多文件:**

```bash
echo 'file A' > /tmp/a.txt
echo 'file B' > /tmp/b.txt
curl -s -o /dev/null -w '%{http_code}\n' -X POST http://127.0.0.1:3000/ \
  -F 'file=@/tmp/a.txt' -F 'file=@/tmp/b.txt'
ls uploads/
```

```
303
a.txt  b.txt
```

**`303 See Other`** 是 `Redirect::to` 产生的状态码。

**验证非文件字段被跳过:**

```bash
curl -s -o /dev/null -w '%{http_code}\n' -X POST http://127.0.0.1:3000/ \
  -F 'title=hello' -F 'file=@/tmp/a.txt'
```

```
303
```

**没有 panic**,`title` 字段被 `continue` 跳过了。对比[第 13 章](13-multipart-form.md#步骤-2接收上传的文件)那个例子,同样的请求会让服务端 panic。

**大文件也走流式:**

```bash
curl -s -o /dev/null -X POST http://127.0.0.1:3000/ -F 'file=@/tmp/big.bin'
ls -lh uploads/big.bin
```

内存同样不会随文件大小增长。

**但注意 multipart 有 body 大小限制。** [第 13 章](13-multipart-form.md#两层请求体限制)讲过 `DefaultBodyLimit` 默认 2MB,而本章**没有配置它**:

```bash
curl -i -s -X POST http://127.0.0.1:3000/ -F 'file=@/tmp/big.bin' | head -1
```

如果返回 `413 Payload Too Large`,就是这个限制在起作用。**流式写盘和大小限制是两件事** —— 前者解决内存问题,后者解决"该不该接受这么大的请求"。「代价与边界」会说。

## 完整代码

`src/main.rs`:

```rust
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

    // save files to a separate directory to not override files in the current directory
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
    axum::serve(listener, app).await.unwrap();
}

// Handler that streams the request body to a file.
//
// POST'ing to `/file/foo.txt` will create a file called `foo.txt`.
async fn save_request_body(
    Path(file_name): Path<String>,
    request: Request,
) -> Result<(), (StatusCode, String)> {
    stream_to_file(&file_name, request.into_body().into_data_stream()).await
}

// Handler that returns HTML for a multipart form.
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

// Handler that accepts a multipart form upload and streams each field to a file.
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

// Save a `Stream` to a file
async fn stream_to_file<S, E>(path: &str, stream: S) -> Result<(), (StatusCode, String)>
where
    S: Stream<Item = Result<Bytes, E>>,
    E: Into<BoxError>,
{
    if !path_is_valid(path) {
        return Err((StatusCode::BAD_REQUEST, "Invalid path".to_owned()));
    }

    async {
        // Convert the stream into an `AsyncRead`.
        let body_with_io_error = stream.map_err(io::Error::other);
        let mut body_reader = pin!(StreamReader::new(body_with_io_error));

        // Create the file. `File` implements `AsyncWrite`.
        let path = std::path::Path::new(UPLOADS_DIRECTORY).join(path);
        let mut file = BufWriter::new(File::create(path).await?);

        // Copy the body into the file.
        tokio::io::copy(&mut body_reader, &mut file).await?;

        Ok::<_, io::Error>(())
    }
    .await
    .map_err(|err| (StatusCode::INTERNAL_SERVER_ERROR, err.to_string()))
}

// to prevent directory traversal attacks we ensure the path consists of exactly one normal
// component
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
```

## 代价与边界

**`create_dir` 不幂等。** 步骤 1 的「更好的写法」指出了:第二次启动直接 panic。

**没有请求体大小限制。** 流式写盘解决了内存问题,但**没解决磁盘问题** —— 客户端可以上传一个 100GB 的文件把磁盘写满。[第 13 章](13-multipart-form.md#两层请求体限制)讲的 `DefaultBodyLimit` 对 multipart 路由仍然生效(默认 2MB),但 `POST /file/{name}` 那条路由用的是裸 `Request`,**完全没有限制**。

生产环境必须配 `RequestBodyLimitLayer`,而且要根据磁盘容量和并发量算一个合理的值。

**文件名冲突会静默覆盖。** `File::create` 在文件已存在时**截断重写**。两个用户上传同名文件,后者覆盖前者;恶意用户可以故意覆盖别人的文件。

**正确做法是服务端生成文件名**(UUID 之类),把原始文件名只当元数据存起来——[第 13 章](13-multipart-form.md#代价与边界)提过同样的建议。

**部分写入的文件不会被清理。** 如果上传中途客户端断开,`tokio::io::copy` 会返回错误,但**已经写进磁盘的部分留在那里**。日积月累会攒下一堆残缺文件。

正确做法是**写临时文件,完成后原子重命名**:

```rust
// 伪代码
let tmp = format!("{path}.part");
copy_to(tmp)?;
tokio::fs::rename(tmp, path).await?;
```

这样要么有完整文件,要么什么都没有。

**`BufWriter` 的最后一批数据依赖 drop。** 前置知识提过。`tokio::fs::File` 的 `Drop` 实现会尝试 flush,但**它不能 `.await`**,所以只是"尽力而为"。**稳妥做法是显式调用 `file.flush().await?`**,官方示例没做。

**没有校验文件类型。** 任何内容都能上传,包括可执行文件、脚本。如果这个目录后来被静态文件服务([第 38 章](38-static-file-server.md))暴露出去,就成了任意文件托管——甚至可能被用来传播恶意软件。

至少应该:限制扩展名白名单、检查文件头的魔数([第 13 章](13-multipart-form.md#代价与边界)提过 `Content-Type` 不可信)、以及**绝不把上传目录直接暴露为静态资源**。

**没有认证和限流。** 任何人都能上传任意多的文件。

**同名并发上传有竞态。** 两个请求同时写同一个文件名,内容会交错。用唯一文件名能顺带解决这个。

**`uploads` 目录是相对路径。** 和[第 38 章](38-static-file-server.md#步骤-1最基本的用法)一样的问题:依赖工作目录。

## 动手练习

**1. 让建目录变得幂等,并显式 flush。**

提示:`create_dir` 换成 `create_dir_all`;`stream_to_file` 里在 `copy` 之后加 `file.flush().await?`(需要 `use tokio::io::AsyncWriteExt`)。做完确认能连续 `cargo run` 两次而不 panic。**这是「代价与边界」里最直接的两处。**

**2. 用服务端生成的文件名,避免覆盖和竞态。**

提示:`cargo add uuid --features v4`,文件名用 `Uuid::new_v4().to_string()` 加上原始扩展名。原始文件名可以记进日志或返回给客户端。做完试着两次上传同名文件,确认磁盘上是两个不同的文件。

**3. 实现"写临时文件 + 原子重命名"。**

提示:先写到 `{name}.part`,`copy` 成功后 `tokio::fs::rename` 成正式名字;失败时 `tokio::fs::remove_file` 清掉临时文件。验证:上传大文件时中途 `Ctrl-C` 掉 curl,确认 `uploads/` 里没有残留的半截文件(只有 `.part` 被清掉了)。**这个练习能让你理解为什么"原子重命名"是文件写入的标准模式。**

## 小结与下一章预告

流式写盘的核心是**桥接两套异步 IO 抽象**:HTTP body 是 `Stream<Item = Bytes>`,文件是 `AsyncWrite`。`tokio_util::io::StreamReader` 把前者转成 `AsyncRead`,然后 `tokio::io::copy` 一行完成拷贝——**缓冲区由它管理,内存占用和文件大小无关**。

`BufWriter` 把多次小写攒成一次大写,减少系统调用。但**它的最后一批数据依赖 drop 时的 flush,而 `Drop` 不能 `.await`**,所以稳妥做法是显式 `flush()`。

**路径穿越防护要用白名单式校验**:`path_is_valid` 要求路径恰好由一个 `Component::Normal` 构成。这挡住了 `..`、绝对路径、子目录、`./` 四类攻击——**比"检查字符串里有没有 `..`"可靠得多**。本章逐个验证过。

`stream_to_file` 写成泛型的,所以能同时接受 HTTP body 流和 multipart field 流。**同一个 `Field`,[第 13 章](13-multipart-form.md)用 `bytes()` 读进内存、本章用流式写盘,内存表现天差地别。**

要注意的是**流式只解决了内存问题**:磁盘可以被写满(需要 body 大小限制)、同名文件会被静默覆盖(需要服务端生成文件名)、中途断开会留下残缺文件(需要临时文件 + 原子重命名)、上传目录绝不能直接暴露为静态资源。

前面几章都是"服务端自己产生响应"。[第 41 章 reqwest-response](41-reqwest-response.md) 转向另一类场景:**服务端去请求别的服务,再把结果转发给客户端**。你会看到 `reqwest` 的响应怎么转成 axum 的响应,以及一个和本章呼应的话题——**转发时也要流式,不能把整个响应读进内存**。
