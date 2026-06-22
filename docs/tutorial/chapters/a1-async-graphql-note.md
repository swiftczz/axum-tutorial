# A1. async-graphql-note

对应内容：`examples/async-graphql`

本附录目标：说明为什么本教程没有为 `async-graphql` 写完整代码教程，以及当前仓库中这个 example 的真实状态。

## 当前目录状态

当前仓库中：

```text
examples/async-graphql/
```

只保留了一个文件：

```text
README.md
```

内容是：

```text
See <https://github.com/async-graphql/examples>.
```

没有 `Cargo.toml`，没有 `src/main.rs`，也没有可运行的 Axum example 代码。

## workspace 中的状态

`examples/Cargo.toml` 里明确写了：

````toml
[workspace]
members = ["*"]
# Example has been deleted, but README.md remains
exclude = ["async-graphql", "target"]
````

这表示：

```text
async-graphql example 已经被删除
当前 workspace 排除了 async-graphql 目录
只保留 README 指向外部 async-graphql examples
```

所以它不适合像前面 55 章那样按本仓库源码逐行讲解。

## 为什么不补一个完整 GraphQL 教程

本教程的原则是：

```text
围绕当前仓库里真实存在的 example 源码教学
```

如果这里强行补 GraphQL 代码，会出现两个问题：

1. 教程内容不再对应本仓库 example。
2. 读者无法在当前目录中对照源码手写。

所以本附录只说明状态，不虚构一章完整 GraphQL example。

## 如果你想继续学 GraphQL

建议路线：

1. 先完成本教程前面的 Router、extractor、state、error handling、testing。
2. 再去 async-graphql 官方 examples 看 Axum 集成方式。
3. 学习 GraphQL schema、query、mutation、resolver。
4. 最后再把数据库、认证、错误处理接进 GraphQL。

## 带中文注释的手写版

本附录没有可手写源码。  
唯一需要记住的是 workspace 配置：

````toml
[workspace]
members = ["*"]
# Example has been deleted, but README.md remains
exclude = ["async-graphql", "target"]
````

这说明 `async-graphql` 不属于当前可运行 examples。

## 运行和验证

验证目录内容：

````bash
find examples/async-graphql -maxdepth 2 -type f | sort
````

预期只看到：

```text
examples/async-graphql/README.md
```

验证 workspace 排除：

````bash
rg -n 'exclude = \\["async-graphql", "target"\\]' examples/Cargo.toml
````

## 手写任务

本附录没有 Axum 代码手写任务。

建议做两个检查：

1. 打开 `examples/Cargo.toml`，确认 `async-graphql` 被 exclude。
2. 打开 `examples/async-graphql/README.md`，确认它只指向外部 examples。

## 本章真正要记住什么

教程不要为了“完整”而虚构源码。  
当前仓库里 `async-graphql` 已经不是可运行 example，所以只做状态说明。

## 源码对照

- `examples/Cargo.toml`
- `examples/async-graphql/README.md`
