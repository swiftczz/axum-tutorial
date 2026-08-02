# 第 07 章 handle-head-request:HEAD 方法与自定义响应头

## 本章你将做出什么

一个路由 `GET /get-head`,它同时能正确响应 `HEAD` 请求。`GET` 会做一些"计算"然后返回 body,`HEAD` 则跳过计算只返回头部。

这一章的例子很短,但涉及两个实用知识点:axum 如何自动处理 `HEAD`(以及为什么有时你想手动接管),和如何在返回值里带上自定义响应头。

学完本章你将掌握:

- `HEAD` 方法的用途,以及 axum 对它的默认处理
- 用 `Method` 提取器在 handler 内部区分请求方法
- 数组形式的元组返回类型 `([(&str, &str); N], T)` 如何设置自定义响应头
- 为什么"只在计算 body 有成本时才特殊处理 HEAD"

## 前置知识

### HEAD 方法是干什么的

[第 00 章](00-web-basics.md#方法告诉服务器你想干什么)列过它:`HEAD` 和 `GET` 一样,但**只要响应头部,不要 body**。

它的实际用途是"在不下载内容的前提下查询内容的元信息":

- **检查资源是否存在**——只看状态码是 `200` 还是 `404`,省掉下载
- **查文件大小**——读 `Content-Length` 决定要不要下载、或者显示进度条
- **检查是否更新**——读 `Last-Modified` 或 `ETag`,配合缓存判断本地副本是否还新鲜
- **健康检查**——监控系统探测服务活着就行,不关心返回什么

HTTP 规范要求 `HEAD` 的响应头必须和同一资源的 `GET` 响应头**一致**(包括 `Content-Length` 要报告 body 本该有的长度),只是不发 body。

用 curl 发 `HEAD` 请求是 `-I` 参数(大写 i),或者 `-X HEAD`。前者更常用。

### axum 默认怎么处理 HEAD

这是本章最需要记住的一点:**`get(handler)` 注册的路由会自动响应 `HEAD` 请求。**

axum 的做法是:收到 `HEAD` 请求时,照常调用 `GET` 的 handler,拿到完整响应后**把 body 丢掉**再发出去。

所以你什么都不做,`HEAD` 就已经语义正确了——头部和 `GET` 一致,body 为空。这也解释了[第 01 章](01-hello-world.md)那个只注册了 `get` 的服务,为什么 `curl -I` 也能正常工作。

那本章还讲什么?讲**这个默认行为的代价**。

因为 axum 是"先完整生成响应,再丢掉 body",所以生成 body 的开销一分不少地付了。如果 handler 里要查数据库、渲染大模板、或者做图像处理,`HEAD` 请求会白白消耗这些资源——算完就扔。

对于轻量 handler 这无所谓;但当**计算 body 有明显成本**时,值得手动区分:检测到是 `HEAD` 就跳过计算,直接返回头部。这正是本章示例演示的做法。

反过来说,如果 body 很便宜,**不要**手动区分——多写的代码有维护成本,还容易让 `HEAD` 和 `GET` 的头部不一致(违反规范)。示例源码的注释专门强调了这点:

> it usually only makes sense to special-case HEAD if computing the body has some relevant cost

### Method 提取器

想在 handler 里知道当前请求用的是什么方法,提取 `axum::http::Method` 就行:

```rust
async fn handler(method: Method) -> Response {
    if method == Method::HEAD { /* ... */ }
}
```

`Method` 是 `http` crate 的类型(axum 从它重导出),表示 HTTP 方法。常量有 `Method::GET`、`Method::POST`、`Method::HEAD` 等,支持 `==` 比较。

它属于 `FromRequestParts` 类别的提取器,和[第 06 章](06-global-404-handler.md#fallback匹配不上时交给谁)提到的 `Uri`、`HeaderMap` 一样——axum 直接为这些常用的请求信息类型实现了提取,不用包装。所以参数写法是 `method: Method` 而不是 `Method(method): Method<...>`。

注意这里没有解构,因为 `Method` 本身就是你要的值,不是像 `Form<T>`、`Path<T>` 那样的包装类型。

### 用数组设置响应头

[第 06 章](06-global-404-handler.md#元组作为返回类型)那张表里有一行 `([(&str, &str); N], T)`。本章用它设置自定义响应头:

```rust
([("x-some-header", "header from GET")], "body from GET")
```

第一个元素是一个**数组**,每个元素是 `(头部名, 头部值)` 的二元组;第二个元素是 body。状态码默认 `200`。

数组只有一个元素时也要写方括号——`[("x-some-header", "...")]` 是长度为 1 的数组,不是别的东西。多个头部就多写几项:

```rust
([("x-a", "1"), ("x-b", "2")], "body")
```

也可以只返回头部不带 body:

```rust
([("x-some-header", "header from HEAD")]).into_response()
```

数组本身实现了 `IntoResponse`(生成一个只有头部、body 为空的 `200` 响应),所以能单独用。

> **更好的写法**:示例的头部名叫 `x-some-header`。历史上 `X-` 前缀表示"非标准的自定义头部",但 [RFC 6648](https://www.rfc-editor.org/rfc/rfc6648) 早在 2012 年就建议废弃这个约定——很多 `X-` 头部后来变成了事实标准,前缀反倒成了包袱(`X-Forwarded-For` 就是典型)。现在的建议是**直接用有意义的名字**,不加前缀。示例沿用 `x-` 只是老习惯,你自己命名时不必跟随。

头部名大小写不敏感,axum/hyper 会统一转成小写发出,这点[第 01 章](01-hello-world.md#步骤-1能跑起来的最小服务)见过。

### 返回类型为什么是 Response

本章 handler 的返回类型是 `Response`,不是 `impl IntoResponse`:

```rust
async fn get_head_handler(method: http::Method) -> Response
```

原因是[第 04 章](04-templates.md#步骤-3为自己的类型实现-intoresponse)讲过的限制:**`impl Trait` 要求所有返回分支是同一个具体类型。** 而这个 handler 有两个分支:

- `HEAD` 分支返回 `[(&str, &str); 1]`
- `GET` 分支返回 `([(&str, &str); 1], &str)`

这是两个不同的类型,用 `impl IntoResponse` 编译不过。解法是两个分支都调用 `.into_response()` 转成统一的 `Response` 类型——`Response` 是 axum 里响应的"擦除类型后"的统一表示。

这个套路很常用:**handler 有多个返回形态时,返回 `Response` 并在每个分支末尾加 `.into_response()`。** 另一种解法是用 `Result<T, E>`([第 05 章](05-templates-minijinja.md#步骤-1布局模板--首页-认识-state)用过),但那适合"成功/失败"的二分,不适合本章这种"两种成功形态"。

## 项目搭建

```bash
cargo new handle-head-request
cd handle-head-request
```

本章依赖最少,只有两个——示例用 `println!` 而非 `tracing`:

```bash
cargo add axum
cargo add tokio --features full
```

## 分步构建

### 步骤 1:一个普通的 GET 路由

**要做什么**:先注册一个普通的 `GET` 路由,带一个自定义响应头。然后观察 axum 对 `HEAD` 的默认处理。

```rust
// src/main.rs
use axum::response::{IntoResponse, Response};
use axum::{routing::get, Router};

fn app() -> Router {
    Router::new().route("/get-head", get(get_head_handler))
}

#[tokio::main]
async fn main() {
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app()).await.unwrap();
}

async fn get_head_handler() -> Response {
    do_some_computing_task();

    ([("x-some-header", "header from GET")], "body from GET").into_response()
}

fn do_some_computing_task() {
    // TODO
}
```

`fn app() -> Router` 的用途见[第 03 章](03-form.md#步骤-1显示表单页面),这里注意它被放在了 `main` **之前**——Rust 不在意顶层项的顺序,官方示例这么排是想让读者先看到路由表。

**`do_some_computing_task()` 是个空函数。** 它代表"生成 body 需要付出的成本"——真实场景里可能是数据库查询、模板渲染、外部 API 调用。示例用一个空函数加 `// TODO` 占位,是为了让"这里有成本"这个意图在代码结构上可见,而不必真写一堆无关逻辑。本章保留它原样。

`main` 里 `axum::serve(listener, app())` 直接调用 `app()`,没有中间变量。

**跑起来看**:

```bash
cargo run
```

先发普通 `GET`:

```bash
curl -i http://127.0.0.1:3000/get-head
```

```
HTTP/1.1 200 OK
x-some-header: header from GET
content-type: text/plain; charset=utf-8
content-length: 13

body from GET
```

自定义头部 `x-some-header` 出现了,body 也在。

现在发 `HEAD` 请求,`-I` 参数:

```bash
curl -I http://127.0.0.1:3000/get-head
```

```
HTTP/1.1 200 OK
x-some-header: header from GET
content-type: text/plain; charset=utf-8
content-length: 13
```

**注意三件事:**

1. **没有报 `405`。** 路由只注册了 `get`,但 `HEAD` 请求成功了——这就是前置知识说的 axum 默认行为。
2. **头部完全一致**,包括 `content-length: 13`。规范要求 `HEAD` 报告 body 本该有的长度,axum 做到了。
3. **body 没有了。** curl 显示到 `content-length` 就结束。

但有个问题看不见:**`do_some_computing_task()` 被调用了。** axum 是先完整生成响应再丢 body,所以计算成本照付。当前这个函数是空的,看不出差别;下一步让它变得可见。

### 步骤 2:显式区分 HEAD

**要做什么**:用 `Method` 提取器判断方法,`HEAD` 请求跳过计算。

```rust
// src/main.rs
use axum::response::{IntoResponse, Response};
use axum::{http, routing::get, Router};        // 改动:加上 http

// ...(app 和 main 不变)

// GET routes will also be called for HEAD requests but will have the response body removed.
// You can handle the HEAD method explicitly by extracting `http::Method` from the request.
async fn get_head_handler(method: http::Method) -> Response {   // 改动:加上 method 参数
    // 新增:HEAD 请求直接返回头部,跳过计算
    // it usually only makes sense to special-case HEAD
    // if computing the body has some relevant cost
    if method == http::Method::HEAD {
        return ([("x-some-header", "header from HEAD")]).into_response();
    }

    // then do some computing task in GET
    do_some_computing_task();

    ([("x-some-header", "header from GET")], "body from GET").into_response()
}

// ...(do_some_computing_task 不变)
```

`http::Method` 这个写法(而不是 `use axum::http::Method` 后直接写 `Method`)和[第 04 章](04-templates.md#步骤-1用-path-提取器接收路径参数)的 `extract::Path` 是同一个风格考虑:带上模块前缀,读代码时一眼看出它来自 HTTP 层。

`HEAD` 分支用 `return` 提前返回,只给头部不给 body。注意头部值故意写成 `header from HEAD`,和 `GET` 分支的 `header from GET` 不同——这纯粹是为了让测试和 curl 能区分两条路径确实走了不同分支。

**严格来说这违反了 HTTP 规范**:`HEAD` 和 `GET` 的响应头应该一致。示例这么写是为了教学时可观测,真实代码里两个分支的头部应该完全相同,只是 body 有无之分。这也提醒一件事——手动区分 `HEAD` 的风险恰恰在于容易让两者的头部失去同步。

**跑起来看**:

```bash
cargo run
```

`GET` 的行为不变:

```bash
curl -i http://127.0.0.1:3000/get-head
```

```
HTTP/1.1 200 OK
x-some-header: header from GET
content-type: text/plain; charset=utf-8
content-length: 13

body from GET
```

`HEAD` 现在走了自己的分支:

```bash
curl -I http://127.0.0.1:3000/get-head
```

```
HTTP/1.1 200 OK
x-some-header: header from HEAD
content-length: 0
```

**三处变化值得注意:**

1. **`x-some-header` 变成了 `header from HEAD`**,证明走的是新分支,`do_some_computing_task()` 没被调用。
2. **`content-length` 变成了 `0`**,不再是 `13`。因为这个分支根本没生成 body,axum 也就无从知道 body 本该多长。
3. **`content-type` 消失了。** 同理,没有 body 就没有内容类型。

第 2、3 点正是上面说的"违反规范"的具体表现:一个严格遵循规范的 `HEAD` 响应应该报告 `content-length: 13` 和 `content-type`,让客户端能据此判断"值不值得下载"。示例为了展示分支差异牺牲了这一点。

**所以真实项目里该怎么写?** 如果计算成本确实值得优化,`HEAD` 分支应该手动补上正确的头部:

```rust
if method == http::Method::HEAD {
    return (
        [
            ("x-some-header", "header from GET"),
            ("content-type", "text/plain; charset=utf-8"),
            ("content-length", "13"),
        ],
    )
        .into_response();
}
```

能这么写的前提是你**不算 body 也知道它多长**——比如长度存在数据库里、或者是个已知大小的文件。如果连长度都得算一遍才知道,那手动区分 `HEAD` 就没有意义了,不如让 axum 自动处理。

**这就是本章真正的判断标准:** 手动区分 `HEAD` 只在"生成 body 昂贵、但获知元信息便宜"时才划算。两者成本相当的话,默认行为是更好的选择——代码更少,规范一致性还免费。

> 官方示例的 `main.rs` 末尾还有一个 `#[cfg(test)] mod tests`,两个测试分别断言 `GET` 返回 `header from GET` 加完整 body、`HEAD` 返回 `header from HEAD` 且 body 为空。测试写法见[第 21 章](21-testing.md)。

## 完整代码

`src/main.rs`:

```rust
use axum::response::{IntoResponse, Response};
use axum::{http, routing::get, Router};

fn app() -> Router {
    Router::new().route("/get-head", get(get_head_handler))
}

#[tokio::main]
async fn main() {
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app()).await.unwrap();
}

// GET routes will also be called for HEAD requests but will have the response body removed.
// You can handle the HEAD method explicitly by extracting `http::Method` from the request.
async fn get_head_handler(method: http::Method) -> Response {
    // it usually only makes sense to special-case HEAD
    // if computing the body has some relevant cost
    if method == http::Method::HEAD {
        return ([("x-some-header", "header from HEAD")]).into_response();
    }

    // then do some computing task in GET
    do_some_computing_task();

    ([("x-some-header", "header from GET")], "body from GET").into_response()
}

fn do_some_computing_task() {
    // TODO
}
```


## 动手练习

**1. 让 `do_some_computing_task` 变得可观测,证明 `HEAD` 真的跳过了它。**

提示:在函数体里加一行 `println!("computing...")`,或者 `std::thread::sleep(std::time::Duration::from_secs(2))`。然后分别 `curl -i` 和 `curl -I`,对比终端输出和响应耗时(用 `time curl ...`)。这一步能让"跳过计算"这件事从抽象变具体。

**2. 修正 `HEAD` 分支的规范一致性。**

提示:让 `HEAD` 分支返回和 `GET` 完全相同的 `x-some-header`,并手动补上 `content-type` 和 `content-length: 13`。然后对比 `curl -i` 和 `curl -I` 的头部——除了 body 有无,其他应该完全一样。做完思考:如果 body 长度是动态的(比如取决于数据库里有多少条记录),你还能这么做吗?

**3. 显式注册一个 `head` 路由,看看会发生什么。**

提示:改成 `get(get_head_handler).head(head_handler)`,`head_handler` 返回点别的东西。验证 `curl -I` 走的是哪个 handler。这说明了显式注册和自动处理之间的优先关系。

## 小结与下一章预告

`get(handler)` 注册的路由自动响应 `HEAD` 请求,做法是完整生成响应后丢掉 body——语义正确,但生成 body 的成本照付。用 `Method` 提取器可以在 handler 里显式区分,跳过昂贵的计算;代价是要自己保证 `HEAD` 和 `GET` 的响应头一致,而这很容易写漏。**只在"算 body 贵、知元信息便宜"时才值得手动区分。**

数组形式的返回值 `([(名, 值); N], body)` 用来设置自定义响应头。handler 有多个不同形态的返回分支时,返回类型用 `Response` 并在各分支调用 `.into_response()`。

前几章的提取器都是 axum 内置的现成货。[第 08 章 parse-body-based-on-content-type](08-parse-body-based-on-content-type.md) 开始动手造:你会**自己实现 `FromRequest`**,做一个能同时接受 JSON 和表单的提取器——根据请求的 `Content-Type` 自动选择解析方式。那一章会讲清 `FromRequest` 和 `FromRequestParts` 的实现细节,以及为什么消耗 body 的提取器只能有一个。
