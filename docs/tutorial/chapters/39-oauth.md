# 39. oauth

对应示例：`examples/oauth`

第 38 章用 JWT 做了"无状态认证"——token 自带数据，server 不存 session。这章走 **OAuth 2.0 Authorization Code flow**——让用户用第三方账号（GitHub、Google、Discord）登录，server 端存 session。

OAuth 涉及四件事：**(1) 引导用户去第三方授权 → (2) 第三方回调带 code → (3) 用 code 换 access_token → (4) 用 token 拉用户信息**。session 用 `async-session` + cookie 存。

分 3 步：先搭骨架 + State + 错误类型，再写 OAuth 启动（`/auth/discord`）+ 回调（`/auth/authorized`），最后加自定义 `User` extractor 实现登录态识别。

相比前面章节新引入：**`oauth2` crate、`async_session`（MemoryStore + Session）、`TypedHeader<headers::Cookie>`、Set-Cookie、CSRF token、`OptionalFromRequestParts`、anyhow 错误转 axum 响应**。

## Cargo.toml

````toml
[package]
name = "example-oauth"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
anyhow = "1"
async-session = "3"
axum = { version = "0.8", features = ["macros"] }
axum-extra = { version = "0.12", features = ["typed-header"] }
http = "1"
oauth2 = { version = "5", default-features = false }
reqwest = { version = "0.12", default-features = false, features = ["rustls-tls", "json"] }
serde = { version = "1", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

本章依赖比较多，按职责分组：

- **OAuth 核心**：`oauth2`（OAuth 2.0 客户端，处理授权 URL 生成、code 换 token）+ `reqwest`（HTTP 客户端，调 Discord API 拉用户信息）
- **Session**：`async-session`（session 存储抽象 + 内存实现 `MemoryStore`，存登录态）
- **错误处理**：`anyhow`（动态错误聚合，第 20 章详解）+ `axum-extra` 的 `typed-header`（类型安全提取 Cookie 头）

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：State 骨架 + anyhow 错误类型

先搭基础设施：`AppState` 包含 `MemoryStore`（存 session）和 `BasicClient`（OAuth 客户端）。用 `FromRef` 把它们各自暴露成可提取的 sub-state。再加 `AppError` 类型——把 anyhow 错误转成 axum 响应。

````rust
use anyhow::{Context, Result};
use async_session::MemoryStore;
use axum::{
    extract::{FromRef, State},
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

type BasicClient = oauth2::basic::BasicClient<
    EndpointSet,
    EndpointNotSet,
    EndpointNotSet,
    EndpointNotSet,
    EndpointSet,
>;

#[derive(Clone)]
struct AppState {
    store: MemoryStore,
    oauth_client: BasicClient,
}

// 让 MemoryStore 和 BasicClient 都能被 FromRef 切出来
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
        .set_auth_uri(AuthUrl::new(auth_url).context("failed to create new authorization server URL")?)
        .set_token_uri(TokenUrl::new(token_url).context("failed to create new token endpoint URL")?)
        .set_redirect_uri(RedirectUrl::new(redirect_url).context("failed to create new redirection URL")?))
}

// anyhow 错误包装：让 ? 能在任何 Result<_, anyhow::Error> 上用
#[derive(Debug)]
struct AppError(anyhow::Error);

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

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(/* tracing 初始化 */)
        .with(tracing_subscriber::fmt::layer())
        .init();

    let store = MemoryStore::new();
    let oauth_client = oauth_client().unwrap();
    let app_state = AppState { store, oauth_client };

    let app = Router::new()
        .route("/", get(|| async { "ok" }))
        .with_state(app_state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
````

> **新面孔：`async_session::MemoryStore`**
>
> session 存储（这章用内存版，生产用 Redis）。`store.store_session(session).await` 存入返回 cookie 值；`store.load_session(cookie).await` 用 cookie 取回 session。
>
> **`MemoryStore` 重启就丢，只用于演示**——生产用 `redis-store` 或数据库 store。

> **新面孔：`oauth2::basic::BasicClient`**
>
> OAuth 2.0 客户端。`BasicClient::new(client_id)` 开始构建，链式配置 client_secret、auth_uri（用户授权 URL）、token_uri（换 token URL）、redirect_uri（回调 URL）。
>
> 那串 `EndpointSet, EndpointNotSet, ...` 是类型状态机——编译期保证必填 URL 都设了。Discord 不支持 PKCE/refresh 等，所以 `EndpointNotSet` 多。

> **新面孔：`AppError`（anyhow → IntoResponse 桥接）**
>
> axum handler 返回 `Result<_, E: IntoResponse>`。anyhow 的 `Error` 不实现 `IntoResponse`，所以包装成 `AppError`：
>
> - `impl IntoResponse for AppError`：转成 500 响应并打日志
> - `impl<E: Into<anyhow::Error>> From<E> for AppError`：让 `?` 自动把任何 anyhow 错误转成 AppError
>
> 详细解释见第 20 章 anyhow-error-response。

> **新面孔：`FromRef` 多 sub-state**
>
> `AppState` 包含两个东西（store + client），handler 可能只要其中一个。`impl FromRef<AppState> for MemoryStore` 让 handler 写 `State(store): State<MemoryStore>` 单独取 store。同 ch18 多 sub-state 模式。

---

## 第二步：OAuth 启动 + 回调（核心流程）

这步是 OAuth 的核心。两个 handler 配合：`/auth/discord` 生成授权 URL 跳转过去，`/auth/authorized` 接 Discord 回调用 code 换 token 再拉用户信息。

### `/auth/discord`：生成授权 URL + 存 CSRF

````rust
use async_session::{Session, SessionStore};
use axum::{
    extract::State,
    http::{header::SET_COOKIE, HeaderMap},
    response::{IntoResponse, Redirect},
};
use axum_extra::{headers, TypedHeader};
use oauth2::{CsrfToken, Scope};

static COOKIE_NAME: &str = "SESSION";
static CSRF_TOKEN: &str = "csrf_token";

async fn discord_auth(
    State(client): State<BasicClient>,
    State(store): State<MemoryStore>,
) -> Result<impl IntoResponse, AppError> {
    // 生成授权 URL 和 CSRF token
    let (auth_url, csrf_token) = client
        .authorize_url(CsrfToken::new_random)
        .add_scope(Scope::new("identify".to_string()))
        .url();

    // 把 CSRF token 存进 session
    let mut session = Session::new();
    session.insert(CSRF_TOKEN, &csrf_token).context("failed in inserting CSRF token")?;
    let cookie = store
        .store_session(session).await
        .context("failed to store CSRF token session")?
        .context("unexpected error retrieving CSRF cookie value")?;

    // Set-Cookie 响应头 + 302 跳转到 Discord 授权页
    let cookie = format!("{COOKIE_NAME}={cookie}; SameSite=Lax; HttpOnly; Secure; Path=/");
    let mut headers = HeaderMap::new();
    headers.insert(SET_COOKIE, cookie.parse().context("failed to parse cookie")?);

    Ok((headers, Redirect::to(auth_url.as_ref())))
}
````

> **新面孔：CSRF token + session cookie**
>
> OAuth 流程必须防 CSRF：否则攻击者能把**自己的** authorization code 塞进受害者的回调 URL，受害者就被登录成攻击者账号。
>
> 防御：跳转前生成随机 `CsrfToken` 存 session，回调时验证 query 里 `state` 等于 session 里的 token。
>
> `store_session(session).await` 存的同时返回 cookie 值——session 数据存在 server 端（MemoryStore），cookie 只存 session ID。

> **新面孔：`Set-Cookie` 头**
>
> 服务端通过 `Set-Cookie` 响应头让浏览器存 cookie。格式 `key=value; 属性`：
>
> - `SameSite=Lax`：跨站请求不发 cookie（防 CSRF）
> - `HttpOnly`：JS 读不到 cookie（防 XSS 偷 cookie）
> - `Secure`：只在 HTTPS 发送
> - `Path=/`：所有路径都发
>
> 这些属性是 session cookie 的标配，**少一个就有安全洞**。

### `/auth/authorized`：接回调换 token

````rust
use axum::extract::Query;
use oauth2::{AuthorizationCode, TokenResponse};
use serde::Deserialize;

#[derive(Debug, Deserialize)]
#[allow(dead_code)]
struct AuthRequest {
    code: String,
    state: String,
}

#[derive(Debug, serde::Serialize, serde::Deserialize)]
struct User {
    id: String,
    username: String,
    // ... Discord 用户对象完整字段
}

async fn login_authorized(
    Query(query): Query<AuthRequest>,
    State(store): State<MemoryStore>,
    State(oauth_client): State<BasicClient>,
    TypedHeader(cookies): TypedHeader<headers::Cookie>,
) -> Result<impl IntoResponse, AppError> {
    // 1. CSRF 验证（省略实现，见源码）
    csrf_token_validation_workflow(&query, &cookies, &store).await?;

    let client = reqwest::Client::new();

    // 2. 用 code 换 access_token
    let token = oauth_client
        .exchange_code(AuthorizationCode::new(query.code.clone()))
        .request_async(&client).await
        .context("failed in sending request to authorization server")?;

    // 3. 用 token 拉 Discord 用户信息
    let user_data: User = client
        .get("https://discordapp.com/api/users/@me")
        .bearer_auth(token.access_token().secret())
        .send().await.context("failed in sending request")?
        .json::<User>().await.context("failed to deserialize")?;

    // 4. 把 user 存进新 session，Set-Cookie
    let mut session = Session::new();
    session.insert("user", &user_data).context("failed in inserting user")?;
    let cookie = store
        .store_session(session).await
        .context("failed to store session")?
        .context("unexpected error retrieving cookie")?;
    let cookie = format!("{COOKIE_NAME}={cookie}; SameSite=Lax; HttpOnly; Secure; Path=/");
    let mut headers = HeaderMap::new();
    headers.insert(SET_COOKIE, cookie.parse().context("failed to parse cookie")?);

    Ok((headers, Redirect::to("/")))
}
````

> **新面孔：OAuth 4 步流程**
>
> 这步展开 OAuth Authorization Code flow 的后三步（第一步在 `/auth/discord`）：
>
> 1. `/auth/discord` 生成 URL → 浏览器跳到 Discord 授权页
> 2. 用户在 Discord 点"授权" → Discord 跳回 `/auth/authorized?code=X&state=Y`
> 3. `oauth_client.exchange_code(code)` 用 code POST 到 Discord token URL 换 access_token
> 4. 用 access_token 调 Discord API `/users/@me` 拉用户信息
>
> 这 4 步是**所有 OAuth provider 通用**——GitHub、Google、Slack 都这套。区别只是 URL 和返回的用户对象。

> **新面孔：`TypedHeader<headers::Cookie>`**
>
> `axum-extra::TypedHeader<T>` 类型安全地提取 HTTP 头。`TypedHeader<headers::Cookie>` 提取 cookie 头，`cookies.get("SESSION")` 拿单个 cookie。比手写 `headers.get("cookie")` 安全——类型保证了头存在且格式正确。

---

## 第三步：自定义 `User` extractor——识别登录态

加两个 handler（`/protected` 要登录、`/` 可登录可不登录），核心是自定义 `User` extractor：从 cookie 拿 session，从 session 拿 user。没登录的根据 handler 类型做不同反应（`FromRequestParts` 拒绝 → 跳转登录，`OptionalFromRequestParts` 返回 None）。

````rust
use axum::{
    extract::{FromRequestParts, OptionalFromRequestParts},
    RequestPartsExt,
};
use axum_extra::{headers, typed_header::TypedHeaderRejectionReason, TypedHeader};
use http::{header, request::Parts};
use std::convert::Infallible;

// 没登录就跳转到 /auth/discord
struct AuthRedirect;
impl IntoResponse for AuthRedirect {
    fn into_response(self) -> Response {
        Redirect::temporary("/auth/discord").into_response()
    }
}

// User 是 FromRequestParts：从 session 拿 user，失败就 AuthRedirect
impl<S> FromRequestParts<S> for User
where
    MemoryStore: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = AuthRedirect;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let store = MemoryStore::from_ref(state);

        // 提取 Cookie 头
        let cookies = parts.extract::<TypedHeader<headers::Cookie>>().await
            .map_err(|e| match *e.name() {
                header::COOKIE => match e.reason() {
                    TypedHeaderRejectionReason::Missing => AuthRedirect,
                    _ => panic!("unexpected error: {e}"),
                },
                _ => panic!("unexpected error: {e}"),
            })?;
        let session_cookie = cookies.get(COOKIE_NAME).ok_or(AuthRedirect)?;

        // 从 cookie 找 session，从 session 找 user
        let session = store.load_session(session_cookie.to_string()).await
            .unwrap().ok_or(AuthRedirect)?;
        let user = session.get::<User>("user").ok_or(AuthRedirect)?;

        Ok(user)
    }
}

// OptionalFromRequestParts：失败返回 None（不跳转）
impl<S> OptionalFromRequestParts<S> for User
where
    MemoryStore: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = Infallible;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Option<Self>, Self::Rejection> {
        match <User as FromRequestParts<S>>::from_request_parts(parts, state).await {
            Ok(res) => Ok(Some(res)),
            Err(AuthRedirect) => Ok(None),
        }
    }
}

// 必须登录才能访问
async fn protected(user: User) -> impl IntoResponse {
    format!("Welcome to the protected area :)\nHere's your info:\n{user:?}")
}

// 可登录可不登录
async fn index(user: Option<User>) -> impl IntoResponse {
    match user {
        Some(u) => format!("Hey {}! You're logged in!", u.username),
        None => "You're not logged in.\nVisit `/auth/discord` to do so.".to_string(),
    }
}
````

> **新面孔：`FromRequestParts` vs `OptionalFromRequestParts`**
>
> - `FromRequestParts`：提取失败就拒绝（rejection）——`User` 的 rejection 是 `AuthRedirect`，跳转到登录页
> - `OptionalFromRequestParts`：提取失败返回 `Ok(None)`——handler 用 `Option<User>` 接收
>
> 区别：拒绝 vs 容忍。`/protected` 要登录用前者（拒绝跳转）；`/` 首页两种状态都展示用后者（容忍 None）。

> **新面孔：自定义 `User` extractor**
>
> 这是第 11 章自定义 extractor 模式的进阶版：从 cookie 提取 session，从 session 提取 user。
>
> 关键点：**rejection 类型可以自定义**——这里用 `AuthRedirect` 让"没登录"自动跳转登录页，业务代码不用每个 handler 手动 `if user.is_none() { redirect }`。

---

## 完整代码

完整代码较长（含 CSRF 验证、logout 等细节），见 examples 源码。

````rust
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

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: String,
    avatar: Option<String>,
    username: String,
    discriminator: String,
}

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

    let mut session = Session::new();
    session
        .insert(CSRF_TOKEN, &csrf_token)
        .context("failed in inserting CSRF token into session")?;

    let cookie = store
        .store_session(session)
        .await
        .context("failed to store CSRF token session")?
        .context("unexpected error retrieving CSRF cookie value")?;

    let cookie = format!("{COOKIE_NAME}={cookie}; SameSite=Lax; HttpOnly; Secure; Path=/");
    let mut headers = HeaderMap::new();
    headers.insert(
        SET_COOKIE,
        cookie.parse().context("failed to parse cookie")?,
    );

    Ok((headers, Redirect::to(auth_url.as_ref())))
}

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
    let cookie = cookies
        .get(COOKIE_NAME)
        .context("unexpected error getting cookie name")?
        .to_string();

    let session = match store
        .load_session(cookie)
        .await
        .context("failed to load session")?
    {
        Some(session) => session,
        None => return Err(anyhow!("Session not found").into()),
    };

    let stored_csrf_token = session
        .get::<CsrfToken>(CSRF_TOKEN)
        .context("CSRF token not found in session")?
        .to_owned();

    store
        .destroy_session(session)
        .await
        .context("Failed to destroy old session")?;

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

    let token = oauth_client
        .exchange_code(AuthorizationCode::new(query.code.clone()))
        .request_async(&client)
        .await
        .context("failed in sending request to authorization server")?;

    let user_data: User = client
        .get("https://discordapp.com/api/users/@me")
        .bearer_auth(token.access_token().secret())
        .send()
        .await
        .context("failed in sending request to target Url")?
        .json::<User>()
        .await
        .context("failed to deserialize response as JSON")?;

    let mut session = Session::new();
    session
        .insert("user", &user_data)
        .context("failed in inserting serialized value into session")?;

    let cookie = store
        .store_session(session)
        .await
        .context("failed to store session")?
        .context("unexpected error retrieving cookie value")?;

    let cookie = format!("{COOKIE_NAME}={cookie}; SameSite=Lax; HttpOnly; Secure; Path=/");

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

#[derive(Debug)]
struct AppError(anyhow::Error);

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
````

## 运行

先去 Discord Developer Portal 创建 application，拿 `CLIENT_ID` 和 `CLIENT_SECRET`，OAuth2 tab 加 redirect URI `http://127.0.0.1:3000/auth/authorized`。

````bash
CLIENT_ID=xxx CLIENT_SECRET=yyy cargo run -p example-oauth
````

浏览器访问 `http://127.0.0.1:3000/`，点提示去 `/auth/discord`，跳到 Discord 授权，授权后回到首页看到 "Hey xxx! You're logged in!"。

## 解读

### OAuth 流程全貌

```text
浏览器                    axum server              Discord OAuth
  │                          │                          │
  │ 1. GET /auth/discord     │                          │
  │ ─────────────────────────>│                          │
  │                          │                          │
  │  302 + Set-Cookie        │                          │
  │ <─────────────────────────│                          │
  │                                                     │
  │ 2. GET discord.com/oauth2/authorize?client_id=... ───>│
  │                          │                          │
  │  3. 授权页（用户点同意）                              │
  │ <─────────────────────────────────────────────────── │
  │                                                     │
  │ 4. 用户同意后，302 跳转回                              │
  │  /auth/authorized?code=xxx&state=yyy                │
  │ <─────────────────────────────────────────────────── │
  │                                                     │
  │ 5. GET /auth/authorized?code=xxx&state=yyy          │
  │ ─────────────────────────>│                          │
  │                          │  6. POST token URL       │
  │                          │ ──────────────────────────>│
  │                          │  7. 返回 access_token     │
  │                          │ <──────────────────────────│
  │                          │  8. GET /users/@me        │
  │                          │ ──────────────────────────>│
  │                          │  9. 返回 user data        │
  │                          │ <──────────────────────────│
  │                          │                          │
  │  10. Set-Cookie + 302 / │                          │
  │ <─────────────────────────│                          │
```

### OAuth vs JWT

| 维度 | OAuth（这章） | JWT（ch38） |
| --- | --- | --- |
| 登录方式 | 跳第三方授权 | 自己发 token |
| session | server 存（有状态） | token 自带（无状态） |
| 用户信息来源 | 第三方 API | 自己签发进 token |
| 适用场景 | "用 GitHub 登录" | 自己的账号体系 |

## 常见问题

**`MemoryStore` 能生产用吗？** 不能。重启就丢，多实例不共享。生产用 Redis-backed session store。

**为什么要 CSRF token？** 防止攻击者把自己的 authorization code 塞进受害者浏览器，让受害者登录成攻击者账号（然后攻击者通过自己的账号看到受害者的操作记录）。

**为什么不存 access_token 到 session 而存 user？** 这章只需要登录态识别（拿用户信息），不需要后续调 Discord API。如果要代表用户操作，存 access_token + refresh_token。

**cookie 的 `Secure` 标志会让本地测试失败吗？** `Secure` 表示只在 HTTPS 发送。本例 HTTP 测试用浏览器 dev tools 手动加 cookie，或换 HTTPS 代理。

## 手写任务

1. 把 Discord 换成 GitHub：改 AUTH_URL/TOKEN_URL/redirect_uri，scope 改 `user:email`，API 换 `https://api.github.com/user`。
2. 加 `/logout` 路由清 session（其实源码已有，自己写一遍）。
3. 加 middleware 在每个响应里加 `X-User-Id` 头（如果有登录用户）。
4. 改成 Redis session store（替换 MemoryStore）。

## 小结

这章用 3 步讲了 OAuth 完整流程：

1. **State 骨架 + 错误类型**：`AppState` 含 `MemoryStore` + `BasicClient`，`FromRef` 暴露 sub-state，`AppError` 桥接 anyhow。
2. **OAuth 流程**：`/auth/discord` 生成 URL + CSRF + Set-Cookie；`/auth/authorized` 验证 CSRF → 换 token → 拉 user → 新 session。
3. **自定义 User extractor**：`FromRequestParts` 没登录就 AuthRedirect，`OptionalFromRequestParts` 返回 None 容忍。

核心：OAuth 是协议，4 步流程对所有 provider 通用；CSRF token 防 token injection；session cookie 安全属性（SameSite/HttpOnly/Secure）不能少；自定义 extractor 把"未登录跳转"封装成 rejection 类型。

## 源码对照

- `examples/oauth/Cargo.toml`
- `examples/oauth/src/main.rs`
