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
- [第 07 章 handle-head-request:HEAD 方法与自定义响应头](07-handle-head-request.md)
- [第 08 章 parse-body-based-on-content-type:自己实现 FromRequest](08-parse-body-based-on-content-type.md)
- [第 09 章 consume-body-in-extractor-or-middleware:body 只能读一次](09-consume-body-in-extractor-or-middleware.md)
- [第 10 章 customize-extractor-error:定制提取器错误的三种做法](10-customize-extractor-error.md)
- [第 11 章 customize-path-rejection:区分失败原因,分别给出状态码](11-customize-path-rejection.md)
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

- [第 22 章 key-value-store:共享可变状态](22-key-value-store.md)
- [第 23 章 todos:一个完整的 REST CRUD](23-todos.md)
- [第 24 章 sqlx-postgres:连接池与数据库提取器](24-sqlx-postgres.md)
- [第 25 章 tokio-postgres:换一套驱动与连接池](25-tokio-postgres.md)
- [第 26 章 diesel-postgres:ORM 与同步库的异步困境](26-diesel-postgres.md)
- [第 27 章 diesel-async-postgres:让 ORM 原生异步](27-diesel-async-postgres.md)
- [第 28 章 tokio-redis:内存存储与无类型的值](28-tokio-redis.md)
- [第 29 章 mongodb:文档数据库与完整 CRUD](29-mongodb.md)

### 认证与安全

- [第 30 章 jwt:无状态认证](30-jwt.md)
- [第 31 章 oauth:把认证委托给第三方](31-oauth.md)
- [第 32 章 tls-rustls:给服务启用 HTTPS](32-tls-rustls.md)

### 实时通信

- [第 33 章 sse:服务端主动推送](33-sse.md)
- [第 34 章 websockets:双向通信](34-websockets.md)
- [第 35 章 testing-websockets:两种测法](35-testing-websockets.md)
- [第 36 章 websockets-http2:HTTP/2 上的 WebSocket](36-websockets-http2.md)
- [第 37 章 chat:聊天室(综合项目)](37-chat.md)

### 内容分发与代理

- [第 38 章 static-file-server:静态文件服务](38-static-file-server.md)
- [第 39 章 compression:双向压缩](39-compression.md)
- [第 40 章 stream-to-file:流式写盘](40-stream-to-file.md)
- [第 41 章 reqwest-response:把别人的响应转发出去](41-reqwest-response.md)
- [第 42 章 http-proxy:CONNECT 隧道与连接升级](42-http-proxy.md)
- [第 43 章 reverse-proxy:改写请求再转发](43-reverse-proxy.md)
- [第 44 章 serve-with-hyper:把 `axum::serve` 拆开](44-serve-with-hyper.md)

### 生产部署

- [第 45 章 graceful-shutdown:让进程体面地退出](45-graceful-shutdown.md)
- [第 46 章 tls-graceful-shutdown:另一套停机 API](46-tls-graceful-shutdown.md)
- [第 47 章 prometheus-metrics:指标采集与基数陷阱](47-prometheus-metrics.md)
- [第 48 章 versioning:API 版本与一个静默的兼容性事故](48-versioning.md)
- [第 49 章 auto-reload:改代码自动重启,还不断连](49-auto-reload.md)

### 专题

- [第 50 章 low-level-rustls:自己握手,换来控制权](50-low-level-rustls.md)
- [第 51 章 low-level-native-tls:用系统自带的 TLS](51-low-level-native-tls.md)
- [第 52 章 low-level-openssl:握手拆到最细](52-low-level-openssl.md)
- [第 53 章 unix-domain-socket:不走网络的 HTTP](53-unix-domain-socket.md)
- [第 54 章 simple-router-wasm:没有服务器的 axum](54-simple-router-wasm.md)

## 环境要求

- Rust 1.75+
- curl
- 部分章节需要 Docker(PostgreSQL、Redis、MongoDB)

基于 axum 0.8.x。
