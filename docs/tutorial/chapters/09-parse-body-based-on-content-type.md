# 09. parse-body-based-on-content-type

对应示例：`examples/parse-body-based-on-content-type`

前面学过两种 body 解析:`Json<T>`(ch02)解析 `application/json`,`Form<T>`(ch06)解析 `application/x-www-form-urlencoded`。本章写一个自定义 extractor `JsonOrForm<T>`,根据请求头 `content-type` 自动选择用哪一个。这章比前几章偏 axum 提取器机制。



相比前面章节新引入：**`FromRequest`（消费 body 的 extractor）、`RequestExt::extract`（复用其他 extractor）、415 Unsupported Media Type**。

## Cargo.toml

````toml
[package]
name = "example-parse-body-based-on-content-type"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 关键概念

> **新面孔：`FromRequest`**
>
> 和 `FromRequestParts`（ch10/12）不同，`FromRequest` 能消费整个请求 body。自定义 `JsonOrForm<T>` 需要它来根据 content-type 选择解析方式。

> **新面孔：`RequestExt::extract`**
>
> 在自定义 extractor 内部复用其他 extractor：`req.extract::<Json<T>>().await` 从同一个 request 里运行 `Json<T>` 提取。


## 完整代码

````rust
use axum::{
    extract::{FromRequest, Request},
    http::{header::CONTENT_TYPE, StatusCode},
    response::{IntoResponse, Response},
    routing::post,
    Form, Json, RequestExt, Router,
};
use serde::{Deserialize, Serialize};
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

    let app = Router::new().route("/", post(handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    tracing::debug!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app).await;
}

#[derive(Debug, Serialize, Deserialize)]
struct Payload {
    foo: String,
}

async fn handler(JsonOrForm(payload): JsonOrForm<Payload>) {
    dbg!(payload);
}

struct JsonOrForm<T>(T);

impl<S, T> FromRequest<S> for JsonOrForm<T>
where
    S: Send + Sync,
    Json<T>: FromRequest<()>,
    Form<T>: FromRequest<()>,
    T: 'static,
{
    type Rejection = Response;

    async fn from_request(req: Request, _state: &S) -> Result<Self, Self::Rejection> {
        let content_type_header = req.headers().get(CONTENT_TYPE);
        let content_type = content_type_header.and_then(|value| value.to_str().ok());

        if let Some(content_type) = content_type {
            if content_type.starts_with("application/json") {
                let Json(payload) = req.extract().await.map_err(IntoResponse::into_response)?;
                return Ok(Self(payload));
            }

            if content_type.starts_with("application/x-www-form-urlencoded") {
                let Form(payload) = req.extract().await.map_err(IntoResponse::into_response)?;
                return Ok(Self(payload));
            }
        }

        Err(StatusCode::UNSUPPORTED_MEDIA_TYPE.into_response())
    }
}
````

## 运行

````bash
cd examples
cargo run -p example-parse-body-based-on-content-type
````

发送 JSON:

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'content-type: application/json' \
  -d '{"foo":"from json"}'
````

发送表单:

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'content-type: application/x-www-form-urlencoded' \
  -d 'foo=from-form'
````

两个请求都应成功,服务端终端打印 `Payload { foo: ... }`。这个 handler 不返回响应体,成功主要看 `dbg!` 输出。

发送不支持的 content type:

````bash
curl -i -X POST http://127.0.0.1:3000/ \
  -H 'content-type: text/plain' \
  -d 'foo=bar'
````

预期 `HTTP/1.1 415 Unsupported Media Type`。

## 解读

### `content-type` 决定怎么解析 body

HTTP 请求 body 只是一串字节。同样提交 `foo=bar`,JSON 和表单格式完全不同:

```text
content-type: application/json                    body: {"foo":"bar"}
content-type: application/x-www-form-urlencoded   body: foo=bar
```

`JsonOrForm<T>` 看请求头的 `content-type` 决定用 `Json<T>` 还是 `Form<T>` 解析。

### 自定义 extractor = 实现 `FromRequest`

`FromRequest` 是 axum 的提取器 trait。一个类型实现了 `FromRequest`,就能放进 handler 参数:

````rust
struct JsonOrForm<T>(T);

impl<S, T> FromRequest<S> for JsonOrForm<T>
where
    ...
{
    type Rejection = Response;

    async fn from_request(req: Request, _state: &S) -> Result<Self, Self::Rejection> {
        ...
    }
}
````

`JsonOrForm<T>` 是 tuple struct,只包装最终解析出来的 `T`。为什么不直接返回 `T`?因为 extractor 需要一个类型来实现 `FromRequest`——我们创建 `JsonOrForm<T>` 就是告诉 axum:handler 参数里出现这个类型时,调用我们写的提取逻辑。

泛型约束第一遍不用死记,重点是:`JsonOrForm<T>` 内部复用 axum 已写好的 `Json<T>` 和 `Form<T>`。

### `req.extract()` 复用其他 extractor

````rust
let Json(payload) = req.extract().await.map_err(IntoResponse::into_response)?;
````

`req.extract()` 来自 `RequestExt` trait,作用是"从当前 Request 里运行另一个 extractor"。所以 JSON 分支实际调用 `Json<T>` 的提取逻辑,表单分支实际调用 `Form<T>` 的提取逻辑。`map_err(IntoResponse::into_response)?` 把提取失败的错误转成 `Response`,符合 `type Rejection = Response`。

### `starts_with` 兼容 charset

`content_type.starts_with("application/json")` 能匹配 `application/json` 也能匹配 `application/json; charset=utf-8`。

### 请求体只能消费一次

进入 JSON 分支消费了请求体后,就不能再拿同一个 request 去试 Form 分支。所以必须先判断格式,再选择一种解析方式。

### 415 Unsupported Media Type

不支持的 content-type 返回 415——"请求体格式服务端不支持"。

## 手写任务

跑通后做三个小改动:

1. 给 `Payload` 加 `bar: String` 字段,分别用 JSON 和表单提交。
2. 把不支持类型的错误从 415 改成返回文本 `unsupported content-type`。
3. 确认 `starts_with` 已经覆盖 `application/x-www-form-urlencoded; charset=utf-8`。

## 小结

- `content-type` 告诉服务端如何解释请求 body。
- 自定义 extractor 要实现 `FromRequest`,放进 handler 参数即可生效。
- `RequestExt::extract()` 在自定义 extractor 里复用其他 extractor。
- 请求体只能消费一次,必须先判断格式再选一种解析方式。
- 不支持的 content-type 返回 415。

## 源码对照

- `examples/parse-body-based-on-content-type/Cargo.toml`
- `examples/parse-body-based-on-content-type/src/main.rs`
