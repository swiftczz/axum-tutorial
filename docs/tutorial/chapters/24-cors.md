# 24. cors

对应示例：`examples/cors`

本章目标：理解浏览器为什么会拦截跨域请求，并学会用 `CorsLayer` 允许指定前端访问后端接口。

这章开始进入 Web 后端非常常见的问题：前端页面和后端 API 不在同一个 origin 时，浏览器会检查 CORS 响应头。

## 这个小项目在做什么

这个 example 在同一个 Rust 程序里启动两个服务：

```text
前端服务：http://127.0.0.1:3000/
后端服务：http://127.0.0.1:4000/json
```

前端页面里有一段 JavaScript：

```text
fetch('http://localhost:4000/json')
```

后端返回 JSON：

```json
["one", "two", "three"]
```

因为页面来自 `localhost:3000`，接口来自 `localhost:4000`，端口不同，所以浏览器认为这是跨域请求。

请求主线是：

```text
浏览器打开 http://localhost:3000/
-> 页面里的 fetch 请求 http://localhost:4000/json
-> 浏览器发现这是跨域请求
-> 后端 CorsLayer 返回允许跨域的响应头
-> 浏览器允许 JavaScript 读取 JSON
```

## 先理解什么是 origin

浏览器判断“是不是同源”，看的是 origin。  
origin 由三部分组成：

```text
协议 + 域名/主机 + 端口
```

例如：

```text
http://localhost:3000
```

它的三部分是：

```text
协议：http
主机：localhost
端口：3000
```

下面这些都不是同一个 origin：

```text
http://localhost:3000
http://localhost:4000
https://localhost:3000
http://127.0.0.1:3000
```

所以哪怕都是本机，只要端口不同，浏览器也会当成跨域。

## CORS 是谁在限制？

CORS 是浏览器执行的安全规则。  
不是 Axum 特有，也不是 Rust 特有。

这意味着：

```text
curl 请求后端接口 -> 通常不会被 CORS 拦截
浏览器页面 fetch 后端接口 -> 会检查 CORS
```

所以你可能遇到这种现象：

```text
curl http://localhost:4000/json 成功
前端 fetch http://localhost:4000/json 报 CORS 错误
```

这不是后端接口“不能访问”。  
而是浏览器不允许当前页面读取这个跨域响应。

## 文件和依赖

这个 example 有两个文件：

1. `examples/cors/Cargo.toml`：声明 Axum、Tokio、tower-http cors。
2. `examples/cors/src/main.rs`：同时启动前端服务和后端服务，后端挂载 `CorsLayer`。

关键依赖：

- `axum`：提供 `Router`、`Html`、`Json`。
- `tower-http`：提供 `CorsLayer`。
- `tokio`：异步运行时，并用 `tokio::join!` 同时运行两个服务。

`tower-http` 需要启用 cors feature：

````toml
tower-http = { version = "0.6.1", features = ["cors"] }
````

## 第一步：同时定义前端服务和后端服务

源码：

````rust
let frontend = async {
    let app = Router::new().route("/", get(html));
    serve(app, 3000).await;
};

let backend = async {
    let app = Router::new().route("/json", get(json)).layer(...);
    serve(app, 4000).await;
};

tokio::join!(frontend, backend);
````

这里定义了两个异步任务：

- `frontend`：监听 3000，返回 HTML 页面。
- `backend`：监听 4000，返回 JSON API。

`tokio::join!` 表示两个任务一起运行。  
如果只写：

````rust
frontend.await;
backend.await;
````

第一个服务会一直运行，程序永远不会走到第二个服务。  
所以这里必须用 `tokio::join!`。

## 第二步：前端页面发起跨域请求

源码：

````rust
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
````

这个 handler 返回的是 HTML。  
浏览器打开页面后，会执行里面的 JavaScript。

关键代码是：

````javascript
fetch('http://localhost:4000/json')
````

页面来自：

```text
http://localhost:3000
```

请求发往：

```text
http://localhost:4000
```

端口不同，所以这是跨域请求。

## 第三步：后端返回 JSON

源码：

````rust
async fn json() -> impl IntoResponse {
    Json(vec!["one", "two", "three"])
}
````

`Json(...)` 会把 Rust 数据序列化成 JSON 响应。  
这里返回的是字符串数组：

```json
["one", "two", "three"]
```

这个 handler 本身不关心 CORS。  
CORS 是统一放在 Router 外层的 middleware 里处理的。

## 第四步：给后端加 CorsLayer

源码：

````rust
let app = Router::new().route("/json", get(json)).layer(
    CorsLayer::new()
        .allow_origin("http://localhost:3000".parse::<HeaderValue>().unwrap())
        .allow_methods([Method::GET]),
);
````

这段配置告诉浏览器：

```text
我允许 http://localhost:3000 这个前端 origin 来请求我
我允许它使用 GET 方法
```

后端响应里会带上类似这样的 CORS header：

```text
access-control-allow-origin: http://localhost:3000
```

浏览器看到这个 header 后，才会允许页面里的 JavaScript 读取响应内容。

## 第五步：allow_origin 要精确匹配

源码里写的是：

````rust
.allow_origin("http://localhost:3000".parse::<HeaderValue>().unwrap())
````

这里必须注意：

```text
http://localhost:3000
```

和：

```text
http://127.0.0.1:3000
```

不是同一个 origin。

如果你在浏览器里打开的是：

```text
http://127.0.0.1:3000/
```

但后端只允许：

```text
http://localhost:3000
```

浏览器仍然可能报 CORS 错误。

所以验证本章时，建议打开：

```text
http://localhost:3000/
```

而不是：

```text
http://127.0.0.1:3000/
```

## 第六步：allow_methods 限制请求方法

源码：

````rust
.allow_methods([Method::GET])
````

这表示后端只允许跨域 GET 请求。

如果未来你要支持：

```text
POST /json
PUT /json
DELETE /json
```

就需要把对应方法加进去。

例如：

````rust
.allow_methods([Method::GET, Method::POST])
````

不要为了省事一开始就全部放开。  
后端接口越明确，排查问题越容易。

## 第七步：什么时候需要 allow_headers？

源码注释里提到：

````rust
// pay attention that for some request types like posting content-type: application/json
// it is required to add ".allow_headers([http::header::CONTENT_TYPE])"
````

本章 example 只是普通 GET 请求，所以没有配置 `allow_headers`。

但如果前端这样发请求：

````javascript
fetch('http://localhost:4000/json', {
  method: 'POST',
  headers: {
    'content-type': 'application/json'
  },
  body: JSON.stringify({ name: 'Alice' })
})
````

浏览器会检查后端是否允许 `content-type` 这个 header。  
后端通常需要加：

````rust
.allow_headers([axum::http::header::CONTENT_TYPE])
````

否则你会看到 CORS 相关报错，即使你的 handler 本身写对了。

## 第八步：理解 preflight（预检请求）——CORS 的关键概念

这是 CORS 最容易被初学者忽略、又最容易踩坑的部分。本章的 example 只演示了简单的 GET 请求，所以你没看到 preflight；但一旦你写 POST JSON 或带自定义 header 的请求，浏览器就会先发一个**预检请求**。

### 什么是 preflight

对于"非简单请求"，浏览器不会直接发真正的请求，而是**先发一个 `OPTIONS` 请求**询问后端："我能不能用这种方法、这些 header 跨域访问你？" 这个 OPTIONS 请求就叫 preflight（预检）。

```text
浏览器想发：POST /json  content-type: application/json  (跨域)
       ↓
浏览器先发：OPTIONS /json  (preflight，没有 body)
       ↓
后端回复：HTTP 200，带上 CORS 允许头
       ↓ 浏览器检查通过
浏览器才发真正的：POST /json  content-type: application/json
       ↓
后端处理并返回响应
```

注意 preflight 是 `OPTIONS` 方法，**不是 GET/POST**，而且它不会到达你写的 handler。

### 什么请求会触发 preflight

简单记一个规则：**不是"简单请求"就会触发 preflight**。"简单请求"的条件很严格，必须同时满足：

- 方法是 `GET`、`HEAD` 或 `POST`
- 只能用浏览器自动设置的几个 header（`accept`、`accept-language`、`content-language`、`content-type`）
- `content-type` 只能是 `text/plain`、`multipart/form-data`、`application/x-www-form-urlencoded`
- 请求里没有自定义 header（如 `Authorization`、`X-Requested-With`）

所以这些都会触发 preflight：

| 请求 | 为什么触发 preflight |
| --- | --- |
| `POST` + `content-type: application/json` | content-type 不是允许的三种 |
| 任何带 `Authorization` header 的请求 | 自定义 header |
| `PUT` / `DELETE` / `PATCH` | 方法不在简单请求列表 |
| 带 `X-Requested-With` 等 `X-` 开头 header | 自定义 header |

本章 example 用的是普通 GET，所以没有 preflight。这就是为什么第七步说"POST JSON 需要 `allow_headers`"——本质上是因为 POST JSON 会触发 preflight，而 preflight 会检查后端是否允许 `content-type` 这个 header。

### CorsLayer 如何自动处理 preflight

好消息是：**`tower-http` 的 `CorsLayer` 会自动响应 preflight 请求**。你不需要自己写 `OPTIONS` handler。

当 preflight（`OPTIONS` 请求）到达时，`CorsLayer` 会根据你配置的 `allow_origin`、`allow_methods`、`allow_headers` 自动回复一个带 CORS 响应头的 `200`，告诉浏览器"允许"。你的 handler 根本不会被调用。

所以这段配置：

````rust
CorsLayer::new()
    .allow_origin("http://localhost:3000".parse::<HeaderValue>().unwrap())
    .allow_methods([Method::GET, Method::POST])
    .allow_headers([axum::http::header::CONTENT_TYPE])
````

实际上做了两件事：

1. 给**真正的请求响应**加上 `access-control-allow-origin` 等头。
2. 自动**回复 preflight**：当浏览器发 `OPTIONS` 时，返回 `access-control-allow-methods: GET, POST` 和 `access-control-allow-headers: content-type`。

### 常见的 preflight 报错

如果你在浏览器 Console 看到类似这样的错误：

```text
Access to fetch at 'http://localhost:4000/json' from origin 'http://localhost:3000'
has been blocked by CORS policy: Response to preflight request doesn't pass
access control check: It does not have HTTP ok status.
```

或者：

```text
... has been blocked by CORS policy:
Request header field content-type is not allowed by Access-Control-Allow-Headers
in preflight response.
```

这些都是 **preflight 没通过**。排查方向：

- 后端是否真的挂了 `CorsLayer`？
- `allow_methods` 是否包含了请求要用的方法（POST/PUT/DELETE）？
- `allow_headers` 是否包含了请求要带的 header（特别是 `content-type`）？
- 后端有没有别的 middleware 在 `CorsLayer` 之前拦截了 OPTIONS？（比如认证 middleware 拒绝了未认证的 OPTIONS，导致 preflight 拿不到 200）

最后一条尤其坑：如果你在 `CorsLayer` **之前**套了一个需要认证的 middleware，preflight 的 OPTIONS 请求没有带认证信息，会被认证 middleware 直接拒绝（401/403），CorsLayer 根本没机会回复。解决方法是让 CORS 相关的 layer 在最外层，或者让认证 middleware 放行 OPTIONS 请求。

### `MaxAge`：缓存 preflight 结果

每次发非简单请求都要先 preflight，会很慢。浏览器允许缓存 preflight 的结果，缓存时长由响应头 `access-control-max-age` 控制。`CorsLayer` 提供了 `.max_age(...)` 来设置：

````rust
use std::time::Duration;

CorsLayer::new()
    .allow_origin(...)
    .allow_methods(...)
    .allow_headers(...)
    .max_age(Duration::from_secs(3600))  // 让浏览器缓存 preflight 结果 1 小时
````

生产环境通常会设一个较大的值（如几小时），避免重复 preflight。

## 第九步：serve 函数复用启动逻辑

源码：

````rust
async fn serve(app: Router, port: u16) {
    let addr = SocketAddr::from(([127, 0, 0, 1], port));
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await;
}
````

这个函数把“监听端口并启动服务”的逻辑封装起来。  
前端和后端都可以复用它：

````rust
serve(app, 3000).await;
serve(app, 4000).await;
````

这里监听的是 `127.0.0.1`。  
但浏览器访问 `localhost` 时，通常也会解析到本机地址，所以 example 可以用 `localhost:3000` 和 `localhost:4000`。

## 函数职责速查

- `main`：创建前端服务和后端服务，并用 `tokio::join!` 同时运行。
- `serve`：绑定指定端口并启动 Axum 服务。
- `html`：返回一个会执行 `fetch` 的 HTML 页面。
- `json`：返回 JSON 数组。

## 带中文注释的手写版

````rust
//! 文档注释：说明如何运行这个 example。
//! Run with
//!
//! ```not_rust
//! cargo run -p example-cors
//! ```

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
    // 前端服务：监听 3000 端口，返回一个 HTML 页面。
    let frontend = async {
        let app = Router::new().route("/", get(html));
        serve(app, 3000).await;
    };

    // 后端服务：监听 4000 端口，提供 JSON API。
    let backend = async {
        let app = Router::new().route("/json", get(json)).layer(
            // CorsLayer 用来给响应加 CORS 相关 header。
            //
            // 更多配置可以看 tower-http 的 cors 文档。
            //
            // 注意：如果跨域 POST application/json，通常还需要：
            // .allow_headers([http::header::CONTENT_TYPE])
            CorsLayer::new()
                // 只允许 http://localhost:3000 这个前端 origin 访问。
                .allow_origin("http://localhost:3000".parse::<HeaderValue>().unwrap())
                // 只允许跨域 GET 方法。
                .allow_methods([Method::GET]),
        );

        serve(app, 4000).await;
    };

    // 同时运行前端服务和后端服务。
    // 两个 HTTP 服务都会一直运行，所以不能一个 await 完再启动另一个。
    tokio::join!(frontend, backend);
}

// 启动一个 Axum 服务，监听指定端口。
async fn serve(app: Router, port: u16) {
    let addr = SocketAddr::from(([127, 0, 0, 1], port));
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

// 前端页面。
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

// 后端 JSON 接口。
async fn json() -> impl IntoResponse {
    Json(vec!["one", "two", "three"])
}
````

## 运行和验证

运行：

````bash
cargo run -p example-cors
````

打开浏览器：

```text
http://localhost:3000/
```

打开浏览器开发者工具的 Console，应该能看到：

```text
["one", "two", "three"]
```

也可以直接查看后端响应头：

````bash
curl -i -H 'Origin: http://localhost:3000' http://localhost:4000/json
````

重点看：

```text
access-control-allow-origin: http://localhost:3000
```

再试一个不允许的 origin：

````bash
curl -i -H 'Origin: http://evil.example' http://localhost:4000/json
````

这时响应不应该给出允许 `http://evil.example` 的 CORS header。

## 常见卡点

### 1. 为什么 curl 成功，浏览器还是报 CORS？

因为 CORS 是浏览器规则。  
curl 不会像浏览器一样阻止 JavaScript 读取跨域响应。

### 2. 为什么 localhost 和 127.0.0.1 会影响 CORS？

因为 origin 要精确匹配。  
`localhost` 和 `127.0.0.1` 在浏览器看来是不同主机。

### 3. 为什么 POST JSON 还要配置 allow_headers？

因为 `content-type: application/json` 会让浏览器做更严格的跨域检查。  
后端不仅要允许方法，还要允许请求 header。

更准确地说：POST JSON 是"非简单请求"，会触发 **preflight**（见第八步）。preflight 会检查后端是否允许 `content-type` header，不允许就拒绝。

### 4. 可以直接允许所有 origin 吗？

开发环境可以临时放宽，但生产环境要谨慎。  
公开 API 和带用户登录态的 API，CORS 策略不能随便全放开。

### 5. CORS 能保护后端接口不被调用吗？

不能。  
CORS 主要限制浏览器页面读取响应，不是后端鉴权机制。

如果接口需要保护，仍然要做认证、授权和服务端校验。

### 6. 报错说 "preflight request doesn't pass" 怎么办？

这说明浏览器的 OPTIONS 预检请求没拿到正确的 CORS 响应（见第八步）。最常见的原因是**其他 middleware 在 CorsLayer 之前拦截了 OPTIONS**，比如认证 middleware 把未认证的 OPTIONS 返回了 401，导致 CorsLayer 没机会回复 preflight。解决方法：把 CorsLayer 放在最外层，或让认证 middleware 放行 OPTIONS 方法。

## 手写任务

建议按下面顺序自己敲一遍：

1. 先写一个 `GET /json`，返回 `Json(vec![...])`。
2. 再写一个 `GET /`，返回带 `fetch` 的 HTML。
3. 用 `serve(app, port)` 分别启动 3000 和 4000。
4. 用 `tokio::join!` 同时运行两个服务。
5. 给后端 Router 加 `CorsLayer`。
6. 用浏览器 Console 验证前端能读到 JSON。

加深练习：

1. 把浏览器地址改成 `http://127.0.0.1:3000/`，观察是否还能通过。
2. 把 `.allow_methods([Method::GET])` 改成不包含 GET，观察报错。
3. 尝试改成 POST JSON，并补上 `.allow_headers([axum::http::header::CONTENT_TYPE])`。

## 本章真正要记住什么

CORS 不是"后端接口通不通"的问题。  
它是：

```text
浏览器是否允许当前页面读取另一个 origin 的响应
```

在 Axum 里，最常见的做法是把 CORS 放到后端 Router 的 middleware 上：

````rust
CorsLayer::new()
    .allow_origin(...)
    .allow_methods(...)
````

先精确允许需要的前端 origin 和方法。  
等你真的需要 POST JSON、认证 header、cookie 时，再继续补 `allow_headers`、`allow_credentials` 等配置。

**最重要的一个概念**：非简单请求（POST JSON、自定义 header、PUT/DELETE 等）会先触发 **preflight 预检请求**（一个 `OPTIONS` 请求）。`tower-http` 的 `CorsLayer` 会自动回复 preflight，你不需要自己写 OPTIONS handler。如果浏览器报 "preflight" 相关错误，排查 `allow_methods`、`allow_headers` 是否完整，以及是否有其他 middleware 拦截了 OPTIONS。

## 源码对照

本章手写版对应源码：

- `examples/cors/src/main.rs`
- `examples/cors/Cargo.toml`
