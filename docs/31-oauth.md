# 第 31 章 oauth:把认证委托给第三方

## 本章你将做出什么

一个"用 Discord 账号登录"的服务:

```
GET /              首页,显示登录状态
GET /auth/discord  跳转到 Discord 授权页
GET /auth/authorized  Discord 回调,换 token 并建立会话
GET /protected     需要登录才能访问
GET /logout        销毁会话
```

和[第 30 章](30-jwt.md)自己签发 token 不同,这一章**把"验证用户身份"这件事整个交给 Discord**——你的服务永远不接触用户密码。

而且这一章的会话是**有状态的**(服务端存 session + cookie),正好和上一章的无状态 JWT 形成对照。

学完本章你将掌握:

- OAuth 授权码流程的四次跳转,以及每一步在防什么
- **`state` 参数如何防 CSRF**,为什么它必须和 session 绑定
- cookie 会话:`HttpOnly`、`Secure`、`SameSite` 三个属性各防什么
- `OptionalFromRequestParts`:让提取器变成可选的
- 复合状态下 `FromRef` 的手写实现

## 前置知识

### OAuth 解决什么问题

设想你要做一个"用 GitHub 账号登录"的功能。最朴素的想法是让用户把 GitHub 密码填进你的表单,你拿去验证——**这显然荒谬**:用户凭什么信任你不保存密码?你又凭什么承担保管别人密码的责任?

**OAuth 2.0 的核心思想是:用户在授权方(Discord/GitHub)那里登录,授权方给你一张凭证,你拿凭证去换用户信息。全程你的服务不接触密码。**

三个角色:

- **资源所有者**(用户)—— 拥有账号的人
- **客户端**(你的服务)—— 想获取用户信息的一方
- **授权服务器**(Discord)—— 验证用户身份、签发凭证的一方

**对你的服务来说,最大的好处是不用管密码:** 不用存密码哈希、不用做找回密码、不用防撞库、密码泄露的责任不在你。代价是依赖第三方——Discord 挂了你的登录就挂了。

### 授权码流程:四次跳转

OAuth 有多种流程,Web 应用用的是**授权码流程**(Authorization Code Flow)。本章五条路由就是它的完整实现,拆开看:

**第 1 步:用户点"用 Discord 登录",你把他重定向到 Discord。**

```
GET /auth/discord
→ 302 到 https://discord.com/api/oauth2/authorize
     ?client_id=你的ID
     &redirect_uri=http://127.0.0.1:3000/auth/authorized
     &scope=identify
     &state=随机字符串
     &response_type=code
```

`scope` 声明你要什么权限(`identify` = 读基本信息)。`state` 是防 CSRF 的随机值,下面详说。

**第 2 步:用户在 Discord 页面上确认授权。** 这一步完全在 Discord 域内发生,用户输密码、看到"某某应用想访问你的基本信息",点同意。

**第 3 步:Discord 把用户重定向回你的服务,带上授权码。**

```
GET /auth/authorized?code=临时授权码&state=刚才那个随机字符串
```

**第 4 步:你用授权码去换 access token,再用 token 拿用户信息。**

```
POST https://discord.com/api/oauth2/token   (带 code + client_secret)
→ { "access_token": "..." }

GET https://discordapp.com/api/users/@me    (带 Bearer token)
→ { "id": "...", "username": "..." }
```

**为什么要"授权码换 token"这么绕,不直接给 token?**

因为第 3 步是通过**浏览器重定向**完成的——URL 会出现在浏览器历史、可能被 referer 泄露、可能被日志记录。授权码是**一次性的、短时效的**(通常 10 分钟),即使泄露危害也有限。

而第 4 步是**你的服务端直接向 Discord 发请求**,不经过浏览器,而且要带上 `client_secret`——**只有你知道这个密钥**。所以真正的 access token 从来不会出现在浏览器里。

### state 参数与 CSRF

**这是 OAuth 里最容易被忽略、也最容易实现错的安全机制。**

**攻击场景:** 攻击者先自己走一遍 OAuth 流程,拿到一个属于**他自己账号**的授权码,但不用。然后诱导受害者访问:

```
http://你的服务/auth/authorized?code=攻击者的授权码
```

如果你的服务不校验来源,就会用这个码去换 token、拿到**攻击者的账号信息**、然后给受害者的浏览器种上"你已登录为攻击者"的 cookie。

之后受害者在你的服务里做的一切(上传文件、绑定支付方式、写私密笔记)**都进了攻击者的账号**。这叫**登录 CSRF**。

**`state` 参数就是防这个的:**

1. 发起授权时生成一个随机值,存进**当前浏览器的 session**,同时放进跳转 URL 的 `state` 参数
2. Discord 回调时会原样带回这个 `state`
3. 你比对"回调带回的 state"和"session 里存的 state",**不一致就拒绝**

攻击者拿不到受害者浏览器的 session,所以伪造的回调 URL 里的 `state` 对不上。

**关键在于 `state` 必须和 session 绑定**,而不是随便存在服务端某个全局集合里——否则攻击者自己发起一次授权就能拿到一个"有效"的 state。本章的实现正是把它存进了 session cookie 对应的 session。

### 有状态会话 vs 无状态 JWT

本章和[第 30 章](30-jwt.md)是两种认证方案的完整对照。

**第 30 章(JWT):** 服务端不存任何东西,用户身份编码在 token 里,靠签名验真伪。

**本章(session):** 服务端存一张表 `session_id → 用户数据`,客户端只拿一个 `session_id`(放在 cookie 里)。

| | JWT(第 30 章) | Session(本章) |
|---|---|---|
| 服务端存储 | 无 | **需要**(内存/Redis/数据库) |
| 客户端拿到的 | 完整身份信息 | 只是一个 ID |
| 能否立即登出 | **不能** | **能**(删掉 session) |
| 水平扩展 | 天然支持 | 需要共享存储 |
| 数据能否被客户端读取 | **能**(base64) | 不能 |
| 每次请求的开销 | 验签(CPU) | 查存储(IO) |
| 撤销/改权限 | 难 | 容易 |

**本章的 `/logout` 能真正生效**,就是因为它是有状态的:

```rust
store.destroy_session(session).await?;
```

删掉服务端记录,那个 session cookie 立刻作废。[第 30 章](30-jwt.md#无状态的代价token-撤销不了)那个"token 撤销不了"的问题在这里不存在。

**这也解释了为什么 OAuth 示例选 session 而不是 JWT:** OAuth 登录之后通常需要能登出、能踢下线、能随权限变化即时生效——这些正是 session 的强项。

### cookie 的三个安全属性

本章设置 cookie 时带了三个属性:

```rust
format!("{COOKIE_NAME}={cookie}; SameSite=Lax; HttpOnly; Secure; Path=/")
```

**`HttpOnly`** —— JavaScript 读不到这个 cookie(`document.cookie` 看不见)。这样即使页面有 XSS 漏洞,攻击者的脚本也偷不走 session ID。**存 session 的 cookie 必须加这个。**

**`Secure`** —— 只在 HTTPS 连接上发送。防止中间人在明文 HTTP 里截获 cookie。

**注意 `Secure` 会让本章示例在 `http://127.0.0.1:3000` 下失效** —— 浏览器不会在明文连接上发送带 `Secure` 的 cookie。「代价与边界」会说这个问题。

**`SameSite=Lax`** —— 跨站请求不带这个 cookie(除了顶层导航的 GET)。这是**防 CSRF 的浏览器级机制**,和[第 20 章](20-cors.md#一个关键区分请求发出去了吗)提到的 CSRF 是同一个问题。

三个取值:

- `Strict` —— 任何跨站请求都不带,最安全但体验差(从外站点链接进来会显示未登录)
- `Lax` —— 顶层导航的 GET 请求会带,其它不带。**默认值,平衡了安全和体验**
- `None` —— 总是带,必须配合 `Secure`。用于确实需要跨站的场景

**`Path=/`** 让 cookie 对整站生效。

### OptionalFromRequestParts

本章的首页要处理两种情况:登录了显示用户名,没登录显示提示。

```rust
async fn index(user: Option<User>) -> impl IntoResponse {
    match user {
        Some(u) => format!("Hey {}! You're logged in!", u.username),
        None => "You're not logged in.".to_string(),
    }
}
```

**参数是 `Option<User>`。** 这需要给 `User` 实现 `OptionalFromRequestParts`:

```rust
impl<S> OptionalFromRequestParts<S> for User {
    type Rejection = Infallible;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Option<Self>, Self::Rejection> {
        match <User as FromRequestParts<S>>::from_request_parts(parts, state).await {
            Ok(res) => Ok(Some(res)),
            Err(AuthRedirect) => Ok(None),
        }
    }
}
```

它复用了必需版本的实现,只是把失败转成 `None` 而不是拒绝。

**`Rejection = Infallible`** 表示"永不失败"——要么有用户,要么没有,不存在错误情况。

**同一个类型同时实现两个 trait 的价值:** `User` 表示"必须登录",`Option<User>` 表示"登录与否都行"。**需求直接写在签名里**,和[第 30 章](30-jwt.md#提取器即认证守卫)那个模式一脉相承。

### 复合状态与手写 FromRef

[第 24 章](24-sqlx-postgres.md#fromref提取器只依赖状态的一部分)讲过 `FromRef` 让提取器只依赖状态的一部分。本章的状态有两个字段:

```rust
#[derive(Clone)]
struct AppState {
    store: MemoryStore,
    oauth_client: BasicClient,
}

impl FromRef<AppState> for MemoryStore {
    fn from_ref(state: &AppState) -> Self { state.store.clone() }
}

impl FromRef<AppState> for BasicClient {
    fn from_ref(state: &AppState) -> Self { state.oauth_client.clone() }
}
```

有了这两个实现,handler 就能各取所需:

```rust
async fn discord_auth(
    State(client): State<BasicClient>,
    State(store): State<MemoryStore>,
) -> ...
```

**同一个 handler 里出现两个 `State`,各自提取不同的部分。**

这两个 impl 其实可以用 `#[derive(FromRef)]` 自动生成([第 24 章](24-sqlx-postgres.md#fromref提取器只依赖状态的一部分)提过),官方示例选择手写是为了直观展示发生了什么。

## 项目搭建

### 注册 Discord 应用

本章需要真实的 OAuth 应用凭据:

1. 访问 <https://discord.com/developers/applications>,登录后点 **New Application**
2. 进入 **OAuth2** 标签页,记下 **CLIENT ID** 和 **CLIENT SECRET**
3. 在 **Redirects** 里添加 `http://127.0.0.1:3000/auth/authorized`,保存

**回调地址必须完全一致**,包括协议、端口、路径——Discord 会严格比对,差一个字符就拒绝。

### 创建项目

```bash
cargo new oauth
cd oauth
```

```bash
cargo add anyhow
cargo add async-session
cargo add axum
cargo add axum-extra --features typed-header
cargo add http
cargo add oauth2
cargo add reqwest --no-default-features --features rustls-tls,json
cargo add serde --features derive
cargo add tokio --features full
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`reqwest` 用 `rustls-tls` 而非默认的 `native-tls`,避免依赖系统的 OpenSSL——和[第 27 章](27-diesel-async-postgres.md#diesel-async-改变了什么)那个 libpq 问题同理,官方注释也写着 "makes it easier to cross-compile on CI"。

### 环境变量

```bash
export CLIENT_ID=你的ClientID
export CLIENT_SECRET=你的ClientSecret
```

## 分步构建

### 步骤 1:状态、OAuth 客户端与错误类型

**要做什么**:搭出骨架——应用状态、OAuth 客户端配置、错误处理。

```rust
// src/main.rs
use anyhow::{Context, Result};
use async_session::MemoryStore;
use axum::{
    extract::FromRef,
    response::{IntoResponse, Response},
    routing::get,
    Router,
};
use http::StatusCode;
use oauth2::{
    AuthUrl, ClientId, ClientSecret, EndpointNotSet, EndpointSet, RedirectUrl, TokenUrl,
};
use std::env;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

static COOKIE_NAME: &str = "SESSION";
static CSRF_TOKEN: &str = "csrf_token";

type BasicClient = oauth2::basic::BasicClient<
    EndpointSet,
    EndpointNotSet,
    EndpointNotSet,
    EndpointNotSet,
    EndpointSet,
>;

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // `MemoryStore` is just used as an example. Don't use this in production.
    let store = MemoryStore::new();
    let oauth_client = oauth_client().unwrap();
    let app_state = AppState {
        store,
        oauth_client,
    };

    let app = Router::new()
        .route("/", get(|| async { "not implemented yet" }))
        .with_state(app_state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .context("failed to bind TcpListener")
        .unwrap();

    tracing::debug!(
        "listening on {}",
        listener
            .local_addr()
            .context("failed to return local address")
            .unwrap()
    );

    axum::serve(listener, app).await.unwrap();
}

#[derive(Clone)]
struct AppState {
    store: MemoryStore,
    oauth_client: BasicClient,
}

impl FromRef<AppState> for MemoryStore {
    fn from_ref(state: &AppState) -> Self {
        state.store.clone()
    }
}

impl FromRef<AppState> for BasicClient {
    fn from_ref(state: &AppState) -> Self {
        state.oauth_client.clone()
    }
}

fn oauth_client() -> Result<BasicClient, AppError> {
    // Environment variables (* = required):
    // *"CLIENT_ID"     "REPLACE_ME";
    // *"CLIENT_SECRET" "REPLACE_ME";
    //  "REDIRECT_URL"  "http://127.0.0.1:3000/auth/authorized";
    //  "AUTH_URL"      "https://discord.com/api/oauth2/authorize?response_type=code";
    //  "TOKEN_URL"     "https://discord.com/api/oauth2/token";

    let client_id = env::var("CLIENT_ID").context("Missing CLIENT_ID!")?;
    let client_secret = env::var("CLIENT_SECRET").context("Missing CLIENT_SECRET!")?;
    let redirect_url = env::var("REDIRECT_URL")
        .unwrap_or_else(|_| "http://127.0.0.1:3000/auth/authorized".to_string());

    let auth_url = env::var("AUTH_URL").unwrap_or_else(|_| {
        "https://discord.com/api/oauth2/authorize?response_type=code".to_string()
    });

    let token_url = env::var("TOKEN_URL")
        .unwrap_or_else(|_| "https://discord.com/api/oauth2/token".to_string());

    Ok(oauth2::basic::BasicClient::new(ClientId::new(client_id))
        .set_client_secret(ClientSecret::new(client_secret))
        .set_auth_uri(
            AuthUrl::new(auth_url).context("failed to create new authorization server URL")?,
        )
        .set_token_uri(TokenUrl::new(token_url).context("failed to create new token endpoint URL")?)
        .set_redirect_uri(
            RedirectUrl::new(redirect_url).context("failed to create new redirection URL")?,
        ))
}

// Use anyhow, define error and enable '?'
#[derive(Debug)]
struct AppError(anyhow::Error);

// Tell axum how to convert `AppError` into a response.
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        tracing::error!("Application error: {:#}", self.0);

        (StatusCode::INTERNAL_SERVER_ERROR, "Something went wrong").into_response()
    }
}

impl<E> From<E> for AppError
where
    E: Into<anyhow::Error>,
{
    fn from(err: E) -> Self {
        Self(err.into())
    }
}
```

**`type BasicClient = oauth2::basic::BasicClient<EndpointSet, EndpointNotSet, ...>`** 这五个类型参数是 `oauth2` 5.x 的**类型状态模式**:每个参数表示某个端点"已设置"还是"未设置"。

好处是**编译期保证配置完整**——没设置 `auth_uri` 就调用 `authorize_url()` 会编译失败,而不是运行时报错。代价是类型极长,必须起别名。

**`AppError` 这段和[第 15 章](15-anyhow-error-response.md#步骤-3一个泛型-from-让--接住一切)一模一样**,官方注释也直接指向了那个示例。

**注意这次 `into_response` 里有 `tracing::error!`** —— 和[第 14 章](14-error-handling.md#为什么不能在-into_response-里记日志)强调的"不要在 `into_response` 里记日志"冲突。

> **更好的写法**:[第 14 章](14-error-handling.md#为什么不能在-into_response-里记日志)解释过为什么不该在 `into_response` 里写日志——它给纯转换函数引入了副作用,而且那时拿不到请求上下文(这条日志不会带上是哪个请求出的错)。更好的做法是把错误塞进响应扩展,由中间件记录。示例这么写是图省事,好处是至少**记了日志**——比[第 24~29 章](24-sqlx-postgres.md)那些把错误直接返回给客户端的做法安全。

**`{:#}` 格式化**打印 anyhow 的完整错误链([第 15 章](15-anyhow-error-response.md#anyhow-的两个实用能力)讲过 `{}` 和 `{:?}` 的区别,`{:#}` 是单行版的完整链)。

**跑起来看**:

```bash
export CLIENT_ID=你的ID
export CLIENT_SECRET=你的密钥
cargo run
```

```
DEBUG oauth: listening on 127.0.0.1:3000
```

不设环境变量会在启动时 panic:

```
called `Result::unwrap()` on an `Err` value: AppError(Missing CLIENT_ID!)
```

**这次是启动即失败**,比[第 30 章](30-jwt.md#代价与边界)那个 `LazyLock` 懒加载好——问题在部署时就暴露,而不是等第一个请求。

### 步骤 2:发起授权跳转

**要做什么**:实现 `/auth/discord`,生成 `state`、存进 session、跳转到 Discord。

```rust
// src/main.rs
use async_session::{MemoryStore, Session, SessionStore};        // 改动
use axum::{
    extract::{FromRef, State},                                  // 改动
    http::{header::SET_COOKIE, HeaderMap},                       // 新增
    response::{IntoResponse, Redirect, Response},                // 改动:加上 Redirect
    routing::get,
    Router,
};
use oauth2::{
    AuthUrl, ClientId, ClientSecret, CsrfToken, EndpointNotSet, EndpointSet, RedirectUrl,
    Scope, TokenUrl,                                             // 改动:加上 CsrfToken、Scope
};
// ...(其余 use 不变)

#[tokio::main]
async fn main() {
    // ...(状态构建不变)

    // 改动:挂上授权路由
    let app = Router::new()
        .route("/auth/discord", get(discord_auth))
        .with_state(app_state);

    // ...
}

// ...(AppState、FromRef、oauth_client 不变)

// 新增
async fn discord_auth(
    State(client): State<BasicClient>,
    State(store): State<MemoryStore>,
) -> Result<impl IntoResponse, AppError> {
    let (auth_url, csrf_token) = client
        .authorize_url(CsrfToken::new_random)
        .add_scope(Scope::new("identify".to_string()))
        .url();

    // Create session to store csrf_token
    let mut session = Session::new();
    session
        .insert(CSRF_TOKEN, &csrf_token)
        .context("failed in inserting CSRF token into session")?;

    // Store the session in MemoryStore and retrieve the session cookie
    let cookie = store
        .store_session(session)
        .await
        .context("failed to store CSRF token session")?
        .context("unexpected error retrieving CSRF cookie value")?;

    // Attach the session cookie to the response header
    let cookie = format!("{COOKIE_NAME}={cookie}; SameSite=Lax; HttpOnly; Secure; Path=/");
    let mut headers = HeaderMap::new();
    headers.insert(
        SET_COOKIE,
        cookie.parse().context("failed to parse cookie")?,
    );

    Ok((headers, Redirect::to(auth_url.as_ref())))
}

// ...(AppError 相关不变)
```

**两个 `State` 参数**,靠步骤 1 那两个 `FromRef` 实现各取所需。前置知识讲过。

**`client.authorize_url(CsrfToken::new_random)`** 一次做两件事:生成随机的 `state` 值、构造出完整的授权 URL。返回元组 `(URL, token)`。

**`add_scope(Scope::new("identify"))`** 声明权限范围。Discord 的 `identify` 只给基本信息(用户名、头像、ID),不给邮箱(那需要 `email` scope)。**最小权限原则:只要你真正需要的。**

**把 `csrf_token` 存进 session**,然后 `store_session` 返回一个 cookie 值(session ID)。这就是前置知识强调的"state 必须和 session 绑定"。

**`Ok((headers, Redirect::to(...)))`** 是[第 06 章](06-global-404-handler.md#元组作为返回类型)讲的元组返回:头部 + 响应。`Redirect::to` 产生 `303 See Other`。

**跑起来看**:

```bash
cargo run
```

```bash
curl -i -s http://127.0.0.1:3000/auth/discord | head -5
```

```
HTTP/1.1 303 See Other
set-cookie: SESSION=xxxxx; SameSite=Lax; HttpOnly; Secure; Path=/
location: https://discord.com/api/oauth2/authorize?response_type=code&client_id=...&state=RANDOM&scope=identify&redirect_uri=http%3A%2F%2F127.0.0.1%3A3000%2Fauth%2Fauthorized
```

**看清 `location` 里的参数**:`client_id`、`state`(随机值)、`scope`、`redirect_uri`——正是前置知识讲的第 1 步。

同时 `set-cookie` 种了一个 session,里面存着那个 `state`。

**多请求几次,`state` 每次都不同:**

```bash
for i in 1 2 3; do
  curl -s -o /dev/null -D - http://127.0.0.1:3000/auth/discord | grep -i location | grep -o 'state=[^&]*'
done
```

```
state=aB3xK9mZ...
state=pQ7wR2nL...
state=fH5jT8vC...
```

**随机性是防 CSRF 的基础** —— 攻击者猜不出受害者那次的 `state`。

### 步骤 3:处理回调,校验 state,换 token

**要做什么**:实现 `/auth/authorized`,完成流程的第 3、4 步。

```rust
// src/main.rs
use anyhow::{anyhow, Context, Result};              // 改动:加上 anyhow!
use axum::{
    extract::{FromRef, Query, State},               // 改动:加上 Query
    // ...
    RequestPartsExt, Router,                        // 后面步骤会用
};
use axum_extra::{headers, TypedHeader};             // 新增
use oauth2::{
    AuthUrl, AuthorizationCode, ClientId, ClientSecret, CsrfToken, EndpointNotSet, EndpointSet,
    RedirectUrl, Scope, TokenResponse, TokenUrl,    // 改动:加上 AuthorizationCode、TokenResponse
};
use serde::{Deserialize, Serialize};                // 新增
// ...

#[tokio::main]
async fn main() {
    // ...

    // 改动:加上回调路由
    let app = Router::new()
        .route("/auth/discord", get(discord_auth))
        .route("/auth/authorized", get(login_authorized))
        .with_state(app_state);

    // ...
}

// 新增:The user data we'll get back from Discord.
#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: String,
    avatar: Option<String>,
    username: String,
    discriminator: String,
}

// 新增
#[derive(Debug, Deserialize)]
#[allow(dead_code)]
struct AuthRequest {
    code: String,
    state: String,
}

// 新增
async fn csrf_token_validation_workflow(
    auth_request: &AuthRequest,
    cookies: &headers::Cookie,
    store: &MemoryStore,
) -> Result<(), AppError> {
    // Extract the cookie from the request
    let cookie = cookies
        .get(COOKIE_NAME)
        .context("unexpected error getting cookie name")?
        .to_string();

    // Load the session
    let session = match store
        .load_session(cookie)
        .await
        .context("failed to load session")?
    {
        Some(session) => session,
        None => return Err(anyhow!("Session not found").into()),
    };

    // Extract the CSRF token from the session
    let stored_csrf_token = session
        .get::<CsrfToken>(CSRF_TOKEN)
        .context("CSRF token not found in session")?
        .to_owned();

    // Cleanup the CSRF token session
    store
        .destroy_session(session)
        .await
        .context("Failed to destroy old session")?;

    // Validate CSRF token is the same as the one in the auth request
    if *stored_csrf_token.secret() != auth_request.state {
        return Err(anyhow!("CSRF token mismatch").into());
    }

    Ok(())
}

// 新增
async fn login_authorized(
    Query(query): Query<AuthRequest>,
    State(store): State<MemoryStore>,
    State(oauth_client): State<BasicClient>,
    TypedHeader(cookies): TypedHeader<headers::Cookie>,
) -> Result<impl IntoResponse, AppError> {
    csrf_token_validation_workflow(&query, &cookies, &store).await?;

    let client = reqwest::Client::new();

    // Get an auth token
    let token = oauth_client
        .exchange_code(AuthorizationCode::new(query.code.clone()))
        .request_async(&client)
        .await
        .context("failed in sending request to authorization server")?;

    // Fetch user data from discord
    let user_data: User = client
        // https://discord.com/developers/docs/resources/user#get-current-user
        .get("https://discordapp.com/api/users/@me")
        .bearer_auth(token.access_token().secret())
        .send()
        .await
        .context("failed in sending request to target Url")?
        .json::<User>()
        .await
        .context("failed to deserialize response as JSON")?;

    // Create a new session filled with user data
    let mut session = Session::new();
    session
        .insert("user", &user_data)
        .context("failed in inserting serialized value into session")?;

    // Store session and get corresponding cookie
    let cookie = store
        .store_session(session)
        .await
        .context("failed to store session")?
        .context("unexpected error retrieving cookie value")?;

    // Build the cookie
    let cookie = format!("{COOKIE_NAME}={cookie}; SameSite=Lax; HttpOnly; Secure; Path=/");

    // Set cookie
    let mut headers = HeaderMap::new();
    headers.insert(
        SET_COOKIE,
        cookie.parse().context("failed to parse cookie")?,
    );

    Ok((headers, Redirect::to("/")))
}

// ...(其余不变)
```

**`csrf_token_validation_workflow` 是本章最重要的函数。** 四步:

1. 从 cookie 拿到 session ID
2. 加载 session,取出存的 `csrf_token`
3. **销毁这个 session**(用完即弃,防重放)
4. 比对存的 token 和回调带回的 `state`,不一致就报错

第 3 步容易忽略但很重要:**每个授权码只能用一次,对应的 state session 也该立刻作废**,否则攻击者可以重放同一个回调 URL。

**`exchange_code(...).request_async(&client)`** 是流程的第 4 步:拿授权码去换 token。这个请求由**你的服务端**发出,带着 `client_secret`,不经过浏览器。

**`client.get("https://discordapp.com/api/users/@me").bearer_auth(...)`** 用拿到的 token 请求用户信息。`bearer_auth` 是 `reqwest` 的便利方法,等价于设置 `Authorization: Bearer xxx` 头部。

**然后建立新 session 存用户数据,种新 cookie,跳回首页。** 注意这是**第二个** session——第一个存 `state` 的已经销毁了。

**跑起来看**:

这一步要用浏览器完成,因为需要在 Discord 页面上点授权。

```bash
cargo run
```

浏览器访问 `http://127.0.0.1:3000/auth/discord`,会跳到 Discord 的授权页面,点"授权"后跳回来。

**如果 `Secure` 属性导致失败**(浏览器不在 HTTP 下发送带 `Secure` 的 cookie),会看到 `Something went wrong`。临时把 cookie 里的 `Secure` 去掉:

```rust
let cookie = format!("{COOKIE_NAME}={cookie}; SameSite=Lax; HttpOnly; Path=/");
```

**这是本地开发时必须做的妥协**,「代价与边界」会详细说。

**验证 state 校验真的在工作。** 手工构造一个 state 不匹配的回调:

```bash
curl -i -s 'http://127.0.0.1:3000/auth/authorized?code=fake&state=wrong' | head -1
```

```
HTTP/1.1 500 Internal Server Error
```

服务端日志:

```
ERROR oauth: Application error: unexpected error getting cookie name
```

没带 cookie 就直接失败了。**这正是防 CSRF 的效果** —— 攻击者伪造的回调 URL 里没有受害者的 session cookie(或者即使浏览器带上了,session 里的 state 也和伪造的对不上)。

### 步骤 4:身份提取器与其余路由

**要做什么**:实现 `User` 提取器,补上 `/`、`/protected`、`/logout`。

```rust
// src/main.rs
use axum::{
    extract::{FromRef, FromRequestParts, OptionalFromRequestParts, Query, State},   // 改动
    // ...
};
use axum_extra::{headers, typed_header::TypedHeaderRejectionReason, TypedHeader};   // 改动
use http::{header, request::Parts, StatusCode};                                      // 改动
use std::{convert::Infallible, env};                                                 // 改动

#[tokio::main]
async fn main() {
    // ...

    // 改动:五条路由齐全
    let app = Router::new()
        .route("/", get(index))
        .route("/auth/discord", get(discord_auth))
        .route("/auth/authorized", get(login_authorized))
        .route("/protected", get(protected))
        .route("/logout", get(logout))
        .with_state(app_state);

    // ...
}

// 新增:Session is optional
async fn index(user: Option<User>) -> impl IntoResponse {
    match user {
        Some(u) => format!(
            "Hey {}! You're logged in!\nYou may now access `/protected`.\nLog out with `/logout`.",
            u.username
        ),
        None => "You're not logged in.\nVisit `/auth/discord` to do so.".to_string(),
    }
}

// 新增:Valid user session required. If there is none, redirect to the auth page
async fn protected(user: User) -> impl IntoResponse {
    format!("Welcome to the protected area :)\nHere's your info:\n{user:?}")
}

// 新增
async fn logout(
    State(store): State<MemoryStore>,
    TypedHeader(cookies): TypedHeader<headers::Cookie>,
) -> Result<impl IntoResponse, AppError> {
    let cookie = cookies
        .get(COOKIE_NAME)
        .context("unexpected error getting cookie name")?;

    let session = match store
        .load_session(cookie.to_string())
        .await
        .context("failed to load session")?
    {
        Some(s) => s,
        // No session active, just redirect
        None => return Ok(Redirect::to("/")),
    };

    store
        .destroy_session(session)
        .await
        .context("failed to destroy session")?;

    Ok(Redirect::to("/"))
}

// 新增:没有 session 时重定向到登录页
struct AuthRedirect;

impl IntoResponse for AuthRedirect {
    fn into_response(self) -> Response {
        Redirect::temporary("/auth/discord").into_response()
    }
}

// 新增:必需版本
impl<S> FromRequestParts<S> for User
where
    MemoryStore: FromRef<S>,
    S: Send + Sync,
{
    // If anything goes wrong or no session is found, redirect to the auth page
    type Rejection = AuthRedirect;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let store = MemoryStore::from_ref(state);

        let cookies = parts
            .extract::<TypedHeader<headers::Cookie>>()
            .await
            .map_err(|e| match *e.name() {
                header::COOKIE => match e.reason() {
                    TypedHeaderRejectionReason::Missing => AuthRedirect,
                    _ => panic!("unexpected error getting Cookie header(s): {e}"),
                },
                _ => panic!("unexpected error getting cookies: {e}"),
            })?;
        let session_cookie = cookies.get(COOKIE_NAME).ok_or(AuthRedirect)?;

        let session = store
            .load_session(session_cookie.to_string())
            .await
            .unwrap()
            .ok_or(AuthRedirect)?;

        let user = session.get::<User>("user").ok_or(AuthRedirect)?;

        Ok(user)
    }
}

// 新增:可选版本
impl<S> OptionalFromRequestParts<S> for User
where
    MemoryStore: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = Infallible;

    async fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> Result<Option<Self>, Self::Rejection> {
        match <User as FromRequestParts<S>>::from_request_parts(parts, state).await {
            Ok(res) => Ok(Some(res)),
            Err(AuthRedirect) => Ok(None),
        }
    }
}

// ...(其余不变)
```

**`AuthRedirect` 作为拒绝类型** —— 未登录时不返回 `401`,而是**重定向到登录页**。这是网页应用的常规做法(API 服务才返回 `401`)。

**`type Rejection = AuthRedirect`** 让这个行为写进了类型:`protected(user: User)` 一旦提取失败,用户就被送去登录。

**`OptionalFromRequestParts` 复用必需版本**,前置知识讲过。于是 `index(user: Option<User>)` 和 `protected(user: User)` 表达了两种不同的需求。

**注意提取器里那两个 `panic!`** —— 处理"cookie 头部存在但格式非法"的情况。

> **更好的写法**:提取器里对非预期的 cookie 解析错误直接 `panic!`,意味着**客户端发一个格式错误的 Cookie 头部就能让这个请求 panic**。虽然 axum 会捕获 panic 不让整个服务倒下,但这个请求会返回一个空响应,而且日志里是一段 backtrace 而非清晰的错误。更稳妥的做法是所有解析失败都归为 `AuthRedirect` 或返回 `400`。另外 `store.load_session(...).await.unwrap()` 也是同样的问题——存储层出错会 panic,应该转成错误响应。

**`logout` 里的 `destroy_session`** 是有状态会话的核心优势:**删掉服务端记录,cookie 立刻作废**。[第 30 章](30-jwt.md#无状态的代价token-撤销不了)那个 JWT 做不到这件事。

**跑起来看**:

```bash
cargo run
```

未登录访问首页:

```bash
curl -s http://127.0.0.1:3000/
```

```
You're not logged in.
Visit `/auth/discord` to do so.
```

**`Option<User>` 提取到了 `None`,没有报错。**

未登录访问受保护路由:

```bash
curl -i -s http://127.0.0.1:3000/protected | head -3
```

```
HTTP/1.1 307 Temporary Redirect
location: /auth/discord
```

**重定向到登录页**,这就是 `AuthRedirect` 的效果。

完整流程用浏览器走一遍:访问 `http://127.0.0.1:3000/` → 点 `/auth/discord` → Discord 授权 → 跳回首页,现在显示 `Hey 你的用户名! You're logged in!` → 访问 `/protected` 能看到用户信息 → 访问 `/logout` → 回到首页,又显示未登录。

**验证登出真的生效。** 登录后从浏览器开发者工具复制 session cookie:

```bash
COOKIE="SESSION=你复制的值"
curl -s -H "Cookie: $COOKIE" http://127.0.0.1:3000/protected     # 能访问
curl -s -H "Cookie: $COOKIE" http://127.0.0.1:3000/logout        # 登出
curl -i -s -H "Cookie: $COOKIE" http://127.0.0.1:3000/protected | head -1   # 再试
```

```
Welcome to the protected area :)...
HTTP/1.1 307 Temporary Redirect
```

**同一个 cookie 值,登出后立刻失效** —— 因为服务端的 session 记录被删了。**这是[第 30 章](30-jwt.md) JWT 方案无论如何做不到的。**

## 完整代码

`src/main.rs`:

```rust
use anyhow::{anyhow, Context, Result};
use async_session::{MemoryStore, Session, SessionStore};
use axum::{
    extract::{FromRef, FromRequestParts, OptionalFromRequestParts, Query, State},
    http::{header::SET_COOKIE, HeaderMap},
    response::{IntoResponse, Redirect, Response},
    routing::get,
    RequestPartsExt, Router,
};
use axum_extra::{headers, typed_header::TypedHeaderRejectionReason, TypedHeader};
use http::{header, request::Parts, StatusCode};
use oauth2::{
    AuthUrl, AuthorizationCode, ClientId, ClientSecret, CsrfToken, EndpointNotSet, EndpointSet,
    RedirectUrl, Scope, TokenResponse, TokenUrl,
};
use serde::{Deserialize, Serialize};
use std::{convert::Infallible, env};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

static COOKIE_NAME: &str = "SESSION";
static CSRF_TOKEN: &str = "csrf_token";

type BasicClient = oauth2::basic::BasicClient<
    EndpointSet,
    EndpointNotSet,
    EndpointNotSet,
    EndpointNotSet,
    EndpointSet,
>;

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // `MemoryStore` is just used as an example. Don't use this in production.
    let store = MemoryStore::new();
    let oauth_client = oauth_client().unwrap();
    let app_state = AppState {
        store,
        oauth_client,
    };

    let app = Router::new()
        .route("/", get(index))
        .route("/auth/discord", get(discord_auth))
        .route("/auth/authorized", get(login_authorized))
        .route("/protected", get(protected))
        .route("/logout", get(logout))
        .with_state(app_state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .context("failed to bind TcpListener")
        .unwrap();

    tracing::debug!(
        "listening on {}",
        listener
            .local_addr()
            .context("failed to return local address")
            .unwrap()
    );

    axum::serve(listener, app).await.unwrap();
}

#[derive(Clone)]
struct AppState {
    store: MemoryStore,
    oauth_client: BasicClient,
}

impl FromRef<AppState> for MemoryStore {
    fn from_ref(state: &AppState) -> Self {
        state.store.clone()
    }
}

impl FromRef<AppState> for BasicClient {
    fn from_ref(state: &AppState) -> Self {
        state.oauth_client.clone()
    }
}

fn oauth_client() -> Result<BasicClient, AppError> {
    // Environment variables (* = required):
    // *"CLIENT_ID"     "REPLACE_ME";
    // *"CLIENT_SECRET" "REPLACE_ME";
    //  "REDIRECT_URL"  "http://127.0.0.1:3000/auth/authorized";
    //  "AUTH_URL"      "https://discord.com/api/oauth2/authorize?response_type=code";
    //  "TOKEN_URL"     "https://discord.com/api/oauth2/token";

    let client_id = env::var("CLIENT_ID").context("Missing CLIENT_ID!")?;
    let client_secret = env::var("CLIENT_SECRET").context("Missing CLIENT_SECRET!")?;
    let redirect_url = env::var("REDIRECT_URL")
        .unwrap_or_else(|_| "http://127.0.0.1:3000/auth/authorized".to_string());

    let auth_url = env::var("AUTH_URL").unwrap_or_else(|_| {
        "https://discord.com/api/oauth2/authorize?response_type=code".to_string()
    });

    let token_url = env::var("TOKEN_URL")
        .unwrap_or_else(|_| "https://discord.com/api/oauth2/token".to_string());

    Ok(oauth2::basic::BasicClient::new(ClientId::new(client_id))
        .set_client_secret(ClientSecret::new(client_secret))
        .set_auth_uri(
            AuthUrl::new(auth_url).context("failed to create new authorization server URL")?,
        )
        .set_token_uri(TokenUrl::new(token_url).context("failed to create new token endpoint URL")?)
        .set_redirect_uri(
            RedirectUrl::new(redirect_url).context("failed to create new redirection URL")?,
        ))
}

// The user data we'll get back from Discord.
// https://discord.com/developers/docs/resources/user#user-object-user-structure
#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: String,
    avatar: Option<String>,
    username: String,
    discriminator: String,
}

// Session is optional
async fn index(user: Option<User>) -> impl IntoResponse {
    match user {
        Some(u) => format!(
            "Hey {}! You're logged in!\nYou may now access `/protected`.\nLog out with `/logout`.",
            u.username
        ),
        None => "You're not logged in.\nVisit `/auth/discord` to do so.".to_string(),
    }
}

async fn discord_auth(
    State(client): State<BasicClient>,
    State(store): State<MemoryStore>,
) -> Result<impl IntoResponse, AppError> {
    let (auth_url, csrf_token) = client
        .authorize_url(CsrfToken::new_random)
        .add_scope(Scope::new("identify".to_string()))
        .url();

    // Create session to store csrf_token
    let mut session = Session::new();
    session
        .insert(CSRF_TOKEN, &csrf_token)
        .context("failed in inserting CSRF token into session")?;

    // Store the session in MemoryStore and retrieve the session cookie
    let cookie = store
        .store_session(session)
        .await
        .context("failed to store CSRF token session")?
        .context("unexpected error retrieving CSRF cookie value")?;

    // Attach the session cookie to the response header
    let cookie = format!("{COOKIE_NAME}={cookie}; SameSite=Lax; HttpOnly; Secure; Path=/");
    let mut headers = HeaderMap::new();
    headers.insert(
        SET_COOKIE,
        cookie.parse().context("failed to parse cookie")?,
    );

    Ok((headers, Redirect::to(auth_url.as_ref())))
}

// Valid user session required. If there is none, redirect to the auth page
async fn protected(user: User) -> impl IntoResponse {
    format!("Welcome to the protected area :)\nHere's your info:\n{user:?}")
}

async fn logout(
    State(store): State<MemoryStore>,
    TypedHeader(cookies): TypedHeader<headers::Cookie>,
) -> Result<impl IntoResponse, AppError> {
    let cookie = cookies
        .get(COOKIE_NAME)
        .context("unexpected error getting cookie name")?;

    let session = match store
        .load_session(cookie.to_string())
        .await
        .context("failed to load session")?
    {
        Some(s) => s,
        // No session active, just redirect
        None => return Ok(Redirect::to("/")),
    };

    store
        .destroy_session(session)
        .await
        .context("failed to destroy session")?;

    Ok(Redirect::to("/"))
}

#[derive(Debug, Deserialize)]
#[allow(dead_code)]
struct AuthRequest {
    code: String,
    state: String,
}

async fn csrf_token_validation_workflow(
    auth_request: &AuthRequest,
    cookies: &headers::Cookie,
    store: &MemoryStore,
) -> Result<(), AppError> {
    // Extract the cookie from the request
    let cookie = cookies
        .get(COOKIE_NAME)
        .context("unexpected error getting cookie name")?
        .to_string();

    // Load the session
    let session = match store
        .load_session(cookie)
        .await
        .context("failed to load session")?
    {
        Some(session) => session,
        None => return Err(anyhow!("Session not found").into()),
    };

    // Extract the CSRF token from the session
    let stored_csrf_token = session
        .get::<CsrfToken>(CSRF_TOKEN)
        .context("CSRF token not found in session")?
        .to_owned();

    // Cleanup the CSRF token session
    store
        .destroy_session(session)
        .await
        .context("Failed to destroy old session")?;

    // Validate CSRF token is the same as the one in the auth request
    if *stored_csrf_token.secret() != auth_request.state {
        return Err(anyhow!("CSRF token mismatch").into());
    }

    Ok(())
}

async fn login_authorized(
    Query(query): Query<AuthRequest>,
    State(store): State<MemoryStore>,
    State(oauth_client): State<BasicClient>,
    TypedHeader(cookies): TypedHeader<headers::Cookie>,
) -> Result<impl IntoResponse, AppError> {
    csrf_token_validation_workflow(&query, &cookies, &store).await?;

    let client = reqwest::Client::new();

    // Get an auth token
    let token = oauth_client
        .exchange_code(AuthorizationCode::new(query.code.clone()))
        .request_async(&client)
        .await
        .context("failed in sending request to authorization server")?;

    // Fetch user data from discord
    let user_data: User = client
        // https://discord.com/developers/docs/resources/user#get-current-user
        .get("https://discordapp.com/api/users/@me")
        .bearer_auth(token.access_token().secret())
        .send()
        .await
        .context("failed in sending request to target Url")?
        .json::<User>()
        .await
        .context("failed to deserialize response as JSON")?;

    // Create a new session filled with user data
    let mut session = Session::new();
    session
        .insert("user", &user_data)
        .context("failed in inserting serialized value into session")?;

    // Store session and get corresponding cookie
    let cookie = store
        .store_session(session)
        .await
        .context("failed to store session")?
        .context("unexpected error retrieving cookie value")?;

    // Build the cookie
    let cookie = format!("{COOKIE_NAME}={cookie}; SameSite=Lax; HttpOnly; Secure; Path=/");

    // Set cookie
    let mut headers = HeaderMap::new();
    headers.insert(
        SET_COOKIE,
        cookie.parse().context("failed to parse cookie")?,
    );

    Ok((headers, Redirect::to("/")))
}

struct AuthRedirect;

impl IntoResponse for AuthRedirect {
    fn into_response(self) -> Response {
        Redirect::temporary("/auth/discord").into_response()
    }
}

impl<S> FromRequestParts<S> for User
where
    MemoryStore: FromRef<S>,
    S: Send + Sync,
{
    // If anything goes wrong or no session is found, redirect to the auth page
    type Rejection = AuthRedirect;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let store = MemoryStore::from_ref(state);

        let cookies = parts
            .extract::<TypedHeader<headers::Cookie>>()
            .await
            .map_err(|e| match *e.name() {
                header::COOKIE => match e.reason() {
                    TypedHeaderRejectionReason::Missing => AuthRedirect,
                    _ => panic!("unexpected error getting Cookie header(s): {e}"),
                },
                _ => panic!("unexpected error getting cookies: {e}"),
            })?;
        let session_cookie = cookies.get(COOKIE_NAME).ok_or(AuthRedirect)?;

        let session = store
            .load_session(session_cookie.to_string())
            .await
            .unwrap()
            .ok_or(AuthRedirect)?;

        let user = session.get::<User>("user").ok_or(AuthRedirect)?;

        Ok(user)
    }
}

impl<S> OptionalFromRequestParts<S> for User
where
    MemoryStore: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = Infallible;

    async fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> Result<Option<Self>, Self::Rejection> {
        match <User as FromRequestParts<S>>::from_request_parts(parts, state).await {
            Ok(res) => Ok(Some(res)),
            Err(AuthRedirect) => Ok(None),
        }
    }
}

// Use anyhow, define error and enable '?'
// For a simplified example of using anyhow in axum check /examples/anyhow-error-response
#[derive(Debug)]
struct AppError(anyhow::Error);

// Tell axum how to convert `AppError` into a response.
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        tracing::error!("Application error: {:#}", self.0);

        (StatusCode::INTERNAL_SERVER_ERROR, "Something went wrong").into_response()
    }
}

// This enables using `?` on functions that return `Result<_, anyhow::Error>` to turn them into
// `Result<_, AppError>`. That way you don't need to do that manually.
impl<E> From<E> for AppError
where
    E: Into<anyhow::Error>,
{
    fn from(err: E) -> Self {
        Self(err.into())
    }
}
```

环境变量:`CLIENT_ID`、`CLIENT_SECRET`(必需),`REDIRECT_URL`、`AUTH_URL`、`TOKEN_URL`(可选)

## 代价与边界

**`MemoryStore` 不能用于生产。** 官方注释自己写了 "Don't use this in production"。三个问题:

- **重启丢失**,所有用户被登出
- **多实例不共享**,负载均衡到不同实例会看到不同的登录状态
- **无过期清理**,session 只增不减直到 OOM

生产应该用 `async-redis-session` 或数据库存储。

**`Secure` 属性和本地 HTTP 冲突。** 步骤 3 提过:浏览器不会在明文 HTTP 上发送带 `Secure` 的 cookie,所以示例在 `http://127.0.0.1:3000` 下**实际是跑不通的**,必须临时去掉 `Secure`。

正确做法是**按环境切换**:开发环境不加 `Secure`,生产环境必须加。这也说明了为什么本地开发最好也配上 HTTPS([第 32 章](32-tls-rustls.md)会讲)。

**提取器里的 `panic!` 和 `unwrap()`。** 步骤 4 的「更好的写法」指出了。

**`into_response` 里记日志。** 步骤 1 的「更好的写法」指出了。

**session 没有过期时间。** `async-session` 的 `Session` 支持 `expire_in()`,但示例没设。**永不过期的 session 等于永久登录**,和[第 30 章](30-jwt.md#exp-是必须的)那个"没有 `exp` 的 JWT"是同一类问题。

**没有 PKCE。** OAuth 2.1 建议所有客户端都用 PKCE(Proof Key for Code Exchange)——即使授权码泄露,没有 code_verifier 也换不到 token。`oauth2` crate 支持它(`set_pkce_challenge`),示例没用。对服务端应用来说 `client_secret` 已经提供了类似保护,但加上更稳妥。

**没有处理用户拒绝授权。** 用户在 Discord 页面点"取消"时,回调 URL 是 `?error=access_denied` 而不是 `?code=...`,`Query<AuthRequest>` 会因为缺 `code` 字段而返回 `400`。应该单独处理这种情况给个友好提示。

**`client_secret` 从环境变量读,但没有轮换机制。** 密钥泄露后需要能快速更换,生产环境应该接密钥管理服务。

**scope 应该最小化。** 本例用 `identify` 是对的。别为了"以后可能用得上"申请一堆权限——用户看到长长的授权列表会犹豫,而且泄露时影响面更大。

## 动手练习

**1. 按环境切换 `Secure` 属性。**

提示:写一个 `fn build_cookie(value: &str) -> String`,根据环境变量 `APP_ENV` 决定加不加 `Secure`。两处设置 cookie 的地方都改用它。**这是本地开发必须解决的问题**,而且能避免"开发时删了 `Secure`、上线忘了加回去"这类事故。

**2. 给 session 加过期时间。**

提示:`session.expire_in(std::time::Duration::from_secs(3600))`。同时给 cookie 加 `Max-Age=3600`。做完想想:两个过期时间必须一致吗?(不必须,但 cookie 先过期会让用户重新登录,session 先过期会让 cookie 变成无效值——**session 的过期时间应该 >= cookie 的**。)

**3. 处理用户拒绝授权的情况。**

提示:Discord 在用户点"取消"时回调 `?error=access_denied&state=...`。当前的 `Query<AuthRequest>` 会因为缺 `code` 返回 `400`。改法:给 `AuthRequest` 的 `code` 加 `Option`,再加一个 `error: Option<String>` 字段,handler 里先检查 `error`。验证:在 Discord 授权页点"取消",应该看到友好提示而不是 `400`。

## 小结与下一章预告

OAuth 把"验证用户身份"委托给第三方,**你的服务永远不接触密码**。授权码流程有四步:跳转到授权方 → 用户确认 → 带授权码回调 → 服务端用码换 token。之所以要"授权码换 token"这一层,是因为回调经过浏览器(URL 会进历史、日志、referer),而换 token 是服务端直连、带 `client_secret`——**真正的 token 从不出现在浏览器里**。

**`state` 参数防的是登录 CSRF**:攻击者诱导受害者用攻击者的授权码完成登录,之后受害者的一切操作都进了攻击者账号。防御的关键是 **`state` 必须和当前浏览器的 session 绑定**,而且用完立刻销毁防重放。

本章的会话是**有状态**的,正好和[第 30 章](30-jwt.md)对照:服务端存 `session_id → 用户数据`,客户端只拿一个 ID。代价是需要共享存储,好处是**登出能立即生效**——本章验证过同一个 cookie 值在 `/logout` 后立刻失效,这是 JWT 无论如何做不到的。**OAuth 示例选 session 而非 JWT 正是因为登录场景通常需要能登出、能踢下线。**

cookie 的三个安全属性各司其职:`HttpOnly` 防 XSS 偷 cookie、`Secure` 防明文截获、`SameSite` 防 CSRF。**但 `Secure` 会让示例在本地 HTTP 下跑不通**,这暴露了一个现实:本地开发也该配 HTTPS。

`OptionalFromRequestParts` 让同一个类型能表达两种需求:`User` 是"必须登录",`Option<User>` 是"登录与否都行"。

「代价与边界」列了九条问题,其中 `MemoryStore` 不能用于生产是官方自己标注的。

前两章的认证都建立在一个隐含前提上:**通信是安全的**。JWT 的 token 在头部明文传输、session cookie 也是——HTTP 下任何中间人都能截获并冒用。[第 32 章 tls-rustls](32-tls-rustls.md) 补上这一环:怎么给 axum 服务启用 HTTPS、自签名证书怎么生成、以及 TLS 握手到底在做什么。
