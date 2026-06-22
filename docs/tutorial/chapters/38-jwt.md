# 38. jwt

对应示例：`examples/jwt`

本章目标：理解 JWT 登录授权的基本流程，并学会在 Axum 中用自定义 extractor 验证 `Authorization: Bearer ...` token。

从这一章开始进入认证。  
认证相关代码一定要先建立正确心智模型，否则很容易把“拿到 token”和“信任用户”混在一起。

## 这个小项目在做什么

应用有两个接口：

```text
POST /authorize -> 提交 client_id/client_secret，换取 JWT
GET  /protected -> 携带 JWT，访问受保护内容
```

认证流程是：

```text
客户端 POST /authorize
-> 服务端检查账号密码
-> 服务端创建 Claims
-> 用 JWT_SECRET 签名生成 token
-> 返回 access_token

客户端 GET /protected
-> 请求头带 Authorization: Bearer <token>
-> Claims extractor 从 header 提取 token
-> 验证签名和过期时间
-> 解码出 Claims
-> handler 使用 Claims 返回受保护内容
```

## 先理解 JWT 是什么

JWT 可以先理解成：

```text
一段被签名的 JSON 数据
```

它通常包含三部分：

```text
header.payload.signature
```

每一部分是用 `.` 分隔的 base64 字符串。展开看：

```text
header    = {"alg":"HS256","typ":"JWT"}            // 用什么算法签名
payload   = {"sub":"b@b.com","company":"ACME","exp":...}  // claims（业务数据）
signature = HMAC-SHA256(base64(header) + "." + base64(payload), secret)
```

payload 里放的就是 claims。  
例如本章的 claims：

```json
{
  "sub": "b@b.com",
  "company": "ACME",
  "exp": 2000000000
}
```

签名的作用是：

```text
客户端可以拿着 token
服务端可以验证这个 token 是否由自己签发且没有被篡改
```

JWT 不是加密。  
payload 通常可以被解码查看，所以不要把密码、密钥、银行卡号这类敏感信息放进 JWT。

### 对称签名（HS256）vs 非对称签名（RS256/ES256）

本例用的是 **HS256**（HMAC + SHA-256），属于**对称签名**：签发和验证用的是同一个 `secret`。

| 算法 | 密钥 | 谁能验证 | 适合场景 |
| --- | --- | --- | --- |
| **HS256** | 一个共享 secret | 任何持有 secret 的人 | 单体应用（签发方=验证方） |
| **RS256** | 公私钥对（私钥签发，公钥验证） | 任何拿到公钥的人 | 多服务（签发方保密私钥，其他服务用公钥验证） |
| **ES256** | 椭圆曲线公私钥对 | 同 RS256 | 同 RS256，密钥更小更快 |

怎么选？如果你的后端只有一个服务，HS256 最简单；如果有**多个服务**需要验证同一个 token（比如认证中心签发，其他微服务验证），用 RS256——签发服务持有私钥，其他服务只拿公钥，即使某个服务被攻破也无法伪造 token。

### ⚠️ JWT 最重要的心智：签发后无法主动销毁

这是 JWT 和 session 最大的区别，也是初学者最容易忽略的点：

```text
session：服务端存着会话记录，想注销直接删掉记录即可。
JWT：    服务端不存任何东西，token 一旦签发，在过期前一直有效，服务端无法单方面让它失效。
```

JWT 的"无状态"是它的优势（不需要 session 存储、天然适合分布式），但也是它的代价——**你不能像 session 那样"一键注销"**。常见补救手段：

1. **短 exp + refresh token**：access token 有效期很短（如 15 分钟），用 refresh token 换新的。想注销就不再发 refresh token。
2. **服务端黑名单**：把要作废的 token id 存进 Redis，每次验证都查一下。但这牺牲了"无状态"——又回到了需要存储。
3. **token 版本号**：在用户表里存一个 `token_version`，改密码时 +1，claims 里带版本号，版本不匹配就拒绝。

理解了这个本质，你就知道为什么第 86 行说"要配合合理过期时间、刷新机制、撤销策略"——这不是可有可无的安全设计，而是 JWT 机制本身的必要补充。

## Bearer token 是什么

访问受保护接口时，请求头通常写成：

```text
Authorization: Bearer <token>
```

Bearer 的意思是：

```text
谁持有这个 token，谁就可以用它访问对应资源
```

所以 token 泄露就很危险。  
真实项目里要配合 HTTPS、合理过期时间、刷新机制、撤销策略等安全设计。

## 文件和依赖

这个 example 有两个主要文件：

1. `examples/jwt/Cargo.toml`：声明 Axum、axum-extra typed-header、jsonwebtoken、serde。
2. `examples/jwt/src/main.rs`：实现授权接口、保护接口、JWT claims extractor 和错误响应。

关键依赖：

- `jsonwebtoken`：创建和验证 JWT。
- `axum-extra`：提供 typed header extractor，用于解析 `Authorization: Bearer`。
- `serde` / `serde_json`：claims、请求体和错误响应 JSON。
- `axum`：提供 Router、Json、自定义 extractor、IntoResponse。
- `tokio`：异步运行时。

`Cargo.toml` 里：

````toml
axum-extra = { path = "../../axum-extra", features = ["typed-header"] }
jsonwebtoken = { version = "10", features = ["aws_lc_rs"] }
````

## 第一步：JWT_SECRET 和 Keys

源码：

````rust
static KEYS: LazyLock<Keys> = LazyLock::new(|| {
    let secret = std::env::var("JWT_SECRET").expect("JWT_SECRET must be set");
    Keys::new(secret.as_bytes())
});
````

JWT 签名需要 secret。  
这个 example 要求通过环境变量设置：

```text
JWT_SECRET
```

如果没设置，程序会直接 panic：

```text
JWT_SECRET must be set
```

`LazyLock` 表示第一次使用 `KEYS` 时才初始化。  
这样整个程序共享同一组 signing keys。

## 第二步：EncodingKey 和 DecodingKey

源码：

````rust
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
````

这里用同一个 secret 创建两把 key：

- `encoding`：签发 token 时用。
- `decoding`：验证 token 时用。

本例使用的是对称签名算法。  
签发和验证使用同一个 secret。

真实项目里 secret 必须足够长、足够随机，不能用示例里的 `"secret"` 当生产密钥。

## 第三步：定义 Claims

源码：

````rust
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Claims {
    sub: String,
    company: String,
    exp: usize,
}
````

Claims 是 token 里携带的数据。

本章有三个字段：

- `sub`：subject，通常表示用户身份。
- `company`：示例业务字段。
- `exp`：过期时间，Unix 时间戳。

`Serialize` 用于生成 token。  
`Deserialize` 用于解析 token。

`exp` 很重要。  
`jsonwebtoken` 的默认验证会检查 token 是否过期。

## 第四步：定义登录请求和响应

请求体：

````rust
#[derive(Debug, Deserialize)]
struct AuthPayload {
    client_id: String,
    client_secret: String,
}
````

响应体：

````rust
#[derive(Debug, Serialize)]
struct AuthBody {
    access_token: String,
    token_type: String,
}
````

客户端调用 `/authorize` 时提交：

````json
{"client_id":"foo","client_secret":"bar"}
````

服务端返回：

````json
{"access_token":"...","token_type":"Bearer"}
````

## 第五步：授权接口 authorize

源码：

````rust
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
````

流程是：

1. 从 JSON 请求体解析账号密码。
2. 检查是否为空。
3. 检查是否等于示例账号 `foo` / `bar`。
4. 构造 claims。
5. 用 `encode` 生成 JWT。
6. 返回 token。

示例里账号密码写死是为了教学。  
真实项目应该从数据库查用户，并使用密码哈希验证，不能明文比较密码。

## 第六步：AuthBody::new

源码：

````rust
impl AuthBody {
    fn new(access_token: String) -> Self {
        Self {
            access_token,
            token_type: "Bearer".to_string(),
        }
    }
}
````

这只是一个小构造函数。  
它把 token 包成统一响应格式：

```json
{
  "access_token": "...",
  "token_type": "Bearer"
}
```

客户端后续要把 `access_token` 放进请求头：

```text
Authorization: Bearer <access_token>
```

## 第七步：受保护接口 protected

源码：

````rust
async fn protected(claims: Claims) -> Result<String, AuthError> {
    Ok(format!(
        "Welcome to the protected area :)\nYour data:\n{claims}",
    ))
}
````

这个 handler 的参数是：

````rust
claims: Claims
````

这非常关键。  
它表示：

```text
只有 Claims extractor 成功，handler 才会执行
```

如果请求没有 token、token 无效、token 过期，Axum 会直接返回 `AuthError`，不会进入 handler。

## 第八步：给 Claims 实现 FromRequestParts

源码：

````rust
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
````

这段让 `Claims` 成为 Axum extractor。

步骤是：

1. 从请求头提取 `Authorization: Bearer ...`。
2. 取出 bearer token 字符串。
3. 用 `decode::<Claims>` 验证 token 并解析 claims。
4. 成功则返回 `Claims`。
5. 失败则返回 `AuthError::InvalidToken`。

为什么用 `FromRequestParts`？

因为验证 JWT 只需要请求 header，不需要读取 request body。

## 第九步：TypedHeader 解析 Authorization

源码：

````rust
let TypedHeader(Authorization(bearer)) = parts
    .extract::<TypedHeader<Authorization<Bearer>>>()
    .await
    .map_err(|_| AuthError::InvalidToken)?;
````

这行看起来复杂，但意思很简单：

```text
从请求头里取 Authorization
并且要求格式是 Bearer token
```

如果请求头缺失或格式不对，就返回 invalid token。

比如这些都会失败：

```text
没有 Authorization header
Authorization: Basic xxx
Authorization: Bearer
```

## 第十步：AuthError 转成 HTTP 响应

源码：

````rust
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
````

这让 handler 可以返回：

````rust
Result<_, AuthError>
````

Axum 会自动把错误变成 HTTP 响应。

错误响应格式统一是：

````json
{"error":"..."}
````

## 第十一步：注册路由

源码：

````rust
let app = Router::new()
    .route("/protected", get(protected))
    .route("/authorize", post(authorize));
````

`/authorize` 是公开接口，用来拿 token。  
`/protected` 是受保护接口，因为 handler 参数里要求 `Claims`。

这是一种很 Axum 的写法：

```text
认证逻辑不写在 handler 正文里
而是写成 extractor
```

handler 只处理“认证已经成功之后”的业务。

## 函数职责速查

- `main`：初始化日志，注册授权接口和保护接口，启动服务。
- `authorize`：验证示例账号密码，签发 JWT。
- `protected`：需要有效 Claims 才能访问，返回受保护内容。
- `Claims::from_request_parts`：从 Authorization header 提取并验证 JWT。
- `AuthError::into_response`：把认证错误转成 JSON HTTP 响应。
- `Keys::new`：用 secret 创建编码和解码 key。
- `AuthBody::new`：构造 token 响应体。

## 带中文注释的手写版

````rust
//! JWT 认证示例。
//!
//! ```not_rust
//! JWT_SECRET=secret cargo run -p example-jwt
//! ```

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

// 全局 JWT keys。第一次使用时从 JWT_SECRET 环境变量初始化。
static KEYS: LazyLock<Keys> = LazyLock::new(|| {
    let secret = std::env::var("JWT_SECRET").expect("JWT_SECRET must be set");
    Keys::new(secret.as_bytes())
});

#[tokio::main]
async fn main() {
    // 初始化日志。
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 注册公开授权接口和受保护接口。
    let app = Router::new()
        .route("/protected", get(protected))
        .route("/authorize", post(authorize));

    // 启动服务。
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

// 受保护接口。只有 Claims extractor 验证成功后才会进入这里。
async fn protected(claims: Claims) -> Result<String, AuthError> {
    Ok(format!(
        "Welcome to the protected area :)\nYour data:\n{claims}",
    ))
}

// 授权接口。提交 client_id/client_secret，换取 JWT。
async fn authorize(Json(payload): Json<AuthPayload>) -> Result<Json<AuthBody>, AuthError> {
    // 检查是否缺少账号密码。
    if payload.client_id.is_empty() || payload.client_secret.is_empty() {
        return Err(AuthError::MissingCredentials);
    }

    // 教学示例：写死账号密码。
    // 真实项目应该查数据库，并校验密码哈希。
    if payload.client_id != "foo" || payload.client_secret != "bar" {
        return Err(AuthError::WrongCredentials);
    }

    // 构造 token 里的 claims。
    let claims = Claims {
        sub: "b@b.com".to_owned(),
        company: "ACME".to_owned(),
        exp: 2000000000,
    };

    // 用 secret 签发 JWT。
    let token = encode(&Header::default(), &claims, &KEYS.encoding)
        .map_err(|_| AuthError::TokenCreation)?;

    Ok(Json(AuthBody::new(token)))
}

// 让 Claims 可以被格式化成字符串。
impl Display for Claims {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Email: {}\nCompany: {}", self.sub, self.company)
    }
}

// 构造授权响应体。
impl AuthBody {
    fn new(access_token: String) -> Self {
        Self {
            access_token,
            token_type: "Bearer".to_string(),
        }
    }
}

// 让 Claims 成为 Axum extractor。
impl<S> FromRequestParts<S> for Claims
where
    S: Send + Sync,
{
    type Rejection = AuthError;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        // 从 Authorization header 中提取 Bearer token。
        let TypedHeader(Authorization(bearer)) = parts
            .extract::<TypedHeader<Authorization<Bearer>>>()
            .await
            .map_err(|_| AuthError::InvalidToken)?;

        // 验证 token，并解码 Claims。
        let token_data = decode::<Claims>(bearer.token(), &KEYS.decoding, &Validation::default())
            .map_err(|_| AuthError::InvalidToken)?;

        Ok(token_data.claims)
    }
}

// 把认证错误转换成 HTTP JSON 响应。
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

// JWT 编码和解码 key。
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

// JWT 中携带的数据。
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Claims {
    sub: String,
    company: String,
    exp: usize,
}

// 授权接口响应体。
#[derive(Debug, Serialize)]
struct AuthBody {
    access_token: String,
    token_type: String,
}

// 授权接口请求体。
#[derive(Debug, Deserialize)]
struct AuthPayload {
    client_id: String,
    client_secret: String,
}

// 认证相关错误。
#[derive(Debug)]
enum AuthError {
    WrongCredentials,
    MissingCredentials,
    TokenCreation,
    InvalidToken,
}
````

## 运行和验证

启动时必须设置 `JWT_SECRET`：

````bash
JWT_SECRET=secret cargo run -p example-jwt
````

获取 token：

````bash
curl -s \
  -H 'content-type: application/json' \
  -d '{"client_id":"foo","client_secret":"bar"}' \
  http://127.0.0.1:3000/authorize
````

响应类似：

````json
{"access_token":"...","token_type":"Bearer"}
````

把 token 放进变量后访问受保护接口：

````bash
TOKEN='这里替换成 access_token'

curl -s \
  -H "Authorization: Bearer $TOKEN" \
  http://127.0.0.1:3000/protected
````

无效 token：

````bash
curl -s \
  -H 'Authorization: Bearer blahblahblah' \
  http://127.0.0.1:3000/protected
````

预期返回：

````json
{"error":"Invalid token"}
````

## 常见卡点

### 1. JWT_SECRET 可以用 secret 吗？

只能教学使用。  
生产环境必须使用足够长、随机、保密的 secret，并通过环境变量或密钥系统管理。

### 2. JWT payload 是加密的吗？

不是。  
JWT 是签名，不是默认加密。不要在 claims 里放密码或敏感信息。

### 3. 为什么 protected 里没有手动验证 token？

因为验证逻辑在 `Claims` extractor 里。  
只有 extractor 成功，handler 才会执行。

### 4. InvalidToken 为什么返回 400？

这是 example 的选择。  
真实项目更常见的是对无效或缺失 token 返回 401 Unauthorized，并可能带 `WWW-Authenticate` header。

### 5. exp 应该怎么设置？

本例写死到 2033 年 5 月，只是为了演示。  
真实项目应该按当前时间动态设置较短过期时间，并根据需要设计 refresh token。

## 手写任务

建议按下面顺序自己敲一遍：

1. 定义 `Claims`，包含 `sub` 和 `exp`。
2. 定义 `Keys`，从 `JWT_SECRET` 创建 encoding/decoding key。
3. 写 `/authorize`，先用写死账号密码生成 token。
4. 写 `AuthBody` 返回 `access_token` 和 `Bearer`。
5. 给 `Claims` 实现 `FromRequestParts`。
6. 从 `Authorization: Bearer` header 提取 token。
7. 用 `decode::<Claims>` 验证 token。
8. 写 `/protected`，参数直接使用 `claims: Claims`。

加深练习：

1. 把无效 token 的状态码改成 401。
2. 把写死账号密码改成从数据库查询用户。
3. 把 `exp` 改成基于当前时间动态生成。

## 本章真正要记住什么

JWT 认证的核心流程是两步：

```text
登录/授权接口：验证身份 -> 签发 token
保护接口：提取 Bearer token -> 验证 token -> 得到 Claims
```

在 Axum 里，很适合把验证逻辑写成 extractor：

````rust
async fn protected(claims: Claims) -> Result<String, AuthError>
````

这样 handler 只处理认证成功后的业务，认证失败统一走 `AuthError::into_response`。

## 源码对照

本章手写版对应源码：

- `examples/jwt/src/main.rs`
- `examples/jwt/Cargo.toml`
