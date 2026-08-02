# 第 30 章 jwt:无状态认证

## 本章你将做出什么

一个带认证的服务,两条路由:

```
POST /authorize   用凭据换 token
GET  /protected   带 token 才能访问
```

核心是一个 **`Claims` 提取器**——handler 签名里写上它,axum 就自动从 `Authorization` 头部取出 token、验签、解析出用户身份;token 无效则请求根本进不了 handler。

```rust
async fn protected(claims: Claims) -> Result<String, AuthError> {
    // 能进到这里,说明身份已经验过了
}
```

学完本章你将掌握:

- JWT 的三段结构,签名如何防篡改
- **"无状态"是优点也是最大的坑:签发出去的 token 撤销不了**
- 认证(你是谁)与授权(你能做什么)的区别
- `TypedHeader` 与类型化的 HTTP 头部
- 把身份校验做成提取器的模式

## 前置知识

### 认证与授权

两个词经常混用,但它们是不同的问题:

- **认证**(Authentication)—— **你是谁**。检查用户名密码、验证 token、确认身份。
- **授权**(Authorization)—— **你能做什么**。这个用户能不能删除那篇文章、能不能访问管理后台。

**本章只做认证。** `/protected` 的逻辑是"有合法 token 就能访问",不区分是谁的 token、有没有权限。真实系统里认证之后还要做授权检查。

有意思的是 HTTP 的头部叫 `Authorization`(授权)而状态码 `401` 叫 `Unauthorized`(未授权),但它们实际处理的都是**认证**。而真正表示"授权失败"的状态码是 `403 Forbidden`。**这是 HTTP 规范里一个众所周知的命名失误**,记住实际含义即可:

- `401 Unauthorized` —— 你没证明自己是谁(该去登录)
- `403 Forbidden` —— 我知道你是谁,但你没这个权限

### JWT 的三段结构

**JWT**(JSON Web Token)是一串用点号分成三段的字符串:

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJiQGIuY29tIn0.M3LAZmrzUkXDC1q5mSzFAs
└────────── header ──────────┘ └──── payload ────┘ └───── signature ─────┘
```

**第一段 header** 是 base64 编码的 JSON,说明用什么算法签名:

```json
{"typ":"JWT","alg":"HS256"}
```

**第二段 payload** 也是 base64 的 JSON,装着**声明**(claims)——关于用户的信息:

```json
{"sub":"b@b.com","company":"ACME","exp":2000000000}
```

**第三段 signature** 是对前两段的签名。

**关键认识:前两段只是 base64 编码,不是加密。** 任何人都能解开看到内容:

```bash
echo 'eyJzdWIiOiJiQGIuY29tIn0' | base64 -d
```

所以 **JWT 里绝不能放敏感信息**——密码、身份证号、内部 ID 都不行。它保证的是**完整性**(内容没被篡改),不是**机密性**。

### 签名如何防篡改

签名的计算方式是:

```
signature = HMAC-SHA256(base64(header) + "." + base64(payload), secret)
```

`secret` 只有服务端知道。验证时服务端用同样的 secret 重算一遍,和 token 里的第三段比对:

- 一致 → 内容没被改过
- 不一致 → token 被篡改或伪造

攻击者可以随意修改 payload(比如把 `"role":"user"` 改成 `"role":"admin"`),但**算不出对应的签名**,因为没有 secret。

这就是 JWT "无状态"的基础:**服务端不需要存任何东西,只靠一个 secret 就能验证 token 的真伪。**

`HS256` 是对称签名(签发和验证用同一个 secret)。还有 `RS256` 这类非对称算法:私钥签发、公钥验证,适合"签发方和验证方不是同一个服务"的场景。

### 无状态的代价:token 撤销不了

**这是 JWT 最重要的坑,也是选型时必须想清楚的。**

传统的 session 方案:服务端存一张表记录"哪些 session 有效",用户登出就删掉那条记录,下次请求立刻失效。

JWT 不存任何东西,**验证完全靠签名和 `exp` 字段**。这带来一个直接后果:

**签发出去的 token 在过期前一直有效,你无法让它失效。**

具体场景:

- 用户点了"登出" —— 客户端删掉 token 了,但**那串字符仍然有效**,谁拿到都能用
- 发现账号被盗 —— 改密码也没用,已签发的 token 照样能访问
- 用户被封禁 —— 手里的 token 到期前照样畅通
- 权限被降级 —— token 里的 `role` 是签发时的快照,不会自动更新

**常见的缓解手段**(注意都是"缓解",不是"解决"):

**短过期时间 + refresh token。** access token 只活 15 分钟,过期后用一个长效的 refresh token 换新的。refresh token 存在服务端,可以撤销。这是目前的主流方案——**代价是引入了服务端状态,JWT 的"无状态"优势打了折扣**。

**黑名单。** 把已撤销的 token ID 存进 Redis,验证时查一次。**这直接放弃了无状态**,而且黑名单要保留到 token 自然过期。

**版本号。** 用户表加一个 `token_version` 字段,token 里带上签发时的版本,改密码时递增。验证时查一次数据库比对——**同样引入了状态**。

**什么时候 JWT 合适:** 服务多、需要跨服务传递身份、能接受"撤销有延迟"、token 生命周期短。

**什么时候用 session 更好:** 单体应用、需要立即登出、需要精确控制在线状态。**别因为 JWT 时髦就用它。**

### exp 是必须的

payload 里的 `exp` 是**过期时间**(UTC 时间戳,秒)。

```rust
struct Claims {
    sub: String,
    company: String,
    exp: usize,     // Mandatory expiry time as UTC timestamp
}
```

`jsonwebtoken` 的 `Validation::default()` **默认会校验 `exp`**,而且如果 token 里没有这个字段,验证会直接失败。这是个好的默认值——上面讲过,**没有过期时间的 JWT 等于永久有效的密码**。

JWT 规范定义了一批标准 claim:

| 字段 | 全称 | 含义 |
|---|---|---|
| `sub` | subject | 主体,通常是用户 ID |
| `exp` | expiration | 过期时间 |
| `iat` | issued at | 签发时间 |
| `nbf` | not before | 生效时间 |
| `iss` | issuer | 签发方 |
| `aud` | audience | 接收方 |

除此之外可以放任意自定义字段,本章的 `company` 就是。

### TypedHeader:类型化的头部

从请求里取 `Authorization` 头部有两种写法。

**手动解析:**

```rust
let auth = parts.headers.get("authorization")
    .and_then(|v| v.to_str().ok())
    .and_then(|s| s.strip_prefix("Bearer "))
    .ok_or(AuthError::InvalidToken)?;
```

**用 `TypedHeader`:**

```rust
let TypedHeader(Authorization(bearer)) = parts
    .extract::<TypedHeader<Authorization<Bearer>>>()
    .await
    .map_err(|_| AuthError::InvalidToken)?;
let token = bearer.token();
```

`TypedHeader<T>` 来自 `axum-extra`,`T` 是一个实现了 `headers::Header` 的类型。它把"头部名 + 值的格式"编码进了类型——`Authorization<Bearer>` 意味着"`Authorization` 头部,`Bearer xxx` 格式"。

**好处是解析规则由类型保证**:不用记头部名怎么拼、不用手写 `strip_prefix("Bearer ")`、格式不对时自动拒绝。缺点是类型嵌套比较深,而且要额外依赖 `axum-extra`。

`headers` crate 提供了一批常用头部的类型:`ContentType`、`UserAgent`、`Cookie`、`IfModifiedSince` 等。

### 提取器即认证守卫

**这是本章在 axum 层面的核心模式。**

给 `Claims` 实现 `FromRequestParts`:

```rust
impl<S> FromRequestParts<S> for Claims
where S: Send + Sync
{
    type Rejection = AuthError;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        // 取 token → 验签 → 返回 claims
    }
}
```

于是 handler 只要写:

```rust
async fn protected(claims: Claims) -> ... {
    // 能执行到这里,身份一定验过了
}
```

**这个模式的价值和[第 12 章](12-validator.md#校验放在哪一层)那个 `ValidatedForm` 是同一个道理:把"必须做的检查"编码进类型,而不是依赖每个 handler 记得调用。**

签名里写了 `Claims` 就一定验过身份;没写就一定没验。**要不要认证变成了类型层面的选择,忘不掉。**

对比中间件方案(在 `.layer()` 里做认证),提取器的优势是:

- 哪些路由需要认证一目了然(看签名)
- handler 能直接拿到用户身份,不用从扩展里取
- 不需要考虑中间件顺序

劣势是每个 handler 都要写一遍。路由多时可以两者结合:中间件做粗粒度拦截,提取器提供身份信息。

## 项目搭建

```bash
cargo new jwt
cd jwt
```

```bash
cargo add axum
cargo add axum-extra --features typed-header
cargo add jsonwebtoken --features aws_lc_rs
cargo add serde --features derive
cargo add serde_json
cargo add tokio --features full
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`jsonwebtoken` 的 `aws_lc_rs` feature 指定加密后端。它还支持 `rust_crypto` 等其它后端。

**运行需要设置 `JWT_SECRET` 环境变量**,没有默认值:

```bash
export JWT_SECRET=secret
```

## 分步构建

### 步骤 1:密钥与数据类型

**要做什么**:定义 claims、认证请求/响应类型、错误类型,以及全局密钥。

```rust
// src/main.rs
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::get,
    Json, Router,
};
use jsonwebtoken::{DecodingKey, EncodingKey};
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

    let app = Router::new().route("/", get(|| async { "not implemented yet" }));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
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

impl Display for Claims {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Email: {}\nCompany: {}", self.sub, self.company)
    }
}

#[derive(Debug, Serialize)]
struct AuthBody {
    access_token: String,
    token_type: String,
}

impl AuthBody {
    fn new(access_token: String) -> Self {
        Self {
            access_token,
            token_type: "Bearer".to_string(),
        }
    }
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
```

**`LazyLock` 做全局密钥。** 它是 Rust 1.80 稳定的标准库类型:第一次访问时执行初始化闭包,之后直接用缓存的值,线程安全。

用全局静态而不是 `State` 是个选择:密钥是进程级的、不变的、所有地方都要用,走 `State` 会让每个提取器都要求状态类型。**代价是不好在测试里替换**——[第 16 章](16-dependency-injection.md#依赖注入解决什么问题)讲过依赖注入的价值,这里为了简洁放弃了。

**`.expect("JWT_SECRET must be set")` 让缺环境变量时 panic。** 注意 `LazyLock` 是**懒加载**的——panic 发生在第一次访问 `KEYS` 时,也就是第一个请求到达时,而不是启动时。「代价与边界」会说这个问题。

**`AuthError` 的四个变体各对应一个状态码。** 这是[第 14 章](14-error-handling.md#用一个-enum-收拢应用错误)那套错误处理的直接应用,而且错误响应是 JSON,格式统一。

> **更好的写法**:`AuthError::InvalidToken` 返回的是 `400 Bad Request`,但语义上应该是 **`401 Unauthorized`** —— token 无效意味着"你没证明自己是谁",正是 `401` 的定义。返回 `400` 会让客户端误以为是请求格式问题,而不是去重新登录。规范的客户端会根据 `401` 自动触发刷新 token 的流程,`400` 则不会。这是示例里一个明确的 bug,练习题 1 会让你修。

**跑起来看**:

```bash
export JWT_SECRET=secret
cargo run
```

这一步还没有实际功能,能编译启动即可。

### 步骤 2:签发 token

**要做什么**:实现 `POST /authorize`,用凭据换 token。

```rust
// src/main.rs
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::{get, post},                    // 改动
    Json, Router,
};
use jsonwebtoken::{encode, DecodingKey, EncodingKey, Header};   // 改动:加上 encode、Header
// ...(其余 use 不变)

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    // 改动:挂上 authorize
    let app = Router::new().route("/authorize", post(authorize));

    // ...
}

// 新增
async fn authorize(Json(payload): Json<AuthPayload>) -> Result<Json<AuthBody>, AuthError> {
    // Check if the user sent the credentials
    if payload.client_id.is_empty() || payload.client_secret.is_empty() {
        return Err(AuthError::MissingCredentials);
    }
    // Here you can check the user credentials from a database
    if payload.client_id != "foo" || payload.client_secret != "bar" {
        return Err(AuthError::WrongCredentials);
    }
    let claims = Claims {
        sub: "b@b.com".to_owned(),
        company: "ACME".to_owned(),
        // Mandatory expiry time as UTC timestamp
        exp: 2000000000, // May 2033
    };
    // Create the authorization token
    let token = encode(&Header::default(), &claims, &KEYS.encoding)
        .map_err(|_| AuthError::TokenCreation)?;

    // Send the authorized token
    Ok(Json(AuthBody::new(token)))
}

// ...(其余不变)
```

**`encode(&Header::default(), &claims, &KEYS.encoding)`** 三个参数:签名算法配置(默认 `HS256`)、要放进 payload 的 claims、签名密钥。返回签好名的 token 字符串。

**`Header::default()`** 用 `HS256`。需要非对称签名时改成 `Header::new(Algorithm::RS256)`。

**凭据校验是硬编码的** `foo`/`bar`。注释也写着 "Here you can check the user credentials from a database" —— 真实系统这里要查库并比对密码哈希。

> **更好的写法**:两处问题。一是凭据硬编码,真实系统应该查数据库,而且**密码必须存哈希而非明文**(用 `argon2` 或 `bcrypt`),比对时用库提供的常量时间验证函数。二是 `payload.client_secret != "bar"` 这种**直接字符串比较存在时序攻击风险**——比较耗时会随匹配的前缀长度变化,理论上可被用来逐字符猜测密钥。安全敏感的比较应该用 `subtle` 这类库的常量时间比较。示例这么写是为了让你不配数据库就能跑通流程。

**`exp: 2000000000`** 是 2033 年 5 月的时间戳——**硬编码的固定时间点**,不是"从现在起多久"。这也是个问题,「代价与边界」会讲。

**跑起来看**:

```bash
export JWT_SECRET=secret
cargo run
```

换 token:

```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"client_id":"foo","client_secret":"bar"}' \
  http://localhost:3000/authorize
```

```json
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJiQGIuY29tIiwiY29tcGFueSI6IkFDTUUiLCJleHAiOjIwMDAwMDAwMDB9.4Adcj3UFYzPUVaVF43FmMab6RlaQD8A9V8wFzzht-KQ","token_type":"Bearer"}
```

**亲手拆开这个 token 看看。** 把三段分别解 base64:

```bash
TOKEN=$(curl -s -H 'Content-Type: application/json' \
  -d '{"client_id":"foo","client_secret":"bar"}' \
  http://localhost:3000/authorize | grep -o '"access_token":"[^"]*"' | cut -d'"' -f4)

echo "$TOKEN" | cut -d. -f1 | base64 -d 2>/dev/null; echo
echo "$TOKEN" | cut -d. -f2 | base64 -d 2>/dev/null; echo
```

```
{"typ":"JWT","alg":"HS256"}
{"sub":"b@b.com","company":"ACME","exp":2000000000}
```

**payload 完全是明文可读的。** 这就是前置知识强调的:**JWT 不加密,只防篡改**。任何拿到 token 的人都能看到里面的内容——**所以绝不能放密码、密钥、身份证号这类东西**。

错误凭据:

```bash
curl -i -s -H 'Content-Type: application/json' \
  -d '{"client_id":"foo","client_secret":"wrong"}' \
  http://localhost:3000/authorize | head -1
curl -s -H 'Content-Type: application/json' \
  -d '{"client_id":"foo","client_secret":"wrong"}' \
  http://localhost:3000/authorize
```

```
HTTP/1.1 401 Unauthorized
{"error":"Wrong credentials"}
```

空凭据:

```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"client_id":"","client_secret":""}' \
  http://localhost:3000/authorize
```

```json
{"error":"Missing credentials"}
```

### 步骤 3:把校验做成提取器

**要做什么**:给 `Claims` 实现 `FromRequestParts`,加上受保护的路由。

```rust
// src/main.rs
use axum::{
    extract::FromRequestParts,               // 新增
    http::{request::Parts, StatusCode},      // 改动
    response::{IntoResponse, Response},
    routing::{get, post},
    Json, RequestPartsExt, Router,           // 改动:加上 RequestPartsExt
};
use axum_extra::{                            // 新增
    headers::{authorization::Bearer, Authorization},
    TypedHeader,
};
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation};  // 改动
// ...(其余 use 不变)

#[tokio::main]
async fn main() {
    // ...(日志初始化不变)

    // 改动:加上受保护路由
    let app = Router::new()
        .route("/protected", get(protected))
        .route("/authorize", post(authorize));

    // ...
}

// 新增
async fn protected(claims: Claims) -> Result<String, AuthError> {
    // Send the protected data to the user
    Ok(format!(
        "Welcome to the protected area :)\nYour data:\n{claims}",
    ))
}

// ...(authorize 不变)

// 新增:身份校验做成提取器
impl<S> FromRequestParts<S> for Claims
where
    S: Send + Sync,
{
    type Rejection = AuthError;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        // Extract the token from the authorization header
        let TypedHeader(Authorization(bearer)) = parts
            .extract::<TypedHeader<Authorization<Bearer>>>()
            .await
            .map_err(|_| AuthError::InvalidToken)?;
        // Decode the user data
        let token_data = decode::<Claims>(bearer.token(), &KEYS.decoding, &Validation::default())
            .map_err(|_| AuthError::InvalidToken)?;

        Ok(token_data.claims)
    }
}

// ...(其余不变)
```

**`impl FromRequestParts for Claims`** 是本章的核心。自定义提取器的套路[第 08 章](08-parse-body-based-on-content-type.md#步骤-2实现-jsonorform)讲过,这里实现的是 `FromRequestParts` 因为只看头部、不碰 body。

**`parts.extract::<TypedHeader<...>>()`** 用了 `RequestPartsExt`([第 10 章](10-customize-extractor-error.md#步骤-2做法一手写-fromrequest)见过)——在自己的提取器里**复用另一个提取器**。这是提取器可组合的体现。

**`decode::<Claims>(token, &KEYS.decoding, &Validation::default())`** 三件事一起做了:验签、检查 `exp`、把 payload 反序列化成 `Claims`。任何一步失败都返回 `Err`。

**`Validation::default()`** 默认校验 `exp` 和算法。可以配置成检查 `iss`、`aud`、或者放宽时钟偏移(`leeway`)。

**`.map_err(|_| AuthError::InvalidToken)`** 把所有失败原因都压成同一个错误。这在安全上是**对的**——不该告诉攻击者"签名错了"还是"token 过期了",信息越少越好。但调试时会很痛苦,「代价与边界」会说。

**`protected(claims: Claims)`** —— handler 签名里就一个 `Claims`,没有任何认证代码。**能执行到函数体,身份就一定验过了。**

**跑起来看**:

```bash
export JWT_SECRET=secret
cargo run
```

先拿 token,再访问受保护路由:

```bash
TOKEN=$(curl -s -H 'Content-Type: application/json' \
  -d '{"client_id":"foo","client_secret":"bar"}' \
  http://localhost:3000/authorize | grep -o '"access_token":"[^"]*"' | cut -d'"' -f4)

curl -s -H "Authorization: Bearer $TOKEN" http://localhost:3000/protected
```

```
Welcome to the protected area :)
Your data:
Email: b@b.com
Company: ACME
```

**handler 拿到了 token 里的身份信息。**

不带 token:

```bash
curl -i -s http://localhost:3000/protected | head -1
curl -s http://localhost:3000/protected
```

```
HTTP/1.1 400 Bad Request
{"error":"Invalid token"}
```

胡乱的 token:

```bash
curl -s -H 'Authorization: Bearer blahblahblah' http://localhost:3000/protected
```

```json
{"error":"Invalid token"}
```

**验证签名真的防篡改。** 把 payload 改掉再拼回去——比如把 `company` 从 `ACME` 改成 `EVIL`:

```bash
HEADER=$(echo "$TOKEN" | cut -d. -f1)
SIG=$(echo "$TOKEN" | cut -d. -f3)
FAKE_PAYLOAD=$(echo -n '{"sub":"b@b.com","company":"EVIL","exp":2000000000}' | base64 | tr -d '=' | tr '/+' '_-')

curl -s -H "Authorization: Bearer $HEADER.$FAKE_PAYLOAD.$SIG" http://localhost:3000/protected
```

```json
{"error":"Invalid token"}
```

**被拒了。** 因为签名是对原始 payload 算的,改了 payload 签名就对不上。攻击者**必须有 secret 才能算出新签名**。

**验证 secret 的作用。** 换一个 secret 重启服务,旧 token 立刻失效:

```bash
# Ctrl-C 停掉,然后
JWT_SECRET=another-secret cargo run
```

```bash
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:3000/protected
```

```json
{"error":"Invalid token"}
```

**这是唯一能"批量撤销"所有 token 的手段** —— 换 secret。代价是所有用户都被登出了。前置知识讲的"撤销不了"在这里体现得很直接。

## 完整代码

`src/main.rs`:

```rust
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
    // Send the protected data to the user
    Ok(format!(
        "Welcome to the protected area :)\nYour data:\n{claims}",
    ))
}

async fn authorize(Json(payload): Json<AuthPayload>) -> Result<Json<AuthBody>, AuthError> {
    // Check if the user sent the credentials
    if payload.client_id.is_empty() || payload.client_secret.is_empty() {
        return Err(AuthError::MissingCredentials);
    }
    // Here you can check the user credentials from a database
    if payload.client_id != "foo" || payload.client_secret != "bar" {
        return Err(AuthError::WrongCredentials);
    }
    let claims = Claims {
        sub: "b@b.com".to_owned(),
        company: "ACME".to_owned(),
        // Mandatory expiry time as UTC timestamp
        exp: 2000000000, // May 2033
    };
    // Create the authorization token
    let token = encode(&Header::default(), &claims, &KEYS.encoding)
        .map_err(|_| AuthError::TokenCreation)?;

    // Send the authorized token
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
        // Extract the token from the authorization header
        let TypedHeader(Authorization(bearer)) = parts
            .extract::<TypedHeader<Authorization<Bearer>>>()
            .await
            .map_err(|_| AuthError::InvalidToken)?;
        // Decode the user data
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
```

环境变量:`JWT_SECRET`(必需)

## 代价与边界

**这是本教程安全相关问题最集中的一章**,逐条说。

**`exp` 是硬编码的固定时间点。** `exp: 2000000000` 是 2033 年 5 月——**签发的 token 有效期长达数年**。正确做法是"从现在起 N 分钟":

```rust
let exp = (std::time::SystemTime::now() + Duration::from_secs(900))
    .duration_since(std::time::UNIX_EPOCH).unwrap().as_secs() as usize;
```

而且既然[前置知识](#无状态的代价token-撤销不了)讲过 token 撤销不了,**过期时间越短风险越小**。生产环境的 access token 通常 15 分钟到 1 小时。

**没有 refresh token。** 只有一个长效 token,用户体验和安全性无法兼顾:短过期就要频繁重新登录,长过期就有风险。生产方案需要 access + refresh 两级。

**`InvalidToken` 返回 `400` 而非 `401`。** 步骤 1 的「更好的写法」指出了。

**凭据硬编码 + 明文比较。** 步骤 2 的「更好的写法」指出了。

**密钥从环境变量读,但 panic 时机很晚。** `LazyLock` 是懒加载的——没设 `JWT_SECRET` 时,**服务能正常启动、能监听端口、健康检查也通过**,直到第一个需要 token 的请求到达才 panic。这比启动即失败糟糕得多:容器编排系统会认为服务是健康的,不会重启它。修法是在 `main` 开头显式访问一次 `KEYS`(比如 `LazyLock::force(&KEYS)`)。

**密钥强度没有校验。** `JWT_SECRET=secret` 这种弱密钥能被暴力破解——HS256 的安全性完全取决于密钥熵。生产环境应该用至少 256 位的随机密钥(`openssl rand -base64 32`)。

**没有区分错误原因,调试困难。** `.map_err(|_| AuthError::InvalidToken)` 把签名错误、过期、格式错误全压成一个。**对外这么做是对的**(不给攻击者线索),但**对内应该记日志**:

```rust
.map_err(|e| { tracing::debug!(?e, "token validation failed"); AuthError::InvalidToken })?
```

否则线上出现"用户说登录不了"时,你完全不知道是 token 过期了还是密钥配错了。

**只有认证没有授权。** 前置知识讲过。任何合法 token 都能访问 `/protected`,不区分用户和权限。

**token 从 `Authorization` 头部传,不涉及 cookie。** 这避开了 CSRF 问题([第 20 章](20-cors.md#代价与边界)提过),但意味着**前端要自己存 token**。存 `localStorage` 有 XSS 风险(脚本能读),存内存则刷新页面就丢。这是个没有完美答案的取舍。

**必须走 HTTPS。** token 在头部明文传输,HTTP 下任何中间人都能截获并冒用。[第 32 章](32-tls-rustls.md)会讲怎么启用 TLS。

## 动手练习

**1. 把 `InvalidToken` 改成 `401`,并加上诊断日志。**

提示:`AuthError::InvalidToken => (StatusCode::UNAUTHORIZED, "Invalid token")`。同时把 `.map_err(|_| ...)` 改成保留错误的形式,用 `tracing::debug!(?e, ...)` 记下真实原因。做完对比:客户端仍然只看到 "Invalid token",但服务端日志能区分"过期"和"签名错误"。**这是「代价与边界」里最该改的两处。**

**2. 让 `exp` 变成"从现在起 15 分钟"。**

提示:

```rust
use std::time::{Duration, SystemTime, UNIX_EPOCH};

let exp = (SystemTime::now() + Duration::from_secs(900))
    .duration_since(UNIX_EPOCH).unwrap().as_secs() as usize;
```

验证过期行为:把 900 改成 5,拿到 token 后 `sleep 6` 再访问 `/protected`,应该被拒。**这个练习能让你亲眼看到 `exp` 校验在工作。**

**3. 加上授权检查。**

提示:给 `Claims` 加一个 `role: String` 字段,再写一条 `/admin` 路由,handler 里检查 `claims.role == "admin"`,否则返回 `403 Forbidden`。做完想想:能不能把这个检查也做成提取器?(可以——定义一个 `AdminClaims` 类型,在它的 `FromRequestParts` 里先提取 `Claims` 再校验 role。这样"需要管理员权限"也变成了类型层面的声明。)

## 小结与下一章预告

JWT 是三段 base64 拼成的字符串:header 说明算法、payload 装着 claims、signature 保护前两段不被篡改。**前两段只是编码不是加密,任何人都能解开看** —— 所以里面绝不能放敏感信息。本章亲手解开过 token 验证了这一点,也验证了改了 payload 后签名对不上会被拒。

**"无状态"是 JWT 的核心卖点,也是它最大的坑:签发出去的 token 在过期前撤销不了。** 用户登出、账号被盗、权限降级,手里的 token 照样有效。唯一的批量撤销手段是换 secret——代价是所有人被登出。缓解方案(短过期 + refresh token、黑名单、版本号)**全都要引入服务端状态**,也就是部分放弃了无状态。单体应用、需要立即登出的场景,session 反而更合适。

axum 层面的核心模式是**把身份校验做成提取器**:给 `Claims` 实现 `FromRequestParts`,handler 签名里写上它就自动完成取 token、验签、解析。这和[第 12 章](12-validator.md#校验放在哪一层)的 `ValidatedForm` 一个道理——**把必须做的检查编码进类型,而不是靠人记得调用**。

顺带认识了 `TypedHeader`:把"头部名 + 值格式"编码进类型,省掉手写 `strip_prefix("Bearer ")` 这类解析。

本章的安全问题相当集中:`exp` 硬编码成 2033 年、`InvalidToken` 返回 `400` 而非 `401`、凭据硬编码且明文比较、`LazyLock` 让缺密钥的 panic 推迟到第一个请求、错误原因既不告诉客户端也不记日志。「代价与边界」逐条列了。

本章的认证是自己签发 token。[第 31 章 oauth](31-oauth.md) 换成**委托给第三方**:让用户用 Discord、GitHub 这类账号登录。你会看到授权码流程的四次跳转、`state` 参数怎么防 CSRF,以及一个和本章形成对比的选择——**OAuth 示例用的是有状态的 session,而不是 JWT**。
