# 第 39 章 compression:双向压缩

## 本章你将做出什么

一个只有 30 行的 echo 服务,但它同时开了**两个方向的压缩**:

```
POST /   收到压缩的请求体自动解压,返回时按客户端能力自动压缩
```

客户端可以发 gzip 压缩的 JSON,服务端自动解开;客户端声明支持 br,服务端就用 br 压缩响应。**handler 本身对此一无所知** —— 它只看到普通的 `Json<Value>`。

学完本章你将掌握:

- HTTP 的内容协商:`Accept-Encoding` 与 `Content-Encoding`
- 四种常见压缩算法的取舍
- `CompressionLayer` 与 `RequestDecompressionLayer` 的分工
- **中间件顺序在这里为什么无关紧要**(和[第 19 章](19-request-id.md)形成对比)
- 压缩与流式响应的冲突,以及 BREACH 这类安全考量

## 前置知识

### 两个方向,两组头部

压缩在 HTTP 里是**内容协商**的一部分,两个方向各有一组头部。

**响应压缩**(服务端 → 客户端),这是最常见的:

```
请求:  Accept-Encoding: gzip, br, zstd     ← 客户端声明"我能解开这些"
响应:  Content-Encoding: br                 ← 服务端声明"我用了这个"
       Vary: accept-encoding                ← 告诉缓存"响应随这个头部变化"
```

**请求压缩**(客户端 → 服务端),少见但存在:

```
请求:  Content-Encoding: gzip               ← 客户端声明"请求体是压缩的"
```

**注意不对称:** 响应压缩是**协商**的(客户端说能力,服务端选一个),请求压缩是**单方面通知**的——客户端直接压了,服务端要么能解要么报错(`415 Unsupported Media Type`)。

因为没有协商机制,**请求压缩在公网上很少用** —— 客户端不知道服务端支不支持。它主要出现在内部服务之间,或者约定好的 API 场景(比如批量上传日志)。

**`Vary: accept-encoding` 很重要。** 没有它,CDN 可能把压缩过的响应缓存下来发给不支持压缩的客户端,或者反过来。`CompressionLayer` 会自动加上。

### 四种算法

| 算法 | 编码名 | 压缩率 | 速度 | 支持面 |
|---|---|---|---|---|
| **gzip** | `gzip` | 中 | 快 | **所有客户端** |
| **deflate** | `deflate` | 中 | 快 | 广泛,但实现有歧义 |
| **brotli** | `br` | **高** | 中(压缩慢、解压快) | 现代浏览器 |
| **zstd** | `zstd` | 高 | **很快** | 较新,浏览器支持在推进 |

**gzip 是最低保障** —— 没有任何客户端不支持它。

**brotli 对文本效果最好**,比 gzip 小 15~25%,而且它内置了一个针对 HTML/CSS/JS 优化的字典。代价是压缩慢,所以更适合静态资源(压一次存起来)而非动态响应。

**zstd 是压缩率和速度的平衡点**,Facebook 出品,近年在服务端之间用得越来越多。

`tower-http` 的 `compression-full` feature 一次开启全部四种,`CompressionLayer::new()` 会**根据客户端的 `Accept-Encoding` 自动选择**——通常按 zstd > br > gzip > deflate 的优先级挑一个双方都支持的。

### 谁不该被压缩

**已经压缩过的内容再压一遍是浪费。** 图片(JPEG/PNG/WebP)、视频、音频、`.zip`/`.gz` 文件本身就是压缩格式,再压不但没效果,还可能**变大**(压缩算法要加自己的头部和字典)。

`CompressionLayer` 默认会**按 `Content-Type` 跳过**这类内容——它内置了一个预定义的列表。

**小响应也不值得压缩。** 压缩有固定开销(算法头部、CPU),几十字节的响应压完可能更大。`CompressionLayer` 支持配置最小阈值,但**默认没有设置**——「代价与边界」会提。

### 中间件顺序在这里不重要

[第 19 章](19-request-id.md#为什么必须用-servicebuilder)强调过顺序敏感的中间件必须用 `ServiceBuilder`,因为"生成 ID"必须在"记录 ID"之前。

**本章这两个中间件恰好互不干扰:**

```rust
ServiceBuilder::new()
    .layer(RequestDecompressionLayer::new())   // 只动请求
    .layer(CompressionLayer::new())            // 只动响应
```

`RequestDecompressionLayer` 在**请求路上**解压,`CompressionLayer` 在**响应路上**压缩——[第 17 章](17-tracing-aka-logging.md#洋葱结构与包装顺序)讲的洋葱结构里,它们作用在不同的方向,不存在依赖关系。

**所以调换顺序也能工作。** 本章会验证这一点。

**但用 `ServiceBuilder` 仍然是好习惯** —— 顺序不重要不等于顺序不明确,显式声明的写法在后续加入顺序敏感的中间件时不用重构。

### handler 完全不知道压缩存在

**这是本章最值得体会的一点。**

```rust
async fn root(Json(value): Json<Value>) -> Json<Value> {
    Json(value)
}
```

这个 handler 没有任何压缩相关的代码。它:

- 拿到的 `Json<Value>` 是**已经解压过**的([第 08 章](08-parse-body-based-on-content-type.md#json收-json-与发-json)那个提取器照常工作)
- 返回的 `Json<Value>` 会**在它返回之后**被压缩

**中间件的价值就在这里:横切关注点从业务代码里剥离出去。** [第 09 章](09-consume-body-in-extractor-or-middleware.md#中间件在请求前后插入逻辑)介绍中间件时讲过这个动机,压缩是它最纯粹的一个例子。

## 项目搭建

```bash
cargo new compression
cd compression
```

```bash
cargo add axum
cargo add serde_json
cargo add tokio --features macros,rt-multi-thread
cargo add tower
cargo add tower-http --features compression-full,decompression-full
cargo add tracing
cargo add tracing-subscriber --features env-filter
cargo add --dev assert-json-diff
cargo add --dev brotli
cargo add --dev flate2
cargo add --dev http
cargo add --dev zstd
```

建立测试数据目录并生成 gzip 文件:

```bash
mkdir data
# 写好下文完整代码中的 data/products.json 后执行
gzip -k data/products.json
```

**注意 `tokio` 这次没用 `full`**,只开了 `macros`(提供 `#[tokio::main]`)和 `rt-multi-thread`(多线程运行时)。这是个好习惯——按需开启能减少编译时间。

`tower-http` 的两个 feature 各对应一个方向:`compression-full` 压响应,`decompression-full` 解请求。名字里的 `full` 指四种算法全开;也可以只开 `compression-gzip` 这类单项。

## 分步构建

### 步骤 1:一个不压缩的 echo 服务

**要做什么**:先写出裸的 echo,作为对比基准。

```rust
// src/main.rs
use axum::{routing::post, Json, Router};
use serde_json::Value;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=trace", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app: Router = app();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

fn app() -> Router {
    Router::new().route("/", post(root))
}

async fn root(Json(value): Json<Value>) -> Json<Value> {
    Json(value)
}
```

**`Json<Value>` 进、`Json<Value>` 出** —— 原样返回收到的 JSON。`serde_json::Value` 能装任意 JSON([第 10 章](10-customize-extractor-error.md#步骤-1骨架与默认行为)用过)。

**`let app: Router = app();`** 这个类型标注是必需的:`app()` 的返回类型 `Router` 其实是 `Router<()>`,不标注的话某些情况下推断不出来。官方示例写了它。

**跑起来看**:

```bash
cargo run
```

准备一个稍大的 JSON 便于观察压缩效果:

```bash
python3 -c "
import json
print(json.dumps({'items': [{'id': i, 'name': 'item ' + str(i), 'desc': 'a fairly long description here'} for i in range(100)]}))
" > /tmp/big.json

wc -c /tmp/big.json
```

```
    8090 /tmp/big.json
```

发送并看响应大小:

```bash
curl -s -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: application/json' \
  --data-binary @/tmp/big.json | wc -c
```

```
    8090
```

**原样返回,8090 字节。**

**试试声明支持压缩:**

```bash
curl -s -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: application/json' \
  -H 'Accept-Encoding: gzip' \
  --data-binary @/tmp/big.json \
  -o /dev/null -w 'size=%{size_download}\n'
```

```
size=8090
```

**没有压缩。** 因为服务端还没配 `CompressionLayer` —— 客户端声明了能力,服务端不理会。

### 步骤 2:加上双向压缩

**要做什么**:挂上两个中间件。

```rust
// src/main.rs
use axum::{routing::post, Json, Router};
use serde_json::Value;
use tower::ServiceBuilder;                                                          // 新增
use tower_http::{compression::CompressionLayer, decompression::RequestDecompressionLayer};  // 新增
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// ...(main 不变)

// 改动:挂上两个压缩中间件
fn app() -> Router {
    Router::new().route("/", post(root)).layer(
        ServiceBuilder::new()
            .layer(RequestDecompressionLayer::new())
            .layer(CompressionLayer::new()),
    )
}

// ...(root 不变)
```

**就这几行。** `handler` 一个字都没改——前置知识讲过这正是中间件的价值。

**跑起来看**:

```bash
cargo run
```

**先验证响应压缩。** 分别用四种编码请求:

```bash
for enc in identity gzip deflate br zstd; do
  size=$(curl -s -X POST http://127.0.0.1:3000/ \
    -H 'Content-Type: application/json' \
    -H "Accept-Encoding: $enc" \
    --data-binary @/tmp/big.json \
    -o /dev/null -w '%{size_download}')
  printf "%-10s %s bytes\n" "$enc" "$size"
done
```

```
identity   8090 bytes
gzip        372 bytes
deflate     360 bytes
br          294 bytes
zstd        330 bytes
```

**8090 → 294~372 字节,压缩率超过 95%。** 这个 JSON 重复度很高所以效果夸张,真实数据通常在 60~80%。

**注意四种算法的差异**:`br` 最小,和前置知识那张表一致。`identity` 表示"不压缩",是客户端明确拒绝压缩的方式。

**看响应头:**

```bash
curl -s -i -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: application/json' \
  -H 'Accept-Encoding: br' \
  --data-binary @/tmp/big.json | head -6
```

```
HTTP/1.1 200 OK
content-type: application/json
content-encoding: br
vary: accept-encoding
transfer-encoding: chunked
```

三个关键头部:

- **`content-encoding: br`** —— 服务端选了 brotli
- **`vary: accept-encoding`** —— 前置知识讲过它对缓存的重要性,自动加上的
- **`transfer-encoding: chunked`** —— **没有 `content-length`**,因为压缩是流式进行的,压完之前不知道最终大小

**curl 会自动解压吗?** 加 `--compressed` 会:

```bash
curl -s --compressed -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: application/json' \
  --data-binary @/tmp/big.json | head -c 60
```

```
{"items":[{"desc":"a fairly long description here","id":0,
```

`--compressed` 让 curl 自动发 `Accept-Encoding` 并解压响应。**上面那些测大小的命令没加它,所以量到的是压缩后的字节数。**

**再验证请求压缩。** 把 JSON 压成 gzip 发过去:

```bash
gzip -c /tmp/big.json > /tmp/big.json.gz
wc -c /tmp/big.json.gz
```

```
     372 /tmp/big.json.gz
```

```bash
curl -s --compressed -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: application/json' \
  -H 'Content-Encoding: gzip' \
  --data-binary @/tmp/big.json.gz | head -c 60
```

```
{"items":[{"desc":"a fairly long description here","id":0,
```

**服务端自动解开了压缩的请求体**,`Json` 提取器拿到的是解压后的内容。**只传了 372 字节却处理了 8090 字节的 JSON。**

**验证不支持的编码。** 临时把中间件改成只支持 gzip:

```rust
            .layer(RequestDecompressionLayer::new().gzip(true).br(false).zstd(false).deflate(false)),
```

然后发一个 zstd 压缩的请求体:

```bash
zstd -c /tmp/big.json > /tmp/big.json.zst
curl -s -i -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: application/json' \
  -H 'Content-Encoding: zstd' \
  --data-binary @/tmp/big.json.zst | head -3
```

```
HTTP/1.1 415 Unsupported Media Type
accept-encoding: gzip
```

**`415` 加一个 `accept-encoding` 头部告诉客户端服务端支持什么。** 这就是前置知识说的"请求压缩没有协商机制"的后果——客户端只能试,失败了才知道。改回 `new()`。

### 步骤 3:验证顺序无关

**要做什么**:调换两个中间件的顺序,确认行为不变。

前置知识说过这两个中间件互不干扰。试一下:

```rust
fn app() -> Router {
    Router::new().route("/", post(root)).layer(
        ServiceBuilder::new()
            .layer(CompressionLayer::new())              // 临时:调换顺序
            .layer(RequestDecompressionLayer::new()),
    )
}
```

```bash
cargo run
```

```bash
curl -s --compressed -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: application/json' \
  -H 'Content-Encoding: gzip' \
  --data-binary @/tmp/big.json.gz | head -c 40
```

```
{"items":[{"desc":"a fairly long descr
```

**照常工作。** 响应压缩也仍然生效:

```bash
curl -s -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: application/json' -H 'Accept-Encoding: br' \
  --data-binary @/tmp/big.json -o /dev/null -w '%{size_download}\n'
```

```
294
```

**因为一个只动请求、一个只动响应**,洋葱的两侧互不影响。

**对比[第 19 章](19-request-id.md#步骤-4故意把顺序写反)那个反例**:那边 `SetRequestIdLayer` 和 `TraceLayer` 都在请求路上工作,而且后者依赖前者的产出,所以顺序错了就失效——**而且是静默失效**。

**判断中间件是否顺序敏感的方法:看它们是否作用在同一个方向、以及是否有数据依赖。** 都不满足就可以随意排列。

改回原来的顺序(虽然无所谓,但保持和官方示例一致)。

### 步骤 4:跑一遍官方测试

**要做什么**:把 `tests.rs` 加进来,用 `cargo test` 验证双向压缩和解压都正确。

官方示例带了一个 `tests.rs`,把请求体的 gzip/br/zstd 解压和响应体的 gzip/br/zstd 压缩全覆盖了。先挂上模块声明:

```rust
// src/main.rs
mod tests;   // 新增
```

`src/tests.rs` 内容较长(约 140 行),直接取自官方示例——测试写法(`oneshot` + 解压后断言)在[第 21 章](21-testing.md)讲透过,这里把完整文件一次给出:

```rust
// src/tests.rs —— 新建
use assert_json_diff::assert_json_eq;
use axum::{
    body::{Body, Bytes},
    response::Response,
};
use brotli::enc::BrotliEncoderParams;
use flate2::{read::GzDecoder, write::GzEncoder, Compression};
use http::{header, StatusCode};
use serde_json::{json, Value};
use std::io::{Read, Write};
use tower::ServiceExt;

use super::*;

#[tokio::test]
async fn handle_uncompressed_request_bodies() {
    // Given

    let body = json();

    let compressed_request = http::Request::post("/")
        .header(header::CONTENT_TYPE, "application/json")
        .body(json_body(&body))
        .unwrap();

    // When

    let response = app().oneshot(compressed_request).await.unwrap();

    // Then

    assert_eq!(response.status(), StatusCode::OK);
    assert_json_eq!(json_from_response(response).await, json());
}

#[tokio::test]
async fn decompress_gzip_request_bodies() {
    // Given

    let body = compress_gzip(&json());

    let compressed_request = http::Request::post("/")
        .header(header::CONTENT_TYPE, "application/json")
        .header(header::CONTENT_ENCODING, "gzip")
        .body(Body::from(body))
        .unwrap();

    // When

    let response = app().oneshot(compressed_request).await.unwrap();

    // Then

    assert_eq!(response.status(), StatusCode::OK);
    assert_json_eq!(json_from_response(response).await, json());
}

#[tokio::test]
async fn decompress_br_request_bodies() {
    // Given

    let body = compress_br(&json());

    let compressed_request = http::Request::post("/")
        .header(header::CONTENT_TYPE, "application/json")
        .header(header::CONTENT_ENCODING, "br")
        .body(Body::from(body))
        .unwrap();

    // When

    let response = app().oneshot(compressed_request).await.unwrap();

    // Then

    assert_eq!(response.status(), StatusCode::OK);
    assert_json_eq!(json_from_response(response).await, json());
}

#[tokio::test]
async fn decompress_zstd_request_bodies() {
    // Given

    let body = compress_zstd(&json());

    let compressed_request = http::Request::post("/")
        .header(header::CONTENT_TYPE, "application/json")
        .header(header::CONTENT_ENCODING, "zstd")
        .body(Body::from(body))
        .unwrap();

    // When

    let response = app().oneshot(compressed_request).await.unwrap();

    // Then

    assert_eq!(response.status(), StatusCode::OK);
    assert_json_eq!(json_from_response(response).await, json());
}

#[tokio::test]
async fn do_not_compress_response_bodies() {
    // Given
    let request = http::Request::post("/")
        .header(header::CONTENT_TYPE, "application/json")
        .body(json_body(&json()))
        .unwrap();

    // When

    let response = app().oneshot(request).await.unwrap();

    // Then

    assert_eq!(response.status(), StatusCode::OK);
    assert_json_eq!(json_from_response(response).await, json());
}

#[tokio::test]
async fn compress_response_bodies_with_gzip() {
    // Given
    let request = http::Request::post("/")
        .header(header::CONTENT_TYPE, "application/json")
        .header(header::ACCEPT_ENCODING, "gzip")
        .body(json_body(&json()))
        .unwrap();

    // When

    let response = app().oneshot(request).await.unwrap();

    // Then

    assert_eq!(response.status(), StatusCode::OK);
    let response_body = byte_from_response(response).await;
    let mut decoder = GzDecoder::new(response_body.as_ref());
    let mut decompress_body = String::new();
    decoder.read_to_string(&mut decompress_body).unwrap();
    assert_json_eq!(
        serde_json::from_str::<serde_json::Value>(&decompress_body).unwrap(),
        json()
    );
}

#[tokio::test]
async fn compress_response_bodies_with_br() {
    // Given
    let request = http::Request::post("/")
        .header(header::CONTENT_TYPE, "application/json")
        .header(header::ACCEPT_ENCODING, "br")
        .body(json_body(&json()))
        .unwrap();

    // When

    let response = app().oneshot(request).await.unwrap();

    // Then

    assert_eq!(response.status(), StatusCode::OK);
    let response_body = byte_from_response(response).await;
    let mut decompress_body = Vec::new();
    brotli::BrotliDecompress(&mut response_body.as_ref(), &mut decompress_body).unwrap();
    assert_json_eq!(
        serde_json::from_slice::<serde_json::Value>(&decompress_body).unwrap(),
        json()
    );
}

#[tokio::test]
async fn compress_response_bodies_with_zstd() {
    // Given
    let request = http::Request::post("/")
        .header(header::CONTENT_TYPE, "application/json")
        .header(header::ACCEPT_ENCODING, "zstd")
        .body(json_body(&json()))
        .unwrap();

    // When

    let response = app().oneshot(request).await.unwrap();

    // Then

    assert_eq!(response.status(), StatusCode::OK);
    let response_body = byte_from_response(response).await;
    let decompress_body = zstd::stream::decode_all(std::io::Cursor::new(response_body)).unwrap();
    assert_json_eq!(
        serde_json::from_slice::<serde_json::Value>(&decompress_body).unwrap(),
        json()
    );
}

fn json() -> Value {
    json!({
      "name": "foo",
      "mainProduct": {
        "typeId": "product",
        "id": "p1"
      },
    })
}

fn json_body(input: &Value) -> Body {
    Body::from(serde_json::to_vec(&input).unwrap())
}

async fn json_from_response(response: Response) -> Value {
    let body = byte_from_response(response).await;
    body_as_json(body)
}

async fn byte_from_response(response: Response) -> Bytes {
    axum::body::to_bytes(response.into_body(), usize::MAX)
        .await
        .unwrap()
}

fn body_as_json(body: Bytes) -> Value {
    serde_json::from_slice(body.as_ref()).unwrap()
}

fn compress_gzip(json: &Value) -> Vec<u8> {
    let request_body = serde_json::to_vec(&json).unwrap();

    let mut encoder = GzEncoder::new(Vec::new(), Compression::default());
    encoder.write_all(&request_body).unwrap();
    encoder.finish().unwrap()
}

fn compress_br(json: &Value) -> Vec<u8> {
    let request_body = serde_json::to_vec(&json).unwrap();
    let mut result = Vec::new();

    let params = BrotliEncoderParams::default();
    let _ = brotli::enc::BrotliCompress(&mut &request_body[..], &mut result, &params).unwrap();

    result
}

fn compress_zstd(json: &Value) -> Vec<u8> {
    let request_body = serde_json::to_vec(&json).unwrap();
    zstd::stream::encode_all(std::io::Cursor::new(request_body), 4).unwrap()
}

```

**七个测试分两组**:前四个带 `CONTENT_ENCODING` 的请求测"请求体解压",后三个带 `ACCEPT_ENCODING` 的请求测"响应体压缩",外加一个不带任何头的 `do_not_compress` 兜底。请求组的断言是"解压后原样到达 handler",响应组的断言是"压缩后能解压回原 JSON"——和步骤 2 里 curl 验证的是同一件事,只是不用起服务器。

**每个测试都是 Given/When/Then 三段式**:`// Given` 构造输入、`// When` 发起请求、`// Then` 断言输出,和官方源码逐字一致。

**跑起来看**:

```bash
cargo test
```

```
running 8 tests
test compress_response_bodies_with_br ... ok
test compress_response_bodies_with_gzip ... ok
test compress_response_bodies_with_zstd ... ok
test decompress_br_request_bodies ... ok
test decompress_gzip_request_bodies ... ok
test decompress_zstd_request_bodies ... ok
test do_not_compress_response_bodies ... ok
test handle_uncompressed_request_bodies ... ok
```

顺便注意辅助函数 `compress_gzip`/`compress_br`/`compress_zstd` 就是步骤 2 里"手工压缩一个请求体"的复用——官方把验证压缩用的工具也写成了可测代码,这就是[第 21 章](21-testing.md)说的"为可测性而设计"。

## 完整代码

`src/main.rs`:

```rust
use axum::{routing::post, Json, Router};
use serde_json::Value;
use tower::ServiceBuilder;
use tower_http::{compression::CompressionLayer, decompression::RequestDecompressionLayer};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[cfg(test)]
mod tests;

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=trace", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app: Router = app();

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

fn app() -> Router {
    Router::new().route("/", post(root)).layer(
        ServiceBuilder::new()
            .layer(RequestDecompressionLayer::new())
            .layer(CompressionLayer::new()),
    )
}

async fn root(Json(value): Json<Value>) -> Json<Value> {
    Json(value)
}
```

`src/tests.rs`:

```rust
use assert_json_diff::assert_json_eq;
use axum::{
    body::{Body, Bytes},
    response::Response,
};
use brotli::enc::BrotliEncoderParams;
use flate2::{read::GzDecoder, write::GzEncoder, Compression};
use http::{header, StatusCode};
use serde_json::{json, Value};
use std::io::{Read, Write};
use tower::ServiceExt;

use super::*;

#[tokio::test]
async fn handle_uncompressed_request_bodies() {
    // Given

    let body = json();

    let compressed_request = http::Request::post("/")
        .header(header::CONTENT_TYPE, "application/json")
        .body(json_body(&body))
        .unwrap();

    // When

    let response = app().oneshot(compressed_request).await.unwrap();

    // Then

    assert_eq!(response.status(), StatusCode::OK);
    assert_json_eq!(json_from_response(response).await, json());
}

#[tokio::test]
async fn decompress_gzip_request_bodies() {
    // Given

    let body = compress_gzip(&json());

    let compressed_request = http::Request::post("/")
        .header(header::CONTENT_TYPE, "application/json")
        .header(header::CONTENT_ENCODING, "gzip")
        .body(Body::from(body))
        .unwrap();

    // When

    let response = app().oneshot(compressed_request).await.unwrap();

    // Then

    assert_eq!(response.status(), StatusCode::OK);
    assert_json_eq!(json_from_response(response).await, json());
}

#[tokio::test]
async fn decompress_br_request_bodies() {
    // Given

    let body = compress_br(&json());

    let compressed_request = http::Request::post("/")
        .header(header::CONTENT_TYPE, "application/json")
        .header(header::CONTENT_ENCODING, "br")
        .body(Body::from(body))
        .unwrap();

    // When

    let response = app().oneshot(compressed_request).await.unwrap();

    // Then

    assert_eq!(response.status(), StatusCode::OK);
    assert_json_eq!(json_from_response(response).await, json());
}

#[tokio::test]
async fn decompress_zstd_request_bodies() {
    // Given

    let body = compress_zstd(&json());

    let compressed_request = http::Request::post("/")
        .header(header::CONTENT_TYPE, "application/json")
        .header(header::CONTENT_ENCODING, "zstd")
        .body(Body::from(body))
        .unwrap();

    // When

    let response = app().oneshot(compressed_request).await.unwrap();

    // Then

    assert_eq!(response.status(), StatusCode::OK);
    assert_json_eq!(json_from_response(response).await, json());
}

#[tokio::test]
async fn do_not_compress_response_bodies() {
    // Given
    let request = http::Request::post("/")
        .header(header::CONTENT_TYPE, "application/json")
        .body(json_body(&json()))
        .unwrap();

    // When

    let response = app().oneshot(request).await.unwrap();

    // Then

    assert_eq!(response.status(), StatusCode::OK);
    assert_json_eq!(json_from_response(response).await, json());
}

#[tokio::test]
async fn compress_response_bodies_with_gzip() {
    // Given
    let request = http::Request::post("/")
        .header(header::CONTENT_TYPE, "application/json")
        .header(header::ACCEPT_ENCODING, "gzip")
        .body(json_body(&json()))
        .unwrap();

    // When

    let response = app().oneshot(request).await.unwrap();

    // Then

    assert_eq!(response.status(), StatusCode::OK);
    let response_body = byte_from_response(response).await;
    let mut decoder = GzDecoder::new(response_body.as_ref());
    let mut decompress_body = String::new();
    decoder.read_to_string(&mut decompress_body).unwrap();
    assert_json_eq!(
        serde_json::from_str::<serde_json::Value>(&decompress_body).unwrap(),
        json()
    );
}

#[tokio::test]
async fn compress_response_bodies_with_br() {
    // Given
    let request = http::Request::post("/")
        .header(header::CONTENT_TYPE, "application/json")
        .header(header::ACCEPT_ENCODING, "br")
        .body(json_body(&json()))
        .unwrap();

    // When

    let response = app().oneshot(request).await.unwrap();

    // Then

    assert_eq!(response.status(), StatusCode::OK);
    let response_body = byte_from_response(response).await;
    let mut decompress_body = Vec::new();
    brotli::BrotliDecompress(&mut response_body.as_ref(), &mut decompress_body).unwrap();
    assert_json_eq!(
        serde_json::from_slice::<serde_json::Value>(&decompress_body).unwrap(),
        json()
    );
}

#[tokio::test]
async fn compress_response_bodies_with_zstd() {
    // Given
    let request = http::Request::post("/")
        .header(header::CONTENT_TYPE, "application/json")
        .header(header::ACCEPT_ENCODING, "zstd")
        .body(json_body(&json()))
        .unwrap();

    // When

    let response = app().oneshot(request).await.unwrap();

    // Then

    assert_eq!(response.status(), StatusCode::OK);
    let response_body = byte_from_response(response).await;
    let decompress_body = zstd::stream::decode_all(std::io::Cursor::new(response_body)).unwrap();
    assert_json_eq!(
        serde_json::from_slice::<serde_json::Value>(&decompress_body).unwrap(),
        json()
    );
}

fn json() -> Value {
    json!({
      "name": "foo",
      "mainProduct": {
        "typeId": "product",
        "id": "p1"
      },
    })
}

fn json_body(input: &Value) -> Body {
    Body::from(serde_json::to_vec(&input).unwrap())
}

async fn json_from_response(response: Response) -> Value {
    let body = byte_from_response(response).await;
    body_as_json(body)
}

async fn byte_from_response(response: Response) -> Bytes {
    axum::body::to_bytes(response.into_body(), usize::MAX)
        .await
        .unwrap()
}

fn body_as_json(body: Bytes) -> Value {
    serde_json::from_slice(body.as_ref()).unwrap()
}

fn compress_gzip(json: &Value) -> Vec<u8> {
    let request_body = serde_json::to_vec(&json).unwrap();

    let mut encoder = GzEncoder::new(Vec::new(), Compression::default());
    encoder.write_all(&request_body).unwrap();
    encoder.finish().unwrap()
}

fn compress_br(json: &Value) -> Vec<u8> {
    let request_body = serde_json::to_vec(&json).unwrap();
    let mut result = Vec::new();

    let params = BrotliEncoderParams::default();
    let _ = brotli::enc::BrotliCompress(&mut &request_body[..], &mut result, &params).unwrap();

    result
}

fn compress_zstd(json: &Value) -> Vec<u8> {
    let request_body = serde_json::to_vec(&json).unwrap();
    zstd::stream::encode_all(std::io::Cursor::new(request_body), 4).unwrap()
}

```

`data/products.json`:

```json
{
    "products": [
        {
            "id": 1,
            "name": "Product 1"
        },
        {
            "id": 2,
            "name": "Product 2"
        }
    ]
}
```

```bash
gzip -k data/products.json
```

## 代价与边界

**没有设置最小压缩阈值。** 前置知识提过:几十字节的响应压完可能更大,而且白费 CPU。

```bash
curl -s -X POST http://127.0.0.1:3000/ \
  -H 'Content-Type: application/json' -H 'Accept-Encoding: gzip' \
  -d '{"a":1}' -o /dev/null -w '%{size_download}\n'
```

小响应会看到压缩后反而更大或差不多。**生产环境应该配 `CompressionLayer::new().compress_when(SizeAbove::new(1024))`** 之类的条件,只压超过某个大小的响应。

**和流式响应冲突。** 这和[第 33 章](33-sse.md#代价与边界) SSE 那个问题同源:压缩算法需要积累一定数据才能有效压缩,所以会**缓冲**。对 SSE 来说这意味着事件不能及时送达——服务端产生了一条消息,但压缩器还在等更多数据。

`CompressionLayer` 对 `text/event-stream` 有特殊处理(不压缩),但**自定义的流式响应仍然可能受影响**。稳妥做法是给流式路由单独配置,别全局套用——[第 33 章](33-sse.md#代价与边界)对缓冲中间件也是同样的建议。

**BREACH 攻击。** 这是压缩相关最重要的安全问题,而且很少被提及。

**原理**:如果响应里同时包含**攻击者可控的内容**和**秘密**(比如 CSRF token),攻击者可以通过观察压缩后的响应大小来逐字节猜出秘密——猜对的字符会和秘密产生重复,压缩率变高、响应变小。

**受影响的场景**:HTML 页面里既嵌了用户输入的搜索词,又嵌了 CSRF token。

**缓解办法**:

- 对包含敏感 token 的响应关闭压缩
- 给 token 加随机长度的填充,掩盖大小变化
- 每次请求换一个 token
- 把 token 放在响应头而非 body

**纯 JSON API 通常风险较低**(攻击者较难精确控制响应内容),但**混合了用户输入和敏感数据的 HTML 页面要当心**。

**请求解压是个攻击面。** `RequestDecompressionLayer` 会解压客户端发来的数据,这意味着**"压缩炸弹"**:一个几 KB 的 gzip 文件可以解压成几 GB。

`tower-http` 有没有内置限制取决于版本,**稳妥做法是配合 [第 13 章](13-multipart-form.md#两层请求体限制)讲的 `RequestBodyLimitLayer`** —— 但注意它限制的是**压缩后**的大小,解压后可能仍然巨大。真正的防护需要在解压时限制输出大小。

**CPU 开销。** 压缩不是免费的。高 QPS 服务上,brotli 的压缩开销可能相当可观。可以考虑:只用 gzip(最快)、降低压缩级别、或者干脆交给 CDN/反向代理做。

**生产环境通常在代理层压缩。** 和[第 32 章](32-tls-rustls.md#代价与边界) TLS、[第 38 章](38-static-file-server.md#代价与边界)静态文件的结论一致:Nginx/CDN 处理压缩更高效,而且能对静态资源做预压缩(压一次存起来,不用每次请求都压)。

**应用内压缩的合理场景**:没有代理层、动态内容为主、或者需要针对特定路由精细控制。

## 动手练习

**1. 加上最小压缩阈值。**

提示:`CompressionLayer::new().compress_when(tower_http::compression::predicate::SizeAbove::new(1024))`。做完对比小响应(`-d '{"a":1}'`)和大响应的 `content-encoding` 头部——前者应该没有这个头部了。**这是「代价与边界」里最直接的一处改进。**

**2. 只对特定路由启用压缩。**

提示:`CompressionLayer` 可以挂在单条路由上(`post(root).layer(CompressionLayer::new())`),而不是整个 `Router`。加两条路由,一条压一条不压,对比响应。**这是应对流式响应冲突的正确做法。**

**3. 复现压缩率随内容变化的现象。**

提示:准备三个文件——高重复度的 JSON(上面那个)、随机字符串、以及一张真实的 PNG 图片(base64 编码后放进 JSON)。分别 POST 并观察压缩后的大小。你会看到:重复内容压缩率极高、随机内容几乎压不动、已压缩的图片数据**可能变大**。这解释了为什么 `CompressionLayer` 要按 `Content-Type` 跳过某些类型。

## 小结与下一章预告

压缩是 HTTP 内容协商的一部分,两个方向不对称:**响应压缩是协商的**(客户端用 `Accept-Encoding` 声明能力,服务端用 `Content-Encoding` 告知选择),**请求压缩是单方面通知的**(客户端直接压,服务端要么能解要么 `415`)。因为没有协商机制,请求压缩在公网上很少用。

四种算法各有取舍:**gzip 是最低保障**、**brotli 对文本效果最好**(比 gzip 小 15~25%)、**zstd 快且压缩率高**。`CompressionLayer` 会自动选一个双方都支持的。

`Vary: accept-encoding` 会自动加上——**没有它,CDN 可能把压缩响应发给不支持的客户端**。

本章最值得体会的是 **handler 完全不知道压缩存在**:它拿到的是解压后的数据,返回的会被自动压缩。中间件把横切关注点从业务代码里剥离了出去。

**这两个中间件顺序无关**,因为一个只动请求、一个只动响应——和[第 19 章](19-request-id.md)那个顺序敏感的例子形成对比。判断方法:**看它们是否作用在同一方向、是否有数据依赖**。

几个不该忽略的边界:没配最小阈值会让小响应白费 CPU;压缩会缓冲数据,**和 SSE 这类流式响应冲突**;**BREACH 攻击**让"混合了用户输入和 CSRF token 的 HTML 页面"不该压缩;请求解压是**压缩炸弹**的攻击面。

生产环境通常在 CDN 或反向代理层压缩,和 TLS([第 32 章](32-tls-rustls.md))、静态文件([第 38 章](38-static-file-server.md))的结论一致。

本章处理的是"数据在传输中怎么变小"。[第 40 章 stream-to-file](40-stream-to-file.md) 处理另一个数据流动的问题:**上传的文件怎么直接写进磁盘而不经过内存**。[第 13 章](13-multipart-form.md#代价与边界)那个 `field.bytes()` 把整个文件读进内存的问题,在这一章会得到正确的解法。
