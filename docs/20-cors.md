# 第 20 章 cors:浏览器的同源策略

## 本章你将做出什么

**两个服务**跑在同一个进程里:

- `http://localhost:3000` — 前端,返回一个用 `fetch` 调后端的 HTML 页面
- `http://localhost:4000` — 后端,返回一段 JSON

不同端口就是不同源,浏览器默认会拦下这次调用。你会先亲眼看到那条经典的报错,再用 `CorsLayer` 放行。

这一章解决的是几乎每个前后端分离项目都会撞上的问题:**后端明明返回了数据,浏览器控制台却报错,而 curl 一切正常**。

学完本章你将掌握:

- 同源策略是什么,以及它为什么存在
- CORS 的工作机制:简单请求与预检请求(preflight)的区别
- `CorsLayer` 的几个配置项各自对应哪一道检查
- 为什么 `allow_origin(Any)` 加 `allow_credentials(true)` 会直接报错

## 前置知识

### 同源策略:浏览器的基本安全边界

**同源策略**(Same-Origin Policy)是浏览器的一条硬规则:一个页面的 JavaScript **默认不能读取**另一个源的响应内容。

"源"(origin)由三部分构成,**任何一个不同就是跨源**:

| | 协议 | 主机 | 端口 |
|---|---|---|---|
| `http://localhost:3000` | http | localhost | 3000 |
| `http://localhost:4000` | http | localhost | **4000** ← 跨源 |
| `https://localhost:3000` | **https** ← 跨源 | localhost | 3000 |
| `http://127.0.0.1:3000` | http | **127.0.0.1** ← 跨源 | 3000 |

最后一行值得注意:**`localhost` 和 `127.0.0.1` 是不同的源**,尽管它们指向同一台机器。本章示例统一用 `localhost`,换成 `127.0.0.1` 就会失败——这是实际开发中常见的困惑来源。

**为什么需要这条规则。** 设想你登录了网银 `bank.com`,浏览器保存了 cookie。这时你访问了恶意网站 `evil.com`,它的页面里有:

```js
fetch('https://bank.com/api/balance')
  .then(r => r.json())
  .then(data => sendToAttacker(data));
```

浏览器发这个请求时**会自动带上 bank.com 的 cookie**(cookie 是按域名存的,不管谁发起的请求)。如果没有同源策略,`evil.com` 就能读到你的余额。

同源策略切断了这条路:请求可以发出去,但**响应内容不给 JavaScript 读**。

### 一个关键区分:请求发出去了吗

这是最容易误解的地方。

**同源策略拦的是"读取响应",不是"发送请求"。** 大多数跨源请求实际上**已经到达服务器并被处理了**,只是浏览器不把结果交给 JS。

这个区别有实际后果:一个跨源的 `POST /transfer` 可能真的执行了转账,只是攻击者读不到响应。防这类攻击靠的是 CSRF token,不是 CORS。**CORS 保护的是"数据不被读走",不是"操作不被执行"。**

另一个后果:**服务端日志里能看到这些请求**。前端说"CORS 报错了",你去查日志发现请求好好地返回了 200 —— 这不矛盾,问题出在浏览器那一侧。

### CORS:服务端说"我允许"

**CORS**(Cross-Origin Resource Sharing,跨源资源共享)是同源策略的官方放行机制。核心逻辑很简单:**服务端在响应头里声明"我允许某个源读我的内容",浏览器看到这个声明才放行。**

最关键的头部是:

```
Access-Control-Allow-Origin: http://localhost:3000
```

**注意放行的决定权在服务端,执行者是浏览器。** 服务端不加这个头部,浏览器就拦;服务端加了,浏览器就放行。curl、Postman 这类工具**完全不管 CORS** —— 它们不是浏览器,没有同源策略。所以"curl 能通但浏览器报错"是 CORS 问题的典型症状。

### 简单请求与预检请求

CORS 把跨源请求分成两类,处理方式不同。

**简单请求**(simple request)直接发出去,服务端响应里带上允许头部就行。满足以下**全部**条件才算简单请求:

- 方法是 `GET`、`HEAD` 或 `POST`
- 只用了安全的头部(`Accept`、`Accept-Language`、`Content-Language`、`Content-Type`)
- `Content-Type` 只能是 `text/plain`、`multipart/form-data`、`application/x-www-form-urlencoded` 三者之一

**预检请求**(preflight)不满足上述条件时触发。浏览器会**先发一个 `OPTIONS` 请求**问服务端"我能不能这么干",得到肯定答复后才发真正的请求。

```
OPTIONS /api/users HTTP/1.1
Origin: http://localhost:3000
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type
```

服务端要回答:

```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: http://localhost:3000
Access-Control-Allow-Methods: POST
Access-Control-Allow-Headers: content-type
Access-Control-Max-Age: 86400
```

**这里藏着一个极常见的坑**:发 JSON 时 `Content-Type: application/json` **不在简单请求的白名单里**,所以会触发预检。而预检要求服务端明确允许 `content-type` 这个头部 —— 很多人配了 `allow_origin` 和 `allow_methods` 却漏了 `allow_headers`,结果 `GET` 好使、`POST` JSON 就失败。

官方示例的注释专门提到了这件事,还附了 [issue #849](https://github.com/tokio-rs/axum/issues/849) 的链接。

`Access-Control-Max-Age` 让浏览器缓存预检结果,避免每个请求都多一次往返。

### CorsLayer 的配置项

`tower-http` 的 `CorsLayer` 把上面这些头部封装成了配置方法:

| 方法 | 对应头部 | 作用 |
|---|---|---|
| `allow_origin(...)` | `Access-Control-Allow-Origin` | 哪些源可以读响应 |
| `allow_methods([...])` | `Access-Control-Allow-Methods` | 预检时允许哪些方法 |
| `allow_headers([...])` | `Access-Control-Allow-Headers` | 预检时允许哪些请求头 |
| `expose_headers([...])` | `Access-Control-Expose-Headers` | 允许 JS 读取哪些**响应**头 |
| `allow_credentials(true)` | `Access-Control-Allow-Credentials` | 允许带 cookie |
| `max_age(...)` | `Access-Control-Max-Age` | 预检结果缓存多久 |

`expose_headers` 容易被忽略:**默认情况下 JS 只能读到几个基本响应头**,自定义头部(比如[第 19 章](19-request-id.md)那个 `x-request-id`)必须显式暴露,否则 `response.headers.get('x-request-id')` 会返回 `null`。

**`allow_credentials(true)` 和 `allow_origin(Any)` 不能共存。** 规范禁止这个组合——允许任意源同时又允许带 cookie,等于把同源策略彻底废掉。`tower-http` 会在运行时 panic 提醒你,这比静默失效好。

## 项目搭建

```bash
cargo new cors
cd cors
```

依赖只有三个,本章示例连日志都没配:

```bash
cargo add axum
cargo add tokio --features full
cargo add tower-http --features cors
```

## 分步构建

### 步骤 1:在一个进程里跑两个服务

**要做什么**:搭出前端和后端两个服务,先不配 CORS,亲眼看到浏览器拦截。

```rust
// src/main.rs
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
```

**`tokio::join!` 同时驱动两个 future。** 两个 `async` 块各自绑定一个端口、跑一个 `Router`,`join!` 让它们并发执行且都不返回(服务器永不停止)。

这是本教程第一次在一个进程里跑两个服务。真实项目里前端和后端当然是分开部署的,示例这么写是为了让你一条 `cargo run` 就能复现跨源场景。

**`serve` 辅助函数**把重复的绑定逻辑收拢。`SocketAddr::from(([127, 0, 0, 1], port))` 是构造地址的简洁写法。

**前端页面只有一个 `<script>`**,用 `fetch` 调后端并把结果打进控制台。没有可见内容,所以页面看起来是空白的——**这一章的观察点在浏览器控制台,不是页面上**。

**跑起来看**:

```bash
cargo run
```

先确认后端本身是好的:

```bash
curl -i http://localhost:4000/json
```

```
HTTP/1.1 200 OK
content-type: application/json
content-length: 23

["one","two","three"]
```

**curl 完全正常**,没有任何 CORS 相关的头部,也不需要——curl 不是浏览器。

现在用浏览器打开 `http://localhost:3000/`,按 F12 打开开发者工具,看 Console 标签:

```
Access to fetch at 'http://localhost:4000/json' from origin 'http://localhost:3000'
has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present
on the requested resource.
```

**这就是那条经典报错。** 意思很直白:响应里没有 `Access-Control-Allow-Origin` 头部,浏览器不允许 JS 读取。

**关键的一步:切到 Network 标签,找到那个 `json` 请求。** 你会看到:

- 状态码是 **200**
- Response 里**有完整的 JSON 数据**

**请求成功了,数据也回来了,只是浏览器不给 JS。** 这就是前置知识说的"拦的是读取,不是发送"。理解这一点,以后遇到 CORS 问题就不会往"请求没发出去"的方向排查了。

再验证一下 `localhost` 和 `127.0.0.1` 是不同源:浏览器访问 `http://127.0.0.1:3000/`,控制台报错依旧,但报错里的 origin 变成了 `http://127.0.0.1:3000`。

### 步骤 2:加上 CorsLayer

**要做什么**:给后端配上 CORS,让前端能读到数据。

```rust
// src/main.rs
use axum::{
    http::{HeaderValue, Method},                    // 新增
    response::{Html, IntoResponse},
    routing::get,
    Json, Router,
};
use std::net::SocketAddr;
use tower_http::cors::CorsLayer;                    // 新增

#[tokio::main]
async fn main() {
    let frontend = async {
        let app = Router::new().route("/", get(html));
        serve(app, 3000).await;
    };

    let backend = async {
        // 改动:挂上 CorsLayer
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

// ...(其余不变)
```

**`allow_origin` 接受一个 `HeaderValue`**,所以要 `.parse::<HeaderValue>()`。这里写死了 `http://localhost:3000`——只有这个源能读。

它也接受其它形式:`Any`(任意源)、一个源列表、或者一个动态判断的闭包。生产环境**不要用 `Any`**,那等于放弃了这层保护;应该配置成明确的白名单。

**`allow_methods([Method::GET])`** 声明允许的方法。本例只有 `GET`,所以够用。

**注释里那段警告很重要**:发 `application/json` 会触发预检,而预检需要 `allow_headers([CONTENT_TYPE])`。本例的 `fetch` 是个不带自定义头部的 `GET`,属于**简单请求**,所以不需要——但这也意味着这个配置一旦用到 POST JSON 就会失败。步骤 3 会验证这一点。

**跑起来看**:

```bash
cargo run
```

浏览器重新访问 `http://localhost:3000/`,看 Console:

```
Array(3) [ "one", "two", "three" ]
```

**数据读到了。** 切到 Network 标签看那个 `json` 请求的响应头,多了一行:

```
access-control-allow-origin: http://localhost:3000
```

用 curl 确认(要手动带上 `Origin` 头部,curl 默认不带):

```bash
curl -i -H 'Origin: http://localhost:3000' http://localhost:4000/json
```

```
HTTP/1.1 200 OK
content-type: application/json
vary: origin
access-control-allow-origin: http://localhost:3000
content-length: 23

["one","two","three"]
```

**注意还多了一个 `vary: origin`。** 这是 `CorsLayer` 自动加的,告诉缓存(CDN、代理)"这个响应的内容取决于请求的 `Origin` 头部,不同 Origin 要分开缓存"。漏了它会导致 A 源的响应被缓存后错误地返回给 B 源——这是个真实存在的缓存投毒风险,`tower-http` 替你处理了。

**试试不被允许的源**:

```bash
curl -i -H 'Origin: http://evil.com' http://localhost:4000/json
```

```
HTTP/1.1 200 OK
content-type: application/json
vary: origin
content-length: 23

["one","two","three"]
```

**没有 `access-control-allow-origin` 头部了。** 数据仍然返回(curl 不管这些),但浏览器看到响应里没有允许头部就会拦下。

### 步骤 3:撞一次预检的坑

**要做什么**:验证前置知识说的那个常见错误——`GET` 好使但 POST JSON 失败。

在后端加一条 POST 路由,前端改成发 JSON:

```rust
// src/main.rs
use axum::{
    http::{HeaderValue, Method},
    response::{Html, IntoResponse},
    routing::{get, post},                          // 临时改动
    Json, Router,
};
// ...

    let backend = async {
        let app = Router::new()
            .route("/json", get(json))
            .route("/echo", post(echo))            // 临时新增
            .layer(
                CorsLayer::new()
                    .allow_origin("http://localhost:3000".parse::<HeaderValue>().unwrap())
                    .allow_methods([Method::GET, Method::POST]),   // 临时:加上 POST
            );
        serve(app, 4000).await;
    };

// 临时新增
async fn echo(Json(v): Json<serde_json::Value>) -> impl IntoResponse {
    Json(v)
}
```

需要 `cargo add serde_json`。前端页面改成:

```rust
async fn html() -> impl IntoResponse {
    Html(
        r#"
        <script>
            fetch('http://localhost:4000/echo', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ hello: 'world' })
            })
              .then(response => response.json())
              .then(data => console.log(data));
        </script>
        "#,
    )
}
```

**跑起来看**:

```bash
cargo run
```

浏览器访问 `http://localhost:3000/`,Console:

```
Access to fetch at 'http://localhost:4000/echo' from origin 'http://localhost:3000'
has been blocked by CORS policy: Request header field content-type is not allowed
by Access-Control-Allow-Headers in preflight response.
```

**方法已经允许了,但 `content-type` 这个请求头没被允许。** 报错信息里明确出现了 `preflight response` —— 这次失败发生在预检阶段。

切到 Network 标签,你会看到**两条**记录:一条 `OPTIONS /echo`(预检),一条被取消的 `POST /echo`。**预检没通过,真正的请求根本没发出去。**

手动复现预检请求:

```bash
curl -i -X OPTIONS http://localhost:4000/echo \
  -H 'Origin: http://localhost:3000' \
  -H 'Access-Control-Request-Method: POST' \
  -H 'Access-Control-Request-Headers: content-type'
```

```
HTTP/1.1 200 OK
vary: origin, access-control-request-method, access-control-request-headers
access-control-allow-origin: http://localhost:3000
access-control-allow-methods: GET,POST
content-length: 0
```

**响应里没有 `access-control-allow-headers`。** 补上配置:

```rust
use axum::http::header::CONTENT_TYPE;

                CorsLayer::new()
                    .allow_origin("http://localhost:3000".parse::<HeaderValue>().unwrap())
                    .allow_methods([Method::GET, Method::POST])
                    .allow_headers([CONTENT_TYPE]),        // 补上这一行
```

再跑,Console 变成:

```
Object { hello: "world" }
```

预检响应也多了那一行:

```
access-control-allow-headers: content-type
```

**这就是官方注释里警告的那个坑。** 记住这个判断顺序:配了 CORS 但 POST JSON 还是失败 → 看报错里有没有 `preflight` → 有就检查 `allow_headers`。

把步骤 3 的临时改动全部撤销,回到官方示例的形态。

## 代价与边界

**`allow_origin(Any)` 在生产环境是危险的。** 它等于宣布"任何网站都可以读我的响应"。对完全公开的只读 API(比如公开的天气数据)可以接受;只要接口涉及用户身份或私有数据,就必须用白名单。

**`allow_credentials(true)` 要格外谨慎。** 它允许跨源请求携带 cookie。开了它就意味着别的源可以用**用户已登录的身份**调你的接口——这正是前置知识里那个网银场景。规范禁止它和 `Any` 共存,`tower-http` 会 panic 提醒。真要用,必须配合严格的源白名单。

**CORS 不能替代认证和 CSRF 防护。** 前置知识强调过:CORS 拦的是"读取响应",不是"执行操作"。一个跨源 `POST /delete-account` 即使读不到响应,删除动作也已经发生了。防这类攻击要靠 CSRF token 或 `SameSite` cookie 属性。**把 CORS 当成安全边界是常见的误解。**

**别在服务端"手动"加 CORS 头部。** 常见的错误做法是在每个 handler 里手写 `Access-Control-Allow-Origin: *`。这样做会漏掉预检处理(`OPTIONS` 请求没人应答)、漏掉 `Vary: Origin`(缓存投毒)、也漏掉各种边界情况。用 `CorsLayer`。

**开发环境放宽、生产环境收紧,配置要能区分。** 常见做法是从环境变量读允许的源列表,本地开发时包含 `http://localhost:5173`(Vite 默认端口)之类,生产只留正式域名。

**预检有性能成本。** 每个非简单请求都多一次往返。用 `max_age` 让浏览器缓存预检结果能显著改善,但注意浏览器对这个值有上限(Chrome 是 2 小时),设再大也没用。

## 完整代码

`src/main.rs`:

```rust
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
```

依赖:

- `axum` 0.8.x
- `tokio` 1.x,feature `full`
- `tower-http` 0.6.x,feature `cors`

## 动手练习

**1. 暴露一个自定义响应头给 JS。**

提示:在 `json` handler 里返回一个带自定义头部的响应,比如 `([("x-total-count", "3")], Json(vec![...]))`([第 06 章](06-global-404-handler.md#元组作为返回类型)的元组返回)。然后在前端页面里 `console.log(response.headers.get('x-total-count'))`。你会发现是 `null`——因为默认只有几个基本响应头对 JS 可见。加上 `.expose_headers([HeaderName::from_static("x-total-count")])` 再试。这个坑在[第 19 章](19-request-id.md)那种想让前端读到 `x-request-id` 的场景下一定会遇到。

**2. 用闭包做动态源判断。**

提示:`allow_origin` 接受 `AllowOrigin::predicate(|origin, _parts| { ... })`,可以在运行时决定是否放行。实现一个"允许所有 `localhost` 的任意端口"的规则,方便本地开发。做完想想:这个规则能用在生产环境吗?

**3. 验证 `Any` 加 `allow_credentials` 会 panic。**

提示:把配置改成 `.allow_origin(tower_http::cors::Any).allow_credentials(true)`,然后 `cargo run`。程序会在启动时 panic 并给出解释。读一遍那条消息——`tower-http` 选择让你在启动时就崩,而不是运行时静默降级,这个设计取向值得体会。

## 小结与下一章预告

同源策略是浏览器的硬规则:一个页面的 JS 默认不能读取另一个源的响应。协议、主机、端口任一不同就是跨源,**`localhost` 和 `127.0.0.1` 也算不同源**。

**它拦的是"读取响应"而不是"发送请求"** —— 跨源请求通常已经到达服务器并被处理了,只是浏览器不把结果交给 JS。所以"curl 能通、浏览器报错"和"服务端日志里明明有这个请求"都不矛盾。这也意味着 **CORS 不能替代 CSRF 防护**:操作已经执行了,只是读不到响应。

CORS 让服务端用响应头声明放行。简单请求直接发,不满足条件(比如 `Content-Type: application/json`)时浏览器会**先发一个 `OPTIONS` 预检**。最常见的配置错误就是漏了 `allow_headers([CONTENT_TYPE])` —— 表现为 `GET` 正常但 POST JSON 失败,报错里带 `preflight` 字样。

`CorsLayer` 还会自动加上 `Vary: Origin`,避免不同源的响应被缓存串了。生产环境不要用 `Any`,尤其不能和 `allow_credentials(true)` 组合(规范禁止,`tower-http` 会 panic)。

「错误处理与工程化」阶段到这里只剩最后一块拼图。前面十几章的验证全靠 `cargo run` 加 curl 手工敲,而且每章都留了一句"测试写法见第 21 章"。[第 21 章 testing](21-testing.md) 把这笔债还清:`Router` 本身就是一个 tower `Service`([第 17 章](17-tracing-aka-logging.md#towerservice-与-layer)讲过),所以**不启动服务器就能直接对它发请求**。你会看到 `oneshot` 的用法、需要连续发多个请求时为什么要用 `ready().call()`、以及 `ConnectInfo` 这类依赖真实连接的提取器在测试里怎么模拟。
