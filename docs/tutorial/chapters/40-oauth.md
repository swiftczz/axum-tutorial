# 40. oauth

对应示例：`examples/oauth`

上一章 JWT 是"服务端自己签发 token",这章 OAuth 是"把用户带到第三方平台登录,第三方回调后再建立本应用自己的 session"。理解 OAuth 授权码流程,在 axum 中处理第三方登录跳转、回调、CSRF state、session cookie、受保护页面。示例用 Discord OAuth。

## Cargo.toml

````toml
[package]
name = "example-oauth"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
anyhow = "1"
async-session = "3.0.0"
axum = "0.8"
axum-extra = { version = "0.12", features = ["typed-header"] }
http = "1.0.0"
oauth2 = "5"
reqwest = { version = "0.12", default-features = false, features = ["rustls-tls", "json"] }
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs(核心版)

````rust
// 1. AppState:保存 MemoryStore 和 BasicClient
#[derive(Clone)]
struct AppState {
    store: MemoryStore,
    oauth_client: BasicClient,
}

// 2. /auth/discord:生成 auth_url + csrf_token,保存 session,重定向到第三方
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

// 3. /auth/authorized:校验 state,用 code 换 token,再拉取用户信息
async fn login_authorized(
    Query(query): Query<AuthRequest>,
    State(store): State<MemoryStore>,
    State(oauth_client): State<BasicClient>,
    TypedHeader(cookies): TypedHeader<headers::Cookie>,
) -> Result<impl IntoResponse, AppError> {
    csrf_token_validation_workflow(&query, &cookies, &store).await?;

    // 用 code 换 access token,async_http_client 是 oauth2 crate 提供的异步 HTTP client
    let token = oauth_client
        .exchange_code(AuthorizationCode::new(query.code.clone()))
        .request_async(async_http_client)
        .await?;

    // 用 access token 请求 Discord 用户信息
    let client = reqwest::Client::new();
    let user_data: User = client
        .get("https://discordapp.com/api/users/@me")
        .bearer_auth(token.access_token().secret())
        .send().await?
        .json::<User>().await?;

    let mut session = Session::new();
    session.insert("user", &user_data)?;

    let cookie = store.store_session(session).await?.unwrap();
    let cookie = format!("{COOKIE_NAME}={cookie}; SameSite=Lax; HttpOnly; Secure; Path=/");

    let mut headers = HeaderMap::new();
    headers.insert(SET_COOKIE, cookie.parse()?);

    Ok((headers, Redirect::to("/")))
}

// 4. User extractor:从 SESSION cookie 加载 session,再取出 user
impl<S> FromRequestParts<S> for User
where MemoryStore: FromRef<S>, S: Send + Sync,
{
    type Rejection = AuthRedirect;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let store = MemoryStore::from_ref(state);
        let TypedHeader(cookies) = parts
            .extract::<TypedHeader<headers::Cookie>>().await
            .map_err(|_| AuthRedirect)?;
        let session_cookie = cookies.get(COOKIE_NAME).ok_or(AuthRedirect)?;
        let session = store.load_session(session_cookie.to_string()).await
            .unwrap().ok_or(AuthRedirect)?;
        let user = session.get::<User>("user").ok_or(AuthRedirect)?;
        Ok(user)
    }
}

// 5. protected:参数直接写 User,表示必须登录
async fn protected(user: User) -> impl IntoResponse {
    format!("Welcome to the protected area :)\nHere's your info:\n{user:?}")
}
````

> 上面是便于手写理解的核心版。完整代码(含 `csrf_token_validation_workflow`、`User` 完整定义、`OptionalFromRequestParts`、`logout` 等)见源码对照。

## 运行

先在 [Discord Developer Portal](https://discord.com/developers/applications) 创建应用,配置 redirect URI 为 `http://127.0.0.1:3000/auth/authorized`:

````bash
CLIENT_ID=你的_CLIENT_ID \
CLIENT_SECRET=你的_CLIENT_SECRET \
cargo run -p example-oauth
````

浏览器流程:

````text
http://127.0.0.1:3000/                  首页(未登录)
http://127.0.0.1:3000/auth/discord      跳转 Discord 授权
                                        授权后自动回到首页(已登录)
http://127.0.0.1:3000/protected         受保护页面
http://127.0.0.1:3000/logout            退出
````

## 解读

### OAuth vs JWT

```text
JWT:   本服务验证账号密码 → 本服务签发 token → 客户端带 token 访问本服务
OAuth: 用户去第三方平台登录 → 第三方给本服务 code → 本服务用 code 换 access token → 本服务拿 token 获取用户信息 → 本服务建立自己的 session
```

本章 `/protected` 靠**本服务的 session cookie**,不是让浏览器每次拿 Discord access token 访问业务接口。

### 为什么不直接让前端拿 access token?

因为 **`client_secret` 不能放到前端**。前端代码(JS/移动 App)可被反编译,secret 泄露后任何人都能伪造你的应用换 token。

授权码流程的核心设计:**前端只拿到 `code`(一次性、短期、和应用绑定),`code` 必须回传后端,后端用 `client_secret` 才能换 `access token`**。secret 永远留在后端。

```text
前端 → 第三方登录 → 拿到 code(无害,泄露也没用)
前端 → 把 code 回传后端
后端 → 用 code + client_secret 换 access token(前端看不到)
后端 → 用 access token 取用户信息,建立自己的 session
```

这也是为什么有 **PKCE**(Proof Key for Code Exchange):SPA/移动端这类"没后端能存 secret"的场景用动态 challenge 替代 secret。现代 OAuth 教程几乎都推荐即使有 secret 也加 PKCE。本例后端持 secret 没用 PKCE,做 SPA 登录应补 `set_pkce_verifier(...)`。

> 扩展:OAuth 2.0 还有 client credentials(服务间调用)、device flow(电视/IoT)等流程,本章用**授权码流程**(authorization code flow),最安全最常用,专门给"有后端的 Web 应用"用。

### 授权码流程四个角色

```text
client_id      第三方平台给你的应用 ID
client_secret  第三方平台给你的应用密钥(不能泄露)
redirect_uri   第三方登录完成后回调你的地址
code           第三方回调时给你的临时授权码
```

`redirect_uri` 必须和 Discord 后台配置完全一致(协议/域名/端口/路径)。

### 完整流程

```text
用户访问 /auth/discord
  → 服务端生成 auth_url 和 csrf_token
  → csrf_token 存入 session
  → SESSION cookie 写给浏览器
  → 重定向到 Discord
  → 用户在 Discord 授权
  → Discord 回调 /auth/authorized?code=...&state=...
  → 服务端校验 state 是否等于之前的 csrf_token
  → 用 code 换 access token
  → 用 access token 请求 Discord 用户信息
  → 把 User 存入新 session
  → 写 SESSION cookie
  → 重定向回 /
```

### CSRF state 校验(本章安全核心)

光说"防 CSRF"太抽象,讲清楚攻击场景:

```text
1. 攻击者用自己的 Discord 账号走授权,拿到一个 code
2. 攻击者把 code 嵌进钓鱼链接:http://你的网站/auth/authorized?code=攻击者的code
3. 受害者点链接,你的网站用这个 code 完成登录
4. 受害者浏览器里登录的是【攻击者的 Discord 账号】
5. 受害者以为是自己的账号,操作(绑定邮箱、充值)
6. 这些操作都被记到【攻击者账号】名下
```

这叫 **login CSRF**。`state` 的作用就是防它:

```text
每次发起授权,后端生成随机 csrf_token 存 session
授权 URL 带 state=csrf_token
回调时,后端比对 session 里的 token 和 URL 里的 state
攻击者拿不到受害者的 session,无法伪造匹配的 state
```

所以 `csrf_token_validation_workflow` 不是走形式,是 OAuth 安全的核心环节。漏了 state 校验等于给 login CSRF 留门。

### 用 code 换 token + 请求用户信息

````rust
// async_http_client 是 oauth2 crate 提供的异步 HTTP client
let token = oauth_client
    .exchange_code(AuthorizationCode::new(query.code))
    .request_async(async_http_client).await?;

// 用 access token 请求 Discord 用户信息
let client = reqwest::Client::new();
let user_data: User = client
    .get("https://discordapp.com/api/users/@me")
    .bearer_auth(token.access_token().secret())   // Authorization: Bearer <discord_token>
    .send().await?
    .json::<User>().await?;
````

### 把 User 存入新 session

OAuth 登录完成后,本应用建立自己的 session。之后访问 `/protected` 不需重新走 Discord OAuth,只要浏览器带 `SESSION` cookie,服务端就能从 session store 取 `User`。

### `User` extractor

````rust
impl<S> FromRequestParts<S> for User
where MemoryStore: FromRef<S>, S: Send + Sync,
{
    type Rejection = AuthRedirect;
    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let store = MemoryStore::from_ref(state);
        ... // 从 cookie 取 SESSION → 从 store 加载 session → 取 user
        let user = session.get::<User>("user").ok_or(AuthRedirect)?;
        Ok(user)
    }
}

async fn protected(user: User) -> impl IntoResponse { ... }
````

没 session 或 session 没 user → `AuthRedirect` → 重定向到 `/auth/discord`。首页用 `Option<User>` 实现"可选登录态"(实现 `OptionalFromRequestParts`,取到 Some/取不到 None)。

### cookie 安全属性

```text
SameSite=Lax  降低跨站请求风险
HttpOnly      浏览器 JS 不能读取
Secure        只在 HTTPS 下发送
Path=/        整个站点可用
```

⚠️ 本地 HTTP 开发时 `Secure` cookie 可能影响发送行为,根据开发环境临时调整。

### logout

```text
从 cookie 找 session → 从 store 加载 → 销毁 → 重定向首页
```

## 常见问题

**redirect URI 不匹配?** Discord 后台配置必须和代码 `REDIRECT_URL` 完全一致(协议/域名/端口/路径)。

**MemoryStore 能用于生产?** 不能,只是 example。生产用 Redis/数据库/其他共享 session store。

**为什么需要 CSRF state?** 防 login CSRF——攻击者用自己的 code 诱导受害者登录攻击者账号。state 让攻击者拿不到受害者 session 就无法伪造。

**OAuth access token 存到浏览器吗?** 本 example 没有。拿到用户信息后建立本应用自己的 session cookie。

**Secure cookie 本地开发有问题?** Secure 表示只 HTTPS 发送,本地 HTTP 调试浏览器可能不带 cookie,根据环境临时调整。

## 手写任务

按下面顺序敲:

1. 写 `oauth_client()` 读 `CLIENT_ID`/`CLIENT_SECRET`。
2. 写 `/auth/discord` 生成 auth URL 和 CSRF token。
3. CSRF token 存 session,设置 cookie。
4. 写 `AuthRequest { code, state }`。
5. 写 CSRF 校验函数。
6. 用 code 换 access token。
7. 用 access token 请求用户信息。
8. 用户信息写入新 session。
9. 实现 `User` extractor 保护 `/protected`。

加深练习:

1. MemoryStore 换成 Redis-backed session store。
2. 给 session 增加过期时间。
3. 错误响应改成更友好的页面。

## 小结

- OAuth 授权码流程不是"直接相信第三方回调",完整流程:生成授权 URL → 保存 CSRF state → 第三方回调 code+state → 校验 state → 用 code 换 token → 用 token 获取用户信息 → 建立本应用 session。
- **`client_secret` 不能放前端**,授权码流程核心设计是前端只拿 code,后端用 secret 换 token;SPA/移动端用 PKCE。
- **CSRF state 防 login CSRF**(攻击者用自己 code 诱导受害者登录攻击者账号),不是走形式是 OAuth 安全核心。
- 拿到第三方用户信息后建立**本应用自己的 session**,之后靠 session cookie 访问业务接口,不是每次拿第三方 token。
- 很 axum 的写法:`User` extractor 封装登录状态,`async fn protected(user: User)` 表示必须登录,认证失败统一重定向到登录入口。

## 源码对照

- `examples/oauth/Cargo.toml`
- `examples/oauth/src/main.rs`
