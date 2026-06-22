# 09. parse-body-based-on-content-type

对应示例：`examples/parse-body-based-on-content-type`

本章目标：自定义 `JsonOrForm<T>` 提取器，根据 `content-type` 自动选择用 JSON 还是表单解析请求体。

这一章比前几章更偏 Axum 提取器机制。第一次学习可以先抓住主线：同一个接口既能接 JSON，也能接普通表单，判断依据是请求头里的 `content-type`。

## 这个小项目在做什么

前面已经学过两种 body 解析方式：

- 第 02 章：`Json<T>` 解析 `application/json`。
- 第 06 章：`Form<T>` 解析 `application/x-www-form-urlencoded`。

这一章把它们合在一起：

```text
如果 content-type 是 application/json
-> 用 Json<T> 解析

如果 content-type 是 application/x-www-form-urlencoded
-> 用 Form<T> 解析

其他 content-type
-> 返回 415 Unsupported Media Type
```

请求主线是：

```text
客户端 POST /
-> JsonOrForm<Payload> 读取 content-type
-> 根据 content-type 选择 Json<Payload> 或 Form<Payload>
-> handler 拿到 Payload
-> dbg!(payload) 打印出来
```

这个 example 没有返回业务响应，重点是演示自定义 extractor。

## 先理解 content-type 的作用

HTTP 请求 body 只是一串字节。服务端要知道怎样解释这些字节，就要看 `content-type`。

同样是提交 `foo = bar`，JSON 和表单格式完全不同：

```text
content-type: application/json
body: {"foo":"bar"}
```

```text
content-type: application/x-www-form-urlencoded
body: foo=bar
```

所以后端不能只看 body 内容，还要看 header：

```text
Header 决定怎么解析 Body。
```

`JsonOrForm<T>` 做的就是这件事。

## 文件和依赖

这个 example 有两个文件：

1. `examples/parse-body-based-on-content-type/Cargo.toml`：声明 Axum、Serde、Tokio、tracing。
2. `examples/parse-body-based-on-content-type/src/main.rs`：实现路由、数据结构和自定义提取器。

关键依赖：

- `axum`：提供 `FromRequest`、`Request`、`Json`、`Form`、`RequestExt`、`IntoResponse`。
- `serde`：让 `Payload` 能从 JSON 或表单反序列化，也能序列化。
- `tokio`：提供异步运行时。
- `tracing` / `tracing-subscriber`：初始化日志。

## 第一步：注册一个 POST 接口

源码：

````rust
let app = Router::new().route("/", post(handler));
````

这个应用只有一个接口：

```text
POST / -> handler
```

它不像第 02 章写成 `Json<Payload>`，也不像第 06 章写成 `Form<Payload>`。  
它会写成：

````rust
async fn handler(JsonOrForm(payload): JsonOrForm<Payload>) {
    dbg!(payload);
}
````

也就是说，handler 不关心客户端到底用了 JSON 还是表单。  
handler 只想拿到最终的 `Payload`。

## 第二步：定义输入类型 Payload

源码：

````rust
#[derive(Debug, Serialize, Deserialize)]
struct Payload {
    foo: String,
}
````

这个结构体表示请求里应该有一个字段：

```text
foo
```

JSON 请求长这样：

````json
{"foo":"bar"}
````

表单请求长这样：

````text
foo=bar
````

为什么同时 derive `Serialize` 和 `Deserialize`？

- `Deserialize`：让 JSON 或表单 body 能变成 `Payload`。
- `Serialize`：示例里虽然没有返回 JSON，但这个类型通常也可能用于响应；官方 example 保留了它。
- `Debug`：让 `dbg!(payload)` 可以打印结构体。

## 第三步：定义包装类型 JsonOrForm

源码：

````rust
struct JsonOrForm<T>(T);
````

这是一个 tuple struct。它只有一个字段，就是最终解析出来的 `T`。

为什么不直接返回 `T`？

因为 Axum 的 extractor 需要一个类型来实现 `FromRequest`。  
我们创建 `JsonOrForm<T>`，就是在告诉 Axum：

```text
以后 handler 参数里出现 JsonOrForm<T>
请调用我们自己写的提取逻辑
```

handler 里这句：

````rust
async fn handler(JsonOrForm(payload): JsonOrForm<Payload>) {
````

左边的 `JsonOrForm(payload)` 是模式匹配。  
它把包装器拆开，直接拿到里面的 `payload`。

## 第四步：实现 FromRequest

核心代码：

````rust
impl<S, T> FromRequest<S> for JsonOrForm<T>
where
    S: Send + Sync,
    Json<T>: FromRequest<()>,
    Form<T>: FromRequest<()>,
    T: 'static,
{
    type Rejection = Response;

    async fn from_request(req: Request, _state: &S) -> Result<Self, Self::Rejection> {
        ...
    }
}
````

`FromRequest` 是 Axum 的提取器 trait。  
一个类型实现了 `FromRequest`，就可以放进 handler 参数里。

这段泛型约束可以先这样理解：

- `S: Send + Sync`：应用 state 要能安全用于异步环境。
- `Json<T>: FromRequest<()>`：`T` 必须能被 `Json<T>` 提取。
- `Form<T>: FromRequest<()>`：`T` 必须能被 `Form<T>` 提取。
- `T: 'static`：提取过程里不依赖短生命周期引用。

第一次读不需要死记这些约束。重点是：

```text
JsonOrForm<T> 里面会复用 Axum 已经写好的 Json<T> 和 Form<T>。
```

## 第五步：读取 content-type

源码：

````rust
let content_type_header = req.headers().get(CONTENT_TYPE);
let content_type = content_type_header.and_then(|value| value.to_str().ok());
````

这一段做两件事：

1. 从请求头里取 `content-type`。
2. 把 header value 转成 Rust 字符串。

为什么要 `.ok()`？

HTTP header 不一定总是合法 UTF-8 字符串。  
如果转换失败，这里就当成没有可用的 content type，最后返回 415。

## 第六步：根据 content-type 选择解析方式

源码：

````rust
if content_type.starts_with("application/json") {
    let Json(payload) = req.extract().await.map_err(IntoResponse::into_response)?;
    return Ok(Self(payload));
}

if content_type.starts_with("application/x-www-form-urlencoded") {
    let Form(payload) = req.extract().await.map_err(IntoResponse::into_response)?;
    return Ok(Self(payload));
}
````

这里的 `req.extract().await` 是关键。  
它来自 `RequestExt`，作用是：

```text
从当前 Request 里运行另一个 extractor
```

也就是说：

- JSON 分支里，实际调用的是 `Json<T>` 的提取逻辑。
- 表单分支里，实际调用的是 `Form<T>` 的提取逻辑。

`map_err(IntoResponse::into_response)?` 的作用是把提取失败的错误转成 `Response`，这样符合：

````rust
type Rejection = Response;
````

如果解析成功，就用 `Ok(Self(payload))` 包起来返回。

## 第七步：不支持的媒体类型返回 415

源码：

````rust
Err(StatusCode::UNSUPPORTED_MEDIA_TYPE.into_response())
````

HTTP 415 的含义是：

```text
Unsupported Media Type
请求体的格式服务端不支持
```

比如你发：

```text
content-type: text/plain
body: foo=bar
```

这个 example 不知道该按 JSON 解析，还是按表单解析，所以返回 415。

## 函数职责速查

- `main`：初始化日志，注册 `POST /`，绑定端口并启动服务。
- `Payload`：描述 JSON 或表单都要解析出来的数据结构。
- `handler`：使用 `JsonOrForm<Payload>`，拿到解析后的 payload。
- `JsonOrForm<T>`：自定义 extractor 的包装类型。
- `from_request`：根据 `content-type` 选择 `Json<T>` 或 `Form<T>`。

## 带中文注释的手写版

````rust
//! 这个 example 演示根据 content-type 解析 JSON 或表单 body。
//!
//! Run with
//!
//! ```not_rust
//! cargo run -p example-parse-body-based-on-content-type
//! ```

// 引入 FromRequest、Request、content-type header、状态码、响应转换、路由和 Router。
use axum::{
    extract::{FromRequest, Request},
    http::{header::CONTENT_TYPE, StatusCode},
    response::{IntoResponse, Response},
    routing::post,
    Form, Json, RequestExt, Router,
};
// serde 负责 JSON/表单和 Rust struct 之间的转换。
use serde::{Deserialize, Serialize};
// tracing_subscriber 用来初始化日志。
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// 创建 Tokio 异步运行时。
#[tokio::main]
async fn main() {
    // 初始化日志系统。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 注册一个 POST / 接口。
    let app = Router::new().route("/", post(handler));

    // 绑定本机 3000 端口。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    // 输出监听地址。
    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    // 启动 HTTP 服务。
    axum::serve(listener, app).await;
}

// 请求数据类型：JSON 和表单里都要有 foo 字段。
#[derive(Debug, Serialize, Deserialize)]
struct Payload {
    foo: String,
}

// POST / 的 handler。JsonOrForm 会根据 content-type 自动解析 body。
async fn handler(JsonOrForm(payload): JsonOrForm<Payload>) {
    // 打印解析后的 payload。
    dbg!(payload);
}

// 自定义 extractor 的包装类型。
struct JsonOrForm<T>(T);

// 让 JsonOrForm<T> 成为 Axum extractor。
impl<S, T> FromRequest<S> for JsonOrForm<T>
where
    // state 要能安全用于异步环境。
    S: Send + Sync,
    // T 必须能通过 Json<T> 从请求中提取。
    Json<T>: FromRequest<()>,
    // T 也必须能通过 Form<T> 从请求中提取。
    Form<T>: FromRequest<()>,
    // T 不依赖短生命周期引用。
    T: 'static,
{
    // 提取失败时直接返回一个 HTTP Response。
    type Rejection = Response;

    // Axum 看到 handler 参数是 JsonOrForm<T> 时，会调用这个函数。
    async fn from_request(req: Request, _state: &S) -> Result<Self, Self::Rejection> {
        // 从请求头中读取 content-type。
        let content_type_header = req.headers().get(CONTENT_TYPE);
        // 把 header value 转成字符串；失败就当成没有 content-type。
        let content_type = content_type_header.and_then(|value| value.to_str().ok());

        // 如果有 content-type，就根据它选择解析方式。
        if let Some(content_type) = content_type {
            // application/json 用 Json<T> 解析。
            if content_type.starts_with("application/json") {
                let Json(payload) = req.extract().await.map_err(IntoResponse::into_response)?;
                return Ok(Self(payload));
            }

            // application/x-www-form-urlencoded 用 Form<T> 解析。
            if content_type.starts_with("application/x-www-form-urlencoded") {
                let Form(payload) = req.extract().await.map_err(IntoResponse::into_response)?;
                return Ok(Self(payload));
            }
        }

        // 其他 content-type 都不支持，返回 415。
        Err(StatusCode::UNSUPPORTED_MEDIA_TYPE.into_response())
    }
}
````

## 运行和验证

运行前先确认 `examples/parse-body-based-on-content-type/Cargo.toml` 里的 `axum = { path = "../../axum" }` 能找到项目根目录下的 `axum/` crate。否则需要先按 `docs/tutorial/README.md` 里的“运行前提”处理依赖。

启动：

````bash
cd examples
cargo run -p example-parse-body-based-on-content-type
````

发送 JSON：

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'content-type: application/json' \
  -d '{"foo":"from json"}'
````

发送表单：

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'content-type: application/x-www-form-urlencoded' \
  -d 'foo=from-form'
````

这两个请求都应该成功，并在服务端终端看到 `Payload { foo: ... }`。

发送不支持的 content type：

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'content-type: text/plain' \
  -d 'foo=bar'
````

预期状态码：

````text
HTTP/1.1 415 Unsupported Media Type
````

常见卡点：

- 这个 handler 没有返回响应体，成功时主要看终端里的 `dbg!` 输出。
- `content-type` 决定走 JSON 分支还是 Form 分支。
- 请求体只能被消费一次，所以进入 JSON 分支后就不能再拿同一个 request 去试 Form 分支。
- `starts_with("application/json")` 可以兼容 `application/json; charset=utf-8` 这类 header。

## 手写任务

完成本章代码后，试着做三个小改动：

1. 给 `Payload` 增加 `bar: String` 字段，并分别用 JSON 和表单提交。
2. 把不支持类型的错误从 415 改成返回文本 `unsupported content-type`。
3. 增加一个分支，尝试支持 `application/x-www-form-urlencoded; charset=utf-8`，确认 `starts_with` 已经覆盖这个场景。

## 本章真正要记住什么

- `content-type` 告诉服务端如何解释请求 body。
- `Json<T>` 和 `Form<T>` 都是 Axum 已经提供的 extractor。
- 自定义 extractor 要实现 `FromRequest`。
- `RequestExt::extract()` 可以在自定义 extractor 里复用其他 extractor。
- 请求体只能消费一次，所以必须先判断格式，再选择一种解析方式。

## 源码对照

手写完成后，可以对照官方 example 源码：

- `examples/parse-body-based-on-content-type/Cargo.toml`
- `examples/parse-body-based-on-content-type/src/main.rs`
