# 第 48 章 versioning:API 版本与一个静默的兼容性事故

## 本章你将做出什么

一个从 URL 里认出 API 版本的服务:

```
GET /v1/foo   →  200  received request with version V1
GET /v2/foo   →  200  received request with version V2
GET /v3/foo   →  200  received request with version V3
GET /v4/foo   →  404  unknown version
```

版本不是在每个 handler 里手工解析的,而是被抽成了一个**提取器**:handler 只要在参数里写 `version: Version`,就能拿到一个已经校验过的枚举。

学完本章你将掌握:

- **什么算破坏性变更**,以及为什么 API 需要版本
- 四种版本化策略的取舍(URL 路径 / 请求头 / 媒体类型 / 查询参数)
- **`RequestPartsExt::extract()`:在提取器内部再跑一个提取器**
- `Path<HashMap<String, String>>`:不预先知道参数名时的通用提取法
- `type Rejection = Response` 的取舍
- **示例这个 `{version}` 通配写法会造成的一个静默兼容性事故**,以及 `Router::nest` 为什么是更好的答案

## 前置知识

### 什么算破坏性变更

API 一旦发布出去,就有别人的代码在依赖它。**破坏性变更**是指"老客户端按原样调用会出错或拿到错误结果"的改动:

| 破坏性 | 非破坏性 |
|---|---|
| 删除字段、重命名字段 | **新增可选字段** |
| 改变字段类型(`"1"` → `1`) | **新增端点** |
| 新增必填参数 | 新增可选参数 |
| 收窄取值范围(原来接受负数,现在不接受) | 放宽取值范围 |
| 改变错误码的语义 | 修复明显的 bug(通常) |

**"新增可选字段是非破坏性的"有一个前提:客户端遵守宽容读取(tolerant reader)原则——忽略自己不认识的字段。** 如果客户端用了严格模式的反序列化(`serde` 的 `deny_unknown_fields`),那连加字段都是破坏性的。所以**公开 API 的文档里应该明确要求客户端忽略未知字段**。

只要能避免破坏性变更,就不需要新版本。**版本化是不得已时的逃生口,不是常规操作** —— 每多一个版本,后端就要多维护一套代码路径。

### 四种版本化策略

| 策略 | 长什么样 | 优点 | 缺点 |
|---|---|---|---|
| **URL 路径** | `GET /v1/users` | 直观;浏览器地址栏能直接试;日志和监控里一眼看出版本;缓存天然按版本隔离 | 严格说不 RESTful(同一个资源有多个 URL);版本渗透进每一条路径 |
| **自定义请求头** | `X-API-Version: 1` | URL 干净,资源标识唯一 | 浏览器里试不了;中间的代理和 CDN 可能忽略它导致缓存串味(除非配 `Vary`);调试麻烦 |
| **Accept 媒体类型** | `Accept: application/vnd.example.v1+json` | HTTP 内容协商的标准做法,理论上最"正确" | 写起来啰嗦;工具链支持一般;绝大多数人看不懂 |
| **查询参数** | `GET /users?version=1` | 最简单 | 容易和业务参数混淆;有些缓存不把 query 计入缓存键 |

**实践中 URL 路径最常见**,大多数公司和多数公开 API 都用它——可调试性压倒了理论纯粹性。本章示例走的也是这条路。

值得一提的是另一种思路:**把版本绑在账号上**。客户端注册时锁定一个版本,之后所有请求默认用那个版本,除非显式指定。这样老客户端永远不会被动升级,而新客户端拿到的是最新版。Stripe 用的就是这个模式,代价是后端要长期维护很多个版本。

### 回顾:`FromRequest` 与 `FromRequestParts`

[第 08 章](08-parse-body-based-on-content-type.md#自定义提取器)讲过这对 trait:

| | `FromRequest` | `FromRequestParts` |
|---|---|---|
| 能读到 | **整个请求,含 body** | 只有请求头部分(method、uri、headers、extensions) |
| 消耗请求吗 |  消耗 |  不消耗 |
| 参数位置 | **只能是最后一个** | 任意位置,可以有多个 |

**版本信息在 URL 里,不需要 body,所以实现 `FromRequestParts`。** 这也让 `Version` 能和别的提取器自由组合:

```rust
async fn handler(version: Version, Json(payload): Json<CreateUser>) -> ... 
```

如果实现的是 `FromRequest`,上面这个签名就编译不过了。

### 在提取器里再跑一个提取器

这是本章的新概念。`Version` 需要从路径里取 `{version}` 参数,而"从路径取参数"已经有现成的提取器 `Path<T>` 了 —— **能不能直接复用它?**

能,用 `RequestPartsExt`:

```rust
use axum::RequestPartsExt;

let params: Path<HashMap<String, String>> = parts.extract().await?;
```

`RequestPartsExt` 给 `Parts` 加了一个 `extract::<T>()` 方法,**能运行任何 `FromRequestParts` 提取器**。

**注意提取器是靠类型标注选中的**:

```rust
let params: Path<HashMap<String, String>> = parts.extract().await?;
//          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ 这个标注决定了跑哪个提取器
```

`extract()` 本身是泛型的,返回类型由左边的标注推断。**去掉标注编译器就不知道该跑哪个提取器了。**

对应地还有 `RequestExt`(用于 `FromRequest`,能读 body)。

**这个能力让提取器可以组合而不是重复造轮子。** [第 10 章](10-customize-extractor-error.md)那种"包一层 `Json` 换个错误响应"的写法,也是同一个思路。

### `Path<HashMap<String, String>>`

平时用 `Path` 都是 `Path<u32>` 或 `Path<(String, u32)>` —— 按位置和类型取。但**这里有个特殊困难**:`Version` 这个提取器**不知道它会被用在哪条路由上**。

```rust
.route("/{version}/foo", get(handler))
.route("/{version}/users/{id}", get(other))   // 两个参数!
```

用 `Path<String>` 的话,第二条路由会因为参数个数不匹配而失败([第 11 章](11-customize-path-rejection.md#wrongnumberofparameters)见过这个拒绝类型)。

`Path<HashMap<String, String>>` **把所有路径参数按名字装进一个 map**,不管有几个。然后按名字取:

```rust
params.get("version")
```

**代价是失去了类型转换**(全是 `String`)和编译期检查。但对"我只关心其中一个具名参数"的场景,这是唯一稳妥的写法。

> `Path<T>` 实现了 `Deref<Target = T>`,所以能直接 `params.get(...)` 而不用 `params.0.get(...)`。

### `type Rejection = Response`

自定义提取器要声明失败时返回什么:

```rust
impl<S> FromRequestParts<S> for Version {
    type Rejection = Response;      // ← 直接用 Response
```

对比[第 10 章](10-customize-extractor-error.md)和[第 11 章](11-customize-path-rejection.md)里那种定义一个专门的拒绝类型再实现 `IntoResponse` 的做法:

| | `type Rejection = Response` | 自定义拒绝类型 |
|---|---|---|
| 写起来 | **省事** | 啰嗦 |
| 调用方能否区分失败原因 |  **不能**,只是一个响应 |  能 `match` |
| 能否被中间件二次加工 | 难 | 容易 |
| 适合 | 应用内部的提取器 | **发布给别人用的库** |

**本章用 `Response` 是合理的**——版本解析失败只有一种处理方式,没必要让调用方去分辨。但如果你在写一个给别人用的 crate,应该给一个具体的拒绝类型。

## 项目搭建

```bash
cargo new versioning
cd versioning
```

```bash
cargo add axum
cargo add tokio --features full
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

```bash
cargo add --dev http-body-util
cargo add --dev tower --features util
```

后两个是测试用的,写法和[第 21 章](21-testing.md#项目搭建)一样:`http-body-util` 用来收集响应体,`tower` 的 `util` 提供 `oneshot`。

## 分步构建

### 步骤 1:先用最朴素的方式做出来

**要做什么**:不搞提取器,直接在 handler 里解析版本。跑通之后再体会为什么要抽象。

```rust
// src/main.rs
use axum::{extract::Path, http::StatusCode, response::Html, routing::get, Router};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // build our application with some routes
    let app = app();

    // run it
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

fn app() -> Router {
    Router::new().route("/{version}/foo", get(handler))
}

#[derive(Debug)]
enum Version {
    V1,
    V2,
    V3,
}

// 朴素版:步骤 2 会替换掉
async fn handler(Path(version): Path<String>) -> Result<Html<String>, (StatusCode, &'static str)> {
    let version = match version.as_str() {
        "v1" => Version::V1,
        "v2" => Version::V2,
        "v3" => Version::V3,
        _ => return Err((StatusCode::NOT_FOUND, "unknown version")),
    };

    Ok(Html(format!("received request with version {version:?}")))
}
```

**`fn app() -> Router` 单独抽出来**,是为了让测试能拿到 `Router` 而不用启动服务器——[第 21 章](21-testing.md)讲过这个模式。**这是官方示例的结构,不是我加的。**

**`{version}` 是路径参数的占位语法**([第 03 章](03-form.md)之后一直在用)。它匹配路径的第一段,不管内容是什么。

**验证**:

```bash
cargo run
```

```bash
curl -i http://127.0.0.1:3000/v1/foo
```

```
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8

received request with version V1
```

```bash
curl -i http://127.0.0.1:3000/v9/foo
```

```
HTTP/1.1 404 Not Found

unknown version
```

**功能是对的。那问题在哪?**

想象再加两条路由:

```rust
.route("/{version}/users", get(list_users))
.route("/{version}/users/{id}", get(get_user))
```

**每个 handler 都要重复那段 `match`**,而且 `/{version}/users/{id}` 那条还得改成 `Path<(String, u32)>`,顺序一错就是运行时错误。

**版本解析是横切关注点,不该散在每个 handler 里。** 下一步把它抽成提取器。

### 步骤 2:抽成提取器

**要做什么**:让 `Version` 自己实现 `FromRequestParts`,handler 直接声明它。

```rust
use axum::{
    extract::{FromRequestParts, Path},
    http::{request::Parts, StatusCode},
    response::{Html, IntoResponse, Response},
    routing::get,
    RequestPartsExt, Router,
};
use std::collections::HashMap;
```

```rust
async fn handler(version: Version) -> Html<String> {
    Html(format!("received request with version {version:?}"))
}

#[derive(Debug)]
enum Version {
    V1,
    V2,
    V3,
}

impl<S> FromRequestParts<S> for Version
where
    S: Send + Sync,
{
    type Rejection = Response;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let params: Path<HashMap<String, String>> =
            parts.extract().await.map_err(IntoResponse::into_response)?;

        let version = params
            .get("version")
            .ok_or_else(|| (StatusCode::NOT_FOUND, "version param missing").into_response())?;

        match version.as_str() {
            "v1" => Ok(Version::V1),
            "v2" => Ok(Version::V2),
            "v3" => Ok(Version::V3),
            _ => Err((StatusCode::NOT_FOUND, "unknown version").into_response()),
        }
    }
}
```

**handler 瘦了一大圈:**

```rust
async fn handler(version: Version) -> Html<String> {
    Html(format!("received request with version {version:?}"))
}
```

**没有 `Result` 了** —— 因为失败已经被提取器接管,handler 只在成功时才会被调用。**这就是提取器的价值:把"校验 + 失败响应"从业务逻辑里剥出去。** [第 08 章](08-parse-body-based-on-content-type.md)第一次讲这个思路,这里是它最纯粹的一个应用。

**逐块看 impl:**

**① 泛型参数 `S` 和 `where S: Send + Sync`**

提取器不关心应用状态是什么类型,所以对 `S` 泛型。`Send + Sync` 是 axum 对状态的基本要求。`_state` 加下划线表示不用。

**这个写法保证了 `Version` 能用在任何 `Router<S>` 上**,不管你的 state 是什么。

**② `parts.extract().await`**

前置知识讲过:在提取器内部跑另一个提取器。`Path<HashMap<String, String>>` 的类型标注决定了跑的是 `Path`。

**③ `.map_err(IntoResponse::into_response)`**

`Path` 的拒绝类型是 `PathRejection`([第 11 章](11-customize-path-rejection.md)的主题),而我们声明的 `Rejection` 是 `Response`,所以要转一下。`IntoResponse::into_response` 直接当函数用,比写闭包简洁。

**什么时候会走到这个分支?** 路由里根本没有路径参数的时候(比如把 `Version` 用在 `/health` 上)。

**④ `.ok_or_else(...)` 那一段**

`params.get("version")` 返回 `Option<&String>`。取不到说明**路由里的参数名不叫 `version`**——这是编程错误,不是客户端的问题。

> **更好的写法**:这里返回 **`404 Not Found` 是不对的**。`version` 参数缺失意味着开发者把 `Version` 提取器用在了一条没有 `{version}` 段的路由上,**这是服务端的配置错误,应该是 `500 Internal Server Error`**。返回 404 会让排查的人以为是路由没注册,而实际上路由匹配上了、只是参数名写错了。
>
> 这和[第 11 章](11-customize-path-rejection.md#代价与边界)指出的 `WrongNumberOfParameters` 该用 500 而不是 400 是**同一类问题**:分不清"客户端错了"和"我自己写错了"。

**⑤ 最后的 `match`**

未知版本返回 404。**404 在这里是合理的**——`/v9/foo` 这个资源确实不存在。(也有人主张用 `400 Bad Request`,理由是"版本是参数不是资源"。两种都能说通,**重要的是团队内部统一**。)

> **更好的写法**:错误响应应该**告诉客户端支持哪些版本**,而不是只说 "unknown version"。比如 `"unknown version, supported: v1, v2, v3"`。客户端拿到前者只能去翻文档,拿到后者立刻就知道怎么改。**错误信息的可操作性(actionability)是 API 设计里最容易被忽略的一环。**

**验证**:

```bash
cargo run
```

```bash
curl -i http://127.0.0.1:3000/v1/foo    # 200  received request with version V1
curl -i http://127.0.0.1:3000/v2/foo    # 200  received request with version V2
curl -i http://127.0.0.1:3000/v3/foo    # 200  received request with version V3
curl -i http://127.0.0.1:3000/v4/foo    # 404  unknown version
```

**和步骤 1 行为完全一致,但 handler 干净了。**

再试一个没匹配上路由的:

```bash
curl -i http://127.0.0.1:3000/foo
```

```
HTTP/1.1 404 Not Found
content-length: 0
```

**注意这个 404 的 body 是空的**,而 `/v4/foo` 那个 404 有 `unknown version`。**两种 404 的来源完全不同**:前者是路由根本没匹配上(axum 默认的兜底,[第 06 章](06-global-404-handler.md)讲过),后者是路由匹配上了但提取器拒绝了。

**排查线上问题时先分清是哪一种** —— 和[第 43 章](43-reverse-proxy.md#步骤-6一个必须知道的边界只有--被代理)那个"代理自己 404 vs 上游 404"是同样的道理。

### 步骤 3:写测试

**要做什么**:用[第 21 章](21-testing.md)的 `oneshot` 方式测两条路径,不启动服务器。

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use axum::{body::Body, http::Request, http::StatusCode};
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    #[tokio::test]
    async fn test_v1() {
        let response = app()
            .oneshot(Request::get("/v1/foo").body(Body::empty()).unwrap())
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::OK);
        let body = response.into_body();
        let bytes = body.collect().await.unwrap().to_bytes();
        let html = String::from_utf8(bytes.to_vec()).unwrap();

        assert_eq!(html, "received request with version V1");
    }

    #[tokio::test]
    async fn test_v4() {
        let response = app()
            .oneshot(Request::get("/v4/foo").body(Body::empty()).unwrap())
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::NOT_FOUND);
        let body = response.into_body();
        let bytes = body.collect().await.unwrap().to_bytes();
        let html = String::from_utf8(bytes.to_vec()).unwrap();

        assert_eq!(html, "unknown version");
    }
}
```

**这些写法[第 21 章](21-testing.md)全讲过**,快速回顾:

- **`app().oneshot(request)`** —— `Router` 本身就是一个 tower `Service`,`oneshot` 直接调用它,**不需要 TCP、不需要端口、不需要启动服务器**
- **`Request::get("/v1/foo").body(Body::empty())`** —— 手工构造请求
- **`body.collect().await.to_bytes()`** —— `http_body_util::BodyExt` 提供,把流式 body 收集成 `Bytes`

**测试的价值在这一章特别明显**:`test_v4` 断言的是**错误路径**——状态码是 404、body 是 `unknown version`。**错误路径最容易在重构时被改坏,而且线上很难发现**(没人会盯着 404 的响应体看)。

**验证**:

```bash
cargo test
```

```
running 2 tests
test tests::test_v1 ... ok
test tests::test_v4 ... ok

test result: ok. 2 passed; 0 failed; 0 ignored
```

### 步骤 4:故意加一条路由,暴露一个静默的兼容性事故

**要做什么**:假设现在要加一个**只有 v2 才有**的新接口 `/bar`,看看会发生什么。

**临时**加一条路由:

```rust
fn app() -> Router {
    Router::new()
        .route("/{version}/foo", get(handler))
        .route("/{version}/bar", get(bar_handler))    // 临时:假设这是 v2 新增的接口
}

async fn bar_handler(version: Version) -> Html<String> {
    Html(format!("bar, version {version:?}"))
}
```

**验证**:

```bash
cargo run
```

```bash
curl -i http://127.0.0.1:3000/v2/bar
```

```
HTTP/1.1 200 OK

bar, version V2
```

意料之中。现在试试 v1:

```bash
curl -i http://127.0.0.1:3000/v1/bar
```

```
HTTP/1.1 200 OK

bar, version V1
```

** `/v1/bar` 也返回了 200。**

**这是一个静默的兼容性事故:**

- 你以为 `/bar` 是 v2 的新特性,**实际上 v1 客户端也能调**
- 没有任何报错、没有任何警告 —— **路由匹配上了,提取器也认为 `v1` 是合法版本**
- 半年后你想改 `/bar` 的返回格式,以为"v1 没人用这个接口",结果**一改就出事**

**根本原因**:`{version}` 是一个**通配段**,它对所有版本一视同仁。**"哪些接口属于哪个版本"这个信息在路由表里根本没有表达出来**,只存在于开发者的脑子里。

**同样的问题还有另一面**:如果 v3 **删掉**了 `/foo`,你没有任何办法让 `/v3/foo` 返回 404 —— 除非在 handler 里手工写 `if version == V3 { return 404 }`,而这正是步骤 1 想摆脱的东西。

> **更好的写法**:**用 `Router::nest` 按版本分组,让路由表如实反映每个版本有哪些接口。**
>
> ```rust
> fn app() -> Router {
>     Router::new()
>         .nest("/v1", v1_routes())
>         .nest("/v2", v2_routes())
>         .nest("/v3", v3_routes())
> }
>
> fn v1_routes() -> Router {
>     Router::new().route("/foo", get(foo_v1))
> }
>
> fn v2_routes() -> Router {
>     Router::new()
>         .route("/foo", get(foo_v1))          // v2 沿用 v1 的实现
>         .route("/bar", get(bar_v2))          // v2 新增
> }
>
> fn v3_routes() -> Router {
>     Router::new().route("/bar", get(bar_v2)) // v3 删掉了 /foo
> }
> ```
>
> 好处很实在:
>
> - **`/v1/bar` 自然返回 404**,不需要任何额外判断
> - **`/v3/foo` 自然返回 404**,删除接口变成了删一行路由
> - **每个版本有哪些接口,看代码一目了然** —— 这是可执行的文档
> - **每个 handler 的签名里不再有 `Version`**,版本差异体现在路由表和 handler 的选择上,而不是运行时的 `match`
> - 共享的实现直接复用函数,不需要复制粘贴
>
> 代价是路由表更长,以及**版本很多、差异很小时会显得啰嗦**。这时候可以两种混用:用 `nest` 划分版本边界,再在需要感知版本的少数 handler 里用 `Version` 提取器。
>
> **官方示例演示的是"怎么写一个从路径取版本的提取器",不是"版本化的最佳实践"。** 这个提取器技术本身很有价值(比如版本从请求头来的时候,`nest` 就不适用了),但用它来做路径版本化,不如 `nest` 直接。

**把临时加的 `/bar` 路由和 `bar_handler` 删掉**,恢复原样。

## 完整代码

`src/main.rs`:

```rust
use axum::{
    extract::{FromRequestParts, Path},
    http::{request::Parts, StatusCode},
    response::{Html, IntoResponse, Response},
    routing::get,
    RequestPartsExt, Router,
};
use std::collections::HashMap;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // build our application with some routes
    let app = app();

    // run it
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

fn app() -> Router {
    Router::new().route("/{version}/foo", get(handler))
}

async fn handler(version: Version) -> Html<String> {
    Html(format!("received request with version {version:?}"))
}

#[derive(Debug)]
enum Version {
    V1,
    V2,
    V3,
}

impl<S> FromRequestParts<S> for Version
where
    S: Send + Sync,
{
    type Rejection = Response;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let params: Path<HashMap<String, String>> =
            parts.extract().await.map_err(IntoResponse::into_response)?;

        let version = params
            .get("version")
            .ok_or_else(|| (StatusCode::NOT_FOUND, "version param missing").into_response())?;

        match version.as_str() {
            "v1" => Ok(Version::V1),
            "v2" => Ok(Version::V2),
            "v3" => Ok(Version::V3),
            _ => Err((StatusCode::NOT_FOUND, "unknown version").into_response()),
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use axum::{body::Body, http::Request, http::StatusCode};
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    #[tokio::test]
    async fn test_v1() {
        let response = app()
            .oneshot(Request::get("/v1/foo").body(Body::empty()).unwrap())
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::OK);
        let body = response.into_body();
        let bytes = body.collect().await.unwrap().to_bytes();
        let html = String::from_utf8(bytes.to_vec()).unwrap();

        assert_eq!(html, "received request with version V1");
    }

    #[tokio::test]
    async fn test_v4() {
        let response = app()
            .oneshot(Request::get("/v4/foo").body(Body::empty()).unwrap())
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::NOT_FOUND);
        let body = response.into_body();
        let bytes = body.collect().await.unwrap().to_bytes();
        let html = String::from_utf8(bytes.to_vec()).unwrap();

        assert_eq!(html, "unknown version");
    }
}
```

## 代价与边界

**`{version}` 通配让"哪个接口属于哪个版本"无法表达。** 步骤 4 详细演示了。这是本章最重要的一条。

**`Version` 枚举拿到了却没用来做任何事。** handler 只是把它 `Debug` 打印出来。真实项目里版本要么影响**序列化格式**(v1 返回 `user_name`,v2 返回 `userName`)、要么影响**业务逻辑**(v2 开始校验邮箱格式)、要么影响**默认值**。示例只演示了"怎么拿到",没演示"拿到之后怎么用"——而后者才是版本化真正的工作量。

**加新版本要改代码三处。** 枚举加一个变体、`match` 加一个分支、还要记得更新文档和测试。忘了改 `match` 的话,新版本会返回 `unknown version` —— 这个错误至少是显式的,不算太糟。

**没有版本弃用机制。** 真实的版本化生命周期还包括:标记某版本为 deprecated(响应里加 `Deprecation` 和 `Sunset` 头)、统计各版本的调用量(正好可以用[第 47 章](47-prometheus-metrics.md)的指标,`version` 作为标签——**取值有限,是个合格的标签**)、到期后下线。**没有调用量统计,你永远不敢下线任何一个老版本。**

**`version` 参数缺失返回 404,应该是 500。** 步骤 2 的「更好的写法」说过。

**错误信息没有告诉客户端支持哪些版本。** 同上。

**返回的不是 HTML 却用了 `Html<String>`。** `received request with version V1` 是纯文本,但 `Html` 会设置 `Content-Type: text/html; charset=utf-8`。**响应头在撒谎。** 应该用 `String`(会得到 `text/plain`)。这在浏览器里问题不大,但**如果内容里有用户可控的部分,声明成 `text/html` 会引入 XSS 风险**——浏览器会把 `<script>` 当代码执行。这一章的内容是写死的所以安全,但**"随手用 `Html` 包一下"是个坏习惯**。

**版本号只支持 `v1`/`v2`/`v3` 这种固定字符串。** 有些 API 用日期做版本(`2024-01-15`),有些用语义化版本(`1.2`)。日期版本(Stripe 的做法)的好处是天然有序,而且能表达"某天之后的行为"。

**没有考虑默认版本。** `GET /foo`(不带版本)直接 404。有些 API 会把不带版本的请求默认到最新版或最老版。**默认到最新版很危险**——新版发布会静默改变这些客户端的行为;默认到最老版更安全,但会让老版本永远下不掉。

**没有 CORS、没有认证、没有限流。** 这些在[第 20 章](20-cors.md)、[第 30 章](30-jwt.md)讲过,一个真正的公开 API 都需要。

## 动手练习

**1. 用 `Router::nest` 重写。**

提示:按步骤 4 的「更好的写法」,建三个 `fn vN_routes() -> Router`,`/foo` 在 v1/v2 有、v3 没有,`/bar` 只有 v2/v3 有。写测试断言 `/v1/bar` 和 `/v3/foo` 都是 404。**做完对比一下:哪种写法"某个接口属于哪些版本"更一目了然?**

**2. 让版本真正影响行为。**

提示:加一个 `GET /{version}/user` 路由,v1 返回 `{"user_name": "alice"}`,v2 返回 `{"userName": "alice"}`(字段名从 snake_case 改成 camelCase,这是一个典型的破坏性变更)。用 `match version` 选择不同的响应结构体。**这个练习让你体会到:版本化真正的成本不在路由,在维护多套序列化逻辑。**

**3. 把错误信息改得可操作。**

提示:`unknown version` 改成 `unknown version 'v9', supported versions: v1, v2, v3`;`version param missing` 改成 500 并说明是服务端配置错误。**记得同步改 `test_v4` 的断言** —— 这正好演示了为什么错误路径也要有测试。

**4. 加一个版本调用量指标。**

提示:结合[第 47 章](47-prometheus-metrics.md),在中间件里把版本作为标签记进 `http_requests_total`。 **版本的取值集合是有限的(v1/v2/v3 + `<unknown>`),所以它是一个合格的标签。** 做完你就有了"能不能下线 v1"的依据。

**5. 支持请求头版本化,并和路径版本化共存。**

提示:改造 `Version` 的 `from_request_parts`:先看 `X-API-Version` 头,没有再回退到路径参数,都没有就用默认版本。 注意加上 `Vary: X-API-Version` 响应头,否则缓存会串味([第 39 章](39-compression.md#步骤-2响应压缩)讲过 `Vary` 的作用)。**做完你会发现:`nest` 在这个场景下就不适用了,提取器方案的价值在这里才显现出来。**

## 小结与下一章预告

**版本化是不得已时的逃生口。** 能靠"只做非破坏性变更 + 要求客户端宽容读取"避开的,就不要开新版本——每多一个版本就多一套要长期维护的代码路径。

四种策略里 **URL 路径最常用**,因为可调试性压倒了理论纯粹性;请求头方案 URL 更干净但要小心缓存;媒体类型最"正确"但最啰嗦。还有一种是把版本绑在账号上(Stripe 的做法),老客户端永远不会被动升级。

技术上,本章把版本解析抽成了一个 `FromRequestParts` 提取器。核心的新招是 **`RequestPartsExt::extract()`:在提取器内部再跑一个提取器**,而且**跑哪个提取器由类型标注决定**。`Path<HashMap<String, String>>` 则解决了"提取器不知道自己会被用在哪条路由上"的困难——按名字取参数,不管路由里有几个。

步骤 1 到步骤 2 的对比说明了提取器的本质价值:**把"校验 + 失败响应"从业务逻辑里剥出去**,handler 的返回类型从 `Result<_, _>` 变回了裸类型。

但步骤 4 暴露了这个方案的根本问题:**`{version}` 是通配段,"哪个接口属于哪个版本"在路由表里根本没被表达**。加一个"只有 v2 才有"的 `/bar`,`/v1/bar` 照样返回 200,**没有任何报错**。半年后你以为 v1 没人用这个接口就动手改,一改就出事。

**`Router::nest` 按版本分组是更直接的答案**:`/v1/bar` 自然 404,删接口就是删一行路由,路由表本身成了可执行的文档。**官方示例演示的是提取器技术,不是版本化最佳实践** —— 那个技术在版本来自请求头时仍然不可替代(练习 5)。

其余边界里值得记住的:**没有调用量统计就永远不敢下线老版本**(用[第 47 章](47-prometheus-metrics.md)的指标,`version` 是个合格标签);**`Html<String>` 包纯文本让响应头撒谎**,内容可控时还会引入 XSS。

到这里,「生产部署」阶段只剩最后一块了,而且方向反过来——前面四章都是**让线上跑得更好**,[第 49 章 auto-reload](49-auto-reload.md) 讲的是**让本地开发更顺手**:改一行代码就自动重编译重启,而且**重启期间的请求不会被拒绝**。你会看到一个叫「套接字激活」的机制,它在开发环境解决热重载,在生产环境则是 systemd 实现零停机重启的基础。
