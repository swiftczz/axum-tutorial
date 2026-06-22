# Axum 中文手写教程

通过手写 [Axum][axum] 官方 `examples/` 里的每一个示例，从零学会 Rust Web 后端开发。

这套教程不假设你有后端经验。我们会从"什么是 HTTP 请求"讲起，一路走到数据库、认证、WebSocket、TLS、反向代理。每个示例都是一个小而完整的项目，配一份逐句中文注释的源码、运行验证步骤和手写练习。

> Axum 是 Rust 生态里基于 Tokio + Tower 的 Web 框架，由 tokio-rs 团队维护。

## 这套教程适合谁

- 会一点 Rust（看过 ownership / async / trait），想用 Rust 做 Web 后端。
- 用过其他语言的后端框架（Express、Flask、Spring、Gin），想转 Rust。
- 不想只读 API 文档，想通过"亲手把每个示例敲一遍"建立直觉。

如果你完全没写过 Rust，建议先过一遍 [The Rust Book][rust-book] 的前 10 章，再回来。

## 仓库结构

```
axum-tutorial/
├── examples/              # Axum 官方示例（55 个 crate + async-graphql 说明）
│   ├── hello-world/       # 每个 example 是一个独立 crate
│   ├── readme/            #   有自己的 Cargo.toml 和 src/main.rs
│   ├── ...
│   ├── Cargo.toml         # workspace 根（members = ["*"]）
│   └── Cargo.lock         # workspace 依赖锁定
└── docs/
    └── tutorial/
        ├── README.md      # 教程总目录（学习顺序、进度清单）
        └── chapters/      # 58 篇教程正文（01-55 + 附录 A0/A1/A2）
```

- `examples/` 是**对照源码**：教程章节按官方 example 的真实布局写，手写完后可以和这里对照。
- `docs/tutorial/chapters/` 是**教程正文**：每章一份 Markdown，从"这个项目在做什么"讲到"本章真正要记住什么"。

## 如何阅读

### 快速上手

1. **先读 [教程总目录][tutorial-readme]**，了解 10 个学习阶段的划分。
2. 从 [`01-hello-world.md`][ch01] 开始，按顺序手写。
3. 每章固定流程：

   ```
   读"这个项目在做什么" → 关掉源码 → 自己手写 → 运行验证 → 报错后回文档定位 → 和 examples/ 对照
   ```

4. 卡住时查附录 [A0《Tower 基础：Service 与 Layer》][a0]，它是贯穿多章的核心概念。

### 学习阶段一览

| 阶段 | 章节 | 主题 |
| --- | --- | --- |
| 1 | 01-05 | 最小 HTTP 服务：Router、handler、serve |
| 2 | 06-15 | 请求提取器与响应：Json、Form、Multipart、自定义 extractor |
| 3 | 16-20 | 状态、依赖注入、错误处理 |
| 4 | 21-26 | 中间件、日志、CORS、压缩、监控指标 |
| 5 | 27-31 | 模板、静态文件、自动重载、WASM |
| 6 | 32-37 | 数据库：SQLx、tokio-postgres、Diesel、Redis、MongoDB |
| 7 | 38-39 | 认证授权：JWT、OAuth |
| 8 | 40-44 | 实时通信：SSE、WebSocket、聊天室 |
| 9 | 45-50 | 测试、优雅关闭、TLS、Hyper、Unix domain socket |
| 10 | 51-55 | 代理、低层 TLS（rustls / native-tls / openssl） |
| 附录 | A0-A2 | Tower 基础、async-graphql 说明、练习方法 |

完整的阶段目标、对应 example 和进度清单见 [教程总目录][tutorial-readme]。

## 运行前提

这些示例对照的是 Axum 官方 `examples/` 的源码布局。`examples/*/Cargo.toml` 里目前用的是本地路径依赖：

```toml
axum = { path = "../../axum" }
```

所以 `cargo run` 之前需要满足以下任一条件：

1. **项目根目录下存在 `axum/` crate**（即 `examples/../axum/Cargo.toml` 能找到）。
2. **或者把 example 的 `Cargo.toml` 改成 crates.io 依赖**，例如 `axum = "0.8"`，并按实际版本调整代码。

> 如果当前仓库只有 `examples/` 和 `docs/`，没有根目录 `axum/`，教程里的运行命令会因为找不到本地依赖而失败。这是仓库依赖布局问题，不是章节代码问题。详见 [教程总目录的"运行前提"一节][tutorial-readme]。

```bash
# 进入 examples workspace
cd examples

# 运行某个示例（example crate 名以 example- 为前缀）
cargo run -p example-hello-world

# 运行测试
cargo test -p example-hello-world
```

**工具链**：Rust ≥ 1.75（见 `examples/Cargo.toml` 的 `rust-version`）。

## 教程特点

- **一个 example 就是一个小项目**，不是 API 罗列。每章解决一个真实后端问题。
- **逐句中文注释版源码**，适合边理解边手写。
- **讲"为什么"不只是"怎么做"**：Arc 的作用、body 只能消费一次、Tower Service 抽象、JWT 为什么不能销毁、WebSocket 为什么需要心跳……这些文档里查不到的工程认知，是教程的重点。
- **横向交叉引用**：前 30 章建立了附录 A0（Tower Service/Layer）等基础概念，后续章节直接引用，避免重复。
- **每章有手写任务和源码对照路径**，形成"学—练—验证"闭环。

## 进度

- ✅ 55 个正文章节（01-55）已全部完成
- ✅ 3 个附录（A0 Tower 基础 / A1 async-graphql 说明 / A2 练习方法）已全部完成
- ✅ README 清单已全部落盘

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
