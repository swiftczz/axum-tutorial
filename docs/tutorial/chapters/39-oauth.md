# 39. oauth

对应示例：`examples/oauth`

本章目标：理解 OAuth 授权码流程，并学会在 Axum 中处理第三方登录跳转、回调、CSRF state、session cookie 和受保护页面。

上一章 JWT 是“服务端自己签发 token”。  
这一章 OAuth 是“把用户带到第三方平台登录，第三方回调后再建立本应用自己的 session”。

## 这个小项目在做什么

示例使用 Discord OAuth。

路由：

```text
GET /                -> 首页，显示是否已登录
GET /auth/discord    -> 跳转到 Discord 授权页
GET /auth/authorized -> Discord 回调地址
GET /protected       -> 需要登录才能访问
GET /logout          -> 销毁 session
```

登录主线是：

```text
用户访问 /auth/discord
-> 服务端生成 auth_url 和 csrf_token
-> csrf_token 存入 session
-> SESSION cookie 写给浏览器
-> 重定向到 Discord
-> 用户在 Discord 授权
-> Discord 回调 /auth/authorized?code=...&state=...
-> 服务端校验 state 是否等于之前的 csrf_token
-> 用 code 换 access token
-> 用 access token 请求 Discord 用户信息
-> 把 User 存入新 session
-> 写 SESSION cookie
-> 重定向回 /
```

## 先理解 OAuth 和 JWT 的区别

JWT 章节中：

```text
本服务验证账号密码
本服务签发 token
客户端带 token 访问本服务
```

OAuth 章节中：

```text
用户去第三方平台登录
第三方平台给本服务 code
本服务用 code 换第三方 access token
本服务拿 access token 获取用户信息
本服务再建立自己的 session
```

本章最终访问 `/protected` 靠的是本服务的 session cookie，不是直接让浏览器每次拿 Discord access token 访问业务接口。

### 为什么不直接让前端拿 access token？

你可能会问：第三方平台授权后，为什么不让前端直接拿到 access token、每次请求带上它？

因为 **`client_secret` 不能放到前端**。前端代码（浏览器里的 JS、移动端 App）是可以被反编译的，一旦 secret 泄露，任何人都能伪造你的应用去换 token。

授权码流程的核心设计就是：**前端只拿到 `code`（一次性、短期、和你的应用绑定），`code` 必须回传到你的后端，后端用 `client_secret` 才能换成 `access token`**。这样 secret 永远留在后端。

```text
前端 → 第三方平台登录 → 拿到 code（无害，泄露了也没用）
前端 → 把 code 回传给后端
后端 → 用 code + client_secret 换 access token（这一步前端看不到）
后端 → 用 access token 取用户信息，建立自己的 session
```

这也是为什么有 **PKCE**（Proof Key for Code Exchange）：对于 SPA、移动端这类"没有后端能存 secret"的场景，PKCE 用一个动态生成的 challenge 替代 secret 的作用。现代 OAuth 教程几乎都推荐即使有 secret 也加上 PKCE，作为额外防护。本例因为后端持有 secret，没用 PKCE，但如果你做 SPA 登录，应该补上 `set_pkce_verifier(...)`。

> 扩展：OAuth 2.0 还有其他流程（client credentials 用于服务间调用、device flow 用于电视/IoT 设备等），本章用的是**授权码流程**（authorization code flow），它是最安全、最常用的，专门给"有后端的 Web 应用"用。

## OAuth 授权码流程里的几个角色

先记住四个词：

```text
client_id
client_secret
redirect_uri
code
```

- `client_id`：第三方平台给你的应用 ID。
- `client_secret`：第三方平台给你的应用密钥，不能泄露。
- `redirect_uri`：第三方平台登录完成后回调你的地址。
- `code`：第三方平台回调时给你的临时授权码。

本示例的 redirect uri 是：

```text
http://127.0.0.1:3000/auth/authorized
```

它必须和 Discord 开发者后台配置一致。

## 文件和依赖

这个 example 有两个主要文件：

1. `examples/oauth/Cargo.toml`：声明 OAuth、session、reqwest、typed-header、anyhow。
2. `examples/oauth/src/main.rs`：实现 OAuth client、session、登录跳转、回调、用户 extractor。

关键依赖：

- `oauth2`：生成授权 URL，用 code 换 access token。
- `async-session`：保存 CSRF token 和登录用户数据。
- `reqwest`：请求 Discord 用户信息接口。
- `axum-extra`：解析 Cookie typed header。
- `anyhow`：简化错误传播。
- `axum`：Router、State、Query、自定义 extractor。

## 第一步：AppState 保存 session store 和 OAuth client

源码：

````rust
#[derive(Clone)]
struct AppState {
    store: MemoryStore,
    oauth_client: BasicClient,
}
````

应用需要共享两类状态：

```text
MemoryStore   -> 保存 session
BasicClient   -> OAuth client
```

`MemoryStore` 只是教学示例。  
真实生产不能用内存 session store，因为服务重启会丢 session，多实例部署也无法共享。

源码还实现了：

````rust
impl FromRef<AppState> for MemoryStore { ... }
impl FromRef<AppState> for BasicClient { ... }
````

这样 handler 可以直接写：

````rust
State(store): State<MemoryStore>
State(client): State<BasicClient>
````

从 `AppState` 中提取子状态。

## 第二步：创建 OAuth client

源码：

````rust
fn oauth_client() -> Result<BasicClient, AppError> {
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
        .set_auth_uri(AuthUrl::new(auth_url)?)
        .set_token_uri(TokenUrl::new(token_url)?)
        .set_redirect_uri(RedirectUrl::new(redirect_url)?))
}
````

必填环境变量：

```text
CLIENT_ID
CLIENT_SECRET
```

可选环境变量：

```text
REDIRECT_URL
AUTH_URL
TOKEN_URL
```

默认就是 Discord 的授权地址和 token 地址。

## 第三步：发起 Discord 授权

源码：

````rust
async fn discord_auth(
    State(client): State<BasicClient>,
    State(store): State<MemoryStore>,
) -> Result<impl IntoResponse, AppError> {
    let (auth_url, csrf_token) = client
        .authorize_url(CsrfToken::new_random)
        .add_scope(Scope::new("identify".to_string()))
        .url();

    ...

    Ok((headers, Redirect::to(auth_url.as_ref())))
}
````

这一步生成两个东西：

```text
auth_url
csrf_token
```

`auth_url` 是要跳转到 Discord 的地址。  
`csrf_token` 会放进 OAuth 的 `state` 参数，用来防止 CSRF 攻击。

请求的 scope 是：

```text
identify
```

表示只请求用户基础身份信息。

## 第四步：把 CSRF token 存入 session

源码：

````rust
let mut session = Session::new();
session
    .insert(CSRF_TOKEN, &csrf_token)
    .context("failed in inserting CSRF token into session")?;

let cookie = store
    .store_session(session)
    .await?
    .context("unexpected error retrieving CSRF cookie value")?;
````

服务端生成的 `csrf_token` 不能只放在 URL 里。  
还要在服务端 session 中存一份。

等 Discord 回调时，服务端会比较：

```text
回调 query 里的 state
session 里保存的 csrf_token
```

一致才继续登录。

## 第五步：设置 SESSION cookie 并跳转

源码：

````rust
let cookie = format!("{COOKIE_NAME}={cookie}; SameSite=Lax; HttpOnly; Secure; Path=/");
let mut headers = HeaderMap::new();
headers.insert(SET_COOKIE, cookie.parse()?);

Ok((headers, Redirect::to(auth_url.as_ref())))
````

响应做两件事：

1. 设置 `SESSION` cookie。
2. 重定向到 Discord 授权页。

cookie 属性：

- `SameSite=Lax`：降低跨站请求风险。
- `HttpOnly`：浏览器 JavaScript 不能读取。
- `Secure`：只在 HTTPS 下发送。
- `Path=/`：整个站点可用。

注意：本地 HTTP 开发时，`Secure` cookie 在部分浏览器场景下可能影响发送行为。这个 example 保留了更接近安全配置的写法。

## 第六步：Discord 回调参数

源码：

````rust
#[derive(Debug, Deserialize)]
struct AuthRequest {
    code: String,
    state: String,
}
````

Discord 回调 `/auth/authorized` 时会带 query：

```text
?code=...&state=...
```

Axum 用：

````rust
Query(query): Query<AuthRequest>
````

把它解析成结构体。

- `code`：后续用来换 access token。
- `state`：用来校验 CSRF。

## 第七步：校验 CSRF state

源码逻辑在：

````rust
csrf_token_validation_workflow(&query, &cookies, &store).await?;
````

它做了几件事：

1. 从 Cookie 中取 `SESSION`。
2. 从 `MemoryStore` 加载 session。
3. 从 session 中取之前保存的 `csrf_token`。
4. 销毁旧 session。
5. 比较 `stored_csrf_token.secret()` 和 `query.state`。

如果不一致：

````rust
return Err(anyhow!("CSRF token mismatch").into());
````

这一步非常重要。  
否则攻击者可能伪造回调，让用户登录到错误的流程里。

### 没有 state 会怎样？（login CSRF 攻击）

光说"防止 CSRF 攻击"太抽象，讲清楚攻击场景：

```text
1. 攻击者用自己的 Discord 账号走授权流程，拿到一个 code
2. 攻击者把这个 code 嵌进钓鱼链接：http://你的网站/auth/authorized?code=攻击者的code
3. 受害者点击链接，你的网站用这个 code 完成登录
4. 现在 受害者的浏览器里，登录的是【攻击者的 Discord 账号】
5. 受害者以为是自己的账号，在网站里操作（比如绑定邮箱、充值）
6. 这些操作都被记到了【攻击者账号】名下
```

这就叫 **login CSRF**（登录态 CSRF）。`state` 的作用就是防止这种攻击：

```text
每次发起授权时，后端生成一个随机 csrf_token，存进 session
授权 URL 里带上 state=csrf_token
回调时，后端比对 session 里的 token 和 URL 里的 state
因为攻击者拿不到受害者的 session，就无法伪造匹配的 state
```

所以第七步的校验不是"走个形式"，而是 OAuth 安全的核心环节之一。如果你的 OAuth 实现里漏了 state 校验，就等于给 login CSRF 留了门。

> 提示：如果你的 OAuth 接口要跨域，`Authorization` header 会触发 CORS preflight（见第 24 章第八步）。记得在 `CorsLayer` 加 `allow_headers([AUTHORIZATION])`。

## 第八步：用 code 换 access token

源码：

````rust
let token = oauth_client
    .exchange_code(AuthorizationCode::new(query.code.clone()))
    .request_async(&client)
    .await
    .context("failed in sending request to authorization server")?;
````

这里把 Discord 回调给你的 `code` 发给 Discord token endpoint。

成功后得到：

```text
access token
```

这个 access token 可以用来请求 Discord API。

## 第九步：请求 Discord 用户信息

源码：

````rust
let user_data: User = client
    .get("https://discordapp.com/api/users/@me")
    .bearer_auth(token.access_token().secret())
    .send()
    .await?
    .json::<User>()
    .await?;
````

这里使用 reqwest 请求 Discord 当前用户接口。

关键是：

````rust
.bearer_auth(token.access_token().secret())
````

它会加上：

```text
Authorization: Bearer <discord_access_token>
```

拿到 JSON 后反序列化成：

````rust
struct User {
    id: String,
    avatar: Option<String>,
    username: String,
    discriminator: String,
}
````

## 第十步：把 User 存入新 session

源码：

````rust
let mut session = Session::new();
session.insert("user", &user_data)?;

let cookie = store
    .store_session(session)
    .await?
    .context("unexpected error retrieving cookie value")?;

let cookie = format!("{COOKIE_NAME}={cookie}; SameSite=Lax; HttpOnly; Secure; Path=/");
````

OAuth 登录完成后，本应用建立自己的 session。  
session 里保存：

```text
user
```

之后访问 `/protected` 时，不需要每次重新走 Discord OAuth。  
只要浏览器带着 `SESSION` cookie，服务端就能从 session store 里取出 `User`。

## 第十一步：User extractor 保护接口

源码：

````rust
impl<S> FromRequestParts<S> for User
where
    MemoryStore: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = AuthRedirect;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let store = MemoryStore::from_ref(state);
        ...
        let user = session.get::<User>("user").ok_or(AuthRedirect)?;

        Ok(user)
    }
}
````

这让 `User` 成为登录用户 extractor。

受保护接口写成：

````rust
async fn protected(user: User) -> impl IntoResponse {
    format!("Welcome to the protected area :)\nHere's your info:\n{user:?}")
}
````

如果没有 session 或 session 里没有 user，就返回 `AuthRedirect`。

`AuthRedirect` 会重定向到：

```text
/auth/discord
```

## 第十二步：首页使用 Option<User>

源码：

````rust
async fn index(user: Option<User>) -> impl IntoResponse {
    match user {
        Some(u) => format!("Hey {}! You're logged in! ...", u.username),
        None => "You're not logged in.\nVisit `/auth/discord` to do so.".to_string(),
    }
}
````

首页不要求必须登录。  
所以参数是：

````rust
Option<User>
````

源码为 `User` 实现了 `OptionalFromRequestParts`：

```text
能取到用户 -> Some(user)
取不到用户 -> None
```

这就是“可选登录态”的写法。

## 第十三步：logout 销毁 session

源码：

````rust
async fn logout(
    State(store): State<MemoryStore>,
    TypedHeader(cookies): TypedHeader<headers::Cookie>,
) -> Result<impl IntoResponse, AppError> {
    let cookie = cookies.get(COOKIE_NAME)?;
    let session = match store.load_session(cookie.to_string()).await? {
        Some(s) => s,
        None => return Ok(Redirect::to("/")),
    };

    store.destroy_session(session).await?;

    Ok(Redirect::to("/"))
}
````

退出登录就是：

```text
从 cookie 找 session
从 store 加载 session
销毁 session
重定向到首页
```

## 函数职责速查

- `main`：初始化日志，创建 session store 和 OAuth client，注册路由并启动服务。
- `oauth_client`：从环境变量创建 Discord OAuth client。
- `index`：根据是否有 `Option<User>` 显示登录状态。
- `discord_auth`：生成授权 URL 和 CSRF token，设置临时 session，重定向到 Discord。
- `login_authorized`：处理 OAuth 回调，校验 CSRF，用 code 换 token，获取用户信息，建立登录 session。
- `csrf_token_validation_workflow`：校验回调 state 是否匹配 session 中的 CSRF token。
- `protected`：需要登录用户才能访问。
- `logout`：销毁 session。
- `User::from_request_parts`：从 session 中提取登录用户。
- `AppError::into_response`：记录内部错误并返回通用 500。

## 带中文注释的手写版

本章源码较长，建议手写时不要一次敲完整文件。先按流程拆成五块：

````rust
// 1. AppState：保存 MemoryStore 和 BasicClient
#[derive(Clone)]
struct AppState {
    store: MemoryStore,
    oauth_client: BasicClient,
}

// 2. /auth/discord：生成 auth_url + csrf_token，保存 session，重定向到第三方
async fn discord_auth(
    State(client): State<BasicClient>,
    State(store): State<MemoryStore>,
) -> Result<impl IntoResponse, AppError> {
    let (auth_url, csrf_token) = client
        .authorize_url(CsrfToken::new_random)
        .add_scope(Scope::new("identify".to_string()))
        .url();

    let mut session = Session::new();
    session.insert(CSRF_TOKEN, &csrf_token)?;

    let cookie = store.store_session(session).await?.unwrap();
    let cookie = format!("{COOKIE_NAME}={cookie}; SameSite=Lax; HttpOnly; Secure; Path=/");

    let mut headers = HeaderMap::new();
    headers.insert(SET_COOKIE, cookie.parse()?);

    Ok((headers, Redirect::to(auth_url.as_ref())))
}

// 3. /auth/authorized：校验 state，用 code 换 token，再拉取用户信息
async fn login_authorized(
    Query(query): Query<AuthRequest>,
    State(store): State<MemoryStore>,
    State(oauth_client): State<BasicClient>,
    TypedHeader(cookies): TypedHeader<headers::Cookie>,
) -> Result<impl IntoResponse, AppError> {
    csrf_token_validation_workflow(&query, &cookies, &store).await?;

    let client = reqwest::Client::new();
    let token = oauth_client
        .exchange_code(AuthorizationCode::new(query.code.clone()))
        .request_async(&client)
        .await?;

    let user_data: User = client
        .get("https://discordapp.com/api/users/@me")
        .bearer_auth(token.access_token().secret())
        .send()
        .await?
        .json::<User>()
        .await?;

    let mut session = Session::new();
    session.insert("user", &user_data)?;

    let cookie = store.store_session(session).await?.unwrap();
    let cookie = format!("{COOKIE_NAME}={cookie}; SameSite=Lax; HttpOnly; Secure; Path=/");

    let mut headers = HeaderMap::new();
    headers.insert(SET_COOKIE, cookie.parse()?);

    Ok((headers, Redirect::to("/")))
}

// 4. User extractor：从 SESSION cookie 加载 session，再取出 user
impl<S> FromRequestParts<S> for User
where
    MemoryStore: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = AuthRedirect;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let store = MemoryStore::from_ref(state);
        let TypedHeader(cookies) = parts
            .extract::<TypedHeader<headers::Cookie>>()
            .await
            .map_err(|_| AuthRedirect)?;
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

// 5. protected：参数直接写 User，表示必须登录
async fn protected(user: User) -> impl IntoResponse {
    format!("Welcome to the protected area :)\nHere's your info:\n{user:?}")
}
````

上面是便于手写理解的核心版。完整代码请对照源码文件。

## 运行和验证

先在 Discord Developer Portal 创建应用，并配置 redirect URI：

```text
http://127.0.0.1:3000/auth/authorized
```

启动：

````bash
CLIENT_ID=你的_CLIENT_ID \
CLIENT_SECRET=你的_CLIENT_SECRET \
cargo run -p example-oauth
````

浏览器访问：

```text
http://127.0.0.1:3000/
```

再访问：

```text
http://127.0.0.1:3000/auth/discord
```

授权完成后会回到首页。  
然后访问：

```text
http://127.0.0.1:3000/protected
```

退出：

```text
http://127.0.0.1:3000/logout
```

## 常见卡点

### 1. redirect URI 不匹配怎么办？

Discord 后台配置的 redirect URI 必须和代码里的 `REDIRECT_URL` 完全一致。  
协议、域名、端口、路径都要一致。

### 2. MemoryStore 能用于生产吗？

不能。  
它只是 example。生产应使用 Redis、数据库或其他共享 session store。

### 3. 为什么需要 CSRF state？

OAuth 登录是浏览器跳转流程。  
`state` 用来确认回调确实对应本服务发起的那次登录请求。

### 4. OAuth access token 会存到浏览器吗？

这个 example 没有把 Discord access token 存到浏览器。  
它拿到用户信息后，建立本应用自己的 session cookie。

### 5. Secure cookie 本地开发有问题怎么办？

`Secure` 表示 cookie 只应通过 HTTPS 发送。  
本地 HTTP 调试时，如果浏览器没有带 cookie，可以先理解原因，再根据开发环境临时调整 cookie 属性。

## 手写任务

建议按下面顺序自己敲一遍：

1. 先写 `oauth_client()`，读取 `CLIENT_ID` 和 `CLIENT_SECRET`。
2. 写 `/auth/discord`，生成 auth URL 和 CSRF token。
3. 把 CSRF token 存进 session，并设置 cookie。
4. 写 `AuthRequest { code, state }`。
5. 写 CSRF 校验函数。
6. 用 code 换 access token。
7. 用 access token 请求用户信息。
8. 把用户信息写入新 session。
9. 实现 `User` extractor，保护 `/protected`。

加深练习：

1. 把 MemoryStore 换成 Redis-backed session store。
2. 给 session 增加过期时间。
3. 把错误响应改成更友好的页面。

## 本章真正要记住什么

OAuth 授权码流程不是“直接相信第三方回调”。  
完整流程至少包括：

```text
生成授权 URL
保存 CSRF state
第三方回调 code + state
校验 state
用 code 换 access token
用 access token 获取用户信息
建立本应用 session
```

Axum 里可以把登录状态封装成 `User` extractor。  
这样受保护接口只需要写：

````rust
async fn protected(user: User) -> impl IntoResponse
````

认证失败就重定向到登录入口。

## 源码对照

本章手写版对应源码：

- `examples/oauth/src/main.rs`
- `examples/oauth/Cargo.toml`
