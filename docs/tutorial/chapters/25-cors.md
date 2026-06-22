# 25. cors

对应示例：`examples/cors`

浏览器的**同源策略**（Same-Origin Policy）阻止网页 JS 跨域请求——`http://localhost:3000` 的 JS 默认不能调 `http://localhost:4000` 的 API。**CORS**（Cross-Origin Resource Sharing）是服务端用响应头告诉浏览器"允许哪些来源跨域访问"的标准机制。

分 2 步：先复现跨域被浏览器拦截的问题，再用 `CorsLayer` 配置允许跨域。

相比前面章节新引入：**同源策略与跨域、`tower-http::cors::CorsLayer`、`Origin`/`Access-Control-Allow-Origin` 头、预检（Preflight）请求**。

## Cargo.toml

````toml
[package]
name = "example-cors"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }
tower-http = { version = "0.6", features = ["cors"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：复现跨域问题

先同时跑前端（端口 3000，提供 HTML）+ 后端（端口 4000，提供 JSON API）。前端 HTML 里的 JS fetch 后端 API——浏览器会拦截跨域请求。

````rust
use axum::{
    response::{Html, IntoResponse},
    routing::get,
    Json, Router,
};
use std::net::SocketAddr;

#[tokio::main]
async fn main() {
    let frontend = async {
        let app = Router::new().route("/", get(html));
        serve(app, 3000).await;
    };

    let backend = async {
        // 注意：这步**没**加 CorsLayer
        let app = Router::new().route("/json", get(json));
        serve(app, 4000).await;
    };

    tokio::join!(frontend, backend);
}

async fn serve(app: Router, port: u16) {
    let addr = SocketAddr::from(([127, 0, 0, 1], port));
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

// 前端页面：JS fetch 后端 API
async fn html() -> impl IntoResponse {
    Html(
        r#"
        <script>
            fetch('http://localhost:4000/json')
              .then(response => response.json())
              .then(data => console.log(data));
        </script>
        "#,
    )
}

// 后端 API：返回 JSON
async fn json() -> impl IntoResponse {
    Json(vec!["one", "two", "three"])
}
````

验证：

````bash
cd examples
cargo run -p example-cors
````

浏览器打开 `http://localhost:3000/`，打开 dev tools console，看到错误：

```text
Access to fetch at 'http://localhost:4000/json' from origin 'http://localhost:3000'
has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present
on the requested resource.
```

浏览器拦截了——后端响应没 CORS 头。下一步加。

> **新面孔：同源策略（Same-Origin Policy）**
>
> 浏览器安全机制：网页 JS 默认只能请求**同源**资源。"同源" = scheme + host + port 完全相同。`http://localhost:3000` 和 `http://localhost:4000` 端口不同，不同源，跨域被拦。
>
> 注意：**这是浏览器行为**，不是 HTTP 协议限制。`curl` 不受同源策略影响——它照样能请求。

> **新面孔：跨域场景**
>
> - 前后端分离：前端 `localhost:3000` 调后端 `localhost:4000`（开发）
> - CDN + API：前端 `cdn.example.com` 调 API `api.example.com`
> - 第三方 API：你的网站调 GitHub/Google API
>
> 这些都需要 CORS 才能在浏览器里工作。

---

## 第二步：加 `CorsLayer` 允许跨域

后端加 `tower-http` 的 `CorsLayer`，配置允许的 origin/method/headers。配置后浏览器看到 CORS 头就放行。

````rust
use axum::http::{HeaderValue, Method};
use tower_http::cors::CorsLayer;

# #[tokio::main]
# async fn main() {
#     // ...
    let backend = async {
        let app = Router::new().route("/json", get(json)).layer(
            CorsLayer::new()
                .allow_origin("http://localhost:3000".parse::<HeaderValue>().unwrap())
                .allow_methods([Method::GET]),
        );
        serve(app, 4000).await;
    };
#     // ...
# }
````

现在浏览器再请求，console 看到响应头：

```text
Access-Control-Allow-Origin: http://localhost:3000
```

fetch 成功，`["one", "two", "three"]` 打印出来。

> **新面孔：`CorsLayer`**
>
> tower-http 提供的 CORS layer。`CorsLayer::new()` 开始构建，链式配置：
>
> - `.allow_origin(origin)`：允许的来源（一个 `HeaderValue`，或 `AllowOrigin::any()` 允许所有）
> - `.allow_methods([Method::GET, Method::POST])`：允许的 HTTP 方法
> - `.allow_headers([CONTENT_TYPE, AUTHORIZATION])`：允许的请求头
> - `.allow_credentials(true)`：是否允许带 cookie
> - `.max_age(Duration::from_secs(3600))`：预检结果缓存时间
>
> axum 收到请求时，CorsLayer 检查 Origin 头，匹配就加 CORS 响应头。

> **新面孔：`allow_origin` 严格 vs 宽松**
>
> - `.allow_origin("http://localhost:3000".parse().unwrap())`：**只允许**这个 origin（严格）
> - `.allow_origin(AllowOrigin::any())`：允许任意 origin（宽松，但**不能同时 `allow_credentials(true)`**）
> - `.allow_origin([origin1, origin2])`：白名单
>
> 生产环境推荐白名单——只允许你自己的前端域名。

> **新面孔：预检（Preflight）请求**
>
> 浏览器对**非简单请求**（如带自定义 header、`Content-Type: application/json`、PUT/DELETE）会先发 **OPTIONS** 请求询问后端"这个跨域请求允许吗"。后端响应允许的方法/headers，浏览器才发真正的请求。
>
> ```text
> 1. OPTIONS /json  + Origin + Access-Control-Request-Method   → 预检
> 2. 后端响应: Access-Control-Allow-Origin + Allow-Methods
> 3. 浏览器看到允许，发真正的 GET /json
> ```
>
> `CorsLayer` 自动处理 OPTIONS——你不用写 OPTIONS handler，layer 会响应预检。

### 为什么 `allow_headers([CONTENT_TYPE])` 重要

如果前端发 `Content-Type: application/json` 的 POST，浏览器会发预检询问"是否允许 Content-Type 头"。`CorsLayer` 默认不允许，预检失败，POST 被拦。所以处理 JSON 请求要：

````rust
CorsLayer::new()
    .allow_origin(...)
    .allow_methods([Method::GET, Method::POST])
    .allow_headers([http::header::CONTENT_TYPE])  // 必须显式允许
````

---

## 完整代码

````rust
use axum::{
    http::{HeaderValue, Method},
    response::{Html, IntoResponse},
    routing::get,
    Json, Router,
};
use std::net::SocketAddr;
use tower_http::cors::CorsLayer;

#[tokio::main]
async fn main() {
    let frontend = async {
        let app = Router::new().route("/", get(html));
        serve(app, 3000).await;
    };

    let backend = async {
        let app = Router::new().route("/json", get(json)).layer(
            // see https://docs.rs/tower-http/latest/tower_http/cors/index.html
            // for more details
            //
            // pay attention that for some request types like posting content-type: application/json
            // it is required to add ".allow_headers([http::header::CONTENT_TYPE])"
            // or see this issue https://github.com/tokio-rs/axum/issues/849
            CorsLayer::new()
                .allow_origin("http://localhost:3000".parse::<HeaderValue>().unwrap())
                .allow_methods([Method::GET]),
        );
        serve(app, 4000).await;
    };

    tokio::join!(frontend, backend);
}

async fn serve(app: Router, port: u16) {
    let addr = SocketAddr::from(([127, 0, 0, 1], port));
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await;
}

async fn html() -> impl IntoResponse {
    Html(
        r#"
        <script>
            fetch('http://localhost:4000/json')
              .then(response => response.json())
              .then(data => console.log(data));
        </script>
        "#,
    )
}

async fn json() -> impl IntoResponse {
    Json(vec!["one", "two", "three"])
}
````

## 运行

````bash
cd examples
cargo run -p example-cors
````

浏览器打开 `http://localhost:3000/`，dev tools console 看到 `["one", "two", "three"]`，没有 CORS 错误。

## 解读

### CORS 工作流

```text
浏览器（http://localhost:3000）             后端（http://localhost:4000）
    │                                            │
    │  GET /json  + Origin: http://localhost:3000 │
    │ ──────────────────────────────────────────>│
    │                                            │
    │  200 OK + Access-Control-Allow-Origin: ... │
    │ <──────────────────────────────────────────│
    │                                            │
    │  浏览器检查 Allow-Origin 包含自己 → 放行    │
```

关键：CORS 是**浏览器实现的**安全机制。后端只是加几个 HTTP 头告诉浏览器"我允许跨域"，浏览器自己决定是否放行 JS 读取响应。

### 为什么 CORS 防御有效

没有 CORS 的话，恶意网站 `evil.com` 的 JS 可以请求 `bank.com/api/transfer`（带用户 cookie），转移用户钱。CORS 让 `bank.com` 显式声明"我只信任 `bank.com` 自己的前端"，`evil.com` 的跨域请求被拦。

## 常见问题

**CORS 错误一定是后端问题吗？** 是——浏览器报 CORS 错说明后端响应头不对。修复必须改后端（这章加 CorsLayer）。

**为什么 `curl` 测试没 CORS 问题？** curl 不实现同源策略——它不是浏览器。CORS 只影响浏览器 JS。

**`allow_credentials(true)` 怎么用？** 允许跨域请求带 cookie。前提是 `allow_origin` 必须是具体值（不能是 `*`）——浏览器不允许 wildcard + credentials。

**生产环境 CORS 怎么配？**
- 网关层（Nginx/CDN）统一处理 CORS
- 或每个 axum app 加 CorsLayer（白名单具体 origin）
- **不要用 `AllowOrigin::any()` + credentials**——不安全

## 手写任务

1. 加 POST 路由，前端改成发 JSON POST，加 `.allow_methods([GET, POST])` 和 `.allow_headers([CONTENT_TYPE])`。
2. 改成白名单：`.allow_origin([origin1, origin2])`，对比效果。
3. 加 `allow_credentials(true)` + cookie，跨域带 session。
4. 用 curl 模拟预检请求：`curl -X OPTIONS -H "Origin: http://localhost:3000" -H "Access-Control-Request-Method: GET" -i http://localhost:4000/json`。

## 小结

这章用 2 步讲了 CORS：

1. **复现问题**：浏览器同源策略阻止跨域请求，前端 JS fetch 后端报 CORS 错。
2. **`CorsLayer`**：tower-http 提供，配置 `allow_origin` / `allow_methods` / `allow_headers`，浏览器看到头就放行。

核心：CORS 是**浏览器**的安全机制（不是 HTTP 协议）；后端用 `CorsLayer` 显式声明允许跨域；生产用白名单具体 origin，不用 wildcard + credentials。

## 源码对照

- `examples/cors/Cargo.toml`
- `examples/cors/src/main.rs`
