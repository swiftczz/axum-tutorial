# A2. how-to-practice

对应内容：全部章节

本附录目标：给出这套 Axum 教程的手写练习方法、复盘清单和学习路线，帮助不会后端的人真正把代码写出来。

## 这套教程应该怎么学

不要只看文档。  
这套教程最有效的学法是：

```text
看一小节
关掉源码
自己手写
运行验证
报错后回文档定位
最后和 examples 源码对照
```

后端学习靠“把请求跑通”建立直觉。

## 每章固定练习流程

建议每章都按这个顺序：

1. 先读“这个小项目在做什么”。
2. 用自己的话画出请求主线。
3. 看“文件和依赖”，确认需要哪些 crate。
4. 手写最小可运行版本。
5. 再补中间件、错误处理、状态、测试等细节。
6. 跑 curl 或浏览器验证。
7. 最后看“本章真正要记住什么”。

## 初学者最该抓住的主线

Axum 后端大多数代码都围绕这条线：

```text
Router
-> route
-> extractor
-> handler
-> response
-> layer/state/error/test
```

只要你能回答下面几个问题，就说明理解在变扎实：

```text
这个请求进哪个 route？
handler 参数从哪里来？
body 是否会被消费？
state 存了什么？
错误如何变成响应？
响应 body 是怎么回到客户端的？
```

## 推荐学习路线

### 第一轮：只求跑通

建议章节：

```text
01-05
06-10
16
21
24
27
29
45
46
```

目标：

```text
会启动服务
会写路由
会读 Path/Query/Json/Form
会返回 JSON/HTML
会写基础测试
```

### 第二轮：补工程能力

建议章节：

```text
11-15
17-20
22-26
30-31
```

目标：

```text
会自定义 extractor
会做错误处理
会写 middleware
会做日志、request id、指标
会理解部署和开发工具边界
```

### 第三轮：接外部系统

建议章节：

```text
32-39
```

目标：

```text
会连接数据库和缓存
会理解连接池
会写认证
会理解 session/JWT/OAuth 的差异
```

### 第四轮：实时通信和底层部署

建议章节：

```text
40-55
```

目标：

```text
会 SSE
会 WebSocket
会测试 WebSocket
理解 TLS
理解低层 Hyper/TCP/TLS 接入
理解代理模型
```

## 每章复盘清单

学完一章后问自己：

1. 这个 example 的入口函数是哪一个？
2. Router 注册了哪些路径？
3. 每个 handler 的参数是什么 extractor？
4. 有没有共享 state？
5. 有没有 middleware/layer？
6. 失败时返回什么状态码？
7. 如何用 curl/browser/test 验证？
8. 真实项目里这章代码有什么边界？

如果答不上来，回到对应章节重新手写一遍。

## 常见学习误区

### 1. 一开始就追求架构完美

先把请求跑通。  
再抽象错误、状态、服务层和测试。

### 2. 只看不写

后端很多知识要靠报错建立直觉。  
不手写，很难真正理解 extractor、body、state、lifetime、async 错误。

### 3. 不做验证

每章都至少做一种验证：

```text
curl
浏览器
cargo test
日志
响应 header
```

### 4. 把示例代码直接当生产代码

examples 为了教学会简化很多东西。  
例如写死账号密码、返回内部错误、使用内存 store、自签名证书。

真实项目要补安全、配置、日志、测试、超时、限流、监控。

## 带中文注释的手写版

本附录没有单一源码。  
可以把下面当成每章练习模板：

````text
1. 新建最小 main.rs
2. 写 Router::new()
3. 注册一个 route
4. 写 handler
5. 启动 listener
6. curl 验证
7. 加 extractor
8. 加 state
9. 加 error
10. 加 test
````

## 运行和验证

检查当前教程文件数量：

````bash
find docs/tutorial/chapters -maxdepth 1 -type f -name '*.md' | sort | wc -l
````

检查 README 是否还有未完成项：

````bash
rg -n '^- \\[ \\]' docs/tutorial/README.md
````

没有输出，说明清单已完成。

## 手写任务

最终练习建议：

1. 不看源码，重写 `01-hello-world`。
2. 不看源码，重写 `16-todos`。
3. 给 todos 加 `TraceLayer`、request id 和错误处理。
4. 给 todos 写测试。
5. 把 todos 接 PostgreSQL。
6. 给 todos 加 JWT 保护。

这条路线能把教程里的核心能力串起来。

## 本章真正要记住什么

学后端不是背 API，而是建立请求生命周期：

```text
请求进来
提取数据
执行业务
访问状态或外部系统
处理错误
返回响应
记录日志和指标
测试验证
部署运行
```

每章 example 都是在这条生命周期上补一个能力。

## 源码对照

- `docs/tutorial/README.md`
- `docs/tutorial/chapters/`
- `examples/`
