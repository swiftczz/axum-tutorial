# Axum 官方 examples 手写教程

这套教程把 `examples/` 目录里的官方示例拆成一个个可手写完成的章节，按 FastAPI 风格组织：**Cargo.toml → 完整代码 → 运行 → 解读 → 手写任务 → 小结**。

## 运行前提

教程里的 `Cargo.toml` 统一用 crates.io 版本（`axum = "0.8"`、`edition = "2024"`），读者 `cargo add axum` 即可安装。examples 源码本身用的是 `path = "../../axum"` 本地路径依赖，教程做了转换。

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

### 阶段 3：状态、依赖注入、业务 API 和测试

| 顺序 | 教程文件 | 对应 example | 目标 |
| --- | --- | --- | --- |
| 16 | `chapters/16-todos.md` | `examples/todos` | 手写完整 REST Todo API |
| 17 | `chapters/17-key-value-store.md` | `examples/key-value-store` | 手写内存 KV 服务，综合状态、Bytes、中间件和 admin routes |
| 18 | `chapters/18-dependency-injection.md` | `examples/dependency-injection` | 理解 trait object 与泛型依赖注入 |
| 19 | `chapters/19-error-handling.md` | `examples/error-handling` | 设计真实项目里的 `AppError` 和错误日志 |
| 20 | `chapters/20-anyhow-error-response.md` | `examples/anyhow-error-response` | 用 `anyhow` 包装内部错误并接入 `?` |
| 21 | `chapters/21-testing.md` | `examples/testing` | 不启动端口测试 Router，并理解 `ServiceExt::oneshot` |

### 阶段 4：中间件、日志、跨域、压缩和指标

| 顺序 | 教程文件 | 对应 example | 目标 |
| --- | --- | --- | --- |
| 22 | `chapters/22-tracing-aka-logging.md` | `examples/tracing-aka-logging` | 使用 `TraceLayer` 做结构化 HTTP 日志 |
| 23 | `chapters/23-print-request-response.md` | `examples/print-request-response` | 手写打印请求/响应 body 的中间件 |
| 24 | `chapters/24-request-id.md` | `examples/request-id` | 为请求生成和传播 request id |
| 25 | `chapters/25-cors.md` | `examples/cors` | 理解浏览器跨域和 `CorsLayer` |
| 26 | `chapters/26-compression.md` | `examples/compression` | 学会请求解压与响应压缩 |
| 27 | `chapters/27-prometheus-metrics.md` | `examples/prometheus-metrics` | 用中间件采集 Prometheus 指标 |

### 阶段 5：页面、模板和静态资源

| 顺序 | 教程文件 | 对应 example | 目标 |
| --- | --- | --- | --- |
| 28 | `chapters/28-templates.md` | `examples/templates` | 用 Askama 渲染 HTML 模板 |
| 29 | `chapters/29-templates-minijinja.md` | `examples/templates-minijinja` | 用 MiniJinja 和 state 管理模板环境 |
| 30 | `chapters/30-static-file-server.md` | `examples/static-file-server` | 用 `ServeDir`/`ServeFile` 提供静态文件和 SPA fallback |
| 31 | `chapters/31-auto-reload.md` | `examples/auto-reload` | 开发期自动重载的使用方式和边界 |

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

### 阶段 9：部署边界

| 顺序 | 教程文件 | 对应 example | 目标 |
| --- | --- | --- | --- |
| 45 | `chapters/45-graceful-shutdown.md` | `examples/graceful-shutdown` | 实现优雅关闭和请求超时 |
| 46 | `chapters/46-tls-rustls.md` | `examples/tls-rustls` | 使用 Rustls 提供 HTTPS |
| 47 | `chapters/47-tls-graceful-shutdown.md` | `examples/tls-graceful-shutdown` | 组合 TLS 与优雅关闭 |
| 48 | `chapters/48-simple-router-wasm.md` | `examples/simple-router-wasm` | 了解 Axum router 在 wasm 场景下的限制和用法 |
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
| A0 | `chapters/a0-tower-fundamentals.md` | 多章节共享的前置概念 | 把贯穿教程的 Tower `Service` 与 `Layer` 抽象集中讲透，被第 17/22/23/25/26/30 章引用。第一次学可跳过，遇到 `.layer(...)`、`nest_service` 看不懂时回查 |
| A1 | `chapters/a1-async-graphql-note.md` | `examples/async-graphql` | 说明该 example 目录在当前 workspace 中被 exclude，只保留 README |
| A2 | `chapters/a2-how-to-practice.md` | 全部章节 | 给出手写练习方法、复盘清单和常见学习路线 |
