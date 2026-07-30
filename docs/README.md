# Axum 从入门到高手

面向**会 Rust 但不懂 Web 开发**的读者。基于 [axum 官方示例](../examples/),每个示例一章,带你从空项目一步步写出来、跑起来、用 curl 验证。

写作约束见 [TUTORIAL_SPEC.md](TUTORIAL_SPEC.md)。

## 目录

### 入门

- [第 00 章 序章:亲手摸一次 HTTP](00-web-basics.md)
- [第 01 章 hello-world:第一个 axum 服务](01-hello-world.md)
- [第 02 章 routes-and-handlers-close-together:让路由和处理器待在一起](02-routes-and-handlers-close-together.md)
- [第 03 章 form:提取器与 HTML 表单](03-form.md)
- [第 04 章 templates:模板引擎与自定义 IntoResponse](04-templates.md)
- [第 05 章 templates-minijinja:运行期模板、模板继承与共享状态](05-templates-minijinja.md)

### 请求与响应

- [第 06 章 global-404-handler:兜底处理器](06-global-404-handler.md)
- [第 07 章 handle-head-request:HEAD 方法处理](07-handle-head-request.md)
- [第 08 章 parse-body-based-on-content-type:自定义提取器与内容协商](08-parse-body-based-on-content-type.md)
- [第 09 章 consume-body-in-extractor-or-middleware:body 只能读一次](09-consume-body-in-extractor-or-middleware.md)
- [第 10 章 customize-extractor-error:定制提取器错误的三种做法](10-customize-extractor-error.md)
- [第 11 章 customize-path-rejection:自定义 Path 拒绝响应](11-customize-path-rejection.md)
- [第 12 章 validator:类型对了不等于数据对了](12-validator.md)
- [第 13 章 multipart-form:文件上传与请求体限制](13-multipart-form.md)

### 错误处理与工程化

- [第 14 章 error-handling:一个应用该怎么组织错误](14-error-handling.md)
- [第 15 章 anyhow-error-response:用 anyhow 装下所有错误](15-anyhow-error-response.md)
- [第 16 章 dependency-injection:trait 对象还是泛型](16-dependency-injection.md)
- [第 17 章 tracing-aka-logging:中间件与结构化日志](17-tracing-aka-logging.md)
- [第 18 章 print-request-response:把两个方向的 body 都打出来](18-print-request-response.md)
- [第 19 章 request-id:给每个请求一个身份](19-request-id.md)
- [第 20 章 cors:浏览器的同源策略](20-cors.md)
- [第 21 章 testing:不启动服务器也能测](21-testing.md)

### 状态与数据库

- 第 22 章 key-value-store:内存 KV 存储
- 第 23 章 todos:完整 CRUD 应用
- 第 24 章 sqlx-postgres:PostgreSQL (sqlx)
- 第 25 章 tokio-postgres:PostgreSQL (tokio-postgres)
- 第 26 章 diesel-postgres:PostgreSQL (diesel)
- 第 27 章 diesel-async-postgres:PostgreSQL (diesel-async)
- 第 28 章 tokio-redis:Redis
- 第 29 章 mongodb:MongoDB

### 认证与安全

- 第 30 章 jwt:JSON Web Token 认证
- 第 31 章 oauth:OAuth 授权
- 第 32 章 tls-rustls:TLS (rustls)

### 实时通信

- 第 33 章 sse:Server-Sent Events
- 第 34 章 websockets:WebSocket
- 第 35 章 testing-websockets:测试 WebSocket
- 第 36 章 websockets-http2:WebSocket over HTTP/2
- 第 37 章 chat:聊天室(综合项目)

### 内容分发与代理

- 第 38 章 static-file-server:静态文件服务
- 第 39 章 compression:响应压缩
- 第 40 章 stream-to-file:流式写文件
- 第 41 章 reqwest-response:转发 reqwest 响应
- 第 42 章 http-proxy:HTTP 代理
- 第 43 章 reverse-proxy:反向代理
- 第 44 章 serve-with-hyper:直接用 hyper 驱动

### 生产部署

- 第 45 章 graceful-shutdown:优雅停机
- 第 46 章 tls-graceful-shutdown:TLS + 优雅停机
- 第 47 章 prometheus-metrics:Prometheus 指标
- 第 48 章 versioning:API 版本管理
- 第 49 章 auto-reload:开发热重载

### 专题

- 第 50 章 low-level-rustls:底层 TLS (rustls)
- 第 51 章 low-level-native-tls:底层 TLS (native-tls)
- 第 52 章 low-level-openssl:底层 TLS (openssl)
- 第 53 章 unix-domain-socket:Unix 域套接字
- 第 54 章 simple-router-wasm:WASM 路由

## 环境要求

- Rust 1.75+
- curl
- 部分章节需要 Docker(PostgreSQL、Redis、MongoDB)

基于 axum 0.8.x。
