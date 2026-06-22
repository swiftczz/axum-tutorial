# 25. cors

对应示例：`examples/cors`

前端页面和后端 API 不在同一个 origin 时,浏览器会检查 CORS 响应头。本章用 `CorsLayer` 允许指定前端访问后端接口,并重点讲清 **preflight 预检请求**这个最容易踩坑的概念。

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
tower-http = { version = "0.6.1", features = ["cors"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## 关键概念

> **新面孔：CORS + preflight**
>
> CORS 是浏览器规则（不是后端鉴权）。非简单请求（POST JSON 等）会先触发 preflight（OPTIONS），`CorsLayer` 自动回复。


## src/main.rs

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
    axum::serve(listener, app).await.unwrap();
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

浏览器打开 `http://localhost:3000/`(注意用 `localhost` 不用 `127.0.0.1`),打开 Console 看到:

````text
["one", "two", "three"]
````

查看后端响应头:

````bash
curl -i -H 'Origin: http://localhost:3000' http://localhost:4000/json
````

重点看:

````text
access-control-allow-origin: http://localhost:3000
````

不允许的 origin:

````bash
curl -i -H 'Origin: http://evil.example' http://localhost:4000/json
# 响应不应给出允许 http://evil.example 的 CORS header
````

## 解读

### origin 和"跨域"

浏览器判断"同源"看 origin = 协议 + 主机 + 端口。下面这些都不是同一个 origin:

```text
http://localhost:3000
http://localhost:4000      (端口不同)
https://localhost:3000     (协议不同)
http://127.0.0.1:3000      (主机不同)
```

哪怕都是本机,只要端口不同就是跨域。

### CORS 是浏览器的规则

CORS 是**浏览器**执行的安全规则,不是 axum 或 Rust 特有:

```text
curl 请求后端接口    → 通常不会被 CORS 拦截
浏览器页面 fetch 后端 → 会检查 CORS
```

所以你会遇到"curl 成功但浏览器 fetch 报 CORS 错"——不是后端接口不通,而是浏览器不允许当前页面读取跨域响应。

### 两个服务同时跑

````rust
let frontend = async { ... serve(app, 3000).await; };
let backend = async { ... serve(app, 4000).await; };
tokio::join!(frontend, backend);
````

两个 HTTP 服务都要一直运行,不能一个 `await` 完再启动另一个(第一个永远不返回),所以用 `tokio::join!`。

### `CorsLayer` 三件套

````rust
CorsLayer::new()
    .allow_origin("http://localhost:3000".parse::<HeaderValue>().unwrap())
    .allow_methods([Method::GET]),
````

告诉浏览器:允许 `http://localhost:3000` 这个 origin 用 GET 方法跨域访问。后端响应带上 `access-control-allow-origin: http://localhost:3000`,浏览器看到才允许页面 JS 读取响应。

**`allow_origin` 要精确匹配**:`http://localhost:3000` 和 `http://127.0.0.1:3000` 不是同一个 origin。浏览器里打开的是 `127.0.0.1` 但后端只允许 `localhost`,仍会报 CORS 错。

### preflight 预检请求(CORS 最关键的概念)

example 演示的是简单 GET 请求,没触发 preflight。但**一旦写 POST JSON 或带自定义 header,浏览器会先发一个预检请求**。

**什么是 preflight**:对"非简单请求",浏览器不直接发真请求,而是先发一个 `OPTIONS` 请求问后端"我能不能用这种方法、这些 header 跨域访问":

```text
浏览器想发:POST /json  content-type: application/json  (跨域)
       ↓
浏览器先发:OPTIONS /json  (preflight,没 body)
       ↓
后端回复:HTTP 200 + CORS 允许头
       ↓ 浏览器检查通过
浏览器才发真正的:POST /json
```

preflight 是 `OPTIONS` 方法,**不会到达你的 handler**。

**什么请求触发 preflight**:"简单请求"条件很严:方法只能是 GET/HEAD/POST、`content-type` 只能是 `text/plain`/`multipart/form-data`/`application/x-www-form-urlencoded`、没有自定义 header。不是简单请求就触发 preflight:

| 请求 | 为什么触发 |
| --- | --- |
| `POST` + `content-type: application/json` | content-type 不在允许的三种 |
| 带 `Authorization` 的请求 | 自定义 header |
| `PUT` / `DELETE` / `PATCH` | 方法不在简单列表 |
| 带 `X-Requested-With` 等 `X-` header | 自定义 header |

**CorsLayer 自动处理 preflight**:`tower-http` 的 `CorsLayer` 会自动响应 preflight——当 `OPTIONS` 请求到达时,它根据你的配置自动返回带 CORS 头的 200。你不需要写 `OPTIONS` handler,handler 根本不会被调用。

所以 POST JSON 时配置:

````rust
CorsLayer::new()
    .allow_origin(...)
    .allow_methods([Method::GET, Method::POST])
    .allow_headers([axum::http::header::CONTENT_TYPE])
````

做了两件事:给真请求响应加 `access-control-allow-origin`,自动回复 preflight(返回 `access-control-allow-methods: GET, POST` 和 `access-control-allow-headers: content-type`)。

**常见 preflight 报错**:

```text
blocked by CORS policy: Response to preflight request doesn't pass...
Request header field content-type is not allowed by Access-Control-Allow-Headers
in preflight response.
```

排查方向:

- 后端是否真的挂了 `CorsLayer`?
- `allow_methods` 是否包含请求要用的方法(POST/PUT/DELETE)?
- `allow_headers` 是否包含请求要带的 header(尤其 `content-type`)?
- **是否有其他 middleware 在 `CorsLayer` 之前拦截了 OPTIONS?**(最坑:认证 middleware 把未认证的 OPTIONS 返回 401,CorsLayer 没机会回复。解决:CorsLayer 放最外层,或认证 middleware 放行 OPTIONS)

**`MaxAge` 缓存 preflight**:每次非简单请求都 preflight 会很慢,`.max_age(Duration::from_secs(3600))` 让浏览器缓存 preflight 结果。

### CORS 的边界

- **CORS 不是后端鉴权机制**——它只限制浏览器页面读取响应,不能保护后端不被调用(接口需要保护仍要做认证授权)。
- **生产环境不能全放开**——公开 API 和带登录态的 API,CORS 策略不能随便 `allow_origin` 全允许。
- **curl 成功但浏览器报 CORS 是正常的**——CORS 是浏览器规则。

## 手写任务

按下面顺序敲:

1. 写一个 `GET /json` 返回 `Json(vec![...])`。
2. 写一个 `GET /` 返回带 `fetch` 的 HTML。
3. 用 `serve(app, port)` 分别启动 3000 和 4000。
4. 用 `tokio::join!` 同时运行。
5. 给后端 Router 加 `CorsLayer`。
6. 浏览器 Console 验证前端能读到 JSON。

加深练习:

1. 浏览器地址改成 `http://127.0.0.1:3000/`,观察是否还能通过。
2. 把 `.allow_methods([Method::GET])` 改成不含 GET,观察报错。
3. 改成 POST JSON,补上 `.allow_headers([axum::http::header::CONTENT_TYPE])`。

## 小结

- CORS 是浏览器规则:浏览器是否允许当前页面读取另一个 origin 的响应。curl 通常不受影响。
- origin = 协议 + 主机 + 端口,任一不同就是跨域;`localhost` 和 `127.0.0.1` 不是同源。
- `CorsLayer` 配置三件套:`allow_origin`(精确匹配)、`allow_methods`、按需 `allow_headers`。
- **非简单请求(POST JSON/自定义 header/PUT/DELETE)会触发 preflight**——浏览器先发 `OPTIONS`。`CorsLayer` 自动回复 preflight,不需写 OPTIONS handler。
- CORS 不是鉴权机制,生产环境不能随便全放开;认证 middleware 拦截 OPTIONS 是常见坑,让 CorsLayer 放最外层。

## 源码对照

- `examples/cors/Cargo.toml`
- `examples/cors/src/main.rs`
