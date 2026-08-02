# 第 22 章 key-value-store:共享可变状态

## 本章你将做出什么

一个内存 KV 存储服务:

```
GET    /{key}           读取
POST   /{key}           写入(body 就是值)
GET    /keys            列出所有 key
DELETE /admin/keys      清空(需要 bearer token)
DELETE /admin/key/{key} 删除单个(需要 bearer token)
```

这是本教程第一个**多个请求会互相看见对方数据**的服务。[第 05 章](05-templates-minijinja.md#state-与-with_state)讲的 `State` 只提供共享**只读**访问,这一章补上"可写"的部分——以及它带来的全部麻烦。

学完本章你将掌握:

- `Mutex` 与 `RwLock` 怎么选,以及 `std` 和 `tokio` 版本的区别
- **持锁跨 `await` 这个能让服务彻底卡死的陷阱**
- `.lock().unwrap()` 为什么会 panic(锁中毒)
- 给单条路由挂中间件,以及 `HandleErrorLayer` 处理中间件产生的错误

## 前置知识

### 从只读到可写

[第 05 章](05-templates-minijinja.md#为什么需要共享状态)用 `Arc<AppState>` 共享了模板引擎。那个场景很简单:启动时初始化一次,之后所有请求只读。

现在的场景不同:**每个请求都可能修改状态,而它们并发执行。** `Arc<T>` 只给共享引用 `&T`,拿不到 `&mut T`——这是 Rust 的核心规则,同一时刻要么多个只读引用,要么一个可写引用。

绕过它的办法是**内部可变性**:用一个类型包住数据,在运行时保证"任一时刻只有一个写者"。[第 16 章](16-dependency-injection.md#步骤-1定义-trait-和一个实现)的 `InMemoryUserRepo` 已经用过这个模式,当时挂了"本章不深入",现在展开。

```rust
type SharedState = Arc<RwLock<AppState>>;
```

三层各司其职:

- **`Arc`** — 让多个 handler 共享同一份数据(克隆只增加引用计数)
- **`RwLock`** — 在运行时保证读写互斥
- **`AppState`** — 实际的数据

### Mutex 与 RwLock

两者都提供内部可变性,区别在于并发度:

| | `Mutex<T>` | `RwLock<T>` |
|---|---|---|
| 获取方法 | `lock()` | `read()` / `write()` |
| 并发读 | **不允许**(读也要独占) | 允许多个 |
| 并发写 | 不允许 | 不允许 |
| 单次开销 | 更小 | 稍大 |
| 适合 | 读写比例接近、或临界区极短 | **读远多于写** |

**默认选 `Mutex`。** 它更简单、开销更小,而且不容易写错(`RwLock` 有个陷阱:同一个线程先 `read()` 再 `write()` 会死锁)。

**读远多于写时选 `RwLock`。** KV 存储、配置缓存、路由表这类场景读操作占绝大多数,允许并发读能显著提升吞吐。本章示例用 `RwLock` 正是这个理由。

**但要注意:`RwLock` 的优势只在临界区足够长时才体现。** 如果锁里只做一次 `HashMap::get`,加解锁的开销可能比操作本身还大,这时 `Mutex` 反而更快。**别凭直觉选,有疑问就测。**

### std 还是 tokio 的锁

`std::sync` 和 `tokio::sync` 都提供 `Mutex`/`RwLock`,这是个**经常选错**的地方。

| | `std::sync::Mutex` | `tokio::sync::Mutex` |
|---|---|---|
| 加锁 | 阻塞线程 | `.await`,让出任务 |
| 能否持锁跨 `await` | **不能** | 能 |
| 开销 | 小 | 大(要维护等待队列) |
| 中毒机制 | 有(返回 `Result`) | 无 |

**官方建议:优先用 `std` 的锁。** tokio 文档自己就是这么说的。理由是 `std` 的锁开销小得多,而且绝大多数场景下临界区很短(操作一下 `HashMap`),阻塞那几纳秒完全可以接受。

**只有一种情况必须用 `tokio` 的锁:临界区里要 `.await`。** 比如"持有锁的同时查数据库"。但更好的做法通常是重构代码避免这种情况——见下一节。

本章示例用的是 `std::sync::RwLock`。

### 持锁跨 await:能让服务卡死的陷阱

这是本章最重要的一节。

`std` 的锁守卫(`MutexGuard`/`RwLockWriteGuard`)**不是 `Send`**,所以下面这段编译不过:

```rust
async fn bad(state: SharedState) {
    let mut db = state.write().unwrap();
    some_async_call().await;        // ← 编译错误
    db.insert(key, value);
}
```

编译器会报一长串关于 `Send` 的错误,核心意思是"这个 future 不能跨线程移动"。**这个编译错误是在救你**,理解它的原因很重要。

**为什么危险。** async 任务在 `.await` 处会让出执行权,tokio 可能把这个任务调度到别的线程继续。如果此时还持有锁:

- 锁被一个**挂起的任务**持有,可能持续很久(等网络、等数据库)
- 其它任务想拿锁就得阻塞**整个工作线程**
- 工作线程数量有限(默认等于 CPU 核数),几个这样的任务就能把线程池占满
- **整个服务停止响应**,而且日志上看不出任何异常——请求只是不返回了

用 `tokio::sync::Mutex` 能通过编译,但问题只是从"编译错误"变成了"运行时性能灾难"——换成锁的排队而已。

**正确做法是缩短临界区**,让锁在 `.await` 之前就释放:

```rust
async fn good(state: SharedState) {
    let value = {
        let db = state.read().unwrap();
        db.get(&key).cloned()        // 复制出来
    };                                // ← 作用域结束,锁在这里释放

    some_async_call().await;          // 现在没持锁了
}
```

用一个块表达式把加锁范围框起来,出了块守卫就被丢弃、锁被释放。**"拿到数据就赶紧放锁"是使用同步锁的基本纪律。**

本章示例的每个 handler 都遵守这条:锁只在一个表达式里存在,拿完就还。

### 锁中毒与 .lock().unwrap()

`std` 的锁方法返回 `Result`,所以到处是 `.unwrap()`:

```rust
state.write().unwrap().db.insert(key, bytes);
```

为什么会失败?因为**锁中毒**(poisoning):如果某个线程在持锁期间 panic 了,锁会被标记为"中毒",之后所有 `lock()`/`read()`/`write()` 都返回 `Err`。

设计意图是:持锁线程 panic 说明它可能只改了一半数据,里面的状态可能已经不一致了,其它线程不该盲目继续用。

**所以 `.unwrap()` 的语义是"如果数据已经损坏,我也跟着 panic"。** 对多数应用这是合理的——数据不一致时继续跑更危险。真要处理,可以用 `into_inner()` 忽略中毒状态取出数据,但那意味着你确信数据仍然可用。

顺带一提,`tokio::sync` 的锁**没有中毒机制**,方法直接返回守卫,不用 `unwrap`。

### HandleErrorLayer:中间件也会出错

本章的中间件栈里有超时和限流,它们会产生错误:

```rust
ServiceBuilder::new()
    .layer(HandleErrorLayer::new(handle_error))
    .load_shed()
    .concurrency_limit(1024)
    .timeout(Duration::from_secs(10))
    .layer(TraceLayer::new_for_http())
```

**问题在于类型不匹配。** axum 的 handler 永不出错(错误已经在 `IntoResponse` 里变成响应了),所以 `Router` 的 `Service::Error` 是 `Infallible`。但 tower 的通用中间件(`timeout`、`load_shed`)会产生 `BoxError`。

`HandleErrorLayer` 就是这个转换器:**接住下游中间件产生的错误,交给一个函数变成响应**,把 `Error` 类型重新变回 `Infallible`。

**它必须是最外层**(`ServiceBuilder` 里第一个声明,见[第 17 章](17-tracing-aka-logging.md#洋葱结构与包装顺序)),这样才能接住内层所有中间件的错误。

三个中间件各自的作用:

- **`load_shed()`** — 服务过载时直接拒绝新请求(返回错误),而不是排队。**快速失败比慢慢超时好**:排队会让所有请求都变慢,拒绝一部分能保住剩下的。
- **`concurrency_limit(1024)`** — 最多同时处理 1024 个请求,超出的排队(配合 `load_shed` 就变成拒绝)
- **`timeout(Duration::from_secs(10))`** — 10 秒还没处理完就放弃

`handle_error` 用 `error.is::<T>()` 判断具体是哪种错误,给出对应的状态码。`BoxError` 是 `Box<dyn Error>`,所以要用这种运行时类型检查。

## 项目搭建

```bash
cargo new key-value-store
cd key-value-store
```

```bash
cargo add axum
cargo add tokio --features full
cargo add tower --features util,timeout,load-shed,limit
cargo add tower-http --features add-extension,auth,compression-full,limit,trace
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

`tower` 的三个 feature 对应三个中间件:`timeout`、`load-shed`、`limit`(提供 `concurrency_limit`)。

`tower-http` 的 `auth` 提供 `ValidateRequestHeaderLayer`,`compression-full` 提供 `CompressionLayer`。

## 分步构建

### 步骤 1:读写共享状态

**要做什么**:实现 `GET /{key}` 和 `POST /{key}`,把共享可变状态跑通。

```rust
// src/main.rs
use axum::{
    body::Bytes,
    extract::{Path, State},
    http::StatusCode,
    routing::get,
    Router,
};
use std::{
    collections::HashMap,
    sync::{Arc, RwLock},
};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let shared_state = SharedState::default();

    let app = Router::new()
        .route("/{key}", get(kv_get).post(kv_set))
        .with_state(Arc::clone(&shared_state));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

type SharedState = Arc<RwLock<AppState>>;

#[derive(Default)]
struct AppState {
    db: HashMap<String, Bytes>,
}

async fn kv_get(
    Path(key): Path<String>,
    State(state): State<SharedState>,
) -> Result<Bytes, StatusCode> {
    let db = &state.read().unwrap().db;

    if let Some(value) = db.get(&key) {
        Ok(value.clone())
    } else {
        Err(StatusCode::NOT_FOUND)
    }
}

async fn kv_set(Path(key): Path<String>, State(state): State<SharedState>, bytes: Bytes) {
    state.write().unwrap().db.insert(key, bytes);
}
```

**`type SharedState = Arc<RwLock<AppState>>`** 给这个三层嵌套起个别名。真实项目里强烈建议这么做——否则每个 handler 签名里都是一长串。

**`AppState` 里包一个 `HashMap` 而不是直接 `Arc<RwLock<HashMap>>`。** 这样将来加字段(比如统计计数器)不用改所有签名。**多一层 struct 是值得的。**

**`kv_get` 里的临界区**:

```rust
let db = &state.read().unwrap().db;
if let Some(value) = db.get(&key) {
    Ok(value.clone())
}
```

`value.clone()` 是必需的——`db.get()` 返回的是引用,而引用不能逃出锁的作用域。好在这里是 `Bytes`,[第 09 章](09-consume-body-in-extractor-or-middleware.md#bytes可以廉价克隆的字节缓冲)讲过它的克隆只增加引用计数,不复制字节。

**`kv_set` 的第三个参数 `bytes: Bytes`** 是提取器——把整个请求体读成字节,不做任何解析。[第 09 章](09-consume-body-in-extractor-or-middleware.md#步骤-1一个缓冲-body-的提取器)见过。它是 `FromRequest` 类别,所以必须放最后。

**`Arc::clone(&shared_state)`** 而不是 `shared_state.clone()`——两者等价,但前者更明确地表达"我在克隆 `Arc`,不是克隆里面的数据"。这是 Rust 社区推荐的写法。

**跑起来看**:

```bash
cargo run
```

写入一个值:

```bash
curl -X POST http://127.0.0.1:3000/foo -d 'hello world'
```

读出来:

```bash
curl http://127.0.0.1:3000/foo
```

```
hello world
```

**数据在两个请求之间保留下来了** —— 这是本教程第一次做到这件事。

读不存在的 key:

```bash
curl -i -s http://127.0.0.1:3000/nope | head -1
```

```
HTTP/1.1 404 Not Found
```

试试二进制数据:

```bash
head -c 100 /dev/urandom > /tmp/bin
curl -X POST http://127.0.0.1:3000/binary --data-binary @/tmp/bin
curl -s http://127.0.0.1:3000/binary | wc -c
```

```
100
```

`Bytes` 不关心内容是什么,原样存取。

### 步骤 2:验证持锁跨 await 编译不过

**要做什么**:亲手触发前置知识说的那个编译错误。这一步不产生最终代码,但能让你以后一眼认出这类问题。

临时把 `kv_set` 改成在持锁时 `.await`:

```rust
async fn kv_set(Path(key): Path<String>, State(state): State<SharedState>, bytes: Bytes) {
    let mut guard = state.write().unwrap();
    tokio::time::sleep(std::time::Duration::from_millis(1)).await;   // 临时:持锁时 await
    guard.db.insert(key, bytes);
}
```

```bash
cargo run
```

```
error: future cannot be sent between threads safely
   |
   = help: within `impl Future<Output = ()>`, the trait `Send` is not implemented
           for `std::sync::RwLockWriteGuard<'_, AppState>`
note: future is not `Send` as this value is used across an await
   |
   |     let mut guard = state.write().unwrap();
   |         --------- has type `RwLockWriteGuard<'_, AppState>` which is not `Send`
   |     tokio::time::sleep(...).await;
   |                            ^^^^^^ await occurs here, with `guard` maybe used later
```

**编译器精确指出了问题**:守卫跨越了 `.await`,而它不是 `Send`。

前置知识解释了后果:锁被一个挂起的任务持有,其它任务想拿锁就阻塞工作线程,几个这样的任务就能拖垮整个服务。**这个编译错误挡住了一类很难排查的线上故障。**

**注意如果换成 `tokio::sync::RwLock` 就能编译通过** —— 但问题只是从编译期挪到了运行期。真正的解法是缩短临界区:

```rust
async fn kv_set(Path(key): Path<String>, State(state): State<SharedState>, bytes: Bytes) {
    tokio::time::sleep(std::time::Duration::from_millis(1)).await;   // 先做异步的事
    state.write().unwrap().db.insert(key, bytes);                    // 再加锁
}
```

或者用块表达式框住加锁范围。改回原来的 `kv_set`。

### 步骤 3:列出所有 key,给单条路由挂中间件

**要做什么**:加上 `GET /keys`,并演示 axum 一个少见但有用的能力——**给单个 handler 挂中间件**。

```rust
// src/main.rs
use axum::{
    body::Bytes,
    extract::{DefaultBodyLimit, Path, State},        // 改动:加上 DefaultBodyLimit
    handler::Handler,                                // 新增
    http::StatusCode,
    routing::get,
    Router,
};
use std::{
    collections::HashMap,
    sync::{Arc, RwLock},
};
use tower_http::{compression::CompressionLayer, limit::RequestBodyLimitLayer};   // 新增
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // ...(日志初始化和 shared_state 不变)

    // 改动:给两个 handler 分别挂不同的中间件
    let app = Router::new()
        .route(
            "/{key}",
            // Add compression to `kv_get`
            get(kv_get.layer(CompressionLayer::new()))
                // But don't compress `kv_set`
                .post_service(
                    kv_set
                        .layer((
                            DefaultBodyLimit::disable(),
                            RequestBodyLimitLayer::new(1024 * 5_000 /* ~5mb */),
                        ))
                        .with_state(Arc::clone(&shared_state)),
                ),
        )
        .route("/keys", get(list_keys))            // 新增
        .with_state(Arc::clone(&shared_state));

    // ...(listener 和 serve 不变)
}

// ...(SharedState、AppState、kv_get、kv_set 不变)

// 新增
async fn list_keys(State(state): State<SharedState>) -> String {
    let db = &state.read().unwrap().db;

    db.keys()
        .map(|key| key.to_string())
        .collect::<Vec<String>>()
        .join("\n")
}
```

这段路由配置是本章最绕的地方,拆开看。

**`kv_get.layer(CompressionLayer::new())`** —— **handler 本身也能 `.layer()`**。这需要 `use axum::handler::Handler`(方法来自这个 trait)。效果是只给这一个 handler 套上压缩,`kv_set` 不受影响。

为什么不压缩 `kv_set`?因为压缩的是**响应**,而 `kv_set` 的响应是空的,压缩毫无意义还白费 CPU。

**`.post_service(...)`** 而不是 `.post(...)`。区别在于:`.post()` 接受 handler,`.post_service()` 接受已经构造好的 `Service`。因为 `kv_set.layer(...).with_state(...)` 已经把 handler 变成了 `Service`,所以要用后者。

**`.with_state(Arc::clone(&shared_state))` 在这里出现了两次** —— 一次给 `kv_set` 那个独立的 service,一次给整个 `Router`。这是因为 `kv_set` 被单独转成 `Service` 之后就脱离了 `Router` 的状态注入,得自己带上状态。

**`.layer((A, B))` 传元组** 是 tower 的一个便利写法:元组也实现了 `Layer`,等价于依次应用。

**`DefaultBodyLimit::disable()` + `RequestBodyLimitLayer::new(5MB)`** 是[第 13 章](13-multipart-form.md#两层请求体限制)讲过的组合:关掉提取器层面的 2MB 默认限制,改用外层中间件设一个更大的上限。KV 存储要能存较大的值,所以放宽到 5MB。

**跑起来看**:

```bash
cargo run
```

```bash
curl -X POST http://127.0.0.1:3000/foo -d 'hello'
curl -X POST http://127.0.0.1:3000/bar -d 'world'
curl http://127.0.0.1:3000/keys
```

```
foo
bar
```

(顺序取决于 `HashMap` 的哈希,不保证。)

**验证压缩确实只对 `kv_get` 生效。** 存一个大一点的值,然后带 `Accept-Encoding` 请求:

```bash
curl -X POST http://127.0.0.1:3000/big -d "$(head -c 5000 /dev/zero | tr '\0' 'a')"
curl -s -i -H 'Accept-Encoding: gzip' http://127.0.0.1:3000/big | grep -i 'content-encoding'
```

```
content-encoding: gzip
```

5000 个 `a` 压缩后只有几十字节。

### 步骤 4:管理接口与 bearer 认证

**要做什么**:加上受保护的 `/admin` 路由。

```rust
// src/main.rs
use axum::{
    // ...
    routing::{delete, get},                          // 改动:加上 delete
    Router,
};
use tower_http::{
    compression::CompressionLayer,
    limit::RequestBodyLimitLayer,
    validate_request::ValidateRequestHeaderLayer,    // 新增
};
// ...

#[tokio::main]
async fn main() {
    // ...

    let app = Router::new()
        .route(/* ...(不变) */)
        .route("/keys", get(list_keys))
        // 新增:Nest our admin routes under `/admin`
        .nest("/admin", admin_routes())
        .with_state(Arc::clone(&shared_state));

    // ...
}

// ...(其余 handler 不变)

// 新增
#[allow(deprecated)]
fn admin_routes() -> Router<SharedState> {
    async fn delete_all_keys(State(state): State<SharedState>) {
        state.write().unwrap().db.clear();
    }

    async fn remove_key(Path(key): Path<String>, State(state): State<SharedState>) {
        state.write().unwrap().db.remove(&key);
    }

    #[allow(deprecated)] // FIXME
    Router::new()
        .route("/keys", delete(delete_all_keys))
        .route("/key/{key}", delete(remove_key))
        // Require bearer auth for all admin routes
        .layer(ValidateRequestHeaderLayer::bearer("secret-token"))
}
```

**`fn admin_routes() -> Router<SharedState>`** 返回一个**带状态类型参数但还没填状态**的 `Router`。[第 05 章](05-templates-minijinja.md#state-与-with_state)讲过 `Router<S>` 的含义:`S` 是它需要的状态类型。这里 `S = SharedState`,状态由父 `Router` 的 `.with_state()` 统一提供。

**两个 handler 定义在函数内部**,这是[第 02 章](02-routes-and-handlers-close-together.md#步骤-2把路由和处理器包进同一个函数)讲过的组织方式。

**`.nest("/admin", ...)`** 给子路由加前缀,[第 16 章](16-dependency-injection.md#routernest-与-merge-的区别)讲过它和 `merge` 的区别。

**`ValidateRequestHeaderLayer::bearer("secret-token")`** 要求请求带 `Authorization: Bearer secret-token`,否则直接返回 `401`。这是最简单的认证形式——**硬编码一个 token**。真实的认证见[第 30 章 jwt](30-jwt.md) 和[第 31 章 oauth](31-oauth.md)。

> **更好的写法**:token 硬编码在源码里,任何能看到代码的人都知道它,提交进 git 之后更是永久留痕。哪怕是最简单的固定 token 方案,也应该从环境变量或配置文件读(`std::env::var("ADMIN_TOKEN")`)。示例这么写是为了让你 `cargo run` 之后立刻能测,不用先配环境。另外 `ValidateRequestHeaderLayer::bearer` 做的是**明文字符串比较**,严格来说存在时序攻击的理论风险;高安全场景应该用常量时间比较。

**两个 `#[allow(deprecated)]`** 和那句 `// FIXME` 是官方示例留下的——某个 API 在新版本里被标记为废弃,但还没找到合适的替代。照抄即可,它不影响功能。

**跑起来看**:

```bash
cargo run
```

先存点数据:

```bash
curl -X POST http://127.0.0.1:3000/foo -d 'hello'
curl -X POST http://127.0.0.1:3000/bar -d 'world'
```

不带 token 访问管理接口:

```bash
curl -i -s -X DELETE http://127.0.0.1:3000/admin/keys | head -1
```

```
HTTP/1.1 401 Unauthorized
```

带上正确的 token:

```bash
curl -i -s -X DELETE http://127.0.0.1:3000/admin/key/foo \
  -H 'Authorization: Bearer secret-token' | head -1
```

```
HTTP/1.1 200 OK
```

确认 `foo` 没了、`bar` 还在:

```bash
curl -s http://127.0.0.1:3000/keys
```

```
bar
```

清空:

```bash
curl -s -X DELETE http://127.0.0.1:3000/admin/keys -H 'Authorization: Bearer secret-token'
curl -s http://127.0.0.1:3000/keys
```

输出为空。

### 步骤 5:全局中间件与错误处理

**要做什么**:加上超时、限流、过载保护,并用 `HandleErrorLayer` 处理它们产生的错误。

```rust
// src/main.rs
use axum::{
    body::Bytes,
    error_handling::HandleErrorLayer,                // 新增
    extract::{DefaultBodyLimit, Path, State},
    handler::Handler,
    http::StatusCode,
    response::IntoResponse,                          // 新增
    routing::{delete, get},
    Router,
};
use std::{
    borrow::Cow,                                     // 新增
    collections::HashMap,
    sync::{Arc, RwLock},
    time::Duration,                                  // 新增
};
use tower::{BoxError, ServiceBuilder};               // 新增
use tower_http::{
    compression::CompressionLayer, limit::RequestBodyLimitLayer, trace::TraceLayer,   // 改动:加上 TraceLayer
    validate_request::ValidateRequestHeaderLayer,
};
// ...

#[tokio::main]
async fn main() {
    // ...

    let app = Router::new()
        .route(/* ...(不变) */)
        .route("/keys", get(list_keys))
        .nest("/admin", admin_routes())
        // 新增:Add middleware to all routes
        .layer(
            ServiceBuilder::new()
                // Handle errors from middleware
                .layer(HandleErrorLayer::new(handle_error))
                .load_shed()
                .concurrency_limit(1024)
                .timeout(Duration::from_secs(10))
                .layer(TraceLayer::new_for_http()),
        )
        .with_state(Arc::clone(&shared_state));

    // ...
}

// ...(其余不变)

// 新增
async fn handle_error(error: BoxError) -> impl IntoResponse {
    if error.is::<tower::timeout::error::Elapsed>() {
        return (StatusCode::REQUEST_TIMEOUT, Cow::from("request timed out"));
    }

    if error.is::<tower::load_shed::error::Overloaded>() {
        return (
            StatusCode::SERVICE_UNAVAILABLE,
            Cow::from("service is overloaded, try again later"),
        );
    }

    (
        StatusCode::INTERNAL_SERVER_ERROR,
        Cow::from(format!("Unhandled internal error: {error}")),
    )
}
```

前置知识讲过这一套的作用。这里补几个细节。

**`ServiceBuilder` 而不是链式 `.layer()`** —— [第 19 章](19-request-id.md#为什么必须用-servicebuilder)讲过,顺序敏感时用它。这里 `HandleErrorLayer` 必须在最外层才能接住内层所有错误。

**`Cow::from(...)`** 是为了让三个分支返回同一个类型。前两个分支是 `&'static str`,第三个是 `String`——`Cow<str>`(写时克隆)能同时装下两者,避免为了统一类型把静态字符串也堆分配一次。

**`error.is::<T>()`** 做运行时类型判断。`BoxError` 是 `Box<dyn Error + Send + Sync>`,编译期不知道具体类型,只能这样区分。

**跑起来看**:

```bash
cargo run
```

正常请求不受影响:

```bash
curl -X POST http://127.0.0.1:3000/foo -d 'hello'
curl http://127.0.0.1:3000/foo
```

```
hello
```

现在日志里多了 `TraceLayer` 的输出:

```
DEBUG request{method=POST uri=/foo version=HTTP/1.1}: tower_http::trace::on_request: started processing request
DEBUG request{method=POST uri=/foo version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=0 ms status=200
```

**验证超时。** 临时给 `kv_get` 加一个长时间等待:

```rust
async fn kv_get(
    Path(key): Path<String>,
    State(state): State<SharedState>,
) -> Result<Bytes, StatusCode> {
    tokio::time::sleep(Duration::from_secs(15)).await;    // 临时:超过 10 秒超时
    // ...(其余不变)
}
```

```bash
cargo run
```

```bash
time curl -i -s http://127.0.0.1:3000/foo | head -1
```

```
HTTP/1.1 408 Request Timeout

real    0m10.02s
```

**10 秒后返回 `408`**,而不是傻等 15 秒。这就是 `timeout` 加 `handle_error` 的效果——`handle_error` 认出了 `Elapsed` 错误并给出对应的状态码。

注意这里的 `sleep` 是在锁**之前**,所以没有触犯步骤 2 那个规则。改回去。

## 完整代码

`src/main.rs`:

```rust
use axum::{
    body::Bytes,
    error_handling::HandleErrorLayer,
    extract::{DefaultBodyLimit, Path, State},
    handler::Handler,
    http::StatusCode,
    response::IntoResponse,
    routing::{delete, get},
    Router,
};
use std::{
    borrow::Cow,
    collections::HashMap,
    sync::{Arc, RwLock},
    time::Duration,
};
use tower::{BoxError, ServiceBuilder};
use tower_http::{
    compression::CompressionLayer, limit::RequestBodyLimitLayer, trace::TraceLayer,
    validate_request::ValidateRequestHeaderLayer,
};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!("{}=debug,tower_http=debug", env!("CARGO_CRATE_NAME")).into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let shared_state = SharedState::default();

    // Build our application by composing routes
    let app = Router::new()
        .route(
            "/{key}",
            // Add compression to `kv_get`
            get(kv_get.layer(CompressionLayer::new()))
                // But don't compress `kv_set`
                .post_service(
                    kv_set
                        .layer((
                            DefaultBodyLimit::disable(),
                            RequestBodyLimitLayer::new(1024 * 5_000 /* ~5mb */),
                        ))
                        .with_state(Arc::clone(&shared_state)),
                ),
        )
        .route("/keys", get(list_keys))
        // Nest our admin routes under `/admin`
        .nest("/admin", admin_routes())
        // Add middleware to all routes
        .layer(
            ServiceBuilder::new()
                // Handle errors from middleware
                .layer(HandleErrorLayer::new(handle_error))
                .load_shed()
                .concurrency_limit(1024)
                .timeout(Duration::from_secs(10))
                .layer(TraceLayer::new_for_http()),
        )
        .with_state(Arc::clone(&shared_state));

    // Run our app with hyper
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

type SharedState = Arc<RwLock<AppState>>;

#[derive(Default)]
struct AppState {
    db: HashMap<String, Bytes>,
}

async fn kv_get(
    Path(key): Path<String>,
    State(state): State<SharedState>,
) -> Result<Bytes, StatusCode> {
    let db = &state.read().unwrap().db;

    if let Some(value) = db.get(&key) {
        Ok(value.clone())
    } else {
        Err(StatusCode::NOT_FOUND)
    }
}

async fn kv_set(Path(key): Path<String>, State(state): State<SharedState>, bytes: Bytes) {
    state.write().unwrap().db.insert(key, bytes);
}

async fn list_keys(State(state): State<SharedState>) -> String {
    let db = &state.read().unwrap().db;

    db.keys()
        .map(|key| key.to_string())
        .collect::<Vec<String>>()
        .join("\n")
}

#[allow(deprecated)]
fn admin_routes() -> Router<SharedState> {
    async fn delete_all_keys(State(state): State<SharedState>) {
        state.write().unwrap().db.clear();
    }

    async fn remove_key(Path(key): Path<String>, State(state): State<SharedState>) {
        state.write().unwrap().db.remove(&key);
    }

    #[allow(deprecated)] // FIXME
    Router::new()
        .route("/keys", delete(delete_all_keys))
        .route("/key/{key}", delete(remove_key))
        // Require bearer auth for all admin routes
        .layer(ValidateRequestHeaderLayer::bearer("secret-token"))
}

async fn handle_error(error: BoxError) -> impl IntoResponse {
    if error.is::<tower::timeout::error::Elapsed>() {
        return (StatusCode::REQUEST_TIMEOUT, Cow::from("request timed out"));
    }

    if error.is::<tower::load_shed::error::Overloaded>() {
        return (
            StatusCode::SERVICE_UNAVAILABLE,
            Cow::from("service is overloaded, try again later"),
        );
    }

    (
        StatusCode::INTERNAL_SERVER_ERROR,
        Cow::from(format!("Unhandled internal error: {error}")),
    )
}
```

## 代价与边界

**数据在内存里,重启就没了。** 这是"内存 KV 存储"的定义,不是缺陷。但要清楚它的适用范围:缓存、会话、临时计算结果可以;任何需要持久化的东西都不行。持久化见[第 24 章](24-sqlx-postgres.md)开始的数据库章节。

**内存会无限增长。** 这个服务没有任何淘汰机制——写进去的 key 永远不会自己消失。持续写入最终会 OOM。真实的 KV 缓存需要 TTL 或 LRU 淘汰,这也是为什么大家用 Redis([第 28 章](28-tokio-redis.md))而不是自己造。

**单个值最大 5MB,但没限制总量。** `RequestBodyLimitLayer` 限制的是单次请求。存一万个 5MB 的值就是 50GB。

**写操作会阻塞所有读。** `RwLock` 的写锁是独占的。如果某个写操作意外变慢(比如插入时触发了 `HashMap` 扩容),所有读请求都会卡住。数据量大时可以考虑分片(把 key 哈希到多个 `RwLock` 上),或者直接用 `dashmap` 这类并发 map。

**`concurrency_limit(1024)` 这个数字要按实际情况调。** 它应该反映"你的服务能同时处理多少请求而不崩",通常由下游依赖(数据库连接池大小、外部 API 的限额)决定,而不是拍脑袋。

**admin token 硬编码。** 上面已经指出,不再重复。

**没有区分"key 不存在"和"值为空"。** `kv_get` 对空值返回 200 加空 body,和某些客户端眼里的"没数据"难以区分。真实 API 应该明确约定。

## 动手练习

**1. 从环境变量读 admin token。**

提示:`std::env::var("ADMIN_TOKEN").unwrap_or_else(|_| "secret-token".to_string())`,然后传给 `ValidateRequestHeaderLayer::bearer`。注意 `bearer` 要求 `&'static str`,你需要用 `Box::leak` 或者改用 `ValidateRequestHeaderLayer::custom` 自己实现校验。验证:`ADMIN_TOKEN=hunter2 cargo run` 之后用新 token 测试。

**2. 加一个 key 数量上限,防止内存无限增长。**

提示:在 `kv_set` 里先检查 `db.len()`,超过比如 1000 就返回 `507 Insufficient Storage`。注意 handler 现在要返回 `Result`。做完想想:这个检查和实际插入之间有没有竞态?(有——两个并发请求可能都通过检查。要真正正确,检查和插入必须在同一个写锁里。)

**3. 把 `RwLock` 换成 `Mutex`,对比代码和行为。**

提示:`Mutex` 只有 `lock()`,读写都用它。改完所有 handler 都能工作。然后想想:什么场景下你会看到性能差异?(提示:大量并发读时。可以用 `ab` 或 `wrk` 压测对比,但注意本例临界区极短,差异可能不明显——这正好印证了前置知识说的"别凭直觉选"。)

## 小结与下一章预告

共享**可变**状态的标准形态是 `Arc<RwLock<T>>` 或 `Arc<Mutex<T>>`:`Arc` 负责共享,锁负责运行时的读写互斥。默认选 `Mutex`,读远多于写时选 `RwLock`,但临界区极短时 `RwLock` 的开销可能反而更大——**别凭直觉,有疑问就测**。

**优先用 `std` 的锁而非 `tokio` 的**,tokio 官方文档自己就这么建议。`std` 锁的守卫不是 `Send`,所以持锁跨 `await` 会**编译失败**——这个错误是在救你,它挡住的是"锁被挂起的任务持有、其它任务阻塞工作线程、整个服务停止响应"这类极难排查的故障。正确做法永远是缩短临界区,拿到数据就放锁。

`.lock().unwrap()` 里的 `unwrap` 应对的是锁中毒:持锁线程 panic 后锁被标记为不可用,因为里面的数据可能已经不一致了。

本章还见到了几个新东西:handler 本身可以 `.layer()`(需要 `axum::handler::Handler`),`HandleErrorLayer` 负责把 tower 中间件产生的 `BoxError` 转成响应——它必须在 `ServiceBuilder` 的最外层。

本章的 KV 存储只有字符串键和字节值,没有结构化数据、没有分页、更新只能整体覆盖。[第 23 章 todos](23-todos.md) 做一个完整的 REST CRUD 应用:`GET`/`POST`/`PATCH`/`DELETE` 四个方法各司其职,带查询参数分页,以及一个值得琢磨的问题——**`PATCH` 的部分更新语义该怎么用类型表达**。
