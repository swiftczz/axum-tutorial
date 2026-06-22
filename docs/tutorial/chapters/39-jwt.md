# 39. jwt

对应示例：`examples/jwt`

进入认证章节。理解 JWT 登录授权的基本流程,在 axum 中用自定义 extractor 验证 `Authorization: Bearer ...` token。认证代码一定要先建立正确心智模型,否则容易把"拿到 token"和"信任用户"混在一起。

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

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

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
        let body = Json(json!({
            "error": error_message,
        }));
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

#[derive(Debug, Serialize, Deserialize)]
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

必须设置 `JWT_SECRET`:

````bash
JWT_SECRET=secret cargo run -p example-jwt
````

获取 token:

````bash
curl -s \
  -H 'content-type: application/json' \
  -d '{"client_id":"foo","client_secret":"bar"}' \
  http://127.0.0.1:3000/authorize
# {"access_token":"...","token_type":"Bearer"}
````

把 token 放进变量访问受保护接口:

````bash
TOKEN='替换成 access_token'
curl -s -H "Authorization: Bearer $TOKEN" http://127.0.0.1:3000/protected
````

无效 token:

````bash
curl -s -H 'Authorization: Bearer blahblahblah' http://127.0.0.1:3000/protected
# {"error":"Invalid token"}
````

## 解读

### JWT 是什么

JWT 是一段被签名的 JSON 数据,三部分用 `.` 分隔:

```text
header    = {"alg":"HS256","typ":"JWT"}                    算法
payload   = {"sub":"b@b.com","company":"ACME","exp":...}    claims(业务数据)
signature = HMAC-SHA256(base64(header) + "." + base64(payload), secret)
```

签名让服务端验证 token 是否由自己签发且未被篡改。**JWT 不是加密**——payload 通常可被解码查看,不要把密码/密钥/银行卡号放进 JWT。

### 对称签名 vs 非对称签名

本例用 **HS256**(HMAC + SHA-256),对称签名:签发和验证用同一个 secret。

| 算法 | 密钥 | 谁能验证 | 适合 |
| --- | --- | --- | --- |
| **HS256** | 一个共享 secret | 任何持有 secret 的人 | 单体应用(签发=验证) |
| **RS256** | 公私钥(私钥签发,公钥验证) | 任何拿公钥的人 | 多服务(签发保密私钥,其他用公钥) |
| **ES256** | 椭圆曲线公私钥 | 同 RS256 | 同 RS256,密钥更小更快 |

单服务用 HS256 最简单;多服务验证同一 token(认证中心签发,微服务验证)用 RS256——签发服务持私钥,其他服务只拿公钥,即使某服务被攻破也无法伪造。

### ⚠️ JWT 最重要的心智:签发后无法主动销毁

这是 JWT 和 session 最大区别:

```text
session:服务端存会话记录,想注销直接删记录
JWT:    服务端不存任何东西,token 一旦签发在过期前一直有效,服务端无法单方面让它失效
```

JWT 的"无状态"是优势(不需 session 存储、天然适合分布式),但也是代价——**不能像 session 那样"一键注销"**。补救手段:

1. **短 exp + refresh token**:access token 有效期短(如 15 分钟),用 refresh token 换新的。想注销就不再发 refresh。
2. **服务端黑名单**:把要作废的 token id 存 Redis,每次验证查一下(牺牲"无状态")。
3. **token 版本号**:用户表存 `token_version`,改密码 +1,claims 带版本号,不匹配就拒。

理解这个本质,你就知道为什么 JWT 要配合"合理过期时间 + 刷新机制 + 撤销策略"——这不是可有可无,而是 JWT 机制本身的必要补充。

### Bearer token

```text
Authorization: Bearer <token>
```

Bearer 意思是"谁持有这个 token,谁就可以用它访问对应资源"。token 泄露就很危险,真实项目要配合 HTTPS、合理过期、刷新机制、撤销策略。

### 认证流程两步

```text
登录/授权(/authorize):验证身份 → 签发 token
保护接口(/protected):提取 Bearer token → 验证 → 得到 Claims
```

`/authorize` 验证 client_id/client_secret(本例写死 `foo`/`bar`,真实项目查数据库 + 密码哈希),构造 Claims,用 `JWT_SECRET` 签发 token 返回 `{"access_token":"...","token_type":"Bearer"}`。

### `Keys` + `LazyLock`

````rust
static KEYS: LazyLock<Keys> = LazyLock::new(|| {
    let secret = std::env::var("JWT_SECRET").expect("JWT_SECRET must be set");
    Keys::new(secret.as_bytes())
});
````

`LazyLock` 第一次使用 `KEYS` 时才初始化,整个程序共享同一组 key。没设 `JWT_SECRET` 直接 panic。`Keys` 用同一个 secret 创建 `EncodingKey`(签发)和 `DecodingKey`(验证)——对称签名。生产 secret 必须足够长、足够随机,不能用示例的 `"secret"`。

### Claims

````rust
#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,      // subject,用户身份
    company: String,  // 业务字段
    exp: usize,       // 过期时间(Unix 时间戳)
}
````

`Serialize` 生成 token,`Deserialize` 解析 token。`exp` 很重要——`jsonwebtoken` 默认验证会检查 token 是否过期。本例 `exp: 2000000000` 写死到 2033 年只为演示,真实项目按当前时间动态设较短过期。

### Claims extractor(本章核心)

````rust
impl<S> FromRequestParts<S> for Claims
where S: Send + Sync,
{
    type Rejection = AuthError;
    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let TypedHeader(Authorization(bearer)) = parts
            .extract::<TypedHeader<Authorization<Bearer>>>().await
            .map_err(|_| AuthError::InvalidToken)?;
        let token_data = decode::<Claims>(bearer.token(), &KEYS.decoding, &Validation::default())
            .map_err(|_| AuthError::InvalidToken)?;
        Ok(token_data.claims)
    }
}
````

让 `Claims` 成为 axum extractor。步骤:从 header 提取 `Authorization: Bearer ...` → 取 bearer token → `decode::<Claims>` 验证签名+过期+解析 → 成功返回 Claims,失败返回 `AuthError::InvalidToken`。

**为什么 `FromRequestParts`?** 验证 JWT 只需请求 header,不需读 body。

**为什么 handler 参数是 `claims: Claims`?** 只有 extractor 成功 handler 才执行。没 token / token 无效 / 过期,axum 直接返回 `AuthError`,不进 handler。这是很 axum 的写法:**认证逻辑不写在 handler 正文,写成 extractor**,handler 只处理认证成功后的业务。

### TypedHeader 解析

````rust
let TypedHeader(Authorization(bearer)) = parts
    .extract::<TypedHeader<Authorization<Bearer>>>().await.map_err(...)?;
````

从请求头取 `Authorization` 且要求格式是 Bearer token。这些都会失败:没 Authorization header、`Authorization: Basic xxx`、`Authorization: Bearer`(没 token)。

### AuthError → HTTP 响应

````rust
impl IntoResponse for AuthError {
    fn into_response(self) -> Response {
        let (status, msg) = match self {
            AuthError::WrongCredentials => (StatusCode::UNAUTHORIZED, "Wrong credentials"),
            AuthError::MissingCredentials => (StatusCode::BAD_REQUEST, "Missing credentials"),
            AuthError::TokenCreation => (StatusCode::INTERNAL_SERVER_ERROR, "Token creation error"),
            AuthError::InvalidToken => (StatusCode::BAD_REQUEST, "Invalid token"),
        };
        (status, Json(json!({"error": msg}))).into_response()
    }
}
````

handler 返回 `Result<_, AuthError>`,axum 自动把错误变 HTTP 响应,格式统一 `{"error":"..."}`。⚠️ example 把 InvalidToken 返回 400,**真实项目更常见是 401 Unauthorized**(可能带 `WWW-Authenticate` header)。

## 常见问题

**JWT_SECRET 能用 `secret` 吗?** 只能教学。生产必须足够长、随机、保密,通过环境变量或密钥系统管理。

**JWT payload 加密吗?** 不,JWT 是签名不是加密。不要在 claims 放密码或敏感信息。

**为什么 protected 里没手动验证 token?** 验证在 `Claims` extractor 里,extractor 成功才进 handler。

**InvalidToken 为什么 400?** example 选择,真实项目更常见 401 + `WWW-Authenticate` header。

**exp 怎么设?** 本例写死到 2033 年只为演示,真实项目按当前时间动态设较短过期 + refresh token。

## 手写任务

按下面顺序敲:

1. 定义 `Claims` 含 `sub` 和 `exp`。
2. 定义 `Keys`,从 `JWT_SECRET` 创建 encoding/decoding key。
3. 写 `/authorize`,先用写死账号密码生成 token。
4. 写 `AuthBody` 返回 `access_token` 和 `Bearer`。
5. 给 `Claims` 实现 `FromRequestParts`。
6. 从 `Authorization: Bearer` header 提取 token。
7. `decode::<Claims>` 验证 token。
8. 写 `/protected` 参数直接用 `claims: Claims`。

加深练习:

1. 无效 token 状态码改 401。
2. 写死账号密码改成从数据库查询用户。
3. `exp` 改成基于当前时间动态生成。

## 小结

- JWT 认证核心两步:授权接口验证身份签发 token,保护接口提取 Bearer token 验证得 Claims。
- JWT 是签名不是加密,payload 可解码,不放敏感信息;对称签名(HS256)适合单体,非对称(RS256)适合多服务。
- **JWT 签发后无法主动销毁**(和 session 最大区别),必须配合短 exp + refresh token / 黑名单 / token 版本号。
- 很 axum 的写法:验证逻辑写成 `Claims` extractor,handler 只处理认证成功后的业务,认证失败统一走 `AuthError::into_response`。
- `FromRequestParts` 合适(只读 header 不读 body);`TypedHeader<Authorization<Bearer>>` 解析 Bearer token。

## 源码对照

- `examples/jwt/Cargo.toml`
- `examples/jwt/src/main.rs`
