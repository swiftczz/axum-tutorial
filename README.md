# Axum 中文手写教程

通过手写 [Axum][axum] 官方 `examples/` 里的每一个示例，从零学会 Rust Web 后端开发。

> Axum 是 Rust 生态里基于 Tokio + Tower 的 Web 框架，由 tokio-rs 团队维护。

## 适合谁

- 会一点 Rust（看过 ownership / async / trait），想用 Rust 做 Web 后端。
- 用过 Express / Flask / Spring / Gin 等其他语言框架，想转 Rust。
- 不想只读 API 文档，想通过"亲手把每个示例敲一遍"建立直觉。

完全没写过 Rust 的话，建议先过一遍 [The Rust Book][rust-book] 前 10 章。

## 教程特点

- **一个 example 一个项目**：不罗列 API，每章解决一个真实后端问题。
- **逐句中文注释版源码**：适合边理解边手写。
- **讲"为什么"不只是"怎么做"**：Arc 的作用、body 只能消费一次、Tower Service 抽象、JWT 为什么不能销毁、WebSocket 为什么需要心跳——这些文档里查不到的工程认知是重点。
- **横向交叉引用**：前 30 章建立的附录 A0（Tower Service / Layer）等基础概念被后续章节直接引用，避免重复。

## 仓库结构

```
axum-tutorial/
├── examples/              # Axum 官方示例（55 个 crate），对照源码
│   ├── hello-world/       # 每个 example 一个独立 crate
│   ├── Cargo.toml         # workspace 根（members = ["*"]）
│   └── Cargo.lock
└── docs/
    └── tutorial/
        ├── README.md      # 教程总目录（学习顺序、进度清单）
        ├── CONVENTIONS.md # 章节编写规范
        └── chapters/      # 58 篇教程正文（01-55 + 附录 A0/A1/A2）
```

- `examples/` 是**对照源码**：教程按官方 example 的真实布局写，手写完可对照。
- `docs/tutorial/chapters/` 是**教程正文**：每章一份 Markdown，从"这个项目在做什么"讲到"本章要记住什么"。

## 学习路线

从 [`01-hello-world.md`][ch01] 开始，按顺序手写。每章固定流程：

```
读"这个项目在做什么" → 关掉源码 → 自己手写 → 运行验证 → 报错后回文档定位 → 和 examples/ 对照
```

卡住时查附录 [A0《Tower 基础：Service 与 Layer》][a0]——它是贯穿多章的核心概念。

### 10 个学习阶段

| 阶段 | 章节 | 主题 |
| --- | --- | --- |
| 1 | 01-05 | 最小 HTTP 服务：Router、handler、serve |
| 2 | 06-15 | 请求提取器与响应：Json、Form、Multipart、自定义 extractor |
| 3 | 16-21 | 状态、依赖注入、错误处理与测试 |
| 4 | 22-27 | 中间件、日志、CORS、压缩、监控指标 |
| 5 | 28-31 | 模板、静态文件、自动重载 |
| 6 | 32-37 | 数据库：SQLx、tokio-postgres、Diesel、Redis、MongoDB |
| 7 | 38-39 | 认证授权：JWT、OAuth |
| 8 | 40-44 | 实时通信：SSE、WebSocket、聊天室 |
| 9 | 45-50 | 部署边界：测试 WebSocket、优雅关闭、TLS、Hyper、Unix domain socket、WASM |
| 10 | 51-55 | 代理、低层 TLS（rustls / native-tls / openssl） |
| 附录 | A0-A2 | Tower 基础、async-graphql 说明、练习方法 |

完整阶段目标、对应 example 和进度清单见 [教程总目录][tutorial-readme]。

## 快速运行

```bash
cd examples
cargo run -p example-hello-world
```

`examples/*/Cargo.toml` 用的是 `axum = { path = "../../axum" }` 本地路径依赖，需要项目根目录存在 `axum/` crate；或改成 crates.io 版本（`axum = "0.8"`）。教程正文已统一转换成 crates.io 版本，详见 [教程总目录的"运行前提"][tutorial-readme]。

**工具链**：Rust ≥ 1.75。

## 进度

- ✅ 55 个正文章节（01-55）已全部完成
- ✅ 3 个附录（A0 Tower 基础 / A1 async-graphql 说明 / A2 练习方法）已全部完成

详见 [教程总目录的写作进度][tutorial-readme-progress]。

## 贡献

发现章节有错误、表述不准，或想补充某个概念？欢迎提 issue 或 PR。尤其是：

- 技术细节的准确性（API 签名、RFC 引用）
- 初学者容易卡住的地方
- 横向交叉引用的补充

## 致谢

- 示例源自 [tokio-rs/axum][axum-repo] 的 `examples/` 目录。
- Axum、Tokio、Tower、Hyper 都是 tokio-rs 团队的开源作品。

[axum]: https://github.com/tokio-rs/axum
[axum-repo]: https://github.com/tokio-rs/axum
[rust-book]: https://doc.rust-lang.org/book/
[tutorial-readme]: docs/tutorial/README.md
[tutorial-readme-progress]: docs/tutorial/README.md#当前写作进度
[ch01]: docs/tutorial/chapters/01-hello-world.md
[a0]: docs/tutorial/chapters/a0-tower-fundamentals.md
