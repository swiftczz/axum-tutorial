# 09. parse-body-based-on-content-type

对应示例：`examples/parse-body-based-on-content-type`

`Json<T>` 解析 JSON body，`Form<T>` 解析表单 body——但**一个 handler 只能接收一种**。这章写一个 `JsonOrForm<T>` extractor，**根据 `Content-Type` 自动选择**解析方式。让同一个接口同时支持 JSON 和表单提交。

分 3 步：先看 axum 的 `Json`/`Form` 各自只认特定 content-type，再写 `JsonOrForm<T>` 按 content-type 分发，最后用 `RequestExt::extract` 在 request 上调用其他 extractor。

相比前面章节新引入：**`RequestExt::extract`（在 request 上调 extractor）、`CONTENT_TYPE` 头检查、`StatusCode::UNSUPPORTED_MEDIA_TYPE` rejection**。

## Cargo.toml

````toml
[package]
name = "example-parse-body-based-on-content-type"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = { version = "0.8", features = ["macros"] }
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：理解 `Json`/`Form` 的 content-type 约束

axum 的 `Json<T>` 解析 body 前会检查 `Content-Type: application/json`；`Form<T>` 检查 `application/x-www-form-urlencoded`。content-type 不对直接 rejection（415）。

````rust
use axum::{routing::post, Json, Form};
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct Payload { foo: String }

async fn json_handler(Json(p): Json<Payload>) -> String {
    p.foo
}

async fn form_handler(Form(p): Form<Payload>) -> String {
    p.foo
}

# fn app() -> axum::Router {
#     axum::Router::new()
#         .route("/json", post(json_handler))
#         .route("/form", post(form_handler))
# }
````

发错 content-type 直接 415：

````bash
curl -X POST http://127.0.0.1:3000/json \
  -H 'content-type: application/x-www-form-urlencoded' \
  -d 'foo=bar'
# 415 Unsupported Media Type
````

> **新面孔：content-type 严格匹配**
>
> axum 的 `Json`/`Form` 都严格匹配 content-type，不是"试试看能不能解析"。这避免了 JSON 接口被发 form 数据污染。
>
> 但有时候想一个接口同时支持两种格式（如兼容旧版 form 客户端 + 新版 JSON 客户端），这就要自定义 extractor。

---

## 第二步：`JsonOrForm<T>` extractor 按 content-type 分发

写一个泛型 extractor `JsonOrForm<T>`：检查 `Content-Type`，是 JSON 就走 `Json<T>`，是 form 就走 `Form<T>`，其他返回 415。

````rust
use axum::{
    extract::{FromRequest, Request},
    http::{header::CONTENT_TYPE, StatusCode},
    response::{IntoResponse, Response},
    Form, Json,
};

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

> **新面孔：`RequestExt::extract`**
>
> `req.extract::<T>().await` 在 `Request` 上直接调用 extractor `T`。这章在 `JsonOrForm` 里调 `req.extract().await` 触发 `Json<T>` 或 `Form<T>`（类型由左侧 `let Json(payload) = ...` 推导）。
>
> 相当于"在当前 request 上重新跑一遍某个 extractor"——避免手写 JSON/form 解析。

> **新面孔：`CONTENT_TYPE` 头检查**
>
> `req.headers().get(CONTENT_TYPE)` 拿 content-type 头。`to_str().ok()` 转 `&str`（头可能含非 ASCII，转失败用 `ok()` 容错成 None）。
>
> `content_type.starts_with("application/json")` 用 starts_with 因为 content-type 可能带 charset：`application/json; charset=utf-8`。

> **新面孔：`StatusCode::UNSUPPORTED_MEDIA_TYPE`（415）**
>
> 客户端发了不支持的 content-type 时返回 415。比 400 Bad Request 更准确——告诉客户端"换个 content-type 重试"。

> **新面孔：`FromRequest<()>` 约束**
>
> `Json<T>: FromRequest<()>, Form<T>: FromRequest<()>` 约束表示这两个 extractor 不依赖 state（`State = ()`）。因为我们用 `req.extract()`（不需要 state），约束成 `()` 保证能调。
>
> 复杂约束写起来繁琐，但保证类型安全。

---

## 第三步：完整 handler

最后用 `JsonOrForm<Payload>` 作为 handler 参数，同时支持 JSON 和表单。

````rust
use axum::{routing::post, Router, RequestExt};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Payload {
    foo: String,
}

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(/* tracing 初始化 */)
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app = Router::new().route("/", post(handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler(JsonOrForm(payload): JsonOrForm<Payload>) {
    dbg!(payload);
}
````

验证：

````bash
cd examples
cargo run -p example-parse-body-based-on-content-type

# JSON
curl -X POST http://127.0.0.1:3000/ \
  -H 'content-type: application/json' \
  -d '{"foo":"bar"}'

# Form
curl -X POST http://127.0.0.1:3000/ \
  -H 'content-type: application/x-www-form-urlencoded' \
  -d 'foo=bar'

# 不支持的 content-type
curl -X POST http://127.0.0.1:3000/ \
  -H 'content-type: text/plain' \
  -d 'plain text'
# 415 Unsupported Media Type
````

服务端两种格式都能看到 `[Payload { foo: "bar" }]`。

---

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

# JSON 和 Form 都接受
curl -X POST http://127.0.0.1:3000/ -H 'content-type: application/json' -d '{"foo":"bar"}'
curl -X POST http://127.0.0.1:3000/ -H 'content-type: application/x-www-form-urlencoded' -d 'foo=bar'
````

## 解读

### 自定义 extractor 复用已有 extractor 模式

这章核心技巧：**在自定义 `FromRequest` 实现里，用 `req.extract()` 调用其他已有 extractor**，避免重写解析逻辑。

```text
JsonOrForm<T>::from_request(req):
    检查 Content-Type
    ↓ JSON
    req.extract::<Json<T>>()  → 复用 axum::Json
    ↓ Form
    req.extract::<Form<T>>()  → 复用 axum::Form
    ↓ 其他
    返回 415
```

同样的模式可写：`CsvOrJson<T>`、`XmlOrJson<T>`、`AutoCompress<T>`（按 Content-Encoding 自动解压）。

### content-type 协商

RESTful API 常用 content-type 协商（一个接口支持多种格式）：

- 现代客户端发 JSON
- 老客户端发 form（HTML 表单）
- 内部 API 可能用 Protobuf/MessagePack

这章的 `JsonOrForm` 是这个模式的简化版。

## 常见问题

**为什么用 `req.extract()` 而不是 `Json::from_request(req, &())`？** `extract()` 是 `RequestExt` 的便捷方法，等价于 `T::from_request(req, &())` 但更简洁。两者本质一样。

**怎么扩展支持更多格式？** 加 `if content_type.starts_with("application/xml")` 分支，调 `req.extract::<Xml<T>>()`（用 `quick-xml` 等库的 extractor）。

**multipart/form-data 怎么办？** 不行——multipart 的 `T` 不是简单的 `Deserialize`，要用 `Multipart` extractor（ch07）单独处理。

## 手写任务

1. 加 XML 支持：`content-type.starts_with("application/xml")` 时调 `quick-xml` 解析。
2. 加错误响应 body：415 时返回 `{"error": "unsupported", "received": "...", "expected": ["json", "form"]}`。
3. 写 `CsvOrJson<T>` extractor 支持 CSV。
4. 把 `JsonOrForm<T>` 改成 middleware（在每个 handler 前运行），对比哪种更合适。

## 小结

这章用 3 步讲了 content-type 协商：

1. **理解约束**：`Json`/`Form` 各自严格匹配 content-type，发错就 415。
2. **`JsonOrForm<T>` extractor**：检查 content-type 分发到 `Json<T>` 或 `Form<T>`，其他返回 415。
3. **`RequestExt::extract`**：在自定义 extractor 里复用已有 extractor，避免重写解析。

核心模式：**自定义 extractor 复用已有 extractor**——`req.extract::<T>().await` 在 request 上调 extractor T，是这章的关键 API。

## 源码对照

- `examples/parse-body-based-on-content-type/Cargo.toml`
- `examples/parse-body-based-on-content-type/src/main.rs`
