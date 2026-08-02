# 第 43 章 reverse-proxy:改写请求再转发

## 本章你将做出什么

一个**反向代理**:客户端以为自己在跟 4000 端口的服务说话,实际上请求被原样转给了 3000 端口。

```
客户端 → 127.0.0.1:4000 (代理)  → 127.0.0.1:3000 (上游)
                               ← Hello, world!
       ← Hello, world!
```

两个服务跑在**同一个进程**里,方便你一条命令跑起来。

和[第 42 章](42-http-proxy.md)的正向代理正好相反:那一章目标地址由客户端给出、代理不解析内容;这一章**目标地址写死,但每个请求都要被解析、改写、再发出去**。

学完本章你将掌握:

- **反向代理的核心动作:改写请求的 URI**
- `hyper-util` 的底层 HTTP 客户端,以及它和 `reqwest` 的分工
- 为什么这个客户端叫 `legacy`
- `Client::<(), ()>::builder(...)` 这个奇怪的 turbofish 是怎么回事
- `tokio::spawn` 在一个进程里跑两个服务
- **`X-Forwarded-For` 这类头为什么是反向代理的必修课**
- 转发请求和响应时,哪些头**不能原样抄**

## 前置知识

### 反向代理在真实系统里做什么

[第 42 章](42-http-proxy.md#正向代理-vs-反向代理)那张表说过:**反向代理替服务端开门**,客户端不知道它的存在。它的用途几乎覆盖了所有生产部署:

| 用途 | 说明 |
|---|---|
| **负载均衡** | 一个入口后面挂 N 个后端实例 |
| **TLS 卸载** | 代理处理 HTTPS,后端只讲明文 HTTP([第 32 章](32-tls-rustls.md)提过) |
| **静态资源** | 静态文件代理直接返回,动态请求才转给后端([第 38 章](38-static-file-server.md)提过) |
| **压缩** | 代理统一压缩,后端不用管([第 39 章](39-compression.md)提过) |
| **路由 / API 网关** | `/api/user/*` 转给用户服务,`/api/order/*` 转给订单服务 |
| **限流、鉴权、WAF** | 在入口统一做 |
| **灰度发布** | 按比例把流量切给新版本 |

前面好几章的「代价与边界」都写过一句「**生产环境通常在代理层做**」——**这一章就是那个代理层。**

写一个玩具反向代理的价值不在于替代 Nginx,而在于**看清代理到底改了请求的什么**。之后再看 Nginx 的 `proxy_pass`、`proxy_set_header` 配置,你会知道每一行在干什么。

### 反向代理的核心动作:改写 URI

客户端发过来的请求长这样:

```
GET /users/1?full=true HTTP/1.1
Host: 127.0.0.1:4000
```

axum 解析后,`req.uri()` 是 **origin-form**:`/users/1?full=true`,**没有 scheme,也没有 authority**。

而 HTTP 客户端要发出一个请求,必须知道**连到哪台机器**。所以代理要把 URI 改成 **absolute-form**:

```
http://127.0.0.1:3000/users/1?full=true
└─scheme─┘└──authority──┘└──path_and_query──┘
```

**这就是反向代理最核心的一行代码**:

```rust
*req.uri_mut() = Uri::try_from(format!("http://127.0.0.1:3000{path_query}")).unwrap();
```

其余的一切——请求头、body、method、HTTP 版本——**原封不动地跟着走**。

### `path_and_query`:为什么不能只用 `path()`

```rust
let path = req.uri().path();                       // "/users/1"
let path_query = req.uri()
    .path_and_query()
    .map(|v| v.as_str())
    .unwrap_or(path);                              // "/users/1?full=true"
```

`path()` **会丢掉查询字符串**。`path_and_query()` 返回 `Option<&PathAndQuery>`,包含两者。

什么时候会是 `None`?URI 是 authority-form(CONNECT,[第 42 章](42-http-proxy.md#请求行里的请求目标有三种形式))或者 asterisk-form(`OPTIONS *`)的时候。这种情况下 fallback 到 `path()`(会得到 `"/"`)。

**忘了带 query 是新手写代理最常见的 bug**,而且很隐蔽:分页、筛选、搜索全都不工作,但请求本身是 200。

### hyper-util 的客户端:和 reqwest 的分工

[第 41 章](41-reqwest-response.md)用的是 `reqwest`,这一章换成 `hyper_util::client::legacy::Client`。为什么?

| | `reqwest` | `hyper-util` 的 `Client` |
|---|---|---|
| 抽象层次 | 高层,面向"我要请求某个 URL" | 底层,面向"我要发出这个 `http::Request`" |
| 输入 | `client.get(url).header(...).json(...)` | `client.request(req)` —— **直接吃 `http::Request`** |
| 自带 | TLS、重定向、cookie、gzip、超时 | **什么都不自带** |
| 适合 | 调第三方 API | **转发已有的请求** |

**代理场景下低层客户端更合适**,因为我们手里已经有一个完整的 `http::Request` 了——不需要把它拆成 URL + 头 + body 再让高层库重新拼回去。`reqwest` 也能做(`Client::execute`),但绕。

反过来,[第 41 章](41-reqwest-response.md)那种"我要去请求某个已知的 API"场景,`reqwest` 明显更顺手。**没有哪个更好,是场景不同。**

### 为什么叫 `legacy`

`hyper_util::client::legacy` 这个模块名让人不安,解释一下:

hyper 1.0 做了一次大清理,**把高层的连接池客户端从核心库里拿掉了**,只留下"在一条连接上收发 HTTP"的底层能力。原来的高层客户端被搬进 `hyper-util`,并**标记为 `legacy`**,意思是"这是从 0.14 搬过来的旧实现,将来会有更好的设计来替代它"。

**它是可以正常使用的,不是废弃的。** 只是 API 将来可能变。目前 Rust 生态里绝大多数需要底层 HTTP 客户端的项目都在用它。

### `Client::<(), ()>::builder(...)` 这个 turbofish

代码里会出现这一行,看着很怪:

```rust
let client: Client =
    hyper_util::client::legacy::Client::<(), ()>::builder(TokioExecutor::new())
        .build(HttpConnector::new());
```

`Client<C, B>` 有两个泛型参数(连接器类型、body 类型),而 `builder` 是它的**关联函数**。Rust 要求调用关联函数时类型参数是确定的,但 `builder` 本身**根本不用这两个参数**——真正决定它们的是后面的 `.build(HttpConnector::new())` 和你把请求塞进去时的 body 类型。

于是就有了这个写法:**先随便填两个 `()` 占位,让编译器闭嘴**,真实类型靠外层的 `let client: Client` 标注反推回来。

这是 `hyper-util` 当前 API 的一个已知瑕疵(和 `legacy` 这个名字一样,都是过渡期的产物)。**照抄就行,不用纠结。**

### `HttpConnector` 与 `TokioExecutor`

```rust
.build(HttpConnector::new())
```

`HttpConnector` 是**明文 TCP 连接器**——它只会 `TcpStream::connect`,**不支持 HTTPS**。上游必须是 `http://`。

要连 HTTPS 上游,得换成 `hyper-rustls` 或 `hyper-tls` 提供的连接器。这在真实部署里很常见(比如代理到一个云服务),「代价与边界」会给出具体写法。

```rust
Client::builder(TokioExecutor::new())
```

`TokioExecutor` 告诉 hyper "**要 spawn 后台任务时,用 `tokio::spawn`**"。hyper 本身不绑定运行时(和[第 42 章](42-http-proxy.md#步骤-1不用-axumserve自己写-accept-循环)的 `TokioIo` 同一个设计思路),所以需要你把执行器传进去。HTTP/2 的连接管理和连接池的后台清理都要用它。

### `Response<Incoming>` 直接就是 axum 的响应

转发的最后一步简单得出人意料:

```rust
Ok(client.request(req).await?.into_response())
```

`client.request()` 返回 `http::Response<hyper::body::Incoming>`,而 axum 为**所有** `http::Response<B>`(只要 `B: http_body::Body<Data = Bytes>`)实现了 `IntoResponse`。所以一个 `.into_response()` 就转好了。

**关键在于:这个转换是流式的。** `Incoming` 被包成 `axum::body::Body`,数据还在网络上一块一块地来。[第 41 章](41-reqwest-response.md#bytes-vs-bytes_stream本章的分歧点)费了一整章讲"别用 `.bytes()`",在这里**根本没有犯错的机会**——因为我们压根没去读 body。

**这是低层客户端的又一个好处:它不诱惑你去读 body。**

### `tokio::spawn`:一个进程跑两个服务

```rust
tokio::spawn(server());     // 上游,监听 3000
// ... 主流程继续,监听 4000
```

`tokio::spawn` 把一个 future 扔进运行时后台执行,立刻返回。这样一个 `main` 就能同时跑两个 `axum::serve`。

[第 44 章](44-serve-with-hyper.md)会看到另一种写法 `tokio::join!`,区别是 `join!` 等两个都完成,`spawn` 不等。**这里用 `spawn` 是因为上游服务是配角**,主流程要继续往下走。

 这只是**演示便利**。真实的反向代理和上游一定是**两个进程**(通常还在两台机器上)。

## 项目搭建

```bash
cargo new reverse-proxy
cd reverse-proxy
```

```bash
cargo add axum
cargo add hyper --features full
cargo add hyper-util --features client-legacy
cargo add tokio --features full
```

`client-legacy` 就是上面说的那个高层客户端,不加这个 feature 就没有 `hyper_util::client`。

**注意本章没有 `tracing`。** 官方示例用的是 `println!`——「代价与边界」会说这为什么是个问题。

## 分步构建

### 步骤 1:先把上游服务跑起来

**要做什么**:先写那个被代理的服务,确认它自己是通的。代理还没影子的时候,先保证目标存在。

```rust
// src/main.rs
use axum::{routing::get, Router};

#[tokio::main]
async fn main() {
    server().await;
}

async fn server() {
    let app = Router::new().route("/", get(|| async { "Hello, world!" }));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

**验证**:

```bash
cargo run
```

```bash
curl -i http://127.0.0.1:3000/
```

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 13

Hello, world!
```

**把这个响应记下来**,一会儿要和经过代理的响应逐字对比。

### 步骤 2:代理骨架 + 后台跑上游

**要做什么**:让 `main` 在后台 spawn 上游服务,自己去监听 4000 端口。

```rust
use axum::{extract::Request, response::Response, routing::get, Router};
use hyper::StatusCode;
```

```rust
#[tokio::main]
async fn main() {
    tokio::spawn(server());

    let app = Router::new().route("/", get(handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:4000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler(req: Request) -> Result<Response, StatusCode> {
    // 先占位
    let _ = req;
    Err(StatusCode::NOT_IMPLEMENTED)
}
```

**`handler(req: Request)`** —— 这是本教程第一次让 handler 直接吃**整个 `Request`**,而不是某个提取器。[第 40 章](40-stream-to-file.md)的裸 body 上传接近,但那里还是配了 `Path`。

原因很直白:**代理要转发整个请求,所以要拿到整个请求。** `Request` 作为提取器必须放在**参数列表的最后**(它消费掉了整个请求,后面的提取器就没东西可提了)——[第 09 章](09-consume-body-in-extractor-or-middleware.md)讲过这个规则。

**验证**:

```bash
cargo run
```

```
listening on 127.0.0.1:4000
listening on 127.0.0.1:3000
```

(两行的顺序可能颠倒,取决于调度。)

```bash
curl -i http://127.0.0.1:4000/     # 501 Not Implemented
curl -i http://127.0.0.1:3000/     # Hello, world!
```

两个端口都活着,骨架搭好了。

### 步骤 3:造一个客户端放进共享状态

**要做什么**:建 `hyper-util` 的客户端,用 `with_state` 共享。

```rust
use axum::extract::State;
use hyper_util::{client::legacy::connect::HttpConnector, rt::TokioExecutor};

type Client = hyper_util::client::legacy::Client<HttpConnector, axum::body::Body>;
```

```rust
    let client: Client =
        hyper_util::client::legacy::Client::<(), ()>::builder(TokioExecutor::new())
            .build(HttpConnector::new());

    let app = Router::new().route("/", get(handler)).with_state(client);
```

```rust
async fn handler(State(client): State<Client>, req: Request) -> Result<Response, StatusCode> {
    let _ = (client, req);
    Err(StatusCode::NOT_IMPLEMENTED)
}
```

**那个 `type Client = ...` 别名很有必要。** 完整类型是 `hyper_util::client::legacy::Client<HttpConnector, axum::body::Body>`,要在 `State<...>`、handler 签名、`let` 标注里各写一遍——不起别名会非常难读。

**`Client` 和[第 41 章](41-reqwest-response.md)的 `reqwest::Client` 一样,内部是 `Arc`,克隆便宜,直接当 state 用**,不用额外包 `Arc<...>`。

**`State` 必须排在 `Request` 前面。** 上一步说过,`Request` 只能在最后。

### 步骤 4:改写 URI,发出去

**要做什么**:实现真正的转发。

```rust
use axum::http::uri::Uri;
use axum::response::IntoResponse;
```

```rust
async fn handler(State(client): State<Client>, mut req: Request) -> Result<Response, StatusCode> {
    let path = req.uri().path();
    let path_query = req
        .uri()
        .path_and_query()
        .map(|v| v.as_str())
        .unwrap_or(path);

    let uri = format!("http://127.0.0.1:3000{path_query}");

    *req.uri_mut() = Uri::try_from(uri).unwrap();

    Ok(client
        .request(req)
        .await
        .map_err(|_| StatusCode::BAD_REQUEST)?
        .into_response())
}
```

**整个反向代理就这么多。** 逐块看:

**① `mut req`** —— 要改 URI,所以提取出来的 `Request` 得是可变的。提取器绑定加 `mut` 是普通的 Rust 语法,不是 axum 的特殊规则。

**② 拼 absolute-form URI**

```rust
let uri = format!("http://127.0.0.1:3000{path_query}");
```

scheme + authority 写死,path 和 query 从原请求抄。

**③ `*req.uri_mut() = ...`**

`uri_mut()` 返回 `&mut Uri`,解引用赋值就替换掉了。**除了 URI,请求的其他部分一个字节没动**——method、headers、body、HTTP 版本全部原样。

**④ `client.request(req).await`**

把改好的请求发出去。这里的 `req` 是 `Request<axum::body::Body>`,正好对上 `type Client` 里的第二个泛型参数。**body 是流式转发的**,大文件上传经过这个代理不会撑爆内存。

**⑤ `.into_response()`**

前置知识讲过:`Response<Incoming>` 直接就能变成 axum 的响应,而且是流式的。

**验证**:

```bash
cargo run
```

```bash
curl -i http://127.0.0.1:4000/
```

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 13
date: ...

Hello, world!
```

**和步骤 1 直连 3000 的响应逐字一致。** 这就是反向代理的目标——**客户端完全不知道中间有一层**。

再确认一下确实转发了,而不是 4000 自己在返回:把上游那行改成 `"Hello from upstream!"`,重新运行,`curl http://127.0.0.1:4000/` 应该跟着变。**改完记得改回来。**

### 步骤 5:故意改坏,看看不改 URI 会怎样

**要做什么**:注释掉 URI 改写那一行,理解它为什么不可省略。

```rust
    // *req.uri_mut() = Uri::try_from(uri).unwrap();
```

(编译器会警告 `uri` 和 `path_query` 没用到,忽略即可。)

重新运行:

```bash
curl -i http://127.0.0.1:4000/
```

**现象**:返回 `400 Bad Request`。

服务端**什么日志都没有**——因为 `.map_err(|_| StatusCode::BAD_REQUEST)` 把错误**整个丢掉了**。

为什么会失败?因为传给客户端的 `req` 的 URI 还是 origin-form 的 `/`,**没有 scheme 也没有 authority**,hyper 的客户端不知道该连哪台机器,于是报错(错误信息大意是 `client requires absolute-form URIs`)。

**这一步同时暴露了两个问题:**

1. **改写 URI 是转发的必要步骤**,不是可选优化
2. **`map_err(|_| ...)` 把诊断信息全扔了** —— 一个真实的排障场景里,你看到 400 却完全不知道为什么

第 2 点比第 1 点更值得警惕。改成这样,现象立刻清楚:

```rust
        .map_err(|err| {
            eprintln!("proxy error: {err}");
            StatusCode::BAD_GATEWAY
        })?
```

**记得把 URI 改写那行放回去。**

> **更好的写法**:和[第 41 章](41-reqwest-response.md#步骤-3转发响应)完全一样的问题——**上游连不上应该是 `502 Bad Gateway`,不是 `400 Bad Request`**。400 会让排障的人去查调用方,而问题在上游。**丢弃错误(`|_|`)比用错状态码更糟**,至少要记一条日志。

> **更好的写法**:`Uri::try_from(uri).unwrap()` 里的 `unwrap()` 会在 URI 拼不出来时**让 handler panic**。虽然 `path_and_query` 来自一个已经解析成功的 URI,拼接后通常仍然合法,但**在处理外部输入的代码里 unwrap 始终是个隐患**。稳妥写法是 `.map_err(|_| StatusCode::BAD_REQUEST)?` —— 注意**这一处**用 400 才是对的,因为确实是客户端给的路径有问题。

### 步骤 6:一个必须知道的边界——只有 `/` 被代理

**要做什么**:试试代理一个别的路径。

```bash
curl -i http://127.0.0.1:4000/anything
```

**结果是 404**,而且是**代理自己返回的 404**,请求根本没到上游。

原因很简单:

```rust
let app = Router::new().route("/", get(handler)).with_state(client);
```

**只注册了 `/` 这一条路由,而且只接受 `GET`。** `POST /`、`GET /foo` 全都进不来。

**这不是 bug,是示例为了聚焦核心逻辑做的简化。** 但一个真正的反向代理必须接管**所有路径、所有方法**。改法是把路由换成兜底:

```rust
let app = Router::new().fallback(handler).with_state(client);
```

`fallback`([第 06 章](06-global-404-handler.md))对**任何没匹配上的请求**生效,而这里一条路由都没注册,所以就是**全部请求**。而且它不限制 method。

改完再试:

```bash
curl -i http://127.0.0.1:4000/anything
```

现在会看到**上游的 404**(`content-length: 0`,没有 axum 默认的错误正文差异)。虽然还是 404,但**这次是转发过去之后上游给的**——给上游加一条 `/anything` 路由就能验证。

**记住这个区别:代理自己 404,和转发后上游 404,是两件完全不同的事。** 排查线上问题时先分清是哪一种,能省很多时间。

### 步骤 7:看看上游到底收到了什么

**要做什么**:临时改一下上游的 handler,把收到的请求头打出来。这一步的目的是引出反向代理最重要的一个话题。

**临时**把 `server()` 里的路由换成:

```rust
async fn server() {
    let app = Router::new().route(
        "/",
        get(|req: Request| async move {
            println!("upstream got: {} {}", req.method(), req.uri());
            for (name, value) in req.headers() {
                println!("  {name}: {value:?}");
            }
            "Hello, world!"
        }),
    );
    // ... 其余不变
}
```

```bash
cargo run
curl -H "X-Custom: hi" http://127.0.0.1:4000/
```

上游打印出来的大致是:

```
upstream got: GET /
  "host": "127.0.0.1:4000"
  "user-agent": "curl/8.x"
  "accept": "*/*"
  "x-custom": "hi"
```

**三个观察:**

**① 自定义头 `x-custom` 原样过来了。** 转发是完整的,不只是 method 和 path。

**② `host` 还是 `127.0.0.1:4000`** —— 也就是**代理的地址,不是上游的地址**。hyper 的客户端只在请求没有 `Host` 头时才根据 URI 补一个,这里已经有了就不动。

这个行为对不对?**看场景。** 很多后端会用 `Host` 做虚拟主机路由或者生成绝对 URL,这时候保留原始 `Host` 正是你要的(Nginx 里对应 `proxy_set_header Host $host;`)。但如果上游按 `Host` 做证书或路由校验,就得改成上游的地址。**这是个需要显式决策的点,而这个示例是默认行为、没做决策。**

**③ 完全没有 `X-Forwarded-For`。**

这是本步骤真正想说的问题:**上游不知道真实客户端是谁。** 它看到的对端地址是代理(`127.0.0.1`),`ConnectInfo`([第 44 章](44-serve-with-hyper.md)会讲)拿到的也是代理的地址。

于是上游没法:按客户端 IP 限流、记录访问来源、做地理位置判断、拉黑恶意 IP——**全都失效了**,因为所有请求看起来都来自同一个 IP。

行业惯例是代理**主动加上这几个头**:

| 头 | 内容 |
|---|---|
| `X-Forwarded-For` | 真实客户端 IP(经过多层代理时是逗号分隔的列表) |
| `X-Forwarded-Proto` | 客户端用的是 `http` 还是 `https` |
| `X-Forwarded-Host` | 客户端请求的原始 Host |
| `Forwarded` | RFC 7239 定义的标准头,把上面几个合并 |

**这个示例一个都没加。** 「动手练习」会补上。

 反过来说,**上游信任这些头也是有条件的**:如果上游能被直接访问,任何人都能伪造 `X-Forwarded-For` 来绕过 IP 限流。所以规矩是:**上游只能信任来自可信代理的转发头**,并且代理必须**覆盖(而不是追加)**客户端自己带来的这些头。

**验证完把 `server()` 改回原样。**

## 完整代码

`src/main.rs`:

```rust
use axum::{
    body::Body,
    extract::{Request, State},
    http::uri::Uri,
    response::{IntoResponse, Response},
    routing::get,
    Router,
};
use hyper::StatusCode;
use hyper_util::{client::legacy::connect::HttpConnector, rt::TokioExecutor};

type Client = hyper_util::client::legacy::Client<HttpConnector, Body>;

#[tokio::main]
async fn main() {
    tokio::spawn(server());

    let client: Client =
        hyper_util::client::legacy::Client::<(), ()>::builder(TokioExecutor::new())
            .build(HttpConnector::new());

    let app = Router::new().route("/", get(handler)).with_state(client);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:4000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler(State(client): State<Client>, mut req: Request) -> Result<Response, StatusCode> {
    let path = req.uri().path();
    let path_query = req
        .uri()
        .path_and_query()
        .map(|v| v.as_str())
        .unwrap_or(path);

    let uri = format!("http://127.0.0.1:3000{path_query}");

    *req.uri_mut() = Uri::try_from(uri).unwrap();

    Ok(client
        .request(req)
        .await
        .map_err(|_| StatusCode::BAD_REQUEST)?
        .into_response())
}

async fn server() {
    let app = Router::new().route("/", get(|| async { "Hello, world!" }));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

## 代价与边界

**只代理 `GET /` 一条路由。** 步骤 6 说了,改成 `fallback` 即可。

**上游地址硬编码。** `"http://127.0.0.1:3000"` 写死在 handler 里。生产环境要做成配置,而且通常不止一个上游——就引出下一条。

**没有负载均衡,没有健康检查。** 真实反向代理后面挂的是一组上游,需要:轮询/最少连接/一致性哈希等策略、周期性健康检查、失败节点摘除、恢复后自动加回。tower 的 `tower::balance` 和 `tower::discover` 提供了这套抽象。

**没有 `X-Forwarded-*` 头。** 步骤 7 详细说了。这是**反向代理最常被遗漏的一件事**。

**逐跳头没有过滤。** 和[第 41 章](41-reqwest-response.md#代价与边界)一样的问题,而且这次是**两个方向**都有:

- **请求方向**:客户端的 `Connection`、`Upgrade`、`TE`、`Proxy-Authorization` 会被原样转给上游
- **响应方向**:上游的 `Connection`、`Transfer-Encoding` 会被原样转给客户端

RFC 9110 规定逐跳头只对单个连接有效,代理应该**消费掉而不是转发**。实践中 hyper 会在写出时修正一部分,但不该依赖这个。

**没有超时。** `client.request(req).await` 会一直等。上游卡死,代理的连接和内存跟着一起卡死。`hyper-util` 的客户端**自己不提供请求超时**(这也是它"低层"的体现),要用 `tokio::time::timeout` 包一层:

```rust
let resp = tokio::time::timeout(Duration::from_secs(30), client.request(req))
    .await
    .map_err(|_| StatusCode::GATEWAY_TIMEOUT)?
    .map_err(|_| StatusCode::BAD_GATEWAY)?;
```

注意 `timeout` 只覆盖到**拿到响应头**为止,body 的流式传输不在里面——想限制整体时长要另外处理。

**没有重试、没有熔断、没有并发限制。** 和[第 41 章](41-reqwest-response.md#代价与边界)列的一样。

**上游只能是明文 HTTP。** `HttpConnector` 不支持 TLS。要连 HTTPS 上游:

```bash
cargo add hyper-rustls --features http1,ring
```

```rust
let https = hyper_rustls::HttpsConnectorBuilder::new()
    .with_native_roots()?
    .https_or_http()
    .enable_http1()
    .build();
let client = Client::builder(TokioExecutor::new()).build(https);
```

底层库和[第 32 章](32-tls-rustls.md)的服务端 TLS 是同一个 `rustls`,只是这次在客户端一侧。

**WebSocket 不会被代理。** WebSocket 需要转发 `Upgrade` 握手**并且**接管升级后的连接做双向搬运——本质上就是[第 42 章](42-http-proxy.md)的隧道逻辑。这个示例只转发普通 HTTP 请求,`Upgrade` 头虽然被抄过去了,但升级后的字节流没人搬,连接会断。

**错误处理有两个问题。** 步骤 5 说了:状态码用了 400 而不是 502,以及 `map_err(|_| ...)` 把错误信息全丢了。

**用 `println!` 而不是 `tracing`。** 这个示例连 `tracing` 依赖都没加。生产代理**必须有结构化访问日志**:请求 ID([第 19 章](19-request-id.md))、上游地址、耗时、状态码、字节数,一样都不能少——代理是排障的第一现场。

**两个服务在一个进程里。** 演示便利,别当真。

**没有优雅停机。** [第 45 章](45-graceful-shutdown.md)会讲。代理停机时应该先停止接受新连接、等在途请求完成,而不是直接砍断。

## 动手练习

**1. 让代理接管所有路径和方法。**

提示:`Router::new().fallback(handler).with_state(client)`。给上游加几条路由(`POST /echo`、`GET /users/{id}`),确认都能透过代理访问。**顺带验证 query 有没有丢**:上游返回 `req.uri().to_string()`,然后 `curl "http://127.0.0.1:4000/users/1?full=true"`,看返回的 URI 里 `?full=true` 在不在。

**2. 加上 `X-Forwarded-For` 和 `X-Forwarded-Proto`。**

提示:需要客户端 IP,所以要用 `ConnectInfo<SocketAddr>` 提取器,并且 `axum::serve` 要写成 `serve(listener, app.into_make_service_with_connect_info::<SocketAddr>())`([第 44 章](44-serve-with-hyper.md)会详细讲)。然后:

```rust
req.headers_mut().insert("x-forwarded-for", client_ip.to_string().parse().unwrap());
req.headers_mut().insert("x-forwarded-proto", "http".parse().unwrap());
```

**用 `insert` 而不是 `append`** —— 步骤 7 结尾说的,要覆盖客户端伪造的值。做完在上游把这两个头打出来验证。

**3. 把错误处理修好。**

提示:上游失败改成 502 并打日志;`Uri::try_from` 的 `unwrap()` 改成返回 400。验证:把上游端口改成一个没人监听的(比如 3999),`curl -i http://127.0.0.1:4000/` 应该看到 502,而且控制台有错误详情。

**4. 加请求超时。**

提示:用「代价与边界」里给的 `tokio::time::timeout` 写法。造一个慢上游:给 `server()` 加一条路由 `sleep 10 秒`再返回。确认代理在超时后返回 504,而不是陪着一起等。

**5. 过滤逐跳头。**

提示:转发前从 `req.headers_mut()` 里 `remove` 掉 `connection`、`keep-alive`、`te`、`trailer`、`transfer-encoding`、`upgrade`、`proxy-authenticate`、`proxy-authorization`;响应方向同理。[第 41 章](41-reqwest-response.md)的练习 3 做过响应方向,这次两个方向都做。

**6.(进阶)加一个第二上游,做轮询负载均衡。**

提示:再 spawn 一个监听 3001 的服务,用一个 `AtomicUsize` 计数器决定这次转给谁。验证:连续 curl 若干次,两个上游的日志应该交替出现。**做完你会发现真正的难点不是轮询,而是"某个上游挂了怎么办"** —— 这就是健康检查存在的理由。

## 小结与下一章预告

反向代理的核心动作是**改写请求的 URI**:客户端发来的是 origin-form(`/users/1?full=true`),HTTP 客户端要的是 absolute-form(`http://上游{path_query}`)。一行 `*req.uri_mut() = ...` 就是全部,**请求的其他部分原封不动**。

`path_and_query()` 而不是 `path()` —— 后者会**丢掉查询字符串**,是写代理最容易踩的坑。

客户端选了 `hyper-util` 的 `legacy::Client` 而不是 `reqwest`,因为它**直接吃 `http::Request`**,正好对上我们手里的东西;而且返回的 `Response<Incoming>` 一个 `.into_response()` 就成了 axum 的响应,**天然流式**——[第 41 章](41-reqwest-response.md)费一整章讲的"别缓冲",在这里没有犯错的余地。`legacy` 只是 hyper 1.0 拆分后的过渡命名,不是废弃。

步骤 6 和步骤 7 揭示了这个示例的两个重要边界:**只代理了 `GET /` 一条路由**(改成 `fallback` 即可),以及**没有加 `X-Forwarded-*` 头**,导致上游完全不知道真实客户端是谁——IP 限流、访问日志、地理判断全部失效。这是反向代理最常被遗漏的一件事。

其余的差距在「代价与边界」里:没有负载均衡和健康检查、没有超时重试熔断、逐跳头没过滤、上游只能是明文 HTTP、WebSocket 不会被代理、错误处理丢信息、日志用 `println!`。**这些正是 Nginx 和 Envoy 存在的理由**——但走过这一遍,你会知道它们的每条配置在解决什么。

[第 44 章 serve-with-hyper](44-serve-with-hyper.md) 是「内容分发与代理」这一阶段的收尾,也是一次揭底:**把 `axum::serve` 彻底拆开**。[第 42 章](42-http-proxy.md)已经手写过一次 accept 循环,下一章会讲透它——`auto::Builder` 怎么同时支持 HTTP/1 和 HTTP/2、`serve_connection_with_upgrades` 到底在做什么,以及本章练习 2 需要的 **`ConnectInfo`:客户端地址是怎么一路传到 handler 里的**。
