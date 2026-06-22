# Axum 官方 examples 手写教程

这套教程的目标不是做 Axum API 总览，而是把 `examples/` 目录里的官方示例拆成一个个可手写完成的章节。

每一章都按“一个 example 就是一个小项目”的方式写：

1. 先说明这一章到底在做什么，以及这个 example 解决的后端问题。
2. 列出项目文件地图，讲清楚每个文件在项目里的职责。
3. 给出运行方式和建议手写顺序。
4. 按请求生命周期说明从 `Router` 到 extractor、handler、`IntoResponse` 的处理流程。
5. 解释为什么要这样写，重点讲 Axum/Tower/Rust 背后的原理。
6. 对新概念多的章节做函数解读，对重复度高的章节改用“函数职责速查”。
7. 展示逐句中文注释版代码，方便新手边理解边手写。
8. 给出运行验证、常见卡点和手写任务。
9. 最后列出源码对照路径，方便和 `examples/` 里的真实文件比较。

先写完一章，再写下一章。每个章节文件都独立保存，避免一次性写超长文档导致中断后难以恢复。

## 运行前提

这些章节对照的是 Axum 官方 examples 的源码布局。当前 `examples/*/Cargo.toml` 里使用的是本地路径依赖：

````toml
axum = { path = "../../axum" }
````

所以运行 `cargo run -p ...` 或 `cargo test -p ...` 前，需要满足其中一种条件：

1. 项目根目录下存在 `axum/` crate，也就是 `examples/foo/../../axum` 能找到 `Cargo.toml`。
2. 或者把对应 example 的 `Cargo.toml` 改成 crates.io 依赖，例如 `axum = "0.x"`，并根据实际版本调整代码。

如果当前仓库只保存了 `examples/` 和 `docs/`，没有根目录 `axum/`，教程里的运行命令会因为找不到本地 `axum` 依赖而失败。这不是章节代码问题，而是仓库依赖布局问题。

## 文件命名规则

- 章节文件放在 `docs/tutorial/chapters/`。
- 文件名前缀使用两位序号，表示建议学习顺序。
- 文件名后半部分尽量和 `examples/` 子目录保持一致。
- 如果一个 example 有多个 bin 或多个源码文件，仍然先用一个章节覆盖，再在章节内拆小节。

## 学习顺序与文件清单

### 阶段 1：从 0 到能写最小 HTTP 服务

| 顺序 | 教程文件 | 对应 example | 目标 |
| --- | --- | --- | --- |
| 01 | `chapters/01-hello-world.md` | `examples/hello-world` | 手写第一个 Axum 服务，理解 `Router`、`route`、`handler`、`serve` |
| 02 | `chapters/02-readme-json-api.md` | `examples/readme` | 手写 GET/POST JSON API，理解 `Json<T>` 和 `IntoResponse` |
| 03 | `chapters/03-routes-and-handlers-close-together.md` | `examples/routes-and-handlers-close-together` | 学会按功能组织路由和 handler |
| 04 | `chapters/04-global-404-handler.md` | `examples/global-404-handler` | 学会用 `fallback` 处理未知路由 |
| 05 | `chapters/05-handle-head-request.md` | `examples/handle-head-request` | 理解 GET 与 HEAD 的关系，以及何时特殊处理 HEAD |

### 阶段 2：请求提取器与响应类型

| 顺序 | 教程文件 | 对应 example | 目标 |
| --- | --- | --- | --- |
| 06 | `chapters/06-form.md` | `examples/form` | 手写 HTML 表单提交，理解 `Form<T>` |
| 07 | `chapters/07-multipart-form.md` | `examples/multipart-form` | 手写 multipart 文件上传，理解 body limit |
| 08 | `chapters/08-stream-to-file.md` | `examples/stream-to-file` | 学会把请求体/上传文件流式写入磁盘 |
| 09 | `chapters/09-parse-body-based-on-content-type.md` | `examples/parse-body-based-on-content-type` | 自定义 `JsonOrForm<T>` 提取器 |
| 10 | `chapters/10-consume-body-in-extractor-or-middleware.md` | `examples/consume-body-in-extractor-or-middleware` | 理解请求体只能消费一次，以及如何重新构造 body |
| 11 | `chapters/11-customize-extractor-error.md` | `examples/customize-extractor-error` | 自定义 JSON 提取失败时的错误响应 |
| 12 | `chapters/12-customize-path-rejection.md` | `examples/customize-path-rejection` | 自定义路径参数解析错误 |
| 13 | `chapters/13-validator.md` | `examples/validator` | 把提取和参数校验组合成 `ValidatedForm<T>` |
| 14 | `chapters/14-versioning.md` | `examples/versioning` | 用自定义提取器实现 API 版本解析 |
| 15 | `chapters/15-reqwest-response.md` | `examples/reqwest-response` | 理解如何把外部 HTTP 响应转成 Axum 响应 |

### 阶段 3：状态、依赖注入和业务 API

| 顺序 | 教程文件 | 对应 example | 目标 |
| --- | --- | --- | --- |
| 16 | `chapters/16-todos.md` | `examples/todos` | 手写完整 REST Todo API |
| 17 | `chapters/17-key-value-store.md` | `examples/key-value-store` | 手写内存 KV 服务，综合状态、Bytes、中间件和 admin routes |
| 18 | `chapters/18-dependency-injection.md` | `examples/dependency-injection` | 理解 trait object 与泛型依赖注入 |
| 19 | `chapters/19-error-handling.md` | `examples/error-handling` | 设计真实项目里的 `AppError` 和错误日志 |
| 20 | `chapters/20-anyhow-error-response.md` | `examples/anyhow-error-response` | 用 `anyhow` 包装内部错误并接入 `?` |

### 阶段 4：中间件、日志、跨域、压缩和指标

| 顺序 | 教程文件 | 对应 example | 目标 |
| --- | --- | --- | --- |
| 21 | `chapters/21-tracing-aka-logging.md` | `examples/tracing-aka-logging` | 使用 `TraceLayer` 做结构化 HTTP 日志 |
| 22 | `chapters/22-print-request-response.md` | `examples/print-request-response` | 手写打印请求/响应 body 的中间件 |
| 23 | `chapters/23-request-id.md` | `examples/request-id` | 为请求生成和传播 request id |
| 24 | `chapters/24-cors.md` | `examples/cors` | 理解浏览器跨域和 `CorsLayer` |
| 25 | `chapters/25-compression.md` | `examples/compression` | 学会请求解压与响应压缩 |
| 26 | `chapters/26-prometheus-metrics.md` | `examples/prometheus-metrics` | 用中间件采集 Prometheus 指标 |

### 阶段 5：页面、模板和静态资源

| 顺序 | 教程文件 | 对应 example | 目标 |
| --- | --- | --- | --- |
| 27 | `chapters/27-templates.md` | `examples/templates` | 用 Askama 渲染 HTML 模板 |
| 28 | `chapters/28-templates-minijinja.md` | `examples/templates-minijinja` | 用 MiniJinja 和 state 管理模板环境 |
| 29 | `chapters/29-static-file-server.md` | `examples/static-file-server` | 用 `ServeDir`/`ServeFile` 提供静态文件和 SPA fallback |
| 30 | `chapters/30-auto-reload.md` | `examples/auto-reload` | 开发期自动重载的使用方式和边界 |
| 31 | `chapters/31-simple-router-wasm.md` | `examples/simple-router-wasm` | 了解 Axum router 在 wasm 场景下的限制和用法 |

### 阶段 6：数据库、缓存和外部存储

| 顺序 | 教程文件 | 对应 example | 目标 |
| --- | --- | --- | --- |
| 32 | `chapters/32-sqlx-postgres.md` | `examples/sqlx-postgres` | 把 `PgPool` 放入 state，并写数据库连接提取器 |
| 33 | `chapters/33-tokio-postgres.md` | `examples/tokio-postgres` | 使用 `tokio-postgres` 接入 PostgreSQL |
| 34 | `chapters/34-diesel-postgres.md` | `examples/diesel-postgres` | 使用 Diesel 同步 PostgreSQL 示例 |
| 35 | `chapters/35-diesel-async-postgres.md` | `examples/diesel-async-postgres` | 使用 async Diesel PostgreSQL 示例 |
| 36 | `chapters/36-tokio-redis.md` | `examples/tokio-redis` | 使用 Redis 连接池和自定义连接提取器 |
| 37 | `chapters/37-mongodb.md` | `examples/mongodb` | 使用 MongoDB client/pool 作为应用状态 |

### 阶段 7：认证、授权和会话

| 顺序 | 教程文件 | 对应 example | 目标 |
| --- | --- | --- | --- |
| 38 | `chapters/38-jwt.md` | `examples/jwt` | 把 JWT Claims 写成认证提取器 |
| 39 | `chapters/39-oauth.md` | `examples/oauth` | 理解 OAuth 登录、CSRF、cookie session 和用户提取器 |

### 阶段 8：实时通信

| 顺序 | 教程文件 | 对应 example | 目标 |
| --- | --- | --- | --- |
| 40 | `chapters/40-sse.md` | `examples/sse` | 使用 SSE 做服务端单向推送 |
| 41 | `chapters/41-websockets.md` | `examples/websockets` | 手写 WebSocket 升级、收发任务和客户端 bin |
| 42 | `chapters/42-chat.md` | `examples/chat` | 手写聊天室，理解广播状态和连接管理 |
| 43 | `chapters/43-websockets-http2.md` | `examples/websockets-http2` | 理解 HTTP/2 TLS 下的 WebSocket 示例 |
| 44 | `chapters/44-testing-websockets.md` | `examples/testing-websockets` | 测试 WebSocket handler |

### 阶段 9：测试、生命周期和部署边界

| 顺序 | 教程文件 | 对应 example | 目标 |
| --- | --- | --- | --- |
| 45 | `chapters/45-testing.md` | `examples/testing` | 不启动端口测试 Router，并理解 `ServiceExt::oneshot` |
| 46 | `chapters/46-graceful-shutdown.md` | `examples/graceful-shutdown` | 实现优雅关闭和请求超时 |
| 47 | `chapters/47-tls-rustls.md` | `examples/tls-rustls` | 使用 Rustls 提供 HTTPS |
| 48 | `chapters/48-tls-graceful-shutdown.md` | `examples/tls-graceful-shutdown` | 组合 TLS 与优雅关闭 |
| 49 | `chapters/49-serve-with-hyper.md` | `examples/serve-with-hyper` | 理解更底层的 Hyper 服务接入 |
| 50 | `chapters/50-unix-domain-socket.md` | `examples/unix-domain-socket` | 用 Unix domain socket 部署本地服务 |

### 阶段 10：代理和底层 TLS

| 顺序 | 教程文件 | 对应 example | 目标 |
| --- | --- | --- | --- |
| 51 | `chapters/51-http-proxy.md` | `examples/http-proxy` | 手写 HTTP 代理的基本请求转发 |
| 52 | `chapters/52-reverse-proxy.md` | `examples/reverse-proxy` | 手写反向代理，理解请求和响应透传 |
| 53 | `chapters/53-low-level-rustls.md` | `examples/low-level-rustls` | 使用低层 Rustls 接入 Axum |
| 54 | `chapters/54-low-level-native-tls.md` | `examples/low-level-native-tls` | 使用 native-tls 低层接入 |
| 55 | `chapters/55-low-level-openssl.md` | `examples/low-level-openssl` | 使用 OpenSSL 低层接入 |

### 附录

| 顺序 | 教程文件 | 对应内容 | 目标 |
| --- | --- | --- | --- |
| A0 | `chapters/a0-tower-fundamentals.md` | 多章节共享的前置概念 | 把贯穿教程的 Tower `Service` 与 `Layer` 抽象集中讲透，被第 17/22/23/25/26/29 章引用。第一次学可跳过，遇到 `.layer(...)`、`nest_service` 看不懂时回查 |
| A1 | `chapters/a1-async-graphql-note.md` | `examples/async-graphql` | 说明该 example 目录在当前 workspace 中被 exclude，只保留 README |
| A2 | `chapters/a2-how-to-practice.md` | 全部章节 | 给出手写练习方法、复盘清单和常见学习路线 |

## 当前写作进度

- [x] 整理文件名和学习顺序。
- [x] 01 `hello-world`
- [x] 02 `readme-json-api`
- [x] 03 `routes-and-handlers-close-together`
- [x] 04 `global-404-handler`
- [x] 05 `handle-head-request`
- [x] 06 `form`
- [x] 07 `multipart-form`
- [x] 08 `stream-to-file`
- [x] 09 `parse-body-based-on-content-type`
- [x] 10 `consume-body-in-extractor-or-middleware`
- [x] 11 `customize-extractor-error`
- [x] 12 `customize-path-rejection`
- [x] 13 `validator`
- [x] 14 `versioning`
- [x] 15 `reqwest-response`
- [x] 16 `todos`
- [x] 17 `key-value-store`
- [x] 18 `dependency-injection`
- [x] 19 `error-handling`
- [x] 20 `anyhow-error-response`
- [x] 21 `tracing-aka-logging`
- [x] 22 `print-request-response`
- [x] 23 `request-id`
- [x] 24 `cors`
- [x] 25 `compression`
- [x] 26 `prometheus-metrics`
- [x] 27 `templates`
- [x] 28 `templates-minijinja`
- [x] 29 `static-file-server`
- [x] 30 `auto-reload`
- [x] 31 `simple-router-wasm`
- [x] 32 `sqlx-postgres`
- [x] 33 `tokio-postgres`
- [x] 34 `diesel-postgres`
- [x] 35 `diesel-async-postgres`
- [x] 36 `tokio-redis`
- [x] 37 `mongodb`
- [x] 38 `jwt`
- [x] 39 `oauth`
- [x] 40 `sse`
- [x] 41 `websockets`
- [x] 42 `chat`
- [x] 43 `websockets-http2`
- [x] 44 `testing-websockets`
- [x] 45 `testing`
- [x] 46 `graceful-shutdown`
- [x] 47 `tls-rustls`
- [x] 48 `tls-graceful-shutdown`
- [x] 49 `serve-with-hyper`
- [x] 50 `unix-domain-socket`
- [x] 51 `http-proxy`
- [x] 52 `reverse-proxy`
- [x] 53 `low-level-rustls`
- [x] 54 `low-level-native-tls`
- [x] 55 `low-level-openssl`
- [x] A0 `tower-fundamentals`
- [x] A1 `async-graphql-note`
- [x] A2 `how-to-practice`

当前仓库实际已写完 `docs/tutorial/chapters/01-hello-world.md` 到 `55-low-level-openssl.md`，并补充 `a0-tower-fundamentals.md`、`a1-async-graphql-note.md`、`a2-how-to-practice.md` 三个附录，README 清单已全部落盘。
