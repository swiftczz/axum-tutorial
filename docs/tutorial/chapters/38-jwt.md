# 38. jwt

对应示例：`examples/jwt`

从这章开始进入认证。JWT（JSON Web Token）是"服务端自己签发 token"的方案：客户端提交账号密码 → 服务端签发 token → 客户端带 token 访问受保护接口。

我们分 4 步从零搭出来。相比前面章节新引入：`jsonwebtoken` crate、`Claims` 结构体、JWT 的 encode/decode、`TypedHeader<Authorization<Bearer>>` 提取器。

## Cargo.toml

````toml
[package]
name = "example-jwt"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
axum-extra = { version = "0.12", features = ["typed-header"] }
jsonwebtoken = { version = "10", features = ["aws_lc_rs"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
````

本章相比前面章节新增：`jsonwebtoken`（JWT 签发和验证，第 38 章第一次出现）和 `axum-extra` 的 `typed-header` feature（类型安全地提取 `Authorization` 请求头，比手写 `headers.get("authorization")` 安全）。`axum-extra` 是 axum 官方的扩展 crate，提供 `TypedHeader`、`WithRejection`（ch11 用过）等额外 extractor。

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

---

## 第一步：理解 JWT 和密钥

JWT 是一段被签名的 JSON 数据，三部分用 `.` 分隔：

```text
header.payload.signature
```

- `header`：算法（如 HS256）
- `payload`：业务数据（叫 Claims，如用户 ID、过期时间）
- `signature`：用密钥对 header+payload 签名，防篡改

签发和验证用同一个密钥（对称签名 HS256）。程序启动时从 `JWT_SECRET` 环境变量读密钥：

````rust
use jsonwebtoken::{DecodingKey, EncodingKey};
use std::sync::LazyLock;

static KEYS: LazyLock<Keys> = LazyLock::new(|| {
    let secret = std::env::var("JWT_SECRET").expect("JWT_SECRET must be set");
    Keys::new(secret.as_bytes())
});

struct Keys {
    encoding: EncodingKey,
    decoding: DecodingKey,
}

impl Keys {
    fn new(secret: &[u8]) -> Self {
        Self {
            encoding: EncodingKey::from_secret(secret),  // 签发 token 用
            decoding: DecodingKey::from_secret(secret),  // 验证 token 用
        }
    }
}
````

> **新面孔：`LazyLock` + `EncodingKey`/`DecodingKey`**
>
> `LazyLock` 是 Rust 的延迟初始化类型——第一次用 `KEYS` 时才执行闭包，之后全局共享。这样整个程序只有一组密钥。
>
> `EncodingKey` 用来签发 token，`DecodingKey` 用来验证 token。对称签名下两者用同一个 secret。

---

## 第二步：定义 Claims 和授权接口

Claims 是 token 里携带的数据：

````rust
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Claims {
    sub: String,       // subject，用户身份
    company: String,   // 示例业务字段
    exp: usize,        // 过期时间（Unix 时间戳）
}
````

> **新面孔：JWT Claims**
>
> `sub`（subject）是 JWT 标准字段，表示"这个 token 属于谁"。`exp`（expiration）也是标准字段，`jsonwebtoken` 库默认验证 token 是否过期。你可以加任意自定义字段（如 `company`）。
>
> `Serialize` 用于生成 token（Claims → JSON → 签名），`Deserialize` 用于解析 token（签名验证 → JSON → Claims）。

授权接口接收账号密码，验证后签发 token：

````rust
use axum::{http::StatusCode, response::IntoResponse, routing::post, Json, Router};
use serde::Deserialize;

#[derive(Deserialize)]
struct AuthPayload {
    client_id: String,
    client_secret: String,
}

#[derive(Serialize)]
struct AuthBody {
    access_token: String,
    token_type: String,
}

async fn authorize(Json(payload): Json<AuthPayload>) -> Result<Json<AuthBody>, AuthError> {
    // 检查账号密码（example 写死，真实项目查数据库）
    if payload.client_id.is_empty() || payload.client_secret.is_empty() {
        return Err(AuthError::MissingCredentials);
    }
    if payload.client_id != "foo" || payload.client_secret != "bar" {
        return Err(AuthError::WrongCredentials);
    }

    // 构造 Claims
    let claims = Claims {
        sub: "b@b.com".to_owned(),
        company: "ACME".to_owned(),
        exp: 2000000000,  // 过期时间（example 写死）
    };

    // 签发 token
    let token = jsonwebtoken::encode(&Header::default(), &claims, &KEYS.encoding)
        .map_err(|_| AuthError::TokenCreation)?;

    Ok(Json(AuthBody::new(token)))
}
````

> **新面孔：`jsonwebtoken::encode`**
>
> `encode(&Header::default(), &claims, &key)` 把 Claims 序列化成 JSON，用密钥签名，生成 JWT 字符串。客户端拿到这个 token 后，后续请求放在 `Authorization: Bearer <token>` header 里。

---

## 第三步：Claims 作为 extractor——验证 token

最有 axum 特色的部分：把 `Claims` 写成 extractor，handler 参数里写 `claims: Claims` 就自动验证 token。

````rust
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
    response::{IntoResponse, Response},
    RequestPartsExt,
};
use axum_extra::{
    headers::{authorization::Bearer, Authorization},
    TypedHeader,
};

impl<S> FromRequestParts<S> for Claims
where
    S: Send + Sync,
{
    type Rejection = AuthError;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        // 1. 从 header 提取 Authorization: Bearer <token>
        let TypedHeader(Authorization(bearer)) = parts
            .extract::<TypedHeader<Authorization<Bearer>>>()
            .await
            .map_err(|_| AuthError::InvalidToken)?;

        // 2. 验证签名和过期时间，解析出 Claims
        let token_data = decode::<Claims>(bearer.token(), &KEYS.decoding, &Validation::default())
            .map_err(|_| AuthError::InvalidToken)?;

        Ok(token_data.claims)
    }
}
````

> **新面孔：`TypedHeader<Authorization<Bearer>>`**
>
> `TypedHeader` 是 axum-extra 提供的类型安全 header 提取器。`Authorization<Bearer>` 要求请求头是 `Authorization: Bearer <token>` 格式。格式不对或缺 header 时返回错误。
>
> **为什么 `FromRequestParts` 不是 `FromRequest`？** 验证 JWT 只需要请求 header，不需要读 body。`FromRequestParts` 只读 header/路径等 parts，不消费 body。

handler 直接用 `Claims` 参数：

````rust
async fn protected(claims: Claims) -> Result<String, AuthError> {
    Ok(format!("Welcome to the protected area :)\nYour data:\n{claims}"))
}
````

没 token / token 无效 / token 过期 → extractor 返回 `AuthError::InvalidToken` → handler 根本不会执行。

---

## 第四步：AuthError 和路由注册

定义错误类型并实现 `IntoResponse`：

````rust
#[derive(Debug)]
enum AuthError {
    WrongCredentials,
    MissingCredentials,
    TokenCreation,
    InvalidToken,
}

impl IntoResponse for AuthError {
    fn into_response(self) -> Response {
        let (status, error_message) = match self {
            AuthError::WrongCredentials => (StatusCode::UNAUTHORIZED, "Wrong credentials"),
            AuthError::MissingCredentials => (StatusCode::BAD_REQUEST, "Missing credentials"),
            AuthError::TokenCreation => (StatusCode::INTERNAL_SERVER_ERROR, "Token creation error"),
            AuthError::InvalidToken => (StatusCode::BAD_REQUEST, "Invalid token"),
        };
        let body = Json(serde_json::json!({ "error": error_message }));
        (status, body).into_response()
    }
}
````

路由——`/authorize` 是公开接口，`/protected` 需要 Claims：

````rust
let app = Router::new()
    .route("/protected", get(protected))
    .route("/authorize", post(authorize));
````

---

## 完整代码

````rust
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
    response::{IntoResponse, Response},
    routing::{get, post},
    Json, RequestPartsExt, Router,
};
use axum_extra::{
    headers::{authorization::Bearer, Authorization},
    TypedHeader,
};
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use serde_json::json;
use std::fmt::Display;
use std::sync::LazyLock;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

static KEYS: LazyLock<Keys> = LazyLock::new(|| {
    let secret = std::env::var("JWT_SECRET").expect("JWT_SECRET must be set");
    Keys::new(secret.as_bytes())
});

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let app = Router::new()
        .route("/protected", get(protected))
        .route("/authorize", post(authorize));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn protected(claims: Claims) -> Result<String, AuthError> {
    Ok(format!(
        "Welcome to the protected area :)\nYour data:\n{claims}",
    ))
}

async fn authorize(Json(payload): Json<AuthPayload>) -> Result<Json<AuthBody>, AuthError> {
    if payload.client_id.is_empty() || payload.client_secret.is_empty() {
        return Err(AuthError::MissingCredentials);
    }
    if payload.client_id != "foo" || payload.client_secret != "bar" {
        return Err(AuthError::WrongCredentials);
    }
    let claims = Claims {
        sub: "b@b.com".to_owned(),
        company: "ACME".to_owned(),
        exp: 2000000000,
    };
    let token = encode(&Header::default(), &claims, &KEYS.encoding)
        .map_err(|_| AuthError::TokenCreation)?;
    Ok(Json(AuthBody::new(token)))
}

impl Display for Claims {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Email: {}\nCompany: {}", self.sub, self.company)
    }
}

impl AuthBody {
    fn new(access_token: String) -> Self {
        Self {
            access_token,
            token_type: "Bearer".to_string(),
        }
    }
}

impl<S> FromRequestParts<S> for Claims
where
    S: Send + Sync,
{
    type Rejection = AuthError;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let TypedHeader(Authorization(bearer)) = parts
            .extract::<TypedHeader<Authorization<Bearer>>>()
            .await
            .map_err(|_| AuthError::InvalidToken)?;

        let token_data = decode::<Claims>(bearer.token(), &KEYS.decoding, &Validation::default())
            .map_err(|_| AuthError::InvalidToken)?;

        Ok(token_data.claims)
    }
}

impl IntoResponse for AuthError {
    fn into_response(self) -> Response {
        let (status, error_message) = match self {
            AuthError::WrongCredentials => (StatusCode::UNAUTHORIZED, "Wrong credentials"),
            AuthError::MissingCredentials => (StatusCode::BAD_REQUEST, "Missing credentials"),
            AuthError::TokenCreation => (StatusCode::INTERNAL_SERVER_ERROR, "Token creation error"),
            AuthError::InvalidToken => (StatusCode::BAD_REQUEST, "Invalid token"),
        };
        let body = Json(json!({ "error": error_message }));
        (status, body).into_response()
    }
}

struct Keys {
    encoding: EncodingKey,
    decoding: DecodingKey,
}

impl Keys {
    fn new(secret: &[u8]) -> Self {
        Self {
            encoding: EncodingKey::from_secret(secret),
            decoding: DecodingKey::from_secret(secret),
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Claims {
    sub: String,
    company: String,
    exp: usize,
}

#[derive(Debug, Serialize)]
struct AuthBody {
    access_token: String,
    token_type: String,
}

#[derive(Debug, Deserialize)]
struct AuthPayload {
    client_id: String,
    client_secret: String,
}

#[derive(Debug)]
enum AuthError {
    WrongCredentials,
    MissingCredentials,
    TokenCreation,
    InvalidToken,
}
````

## 运行

必须设置 `JWT_SECRET`：

````bash
JWT_SECRET=secret cargo run -p example-jwt
````

获取 token：

````bash
curl -s -H 'content-type: application/json' \
  -d '{"client_id":"foo","client_secret":"bar"}' \
  http://127.0.0.1:3000/authorize
# {"access_token":"...","token_type":"Bearer"}
````

访问受保护接口：

````bash
TOKEN='替换成 access_token'
curl -s -H "Authorization: Bearer $TOKEN" http://127.0.0.1:3000/protected
````

## 手写任务

1. 把无效 token 的状态码改成 401。
2. 把写死账号密码改成从数据库查询用户。
3. `exp` 改成基于当前时间动态生成。

## 小结

这章分 4 步搭建了 JWT 认证：

1. **JWT 和密钥**：`LazyLock` 全局持有 `EncodingKey`/`DecodingKey`，对称签名。
2. **Claims 和授权**：`/authorize` 验证账号密码，`encode` 签发 token。
3. **Claims extractor**：`FromRequestParts` 实现，从 `Authorization: Bearer` header 提取并 `decode` 验证。handler 参数写 `claims: Claims` 就自动认证。
4. **AuthError**：统一错误类型，`IntoResponse` 转成 JSON 响应。

核心模式：**认证逻辑不写在 handler 正文，写成 extractor**。handler 只处理认证成功后的业务。

## 源码对照

- `examples/jwt/Cargo.toml`
- `examples/jwt/src/main.rs`
