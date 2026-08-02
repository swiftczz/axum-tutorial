# 第 02 章 routes-and-handlers-close-together:让路由和处理器待在一起

## 本章你将做出什么

一个有三条路由的服务:`GET /`、`GET /foo`、`POST /foo`。

功能很简单,重点在**代码怎么组织**。你会先用第 01 章的平铺写法写一遍,亲身体会它在多路由场景下的问题,然后重构成"每条路由自带处理器"的形式,用 `Router::merge` 组装起来。

学完本章你将掌握:

- `Router` 和 `MethodRouter` 的分工,以及 `MethodRouter<S>` 里那个 `S` 是什么
- `Router::merge` 的语义和它的边界条件(什么情况会 panic)
- 一种把路由声明和处理器实现绑在一起的组织模式,以及它适用/不适用的场合

## 前置知识

### 路由表膨胀带来的问题

第 01 章的写法是这样的:

```rust
let app = Router::new().route("/", get(handler));
// ...
async fn handler() -> Html<&'static str> { /* ... */ }
```

一条路由时毫无问题。但设想一个有 40 条路由的项目,`main` 里是 40 行 `.route(...)` 链式调用,handler 则散落在文件的其他地方甚至其他模块。这会带来两个具体的麻烦:

**改一条路由要改两处,而且这两处离得很远。** 你想给 `/users/{id}` 加个 `DELETE`,得先在 `main` 的路由链里找到那一行,再翻到文件另一处找 handler。改完还得回头确认路径写对了没。

**看到一个 handler,不知道它挂在哪个路径上。** 反过来也一样:看到路由链里的 `get(update_profile)`,不知道 `update_profile` 干了什么、需要哪些参数。

这不是 axum 特有的问题,任何"路由集中声明"的框架都有。常见的应对办法有几种:

- 按业务模块拆成子路由,再用 `Router::nest` 挂到父路由上(第 23 章 todos 和第 48 章 versioning 会用到)
- 用宏或属性注解把路径写在 handler 上方(Rocket、actix-web 的风格,axum 刻意不这么做)
- **把每条路由包成一个返回 `Router` 的函数,路径和 handler 写在一起,再合并** —— 这就是本章示例展示的做法

本章讲第三种。它是 axum 官方示例给出的一种模式,不是唯一正确答案,但理解它能顺带把 `Router` 的组合语义搞清楚。

### Router 与 MethodRouter 的分工

[第 01 章](01-hello-world.md#步骤-1能跑起来的最小服务)提过一句,这里说透。

axum 的路由是两层的:

- **`Router`** 按**路径**分发。内部是一棵前缀树(用 `matchit` crate),负责把 `/users/42` 匹配到 `/users/{id}` 这样的模式上。路径匹配不上就是 `404`。
- **`MethodRouter`** 按**方法**分发。挂在某个具体路径下面,负责把 `GET` 交给这个 handler、`POST` 交给那个。方法匹配不上就是 `405`。

`get(handler)` 返回的就是一个 `MethodRouter`。它可以链式加方法:

```rust
get(show).post(create).delete(remove)
```

这三个方法共享同一个路径,是同一个 `MethodRouter` 里的三个分支。

理解这个两层结构,第 01 章看到的行为就顺理成章了:请求 `POST /` 时,`Router` 成功匹配到 `/`,交给它下面的 `MethodRouter`;`MethodRouter` 发现自己只有 `GET` 分支,于是返回 `405` 并把已有分支列进 `allow` 头部。两层各管一段,各给各自的错误码。

### MethodRouter<S> 里的 S 是什么

本章代码里有个 `MethodRouter<()>`,那个 `()` 需要解释一下。

`Router<S>` 和 `MethodRouter<S>` 都带一个泛型参数 `S`,表示**应用状态**(state)的类型——就是数据库连接池、配置、共享缓存这类需要在 handler 之间共享的东西。handler 通过 `State<S>` 提取器拿到它。

`S = ()` 意思是"没有状态"。`Router::new()` 的返回类型其实是 `Router<()>`,只是 `()` 通常能被推断出来,你平时不用写。

本章的路由不需要任何共享状态,所以是 `MethodRouter<()>`。等到[第 22 章 key-value-store](22-key-value-store.md) 引入 `State`,你会看到这个泛型参数真正发挥作用,以及 axum 如何用 `with_state` 把带状态的 `Router<S>` "消解"成可以 serve 的 `Router<()>`。

现在只需记住:`MethodRouter<()>` = "不需要共享状态的方法路由表"。

## 项目搭建

```bash
cargo new routes-and-handlers-close-together
cd routes-and-handlers-close-together
```

依赖和第 01 章一样:

```bash
cargo add axum
cargo add tokio --features full
```

## 分步构建

### 步骤 1:先用平铺写法写出三条路由

**要做什么**:用第 01 章的风格实现 `GET /`、`GET /foo`、`POST /foo`,然后看看它哪里不舒服。

```rust
// src/main.rs
use axum::{
    routing::{get, post},
    Router,
};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(root_handler))
        .route("/foo", get(get_foo_handler).post(post_foo_handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn root_handler() -> &'static str {
    "Hello, World!"
}

async fn get_foo_handler() -> &'static str {
    "Hi from `GET /foo`"
}

async fn post_foo_handler() -> &'static str {
    "Hi from `POST /foo`"
}
```

`.route("/foo", get(get_foo_handler).post(post_foo_handler))` 展示了 `MethodRouter` 的链式用法:一个路径,两个方法,两个 handler。

`post` 需要从 `axum::routing` 里额外引入,和 `get` 并列。

**跑起来看**:

```bash
cargo run
```

```bash
curl http://127.0.0.1:3000/
curl http://127.0.0.1:3000/foo
curl -X POST http://127.0.0.1:3000/foo
```

```
Hello, World!
Hi from `GET /foo`
Hi from `POST /foo`
```

三条路由都通了。方法不对时:

```bash
curl -i -X DELETE http://127.0.0.1:3000/foo
```

```
HTTP/1.1 405 Method Not Allowed
allow: GET, POST
content-length: 0
date: Thu, 30 Jul 2026 06:31:10 GMT
```

`allow` 头部这次列出了两个方法,因为 `/foo` 的 `MethodRouter` 里确实有两个分支。

**现在看代码的问题。** 三条路由还好,但注意 `main` 里的路由链和下面三个 handler 之间只靠函数名关联。handler 名字里带 `foo`、带 `post` 完全是靠人自觉——编译器不会检查 `post_foo_handler` 是不是真的挂在 `POST /foo` 上。把它错挂到 `get(...)` 里,编译毫无问题,只有运行时才发现不对。

把这个规模放大到 40 条路由,再想象几个人同时改这个文件。这就是本章要解决的问题。

### 步骤 2:把路由和处理器包进同一个函数

**要做什么**:每条路由变成一个返回 `Router` 的函数,handler 定义在函数内部,再用 `merge` 组装。

```rust
// src/main.rs
use axum::{
    routing::{get, post},
    Router,
};

#[tokio::main]
async fn main() {
    // 改动:用 merge 组装三个独立的 Router
    let app = Router::new()
        .merge(root())
        .merge(get_foo())
        .merge(post_foo());

    // ...(listener 和 serve 不变)
}

// 新增:一条路由 + 它的处理器,打包在一个函数里
fn root() -> Router {
    async fn handler() -> &'static str {
        "Hello, World!"
    }

    Router::new().route("/", get(handler))
}

// 新增
fn get_foo() -> Router {
    async fn handler() -> &'static str {
        "Hi from `GET /foo`"
    }

    Router::new().route("/foo", get(handler))
}

// 新增
fn post_foo() -> Router {
    async fn handler() -> &'static str {
        "Hi from `POST /foo`"
    }

    Router::new().route("/foo", post(handler))
}
```

把步骤 1 里的三个顶层 handler 删掉。

这里有三个值得说的点。

**handler 定义在函数体内部。** Rust 允许在函数里定义函数,内层函数不能捕获外层的局部变量(它不是闭包),但作用域被限制在外层函数内。这带来一个实际好处:三个 handler 都可以叫 `handler`,互不冲突。名字不再需要承担"我是哪条路由的处理器"这个信息——**路径就写在它下面三行**,一眼可见。

**返回类型是 `Router`。** 每个函数产出一个只有一条路由的完整 `Router`,然后由 `main` 合并。这让"添加一条路由"变成"加一个函数 + 加一行 `.merge()`",改动局部化了。

**`Router::merge` 做的是同层合并。** 它把参数 `Router` 里的所有路由**原样**搬进调用者的路由表,路径不加前缀。所以 `root()` 里注册的 `/` 合并后还是 `/`,`get_foo()` 里的 `/foo` 还是 `/foo`。

这一点要和 `Router::nest` 区分开。`nest("/api", other)` 会给 `other` 里的所有路径加上 `/api` 前缀,`/foo` 变成 `/api/foo`。两个都是组合 `Router` 的手段,但语义完全不同:

- `merge` — 平级合并,用于把同一层级的路由拆开写
- `nest` — 嵌套挂载,用于按模块划分路径空间(第 23 章、第 48 章会用)

**关于 `/foo` 被注册了两次。** `get_foo()` 和 `post_foo()` 都往 `/foo` 上注册,合并后不会冲突——因为它们占的是**不同的方法分支**。`merge` 在遇到相同路径时会尝试把两边的 `MethodRouter` 合并起来,`GET` 和 `POST` 不重叠,合并成功。

但如果两个 `Router` 在**同一路径的同一方法**上都注册了 handler,`merge` 会 **panic**:

```
Overlapping method route. Handler for `GET /foo` already exists
```

注意这是运行时 panic,不是编译错误。而且因为发生在 `main` 启动阶段,你几乎一定会在第一次 `cargo run` 时就撞上,不会漏到生产环境。这算是 axum 的一个取舍:换取了 `Router` 组合的灵活性,代价是这类冲突只能运行时发现。

**跑起来看**:

```bash
cargo run
```

三条路由的行为应该和步骤 1 完全一致:

```bash
curl http://127.0.0.1:3000/
curl http://127.0.0.1:3000/foo
curl -X POST http://127.0.0.1:3000/foo
```

```
Hello, World!
Hi from `GET /foo`
Hi from `POST /foo`
```

`405` 也一样:

```bash
curl -i -X DELETE http://127.0.0.1:3000/foo
```

```
HTTP/1.1 405 Method Not Allowed
allow: GET, POST
...
```

`allow` 依然是 `GET, POST` 两个——虽然它们来自两个不同的函数、两个不同的 `Router`,`merge` 之后在同一个 `MethodRouter` 里。这印证了上面说的合并语义。

顺便验证一下 panic 行为,加深印象。临时把 `post_foo` 里的 `post(handler)` 改成 `get(handler)`,然后 `cargo run`:

```
thread 'main' panicked at ...:
Overlapping method route. Handler for `GET /foo` already exists
```

改回来。

### 步骤 3:抽出 route 辅助函数

**要做什么**:消掉三处重复的 `Router::new().route(...)`。

三个函数末尾都是同样的形状。抽个小函数:

```rust
// src/main.rs
use axum::{
    routing::{get, post, MethodRouter},   // 改动:加上 MethodRouter
    Router,
};

// ...(main 不变)

fn root() -> Router {
    async fn handler() -> &'static str {
        "Hello, World!"
    }

    route("/", get(handler))     // 改动:用 route 辅助函数
}

fn get_foo() -> Router {
    async fn handler() -> &'static str {
        "Hi from `GET /foo`"
    }

    route("/foo", get(handler))  // 改动
}

fn post_foo() -> Router {
    async fn handler() -> &'static str {
        "Hi from `POST /foo`"
    }

    route("/foo", post(handler)) // 改动
}

// 新增:把"新建 Router 并注册一条路由"这个动作收敛成一个函数
fn route(path: &str, method_router: MethodRouter<()>) -> Router {
    Router::new().route(path, method_router)
}
```

`route` 的第二个参数类型是 `MethodRouter<()>`,也就是前置知识里讲的"不带共享状态的方法路由表"。`get(handler)` 和 `post(handler)` 的返回值正是这个类型。

这一步纯粹是消除重复,没有行为变化。它也顺手让每个路由函数的最后一行读起来更像一句声明:`route("/foo", post(handler))`——路径、方法、处理器,三样东西挤在一行里,这正是本章标题所说的 "close together"。

**跑起来看**:

```bash
cargo run
```

行为和前两步完全一致,三条 curl 都验一遍确认没写坏:

```bash
curl http://127.0.0.1:3000/
curl http://127.0.0.1:3000/foo
curl -X POST http://127.0.0.1:3000/foo
```

```
Hello, World!
Hi from `GET /foo`
Hi from `POST /foo`
```

## 完整代码

`src/main.rs`:

```rust
use axum::{
    routing::{get, post, MethodRouter},
    Router,
};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .merge(root())
        .merge(get_foo())
        .merge(post_foo());

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

fn root() -> Router {
    async fn handler() -> &'static str {
        "Hello, World!"
    }

    route("/", get(handler))
}

fn get_foo() -> Router {
    async fn handler() -> &'static str {
        "Hi from `GET /foo`"
    }

    route("/foo", get(handler))
}

fn post_foo() -> Router {
    async fn handler() -> &'static str {
        "Hi from `POST /foo`"
    }

    route("/foo", post(handler))
}

fn route(path: &str, method_router: MethodRouter<()>) -> Router {
    Router::new().route(path, method_router)
}
```

## 这个模式什么时候值得用

官方示例展示了这个模式,但没说它的适用边界,这里补上。

**它解决的问题**:路径声明和处理器实现的距离。改一条路由只需要动一个函数,读一个 handler 立刻知道它挂在哪。

**它的代价**:

每条路由多一层函数包装,还多一次 `Router` 的创建和合并。启动时的开销可以忽略,但代码行数确实变多了——三条路由从 15 行变成 25 行。

`main` 里仍然有一份 `.merge()` 清单,新增路由还是要改两个地方(加函数 + 加 merge)。它把"改动的距离"缩短了,但没有消除。

内层 `async fn handler` 不能捕获外层变量,所以如果几条路由需要共享一些初始化好的东西,这个模式就不太顺手了——那种情况该用 `State`(第 22 章)。

**更适合的场合**:路由数量中等(十几条到几十条)、按功能横向切分的项目。

**不太适合的场合**:路由极少时(本章这种规模)平铺更清楚;路由极多且有明显的层级结构时(比如 `/api/v1/users/*`、`/api/v1/orders/*`),应该用 `nest` 按模块拆成子路由,那是[第 23 章 todos](23-todos.md) 和[第 48 章 versioning](48-versioning.md) 的做法。

实际项目里这几种手段常常混用:大层级用 `nest` 拆模块,模块内部用本章的方式或者平铺。没有必须二选一的道理。

## 动手练习

**1. 加一条 `DELETE /foo`,复用同一个模式。**

提示:`cargo add` 不需要,`delete` 就在 `axum::routing` 里。写完用 `curl -i -X DELETE http://127.0.0.1:3000/foo` 验证,并注意 `PATCH` 请求的 `allow` 头部现在列出了几个方法。

**2. 把 `get_foo` 和 `post_foo` 合并成一个 `foo()` 函数,内部两个 handler 分别叫 `get_handler` 和 `post_handler`。**

提示:用 `get(get_handler).post(post_handler)` 构造一个含两个分支的 `MethodRouter`,`route` 辅助函数不用改。做完对比一下两种写法:一个路径一个函数(本练习)和一个方法一个函数(原示例),你觉得哪种更好?什么情况下会倒过来?

## 小结与下一章预告

axum 的路由是两层的:`Router` 按路径分发并给出 `404`,`MethodRouter` 按方法分发并给出 `405`,泛型参数 `S` 表示共享状态的类型,不需要状态时是 `()`。`Router::merge` 做平级合并,同路径的不同方法能合并成功,同路径同方法会在启动时 panic。

把"一条路由 + 它的处理器"包进一个返回 `Router` 的函数,是一种缩短路径声明与实现之间距离的组织方式。它有代价,适用于中等规模、横向切分的项目,和 `nest` 拆模块并不冲突。

前两章的 handler 都没有参数,只能返回固定内容。真实的服务要读取客户端发来的数据——查询字符串、表单、JSON。[第 03 章 form](03-form.md) 引入 axum 最重要的机制:**提取器**(extractor)。你会看到 `async fn handler(Form(input): Form<SignUp>)` 这样的签名如何自动完成"解析 body、按字段反序列化、失败时返回 `400`"这一整套动作。
