# A0. Tower 基础：Service 与 Layer

对应内容：多个章节共享的前置概念

本附录目标：把贯穿教程多个章节的 Tower `Service` 和 `Layer` 抽象集中讲清楚。读完这一篇，再回去看第 17、22、23、25、29、26 章的 `.layer(...)`、`nest_service`、`ServiceBuilder` 就不会再是黑盒。

> 本文是"参考性"附录，不需要手写。第一次学可以跳过，等到某章遇到看不懂的 `.layer(...)` 再回来查。

## 为什么单独拿出来讲

你会在很多章节看到这种写法：

````rust
let app = Router::new()
    .route("/", get(handler))
    .layer(TraceLayer::new_for_http())
    .layer(CompressionLayer::new());
````

`TraceLayer`、`CompressionLayer`、`CorsLayer`、`TimeoutLayer` 都来自一个共同的抽象：**Tower 的 Layer**。Axum 的 Router、handler、middleware 全都建立在这个抽象之上。不理解它，你只能"照抄能跑"，换成自由组合就会卡住。

理解了它，你会发现以下问题都有同一个答案：

- 为什么 `ServiceBuilder::new().layer(A).layer(B)` 里 A 在外、B 在内？
- 为什么 `route` 挂 handler，`nest_service` 挂 service？
- 为什么 `axum::serve(listener, app)` 能把 Router 当成服务跑起来？
- 为什么测试里可以 `app.oneshot(request)` 直接喂请求？

答案都指向同一个东西：**Service trait**。

## Service 是什么

一句话：**Service 是一个"接收请求、返回响应"的对象。**

它的核心定义（简化版）是这样的：

````rust
trait Service<Request> {
    type Response;
    type Error;

    fn call(&mut self, req: Request) -> Future<Output<Result<Self::Response, Self::Error>>;
}
````

注意几个关键点：

- 它是**泛型**的，参数化的是"能处理什么类型的请求"。一个 `Service<Request>` 就是"能处理 `Request` 的服务"。
- 它返回一个 `Future`，所以 service 是异步的。
- 它返回 `Result<Response, Error>`，所以可能失败。

建立这个心智模型后，很多东西就通了：

```text
你的 handler         是一个 Service<Request>
加了 middleware 的 Router  是一个 Service<Request>
ServeDir（静态文件）  是一个 Service<Request>
```

它们都是"给一个请求，产出一个响应"。Axum 的工作就是把它们拼起来。

## 为什么 handler 是 Service

你写的 handler 长这样：

````rust
async fn handler(State(db): State<Db>) -> impl IntoResponse { ... }
````

它看起来只是一个普通 async 函数，为什么能当成 Service 用？

因为 Axum 在内部为 handler 实现了 `Service<Request>`。具体地说，任何满足 `Handler<T, S>` trait 的函数，Axum 都会把它**转换**成一个实现了 `Service<Request>` 的对象。转换过程做了这些事：

```text
Request 进来
-> 调用各个 extractor 从 Request 里提取参数（State、Path、Json...）
-> 把提取出的参数传给你的函数
-> 你的函数返回 impl IntoResponse
-> 转成 Response 返回
```

所以你写的是函数，Axum 把它包成一个 service。这就是为什么 `.route("/", get(handler))` 能工作：`get(handler)` 内部把 `handler` 变成了 `MethodRouter`，而 `MethodRouter` 实现了 `Service`。

这也解释了一个常见的困惑：**handler 和 service 有什么区别？**

```text
handler 是 Axum 提供的"语法糖"。
它本质上是 service，但多了 extractor 自动提取参数的能力。
service 更底层，只接收 Request、返回 Response，没有 extractor。
```

## Layer 是什么

如果 Service 是"处理请求的东西"，那 Layer 就是"把一个 Service 包装成另一个 Service 的东西"。

简化定义：

````rust
trait Layer<S> {
    type Service: Service<Request>;

    fn layer(&self, inner: S) -> Self::Service;
}
````

读法：`Layer` 接收一个内层 service `S`，返回一个新的 service。这个新 service 通常会在调用 `inner` 之前或之后做点额外的事。

举个具体例子，`TraceLayer` 包装后的 service 行为是：

```text
收到 Request
-> 记录开始时间、打印"请求来了"
-> 调用 inner service（真正的 handler）
-> 拿到 Response
-> 记录结束时间、打印"请求完成，耗时 X"
-> 返回 Response
```

所以 middleware（中间件）在 Tower 里的本质就是：**一个 Layer，它产生的新 service 在调用内层 service 前后插入了逻辑。**

## 关键：Layer 的"洋葱"顺序

这是初学者最容易卡的地方。看这段代码：

````rust
ServiceBuilder::new()
    .layer(A)
    .layer(B)
    .layer(C)
````

请求流向是 **A → B → C → handler → C → B → A**，像洋葱一样一层层穿过。

```text
请求 →  [A 前] → [B 前] → [C 前] → handler → [C 后] → [B 后] → [A 后] → 响应
```

记住一条规则：

```text
先 layer() 的在外层，后 layer() 的在内层。
请求从外向内穿过去，响应从内向外穿回来。
```

这很反直觉，因为代码是从上往下写的，但执行时"上面的"反而最先看到请求。可以这样理解：`ServiceBuilder::new().layer(A)` 把 A 套在最外面，再加 `.layer(B)` 是在 A 内部再套一层 B，所以 B 在 A 里面。

### 在 Router 上 `.layer(...)` 的顺序

直接在 Router 上调 `.layer(...)` 也遵循同样的规则：

````rust
let app = Router::new()
    .route("/", get(handler))
    .layer(TraceLayer::new_for_http())   // 外层
    .layer(CompressionLayer::new());      // 内层（更靠近 handler）
````

请求先经过 TraceLayer（打日志），再经过 CompressionLayer（压缩），最后到 handler。响应反过来，先被 CompressionLayer 压缩，再被 TraceLayer 记录。

### 为什么 request-id 的顺序很重要

第 24 章会出现这种写法：

````rust
ServiceBuilder::new()
    .layer(SetRequestIdLayer)        // 最外层：先给请求打上 id
    .layer(TraceLayer)                // 中间：打日志时能读到 id
    .layer(PropagateRequestIdLayer);  // 最内层：把 id 写进响应头
````

现在你能理解为什么是这个顺序了：

- 请求进来，SetRequestIdLayer 先执行，生成 id 放进请求。
- 然后 TraceLayer 执行，打日志时从请求里读 id，这样每条日志都带 id。
- 最后到 handler。

如果顺序反了，TraceLayer 在 SetRequestIdLayer 之前，打日志时 id 还没生成，日志里就没有 id。这就是"顺序决定一切"的典型例子。

## `route` vs `nest_service` vs `route_service`

这三个 API 的区别困扰很多人，理解 Service 后就清楚了：

| API | 接收什么 | 什么时候用 |
| --- | --- | --- |
| `.route("/path", get(handler))` | handler（有 extractor 能力） | 写业务接口 |
| `.route_service("/path", service)` | service（无 extractor） | 挂一个现成的 service，如 `ServeFile::new("...")` |
| `.nest_service("/api", service)` | service（无 extractor） | 把整个 service 挂到一个前缀下，如 `ServeDir::new("assets")` |

为什么会有 `route_service` 和 `nest_service`？

因为有些东西天生就是 service，不是 handler。比如：

- `ServeDir`（提供静态文件）——它内部要处理路径匹配、读文件、返回响应，这套逻辑不依赖 extractor，直接实现成 `Service` 更自然。
- `ServeFile`（返回单个文件）——同理。

你不能把它们塞进 `get(handler)`，因为 handler 需要是 `Handler` trait，而这些类型实现的是 `Service` trait。所以 Axum 提供了 `route_service` 和 `nest_service` 来挂 service。

`route_service` 和 `nest_service` 的区别在于前缀：

- `route_service("/foo", svc)`：只匹配精确的 `/foo`。
- `nest_service("/assets", svc)`：匹配 `/assets` 下所有路径，比如 `/assets/a.js`、`/assets/css/b.css`，剩余路径（`a.js`、`css/b.css`）会传给 svc。

## ServiceBuilder：批量套 layer

当你需要套很多 layer 时，写一长串 `.layer().layer()` 会很啰嗦，而且顺序容易搞错。`ServiceBuilder` 是更清晰的写法：

````rust
// 这两种写法等价（但下面的更清晰）
let layered = app
    .layer(A)
    .layer(B)
    .layer(C);

let layered = app.layer(
    ServiceBuilder::new()
        .layer(A)
        .layer(B)
        .layer(C)
);
````

`ServiceBuilder` 把多个 layer 组合成一个 layer，最后用一次 `.layer(...)` 挂上去。顺序规则不变：先 `.layer()` 的在外层。

## `oneshot`：测试时直接喂请求

理解 Service 后，第 5、45 章的测试代码就不神秘了：

````rust
let response = app
    .oneshot(Request::get("/").body(Body::empty()).unwrap())
    .await
    .unwrap();
````

`oneshot` 来自 `tower::ServiceExt`，它的意思是：**用这个 service 处理一次请求**。因为 Router 实现了 `Service<Request>`，所以你可以直接给它一个请求，拿回响应，不用真的启动端口。

这让测试又快又稳：没有网络、没有端口占用、没有超时。

## 一张图总结

```text
你写的 handler
    ↓ Axum 用 Handler trait 把它包成 Service
    ↓
Router（也是一个 Service<Request>）
    ↓ 用 .layer() 套上各种 Layer
    ↓
带 middleware 的 Router（还是一个 Service<Request>）
    ↓
axum::serve(listener, app)
    ↓ 把 Router 当成 Service，对每个进来的请求调一次 .call()
    ↓
HTTP 响应返回给客户端
```

整条链路的底层抽象就是 Service。Axum 的所有功能（路由、extractor、middleware、测试）都是在这个抽象上构建的。

## 什么时候回来看这篇

如果在以下场景卡住了，回来看对应小节：

| 卡住的地方 | 看哪一节 |
| --- | --- |
| `.layer(A).layer(B)` 顺序搞不清 | "Layer 的洋葱顺序" |
| 为什么有 `nest_service` 和 `route` 两种 | "route vs nest_service vs route_service" |
| 测试里 `oneshot` 是什么 | "oneshot：测试时直接喂请求" |
| handler 和 service 到底什么关系 | "为什么 handler 是 Service" |
| request-id 的 layer 顺序 | "为什么 request-id 的顺序很重要" |

## 本附录真正要记住什么

- Service 是"接收请求、返回响应"的对象，Axum 里几乎一切都是 Service。
- handler 是 Axum 提供的语法糖，本质是带 extractor 的 service。
- Layer 把一个 Service 包装成另一个 Service，这就是 middleware 的本质。
- `ServiceBuilder::new().layer(A).layer(B)` 里 A 在外、B 在内，请求从外向内穿过。
- `route` 挂 handler，`route_service` / `nest_service` 挂 service。
- `oneshot` 能不启动端口直接测试，因为 Router 是 Service。
