# 25. compression

对应示例：`examples/compression`

本章目标：理解 HTTP 请求体解压和响应体压缩的区别，并学会用 `RequestDecompressionLayer` 和 `CompressionLayer` 自动处理 gzip、br、zstd 等编码。

上一章 CORS 讲的是浏览器能不能读取响应。  
这一章讲的是请求和响应在网络上传输时能不能更小。

## 这个小项目在做什么

应用只有一个接口：

```text
POST /
```

它接收 JSON，并原样返回 JSON：

```text
客户端 POST JSON
-> 后端解析 JSON
-> 后端返回同一个 JSON
```

但 Router 外面加了两层 middleware：

```text
RequestDecompressionLayer
CompressionLayer
```

它们让服务具备两种能力：

1. 客户端发来压缩过的请求体时，后端先自动解压，再交给 `Json<Value>` 解析。
2. 客户端声明自己能接收压缩响应时，后端自动压缩响应体。

请求主线是：

```text
客户端 POST /
-> 如果请求 header 有 content-encoding，RequestDecompressionLayer 解压请求体
-> Json<Value> 解析 JSON
-> handler 原样返回 Json<Value>
-> 如果请求 header 有 accept-encoding，CompressionLayer 可能压缩响应体
-> 客户端收到响应
```

## 先理解两个 header

压缩相关最容易混淆的是这两个 header：

```text
content-encoding
accept-encoding
```

它们方向不同。

### content-encoding

`content-encoding` 描述“当前请求体或响应体已经用了什么编码”。

例如客户端发送：

```text
content-encoding: gzip
```

意思是：

```text
我发给你的 body 是 gzip 压缩过的，你先解压再读
```

所以它影响的是服务端读取请求体。

### accept-encoding

`accept-encoding` 描述“客户端能接受哪些响应压缩格式”。

例如客户端发送：

```text
accept-encoding: gzip
```

意思是：

```text
你可以把响应体用 gzip 压缩后发给我
```

所以它影响的是服务端返回响应体。

一句话区分：

```text
content-encoding = 这份 body 现在是什么编码
accept-encoding = 我能接受你用什么编码返回
```

## 文件和依赖

这个 example 有三个主要文件：

1. `examples/compression/Cargo.toml`：声明 Axum、tower、tower-http 压缩 feature、测试依赖。
2. `examples/compression/src/main.rs`：实现 Router、压缩 layer 和 JSON handler。
3. `examples/compression/src/tests.rs`：验证未压缩请求、压缩请求、压缩响应等行为。

关键依赖：

- `axum`：提供 `Router`、`Json`。
- `tower`：提供 `ServiceBuilder`，组合多层 middleware。
- `tower-http`：提供 `CompressionLayer` 和 `RequestDecompressionLayer`。
- `serde_json`：使用通用 JSON 值 `Value`。
- `tracing` / `tracing-subscriber`：输出服务日志。
- `brotli`、`flate2`、`zstd`：测试中用来构造和解析压缩数据。

`tower-http` 需要启用完整压缩和解压 feature：

````toml
tower-http = { version = "0.6.1", features = ["compression-full", "decompression-full"] }
````

## 第一步：把应用拆成 app 函数

源码：

````rust
fn app() -> Router {
    Router::new().route("/", post(root)).layer(
        ServiceBuilder::new()
            .layer(RequestDecompressionLayer::new())
            .layer(CompressionLayer::new()),
    )
}
````

这里没有直接在 `main` 里创建 Router，而是单独写了 `app()`。

好处是：

```text
main 可以启动真实服务
tests.rs 可以直接调用 app() 做测试
```

这是写可测试 Axum 项目的常见方式。

## 第二步：注册 POST 路由

源码：

````rust
Router::new().route("/", post(root))
````

只注册一个接口：

```text
POST /
```

它对应的 handler 是 `root`。

为什么用 POST？

因为这一章要演示请求体。  
GET 通常没有 body，不适合作为请求体解压示例。

## 第三步：RequestDecompressionLayer 解压请求体

源码：

````rust
.layer(RequestDecompressionLayer::new())
````

这一层会检查请求 header：

```text
content-encoding: gzip
content-encoding: br
content-encoding: zstd
```

如果发现请求体是压缩过的，就先解压。  
解压之后，后面的 `Json<Value>` 才能正常解析。

没有这一层时，如果客户端发来 gzip 压缩过的 JSON，`Json<Value>` 看到的是压缩后的二进制数据，而不是 JSON 文本，解析就会失败。

## 第四步：CompressionLayer 压缩响应体

源码：

````rust
.layer(CompressionLayer::new())
````

这一层会检查请求 header：

```text
accept-encoding: gzip
accept-encoding: br
accept-encoding: zstd
```

如果客户端表示自己能接收压缩响应，服务端就可以把响应体压缩后再返回。

如果客户端没有发送 `accept-encoding`，就返回普通未压缩响应。

所以压缩响应不是“服务端想压就压”。  
它要看客户端是否声明自己支持对应编码。

## 第五步：为什么用 ServiceBuilder

源码：

````rust
ServiceBuilder::new()
    .layer(RequestDecompressionLayer::new())
    .layer(CompressionLayer::new())
````

`ServiceBuilder` 用来组合多个 middleware。  
这里你可以按请求进入服务的方向理解：

```text
请求进入
-> RequestDecompressionLayer 解压请求体
-> handler 处理业务
-> CompressionLayer 压缩响应体
响应返回
```

这两个 layer 分别处理不同方向：

- `RequestDecompressionLayer`：主要影响 request body。
- `CompressionLayer`：主要影响 response body。

把它们放在 Router 外层后，所有匹配的路由都会拥有压缩和解压能力。

## 第六步：handler 不需要知道压缩细节

源码：

````rust
async fn root(Json(value): Json<Value>) -> Json<Value> {
    Json(value)
}
````

`Json(value): Json<Value>` 是 extractor。  
它负责从请求体里解析 JSON。

因为 `RequestDecompressionLayer` 已经提前解压，handler 看到的就是普通 JSON。  
handler 不需要判断：

```text
这个请求是不是 gzip？
这个请求是不是 br？
这个请求是不是 zstd？
```

返回时也一样。  
handler 只返回普通 `Json(value)`，压缩响应由 `CompressionLayer` 处理。

这就是 middleware 的价值：

```text
通用协议能力放 middleware
业务逻辑保持干净
```

## 第七步：测试覆盖了哪些场景

这个 example 的 `tests.rs` 很有价值。  
它验证了几类行为。

### 未压缩请求也能处理

测试名：

````rust
handle_uncompressed_request_bodies
````

它发送普通 JSON 请求，没有 `content-encoding`。  
预期后端返回 200，并原样返回 JSON。

这说明加了 `RequestDecompressionLayer` 后，并不会破坏普通请求。

### 能解压 gzip、br、zstd 请求体

测试名：

````rust
decompress_gzip_request_bodies
decompress_br_request_bodies
decompress_zstd_request_bodies
````

这些测试会先把 JSON 压缩，然后设置：

```text
content-encoding: gzip
content-encoding: br
content-encoding: zstd
```

预期 handler 仍然能拿到正确 JSON。

### 没有 accept-encoding 时不压缩响应

测试名：

````rust
do_not_compress_response_bodies
````

它没有发送 `accept-encoding`。  
预期响应体可以直接按 JSON 解析。

### 有 accept-encoding 时压缩响应

测试名：

````rust
compress_response_bodies_with_gzip
compress_response_bodies_with_br
compress_response_bodies_with_zstd
````

这些测试发送：

```text
accept-encoding: gzip
accept-encoding: br
accept-encoding: zstd
```

然后把响应体解压，再检查 JSON 是否正确。

测试不是额外负担。  
它刚好把本章的协议规则写成了可验证的例子。

## 函数职责速查

- `main`：初始化日志，调用 `app()` 创建 Router，绑定端口并启动服务。
- `app`：注册 POST 路由，并给 Router 加请求解压和响应压缩 layer。
- `root`：接收 JSON，原样返回 JSON。
- `tests.rs` 中的测试函数：验证普通请求、压缩请求和压缩响应。

## 带中文注释的手写版

````rust
use axum::{routing::post, Json, Router};
use serde_json::Value;
use tower::ServiceBuilder;
use tower_http::{compression::CompressionLayer, decompression::RequestDecompressionLayer};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 测试模块。
// tests.rs 会直接调用 app()，不需要真的启动 TCP 端口。
#[cfg(test)]
mod tests;

#[tokio::main]
async fn main() {
    // 初始化日志。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=trace", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 创建应用。
    let app: Router = app();

    // 绑定本地端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动服务。
    axum::serve(listener, app).await.unwrap();
}

// 单独封装 app，方便 main 启动，也方便测试直接调用。
fn app() -> Router {
    Router::new().route("/", post(root)).layer(
        ServiceBuilder::new()
            // 请求进入时，如果 body 带 content-encoding，就先解压。
            .layer(RequestDecompressionLayer::new())
            // 响应返回时，如果客户端 Accept-Encoding 支持，就压缩响应。
            .layer(CompressionLayer::new()),
    )
}

// 接收 JSON，并原样返回 JSON。
// 压缩和解压都由 middleware 处理，handler 不需要知道细节。
async fn root(Json(value): Json<Value>) -> Json<Value> {
    Json(value)
}
````

## 运行和验证

运行服务：

````bash
cargo run -p example-compression
````

发送普通 JSON 请求：

````bash
curl -s \
  -H 'content-type: application/json' \
  -d '{"name":"foo"}' \
  http://127.0.0.1:3000/
````

预期返回：

````json
{"name":"foo"}
````

查看 gzip 响应 header：

````bash
curl -s -D - -o /tmp/axum-compressed.out \
  -H 'content-type: application/json' \
  -H 'accept-encoding: gzip' \
  -d '{"name":"foo"}' \
  http://127.0.0.1:3000/
````

重点看响应 header 里是否出现：

```text
content-encoding: gzip
```

运行 example 自带测试：

````bash
cargo test -p example-compression
````

这些测试会覆盖 gzip、br、zstd 的请求解压和响应压缩。

## 常见卡点

### 1. 为什么直接 curl 看起来是乱码？

因为你要求后端返回 gzip、br 或 zstd 压缩响应时，响应体本来就是二进制压缩数据。  
终端直接显示会像乱码。

可以只看 header，或者用支持自动解压的客户端。

### 2. 为什么没有 Accept-Encoding 时响应没压缩？

因为服务端不能假设客户端一定能解压。  
客户端必须通过 `accept-encoding` 表示自己支持哪些编码。

### 3. 为什么压缩请求体要设置 Content-Encoding？

否则服务端不知道请求体已经被压缩。  
没有 `content-encoding` 时，`Json<Value>` 会把压缩后的二进制当成 JSON 解析，通常会失败。

### 4. 压缩一定能提升性能吗？

不一定。  
压缩能减少网络传输体积，但会增加 CPU 开销。

一般来说：

- 大文本、JSON、HTML：压缩通常有收益。
- 很小的响应：收益可能不明显。
- 图片、视频、zip 文件：通常已经压缩过，再压缩意义不大。

### 5. 业务 handler 需要手动解压吗？

不需要。  
这一章的核心就是把协议层能力放到 middleware，让 handler 只处理已经解压好的业务数据。

## 手写任务

建议按下面顺序自己敲一遍：

1. 写一个 `POST /`，用 `Json<Value>` 接收 JSON。
2. 让 handler 原样返回 `Json(value)`。
3. 把 Router 封装成 `fn app() -> Router`。
4. 用 `ServiceBuilder` 加 `RequestDecompressionLayer`。
5. 再加 `CompressionLayer`。
6. 用普通 curl 请求确认接口仍然正常。
7. 用 `accept-encoding: gzip` 请求确认响应 header。

加深练习：

1. 阅读 `tests.rs`，把每个测试名翻译成一句中文。
2. 试着新增一个 deflate 相关测试。
3. 写一个较大的 JSON 响应，比较压缩前后的 body 大小。

## 本章真正要记住什么

压缩和解压是 HTTP 协议层能力，不应该散落在业务 handler 里。

这一章最重要的方向关系是：

```text
content-encoding -> 请求体或响应体当前已经怎么编码
accept-encoding  -> 客户端能接受服务端怎么压缩响应
```

在 Axum 项目里，可以用：

````rust
RequestDecompressionLayer::new()
CompressionLayer::new()
````

把请求解压和响应压缩统一放到 Router 外层。

## 源码对照

本章手写版对应源码：

- `examples/compression/src/main.rs`
- `examples/compression/src/tests.rs`
- `examples/compression/Cargo.toml`
