# 26. compression

对应示例：`examples/compression`

上一章 CORS 讲浏览器能不能读取响应,这章讲请求和响应在网络上传输时能不能更小。理解 HTTP 请求体**解压**(`RequestDecompressionLayer`)和响应体**压缩**(`CompressionLayer`)的区别,自动处理 gzip/br/zstd。

## Cargo.toml

````toml
[package]
name = "example-compression"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
serde_json = "1"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
tower = "0.5.2"
tower-http = { version = "0.6.1", features = ["compression-full", "decompression-full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

[dev-dependencies]
assert-json-diff = "2.0"
brotli = "8"
flate2 = "1"
http = "1"
zstd = "0.13"
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

````rust
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
````

## 运行

````bash
cd examples
cargo run -p example-compression
````

普通 JSON 请求:

````bash
curl -s \
  -H 'content-type: application/json' \
  -d '{"name":"foo"}' \
  http://127.0.0.1:3000/
# 预期: {"name":"foo"}
````

查看 gzip 响应 header:

````bash
curl -s -D - -o /tmp/axum-compressed.out \
  -H 'content-type: application/json' \
  -H 'accept-encoding: gzip' \
  -d '{"name":"foo"}' \
  http://127.0.0.1:3000/
````

重点看响应 header:

````text
content-encoding: gzip
````

运行测试:

````bash
cargo test -p example-compression
````

## 解读

### 两个方向不同的 header(关键)

```text
content-encoding = 这份 body 现在已经用了什么编码
accept-encoding  = 我能接受你用什么编码返回
```

- **`content-encoding: gzip`**:我发给你的 body 是 gzip 压缩过的,你先解压再读 → 影响服务端读请求体。
- **`accept-encoding: gzip`**:你可以把响应体用 gzip 压缩后发给我 → 影响服务端返回响应体。

### 两个 layer 对应两个方向

````rust
ServiceBuilder::new()
    .layer(RequestDecompressionLayer::new())  // 请求解压
    .layer(CompressionLayer::new())           // 响应压缩
````

请求进入服务的方向:

```text
请求进入 → RequestDecompressionLayer 解压请求体 → handler 处理 → CompressionLayer 压缩响应体 → 响应返回
```

### `RequestDecompressionLayer` 解压请求体

检查请求 `content-encoding` header(gzip/br/zstd),发现请求体压缩过就先解压,后面的 `Json<Value>` 才能正常解析。没有这层时,客户端发 gzip 压缩的 JSON,`Json<Value>` 看到的是压缩二进制而非 JSON 文本,解析失败。

### `CompressionLayer` 压缩响应体

检查请求 `accept-encoding` header,客户端声明能接收压缩响应就把响应体压缩后返回;没发送 `accept-encoding` 就返回普通未压缩响应。**压缩响应不是"服务端想压就压"**,要看客户端是否声明支持对应编码。

### handler 不用关心压缩细节

````rust
async fn root(Json(value): Json<Value>) -> Json<Value> {
    Json(value)
}
````

`RequestDecompressionLayer` 已提前解压,handler 看到普通 JSON;返回时也只返回普通 `Json(value)`,压缩响应由 `CompressionLayer` 处理。这就是 middleware 的价值:**通用协议能力放 middleware,业务逻辑保持干净**。

### 测试覆盖(`tests.rs`)

`tests.rs` 把本章协议规则写成可验证例子:

- `handle_uncompressed_request_bodies`:未压缩请求也正常处理(加 layer 不破坏普通请求)。
- `decompress_{gzip,br,zstd}_request_bodies`:压缩请求体 + 对应 `content-encoding`,handler 仍拿到正确 JSON。
- `do_not_compress_response_bodies`:没 `accept-encoding` 时不压缩响应。
- `compress_response_bodies_with_{gzip,br,zstd}`:有 `accept-encoding` 时压缩响应,解压后 JSON 正确。

## 常见问题

**直接 curl 看起来是乱码?** 因为你要求后端返回 gzip/br/zstd 响应,响应体是二进制压缩数据。只看 header,或用支持自动解压的客户端。

**没 `accept-encoding` 时响应没压缩?** 服务端不能假设客户端能解压,必须客户端通过 `accept-encoding` 声明支持。

**压缩请求体要设 `content-encoding`?** 否则服务端不知道请求体已压缩,`Json<Value>` 把压缩二进制当 JSON 解析会失败。

**压缩一定提升性能?** 不一定。大文本/JSON/HTML 压缩通常有收益;很小响应收益不明显;图片/视频/zip 已压缩过,再压缩意义不大。压缩减少网络体积但增加 CPU 开销。

## 手写任务

按下面顺序敲:

1. 写 `POST /` 用 `Json<Value>` 接收 JSON。
2. handler 原样返回 `Json(value)`。
3. Router 封装成 `fn app() -> Router`。
4. `ServiceBuilder` 加 `RequestDecompressionLayer`。
5. 再加 `CompressionLayer`。
6. 普通 curl 请求确认接口正常。
7. `accept-encoding: gzip` 请求确认响应 header。

加深练习:

1. 读 `tests.rs`,把每个测试名翻译成一句中文。
2. 新增 deflate 相关测试。
3. 写较大 JSON 响应,比较压缩前后 body 大小。

## 小结

- 两个方向不同的 header:`content-encoding`(body 当前编码,影响读请求体)、`accept-encoding`(客户端能接受的响应编码,影响返回响应体)。
- `RequestDecompressionLayer` 解压请求体,`CompressionLayer` 压缩响应体,放 Router 外层后所有路由都有压缩/解压能力。
- 压缩是协议层能力,放 middleware;handler 不用关心压缩细节,只处理已解压的业务数据。
- 压缩响应要看客户端 `accept-encoding`;不是所有响应都适合压缩(小响应/已压缩文件收益小)。

## 源码对照

- `examples/compression/Cargo.toml`
- `examples/compression/src/main.rs`
- `examples/compression/src/tests.rs`
