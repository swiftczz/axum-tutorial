# Axum 从入门到高手

面向**会 Rust 但不懂 Web 开发**读者的 axum 中文教程。基于 [axum 官方示例](https://github.com/tokio-rs/axum/tree/main/examples),每个示例独立成一章,带领你**从空项目一步步把代码写出来、跑起来、用 curl 验证**,最终能用 axum 独立开发生产级服务。

> axum 是 Rust 生态里基于 Tokio + Tower 的 Web 框架,由 tokio-rs 团队维护。

## 适合谁

- 会 Rust(所有权、trait、泛型、基本 async),想用 Rust 做 Web 后端,但**完全不懂 Web**(HTTP、路由、cookie、CORS……)。
- 用过 Express / Flask / Spring / Gin 等其他语言框架,想对比 Rust 生态的写法。
- 不想只读 API 文档,想通过「亲手敲一遍官方示例」建立工程直觉。

完全没写过 Rust 的话,建议先过一遍 [The Rust Book](https://doc.rust-lang.org/book/) 前 10 章。

## 教程特点

- **这是"带着写",不是"带着读"**:每章从 `cargo new` 开始分步构建,每步都可编译、可运行、可验证,不是先贴完整代码再逐行讲解。
- **原理优先于语法**:讲解的重点是"为什么这么设计"——`Handler` trait 与 `IntoResponse` 如何让任意返回类型变成 HTTP 响应、`Layer` 的嵌套顺序、`State` 的 Clone 语义、body 为什么只能消费一次。
- **故意破坏式教学**:对"静默失败"的易错点,设计专门的步骤让你亲手制造一次错误(比如去掉 `StatusCode::NOT_FOUND` 后 404 变 200),再改回来——比写一句"记得要 XX"有效得多。
- **官方示例的镜像**:章末"完整代码"与 `examples/` 官方源码逻辑一致,读者脱离示例目录也能自查;官方示例的缺陷(不安全、过时写法)会指出但不修改,并给出更好的替代方案。
- **全套 55 章统一约束**:写作规范见 [TUTORIAL_SPEC.md](docs/TUTORIAL_SPEC.md),每章结构、篇幅、交叉引用都有统一约定。

## 仓库结构

```
axum-tutorial/
├── docs/
│   ├── README.md          # 教程总目录(55 章链接与学习顺序)
│   ├── TUTORIAL_SPEC.md   # 教程写作约束(唯一约束来源)
│   ├── 00-web-basics.md   # 序章:Web 基础速成(HTTP、请求/响应、URL、状态码)
│   ├── 01-hello-world.md  # 第 01 章:最小 axum 服务
│   ├── 02-routes-and-handlers-close-together.md
│   └── ...                # 共 55 章,扁平编号,按学习曲线排序
└── examples/              # axum 官方示例源码(学习素材,不修改)
```

- **`docs/`**:教程正文,55 章按学习曲线重排,连续编号即推荐阅读顺序。
- **`examples/`**:来自 axum 官方仓库 main 分支的学习素材(共 54 个有代码的示例),教程每章对应一个。
- 注意:官方示例的 `Cargo.toml` 用 `path = "../../axum"` 本地路径依赖,不可直接运行;教程每章用 `cargo add` 引导读者添加 crates.io 已发布版本,以已发布版 API 为准。

## 快速开始

从[第 00 章 序章](docs/00-web-basics.md)开始,按编号顺序阅读。每章都是独立项目,照章操作:

```bash
cargo new hello-world
cd hello-world
cargo add axum
# ...按章节步骤写代码
cargo run
curl http://127.0.0.1:3000/
```

完整目录见 [docs/README.md](docs/README.md)。

## 学习阶段

| 阶段 | 章节 | 主题 |
|---|---|---|
| 入门 | 00~05 | Web 基础、最小服务、路由组织、表单、模板 |
| 请求与响应 | 06~13 | 404/HEAD、Content-Type、自定义提取器、校验、文件上传 |
| 错误处理与工程化 | 14~21 | 错误模式、依赖注入、日志、请求 ID、CORS、测试 |
| 状态与数据库 | 22~29 | 共享状态、完整 CRUD、PostgreSQL/Redis/MongoDB |
| 认证与安全 | 30~32 | JWT、OAuth、TLS |
| 实时通信 | 33~37 | SSE、WebSocket、聊天室 |
| 内容分发与代理 | 38~44 | 静态文件、压缩、流式转发、代理 |
| 生产部署 | 45~49 | 优雅停机、指标、版本管理、热重载 |
| 专题 | 50~54 | 底层 TLS、Unix 域套接字、WASM |

## 环境要求

- Rust 1.75+(教程基于 axum 0.8.x)
- curl
- 部分章节需要 Docker(PostgreSQL、Redis、MongoDB)
