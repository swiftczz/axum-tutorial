# Axum 教程写作约束文档

> 本文档是整套教程的唯一约束来源。所有文章必须遵守本文档;若有冲突,以本文档为准。
> 修改本文档需双方讨论确认后进行。
>
> 状态:**已定稿**。全书 55 章(第 00~54 章)已全部写完,本文档转为回改与维护的基准。

## 1. 定位

- **教程目标**:带领读者从零 Web 知识到能用 axum 独立开发生产级服务。
- **读者假设**:会 Rust(所有权、trait、泛型、基本 async 不再解释),**完全不懂 Web**(HTTP、路由、cookie、CORS 等概念必须在首次出现时讲透)。
- **素材来源**:本仓库 `examples/` 下的 axum 官方示例。`examples/` 共 56 个目录,逐级排除后收录 54 个,每个示例独立成一章:

  | 阶段 | 数量 | 说明 |
  |---|---|---|
  | `examples/` 下的目录总数 | 56 | |
  | 减:`async-graphql` | −1 | 目录下没有 `src/`,只有一个指向外部仓库的 README |
  | 有实际代码的示例 | 55 | |
  | 减:`readme` | −1 | 入门部分与 `hello-world` 重复;它独有的 `Json<T>` 材料已并入第 08 章(见第 7.7 节) |
  | **教程收录** | **54** | 对应第 01~54 章 |

  加上无对应示例的序章(第 00 章),**全书共 55 章**。注意区分"54 个示例"和"55 章"这两个数字。
- **产出形式**:仓库内 Markdown,统一放在 `docs/` 目录,可直接在 GitHub 阅读。

## 2. 目录组织

按学习曲线重排,不按官方示例的字母序。扁平章节结构,每个示例独立成一章,用连续编号体现推荐阅读顺序。第 00 章是序章,无对应示例;第 01~54 章与 54 个示例一一对应,全书共 55 章。阶段仅作为逻辑分组标记,不影响文件结构。

| 阶段 | 章号 | 示例名 | 主题关键词 |
|---|---|---|---|
| **入门** | 00 | (序章) | Web 基础速成:HTTP、请求/响应、URL、方法、状态码、JSON、环境准备 |
| | 01 | hello-world | 最小 axum 服务 |
| | 02 | routes-and-handlers-close-together | 路由与处理器组织 |
| | 03 | form | HTML 表单提交 |
| | 04 | templates | 服务端模板渲染(askama) |
| | 05 | templates-minijinja | 服务端模板渲染(minijinja) |
| **请求与响应** | 06 | global-404-handler | 全局 404 处理 |
| | 07 | handle-head-request | HEAD 方法处理 |
| | 08 | parse-body-based-on-content-type | 按 Content-Type 解析请求体 |
| | 09 | consume-body-in-extractor-or-middleware | Extractor 与中间件消费 body(**首个中间件**:`middleware::from_fn`) |
| | 10 | customize-extractor-error | 自定义 Extractor 错误 |
| | 11 | customize-path-rejection | 自定义 Path 拒绝响应 |
| | 12 | validator | 请求校验 |
| | 13 | multipart-form | 多部分表单上传(**首个 tower-http 中间件**:`RequestBodyLimitLayer`、`TraceLayer`,均按 B 档处理) |
| **错误处理与工程化** | 14 | error-handling | 错误处理模式 |
| | 15 | anyhow-error-response | 用 anyhow 做错误响应 |
| | 16 | dependency-injection | 依赖注入 |
| | 17 | tracing-aka-logging | 日志与追踪(tower `Layer` / `Service` 抽象与叠加顺序的主题章) |
| | 18 | print-request-response | 打印请求/响应(自定义中间件) |
| | 19 | request-id | 请求 ID |
| | 20 | cors | 跨域资源共享(中间件配置) |
| | 21 | testing | 测试 |
| **状态与数据库** | 22 | key-value-store | 内存 KV 存储 |
| | 23 | todos | 完整 CRUD 应用 |
| | 24 | sqlx-postgres | PostgreSQL (sqlx) |
| | 25 | tokio-postgres | PostgreSQL (tokio-postgres) |
| | 26 | diesel-postgres | PostgreSQL (diesel) |
| | 27 | diesel-async-postgres | PostgreSQL (diesel-async) |
| | 28 | tokio-redis | Redis |
| | 29 | mongodb | MongoDB |
| **认证与安全** | 30 | jwt | JSON Web Token 认证 |
| | 31 | oauth | OAuth 授权 |
| | 32 | tls-rustls | TLS (rustls) |
| **实时通信** | 33 | sse | Server-Sent Events |
| | 34 | websockets | WebSocket |
| | 35 | testing-websockets | 测试 WebSocket |
| | 36 | websockets-http2 | WebSocket over HTTP/2 |
| | 37 | chat | 聊天室(综合项目) |
| **内容分发与代理** | 38 | static-file-server | 静态文件服务 |
| | 39 | compression | 响应压缩 |
| | 40 | stream-to-file | 流式写文件 |
| | 41 | reqwest-response | 转发 reqwest 响应 |
| | 42 | http-proxy | HTTP 代理 |
| | 43 | reverse-proxy | 反向代理 |
| | 44 | serve-with-hyper | 直接用 hyper 驱动 |
| **生产部署** | 45 | graceful-shutdown | 优雅停机 |
| | 46 | tls-graceful-shutdown | TLS + 优雅停机 |
| | 47 | prometheus-metrics | Prometheus 指标 |
| | 48 | versioning | API 版本管理 |
| | 49 | auto-reload | 开发热重载 |
| **专题** | 50 | low-level-rustls | 底层 TLS (rustls) |
| | 51 | low-level-native-tls | 底层 TLS (native-tls) |
| | 52 | low-level-openssl | 底层 TLS (openssl) |
| | 53 | unix-domain-socket | Unix 域套接字 |
| | 54 | simple-router-wasm | WASM 路由 |

排序的第一原则是**知识依赖**:讲完 `Router` 和 handler 才能讲 extractor,讲完 `Layer` 才能讲基于中间件的功能,讲完 `State` 才能讲数据库,讲完 TLS 才能讲 `tls-graceful-shutdown`。调整章序时以此为准。

### 相近示例的处理规则

相近示例(diesel 两章、low-level-tls 三章、templates 两章等)各自独立成章,但**后章只讲与前章的差异**,不重复逐行讲解,并在开头链接前章。

### 前向引用的触发条件

知识依赖排序的必然结果是:一部分能力出现得比讲解早,这些位置必须主动把读者引到后面的章节。

**不维护逐章清单**——清单必然漏、必然过期。改为按条件自查:**本章代码里出现了某个能力,而讲透它的主题章在本章之后,就必须给出带章号的前向引用**(格式见第 4.2 节 B 档,写法见第 5.5 节)。

判断方法是机械的:对照第 7.7 节的概念归属表和下面这张主题章对照表,凡是"用到了但主题章在后面"的,一律加前向引用。

| 能力 | 主题章 |
|---|---|
| 静态资源托管(`ServeDir`、页面配 CSS/JS) | 38 static-file-server |
| tower `Layer` / `tower-http` 现成中间件 | 17 tracing-aka-logging |
| 日志与 `tracing_subscriber` 初始化样板(见第 7.5 节) | 17 tracing-aka-logging |
| 测试写法与示例自带的 `mod tests`(见第 7.6 节) | 21 testing |
| 可变共享状态与锁(见第 7.7 节) | 22 key-value-store |

新增能力时补这张表,不要在这里登记"哪些章用到了它"。

## 3. 文件与命名规范

- 目录结构:`docs/<两位章号>-<示例名>.md`,扁平放置,不分子目录。
  - 例:`docs/01-hello-world.md`、`docs/00-web-basics.md`
- `docs/README.md` 作为总目录,列出全部章节链接,随章节推进更新。

## 4. 单篇文章模板

### 4.0 核心原则:这是"带着写",不是"带着读"

这一条是整套教程的灵魂,优先于其他所有格式约束。

**教程不是对官方示例的逐行解读,而是带领读者从空项目一步步把代码写出来。** 读者的体验应该是:跟着敲一段代码 → 立刻跑起来 → 用 curl 看到效果 → 理解刚才这段代码为什么这么写 → 再敲下一段。而不是先看到一份完整代码,再听人解释每一行。

由此派生三条硬性要求:

1. **分步递进**:代码分若干步骤写出。每个步骤的产出应当是**可编译、可运行、可验证**的——哪怕功能还不完整。读者永远不应该处于"敲了一半代码,编译不过,不知道对不对"的状态。
2. **原理优先于语法**:讲解的重点是"为什么这么设计",而不是"这行代码做了什么"。`async fn root() -> &'static str` 能直接当 handler,重点是解释 axum 的 `Handler` trait 与 `IntoResponse` 如何让任意返回类型自动变成 HTTP 响应,而不是复述"这个函数返回一个字符串"。
3. **摆脱示例目录**:读者读完一章不需要打开 `examples/` 也能得到完整、可运行、与官方示例逻辑一致的代码。章末的"完整代码"就是这个保证。

### 4.1 章节结构

每篇文章包含以下部分,顺序固定。第 (6) 段条件性必填,其余必填:

#### (1) 本章你将做出什么

不是罗列知识点,而是先告诉读者**这一章最终会跑起来一个什么东西**:一句话描述成品,再跟 2~4 条学习目标。让读者带着目标感往下读。

#### (2) 前置知识

列出本章引入的、前面章节未覆盖的知识点。涵盖所有新概念:Web 概念(HTTP 方法、Content-Type、CORS……)、首次出现的 crate(serde、tower-http……)、首次出现的模式(中间件、extractor……)、首次出现的工具(Docker、openssl……)。

**每个概念全套教程只讲透一次**,这是控制全书 55 章总篇幅的关键。但"讲透"不是唯一的处理方式——见下面第 4.2 节的三档处理。

#### (3) 项目搭建

引导读者创建本章的独立项目:

1. **`cargo new` 创建项目,项目名直接用示例名**,不加 `example-` 前缀、不改写、不简化:

   ```bash
   cargo new hello-world
   cd hello-world
   ```

   序章无对应示例,项目名用 `web-basics`。其余各章一律与第 2 节表格中的"示例名"列完全一致(如 `cargo new customize-extractor-error`、`cargo new tokio-redis`)。

   注意官方示例的 `Cargo.toml` 里 `package.name` 是 `example-hello-world` 形式,教程中**不采用**这个命名——读者的项目名就是示例名本身。这一差异属于第 7.1 节允许的偏离。

2. 用 `cargo add` 添加依赖,明确 feature flags(详见第 7.2 节)
3. 如果本章需要非 Rust 文件(模板、HTML、证书、迁移脚本等),此处只创建目录骨架;文件内容留到用得上它的那个步骤再写,避免读者茫然地复制一堆还没派上用场的文件

#### (4) 分步构建(本章主体)

这是文章的核心,详细约定见第 5 节。

#### (5) 完整代码

本章所有文件的完整内容,不省略、不折叠。这一段的作用是让读者脱离 `examples/` 目录也能对照检查自己的成果,同时作为日后回查的索引。

要求:

- 每个文件一个代码块,块前标注文件路径(如 `src/main.rs`)
- 代码逻辑必须与 `examples/<示例名>/` 中的官方源码一致
- 多文件示例按依赖顺序排列
- 非 Rust 资源文件(模板、HTML、SQL 等)同样完整列出;二进制文件(证书、.gz)给出生成命令而非内容
- 本段不含讲解文字,纯代码 + 文件路径标注

#### (6) 代价与边界(默认必填)

**默认必填,标题统一写作 `## 代价与边界`。** 只有确实找不出性能陷阱、安全风险或适用边界的章节才整段省略——通常是知识点单一的早期章节。

这是本教程相对官方示例的核心增量——**官方示例只演示"怎么做",不告诉读者"什么时候别这么做"**。照抄示例进生产环境踩坑,往往就踩在这些没被说出口的前提上。

内容要求:

- 说清代价是什么(内存、并发、安全面、可维护性)
- 给出**什么时候可以这么用 / 什么时候绝对不能**的具体清单,不要泛泛而谈
- 如果有更稳妥的替代方案,指出它在哪一章

**标题例外**:早期章节有若干按内容自拟的标题(如"缓冲整个 body 的代价"、"这个模式什么时候值得用"),保持原样不改——其中一部分已被别章当作锚点链接,改标题会断链。新写或重写的章节一律用统一标题。

#### (7) 动手练习

基于本章代码的小改造题,给思路提示但不给完整答案。

数量基准:**2~4 个**。满足第 4.3 节上限放宽条件、或本章"代价与边界"里点出了多处可动手修的问题、或官方示例故意留下多处生产级缺陷时,可到 **6 个**(其中标 `(进阶)` 的不计入对普通读者的负担预期——做不做由读者自选)。

不要为凑数灌水题;也不要因为题多就删掉已经写好、能练到真实取舍的题。

#### (8) 小结与下一章预告

3~5 句小结,点明本章掌握的核心机制;再说下一章是什么、为什么接着读它。同样按第 4.3 节的放宽条件,可到 6 句。

### 4.2 概念的三档处理

按学习曲线排序的必然结果是:某些概念会在它的"主题章"之前就出现在代码里。硬要在早期章节讲透会让文章跑题、篇幅失控;完全不提又会让读者对着看不懂的代码发懵。

所以概念有三档处理方式,**按"本章要不要理解它才能往下走"来选**:

**A 档 · 讲透。** 本章的主题概念,或者不理解就无法继续的概念。每个概念全套教程只有一章做这件事。

**讲透的落点分两种,不要混:**

- **概念性的**——Web 概念、crate 的定位、某个模式是什么(如"提取器是什么"、"中间件的洋葱结构"、"body 为什么只能读一次")。放"前置知识"段,用完整段落展开。读者得先有这个概念,才看得懂后面的代码。
- **与具体代码强绑定的类型机制**——`Handler` trait 为什么能接受任意签名、`IntoResponse` 如何把返回值变成响应、`with_state` 如何把 `Router<S>` 消解成 `Router<()>`。放"分步构建"里代码出现的那一步**就地**讲透(见第 5.2 节第 2 点)。脱离代码讲这些只会更抽象。

**B 档 · 点到为止 + 前向引用。** 代码里出现了,但读者本章不需要真正理解它——照抄就能跑,理解它属于另一章的任务。用统一的引用块格式标出:

```markdown
> **本章不深入**:`Layer` 是 tower 定义的中间件抽象。这里只需知道 `.layer(...)` 能给整个 `Router` 套一层处理逻辑,照抄即可。它的组合语义和执行顺序见[第 17 章](17-tracing-aka-logging.md)。
```

格式要求:

- 用 Markdown 引用块(`>`)包裹,让读者视觉上一眼识别"这里被刻意跳过了"
- 以粗体 **本章不深入** 开头,统一措辞,不要写成"稍后会讲""后面再说"这类模糊表达
- 三句话以内:是什么(一句)、本章怎么用(一句)、去哪看细节(带章节链接)
- **必须给出具体章号和链接**,不能只写"后面章节会讲"

**例外:语法糖不走三档流程。** 不构成独立概念的小知识点——如 tracing 的字段前缀 `?body`(按 `Debug` 记录)/ `%uri`(按 `Display` 记录)、`r#"..."#` 原始字符串、变量遮蔽、字段初始化简写——允许在首次用到的地方就地一句话解释,不必起引用块也不必列进"前置知识"。判断标准:**一句话能说完、且不理解也不影响读懂本章逻辑**。同一个语法糖全书只在首次出现处解释,后面直接用。

**C 档 · 一句话回引。** 前面已经讲透过的概念,写成一句话带链接,不重复展开:

```markdown
这里又用到了 `Form` 提取器,原理见[第 03 章](03-form.md#提取器axum-的核心机制)。
```

判断标准举例。下表用早期章节的几个概念示范怎么定档,**是尺度参照,不是概念登记表**——概念归属该查第 7.7 节:

| 概念 | 出现章节 | 档位 | 理由 |
|---|---|---|---|
| `Handler` / `IntoResponse` | 01 | A | 本章主题,不懂就没法理解 handler 为何能这么写 |
| tower 的 `Service` / `Layer` | 01 提及 | B | 只是介绍生态分层,本章不用它 |
| `tracing_subscriber` 初始化 | 03 | B | 照抄就能跑,日志机制是第 17 章的主题 |
| `Form` 提取器 | 03 | A | 本章主题 |
| `State` | 05 | A | 不理解就没法理解 minijinja 的引擎为何要共享 |
| `Arc<RwLock<T>>` 可变共享状态 | 05 提及 | B | 本章状态只读,可变状态是第 22 章主题 |
| `#[cfg(test)] mod tests` | 03/04 | B | 见第 7.6 节,完整代码里省略并指向第 21 章 |

同一个概念在不同章节可以是不同档位:B 档提及过的概念,到它的主题章仍然要 A 档讲透(那一章不必回避"前面提过"),之后再出现就是 C 档。

### 4.3 篇幅基准

正文(不含"完整代码"段)**2500~4500 字**。"完整代码"段不计入。

放宽规则:

- **下限**可放到 2000。知识点单一的短章硬凑字数只会灌水。
- **上限**可放到 5000,但必须满足下列任一条件,不是"觉得复杂"就行:
  - 多文件示例(见第 7.3 节)
  - 需要读者额外跑起一个进程或服务才能验证(见第 7.8 节)
  - 一组相近示例里的**首章**——要为后续章节打底
  - "代价与边界"段涉及安全面,需要展开具体清单

注意**"复杂"和"长"不是一回事**:业务逻辑多的章节未必费笔墨,反倒是要讲清安全边界、要引入 axum 之外运行环境的章节容易超。写完超过 5000 字的,先回头找能降到 B 档的内容,而不是直接套用上面的放宽条件。

B 档处理的作用之一就是守住这个篇幅——遇到"这个也想讲清楚"的冲动时,先判断它是不是本章必需,不是就降到 B 档。

## 5. 分步构建的写法约定

### 5.1 步骤划分原则

- **每个步骤都要能跑**:步骤结束时 `cargo run` 必须能编译通过并启动。如果某步骤在技术上确实无法独立运行(例如必须同时定义 struct 和 impl 才能编译),把它们合并成一个步骤,而不是留一个编译不过的中间态。
- **每个步骤都要能验证**:步骤结束时给出验证方式——curl 命令、浏览器访问、日志输出、或 `cargo test`。让读者拿到即时反馈。
- **粒度参考**:一个步骤新增 20~80 行代码。只有一两个 handler 的最小示例可能只需 1~2 步;满足第 4.3 节上限放宽条件的章节可能需要 5~8 步。
- **按功能切分,不按文件切分**:步骤的边界是"完成了一个可观察的能力",而不是"写完了一个文件"。多文件示例中,一个步骤可以同时改动多个文件。

### 5.2 每个步骤的内部结构

固定三段,顺序固定:

1. **要做什么**:一句话说明本步骤要实现的能力。
2. **写代码 + 讲原理**:代码分块给出,每块后紧跟讲解。讲解重点是设计原理和"为什么",而非逐行翻译语法。axum 的类型机制(`Handler` trait、`IntoResponse`、`FromRequest`/`FromRequestParts`、`State` 的 Clone 语义、`Layer` 的嵌套顺序等)在首次出现时必须**就地**讲透,不要挪到"前置知识"段——两者的分工见第 4.2 节 A 档。
3. **跑起来看**:`cargo run` 启动,然后给出验证命令和**预期输出**。curl 命令要完整可复制;预期输出要包含响应体和关键的状态码/响应头。有客户端程序(如 websockets 的 `client.rs`)的说明如何在另一个终端运行;有浏览器页面(如 chat.html)的说明访问哪个地址、看到什么。

### 5.2.1 故意破坏式验证步骤

**当某个易错点的失败后果是"静默失败"——不报错、不 panic、只是结果悄悄不对——应当设计一个让读者故意破坏它的步骤。**

理由:静默失败是最难查的一类 bug,读者只有亲手制造一次、看到"什么都没发生但数据没了",才会真正记住。单纯写一句"记得要 XX"没有同等效果。

写法要求:

- 明确说"临时改成……",给出改动后的代码片段
- 给出破坏后的实际输出,并指出**关键在于没有任何报错**
- 简短说明这类 bug 在真实项目里会以什么面貌出现(便于日后对号入座)
- **必须以"改回来"结尾**,不能让读者的代码停在破坏状态

代表性写法(这个手法在全书大量使用,这里只列典型,不维护全量清单):

| 章节 | 破坏什么 | 暴露的静默失败 |
|---|---|---|
| 05 templates-minijinja | 模板变量名写错 | 编译通过、启动正常、页面静默空白 |
| 06 global-404-handler | 去掉 `StatusCode::NOT_FOUND` | 404 变成 200,前端与监控全被误导 |
| 09 consume-body-in-extractor-or-middleware | 不归还 body | 下游拿到空 body,不报错、状态码仍是 200 |

不是每章都需要这一步。判断标准:**错误后果是"响一声"(编译失败、panic、明确的错误响应)就不必设计,是"静悄悄"才需要。**

有一种情况看着像例外,其实不是:为了对比"编译期就报错"和"运行期静默出错"的差别,可以在前一类章节里也演一遍同样的错误。那是**对照组**,目的是衬托后一章,不受本节约束,也不要当成本节的写法模板照搬。

### 5.3 代码增量的展示规则

这是控制文档体积的关键约定。

**新增代码要完整展示,已写过的代码用省略号折叠,只保留定位所需的上下文锚点。**

约定写法:

```rust
// src/main.rs
use axum::{routing::get, Router};
use axum::routing::post;              // 新增

async fn main() {
    let app = Router::new()
        .route("/", get(root))
        .route("/users", post(create_user));   // 新增
    // ...(其余不变)
}

// 新增:处理 POST /users
async fn create_user(/* ... */) {
    // ...
}
```

具体规则:

- 折叠已有代码统一用 `// ...` 加中文说明,如 `// ...(前面写的 root handler 不变)`
- 保留足够的上下文锚点(所在函数签名、相邻的一两行)让读者能准确定位插入位置
- 新增或修改的行加行尾注释标记(`// 新增`、`// 改动`),或在代码块前用文字说明"在 `main` 函数的 `Router::new()` 链上追加一行"
- **不要在每个步骤都重贴全文**。第 N 步的代码块只展示第 N 步的增量 + 锚点。想看全貌的读者去看章末"完整代码"段。
- 如果某一步改动很分散(超过 3 处),允许贴出该文件的完整当前状态,但要在文字里说明"这一步改动较多,以下是 `src/main.rs` 的当前完整内容"

### 5.4 多文件示例的处理

多文件示例(见第 7.3 节列表)按功能推进,不按文件顺序推进:

- 每个代码块前必须标注文件路径,读者要清楚这段代码写在哪个文件
- 新建文件的那一步,说明为什么此时要拆出新文件(模块职责、复用需求等),并给出 `mod` 声明的改动
- `customize-extractor-error` 这类"同一问题的多种解法"示例,每种解法作为一个独立步骤,步骤之间横向对比优劣,而不是简单罗列

### 5.5 交叉引用规则

概念的三档处理见第 4.2 节。这里规定引用本身的写法,前向(指向后面章节)和回引(指向前面章节)都适用:

- **链接必须指向具体章节,能定位到小节就带锚点**,不能只写"见前文""后面会讲"
- 一句话点明"这里用到了什么",让读者自己判断是否需要跳转
- B 档的前向引用用引用块 + 粗体 **本章不深入** 开头(格式见第 4.2 节);C 档的回引直接写在正文里,不用引用块
- 前向引用**不要许诺过多**:只说那一章会讲这个概念,不预告细节

回引示例(C 档,写在正文中):

> 这里又用到了 `State` 提取器共享应用状态,`Arc` 的作用和 `with_state` 的类型变化见[第 05 章](05-templates-minijinja.md#state-与-with_state)。

相近示例(diesel 两章、low-level-tls 三章、templates 两章)的后续章节,整章都以"与前章的差异"为主线组织,共同部分全部走 C 档回引。

## 6. 写作风格约束

- 语言:简体中文;代码、命令、类型名、crate 名保留英文原文。
- Web 术语和新概念首次出现时给出英文原词,如"路由(routing)",之后直接用中文。
- Rust 语言本身的知识(所有权、生命周期、trait)不解释;但 **axum 特有的类型机制必须讲透**,这是本教程的核心价值。
- 讲解回答"为什么这么设计",不复述"这行代码做了什么"。避免"这里我们定义了一个函数"这类没有信息量的句子。
- 每个概念(Web 概念、crate 用法、工具用法、axum 机制)全套教程只讲一次,靠交叉链接复用。
- 不使用 emoji;少用列表,讲解以段落叙述为主,列表仅用于步骤和清单。
- 用第二人称"你"称呼读者,语气像同事在旁边带着写,不用"我们来看一下"这类课堂腔。

## 7. 技术基线

### 7.1 源码一致性

**章末"完整代码"段的内容,逻辑上必须与 `examples/<示例名>/` 中的官方源码一致**(依赖声明除外,见第 7.2 节)。这是"脱离 examples 也能学"的前提,也是读者自查的基准。

允许的偏离范围:

- `Cargo.toml` 不出现在教程中(改为 `cargo add` 命令)
- 项目名(`package.name`)用示例名而非官方的 `example-<示例名>`,详见第 4.1 节 (3)
- 为讲解需要补充的中文注释
- 官方示例中与教学无关的样板(如 `#![allow(...)]`、workspace lint 相关的属性)可省略
- **为适配 crates.io 已发布版本而必需的修正**,见下方 7.1.1

不允许的偏离:改变代码结构、重命名函数/类型、简化或"优化"官方写法。

#### 7.1.1 examples 是 main 分支代码,教程对齐已发布版本

仓库 `examples/` 下的代码取自 axum **main 分支**,可能领先于 crates.io 上已发布的版本。读者用 `cargo add axum` 拿到的是**已发布的稳定版**,因此教程必须以已发布版本的 API 为准,而不是照抄 examples。

已知的必须修正项:

| 位置 | examples(main 分支)写法 | 教程写法(已发布版) | 原因 |
|---|---|---|---|
| `axum::serve` | `axum::serve(listener, app).await;` | `axum::serve(listener, app).await.unwrap();` | 已发布版的 `serve` 返回 `io::Result<()>`,不处理会触发 `unused_must_use` 警告 |

处理规则:

- 这类修正**不需要在每章正文里解释**,直接按已发布版本写即可
- 仅在首次出现的章节(第 01 章)用一处引用块简短说明差异,让读者对照 examples 时不困惑;后续章节不重复提
- 写作时如发现新的此类差异,补进上表,并核对 <https://docs.rs/axum> 上已发布版本的签名后再定写法

#### 7.1.2 示例不是最佳实践时:指出来,但不改代码

`examples/` 是从 axum 官方仓库取来的学习素材,**不是生产模板**。示例为聚焦单一知识点常做简化,部分代码也可能沿用了较老的写法。发现这类情况时,**一律指出,一律不改**。

适用的三类情况。下表的"典型表现"是**用来校准判断尺度的举例,不是已知问题清单**——这条规则全书通用,不要以为表里没列的章节就没有:

| 类型 | 典型表现 |
|---|---|
| **A. 生产级缺陷** — 有安全或性能后果 | 无上限地缓冲请求体;`unwrap` 会让请求打 panic;用户可控的文件名直接落盘(路径穿越);把内部错误详情返回给客户端 |
| **B. 过时或有更好替代的写法** — 能跑,但现在有更好的做法 | 手写 `From` impl 而不用 `thiserror` 的 `#[from]`;绕远路的配置写法(如"关掉默认限制再加一层"而非直接调额度) |
| **C. 示例自身的瑕疵** — 简化过头或笔误 | 签名是 `Result` 却全程 `.unwrap()`;多写的分号导致返回空 body;校验消息与实际规则对不上;残留的 `dbg!` |

**统一处理方式(三步,不修改"完整代码"段):**

1. **在最贴近问题的位置指出**——局部写法就在"写代码 + 讲原理"里就地指出;影响全局的就放"代价与边界"段
2. **给出更好的写法**,可以附代码片段作为对照
3. **修复放进"动手练习"**,让读者自己动手;如果正确做法在后面某章讲,给出章节链接

**局部指出时用统一的引用块格式:**

```markdown
> **更好的写法**:这里手写了两个 `impl From`,用 `thiserror` 的 `#[from]` 属性可以自动生成。官方示例保留手写版本是为了直白展示 `?` 背后发生了什么,实际项目建议用 `thiserror`,见[第 10 章](10-customize-extractor-error.md#thiserror定义错误类型的省事办法)。
```

格式要求:

- 引用块 + 粗体 **更好的写法** 开头,和 B 档的 **本章不深入** 区分开
- 说清三件事:示例怎么写的、更好的写法是什么、**示例为什么这么写**(往往是为了教学清晰,不是作者疏忽)
- 不要居高临下。多数简化是有意为之,写成"官方示例这里偷懒了"既不准确也无必要

**为什么不直接改代码:**

本教程的定位是**学会 axum**,不是提供可直接上线的模板。一旦允许"改进版",章末"完整代码"就不再是官方示例的镜像,读者拿它和 `examples/` 对照时会产生困惑,"脱离 examples 也能学"的一致性基准也就失效了。

而且**缺陷本身是优质教学素材**:读者将来读到别人写的类似代码,能认出问题所在——这比给他一份完美代码更有价值。指出来比悄悄修掉强。

**边界:不要过度挑刺。** 只指出有实际后果的问题(会出安全事故、会误导读者、有明确更优的现代替代)。纯风格偏好(变量命名、是否用链式调用、注释密度)不算,不必提。

写作前必须先读对应示例的所有文件:

- `Cargo.toml` — 确认依赖列表和 feature flags
- `src/` 下所有 `.rs` 文件 — 确认代码逻辑
- 非 Rust 资源文件(模板、HTML、证书、迁移脚本、JSON 等)— 确认内容和路径

### 7.2 依赖处理

官方示例的 `Cargo.toml` 使用 `path = "../../axum"` 形式的本地路径依赖,学习者无法直接使用。教程中必须做如下转换:

- **教程中不出现完整的 `Cargo.toml`**。依赖一律通过"项目搭建"段的 `cargo add` 命令引导读者添加,明确指定 feature flags。
  - **例外**:当 `cargo add` 表达不了(如 `default-features = false`、`[lib]`/`[[bin]]` 配置),或读者需要核对 `cargo add` 生成的结果(如 feature 组合、dev-dependencies 版本)时,允许贴出 `Cargo.toml` 的**相关几行**——不贴整份文件,并说明为什么要手工确认。
- 涉及的 crate 及写法示例:
  - `axum`:  `cargo add axum` 或 `cargo add axum -F ws,macros`(按示例所需 feature)
  - `axum-extra`: `cargo add axum-extra -F typed-header`(按示例所需 feature)
  - 其他 crate(tokio、serde、tower-http 等):按示例 `Cargo.toml` 中的版本和 feature 转换为 `cargo add` 命令
- axum 版本以 crates.io 最新稳定版为准,在序章标注一次,各章不重复声明。

### 7.3 多文件示例

以下示例包含多个 `.rs` 源文件,处理方式见第 5.4 节:

| 示例 | 文件列表 |
|---|---|
| customize-extractor-error | main.rs, custom_extractor.rs, derive_from_request.rs, with_rejection.rs |
| websockets | main.rs, client.rs |
| compression | main.rs, tests.rs |
| static-file-server | main.rs, tests.rs |

### 7.4 非 Rust 资源文件

以下示例包含非 Rust 文件。目录骨架在"项目搭建"段建立,**文件内容在用得上它的那个步骤才写**:

| 示例 | 资源文件 |
|---|---|
| templates | templates/hello.html |
| templates-minijinja | templates/layout.jinja, home.jinja, about.jinja, content.jinja |
| static-file-server | assets/index.html, assets/script.js |
| sse | assets/index.html, assets/script.js |
| websockets | assets/index.html, assets/script.js |
| websockets-http2 | assets/index.html, assets/script.js, self_signed_certs/cert.pem, key.pem |
| chat | chat.html |
| compression | data/products.json, data/products.json.gz |
| diesel-postgres | migrations/…/up.sql, down.sql |
| diesel-async-postgres | migrations/…/up.sql, down.sql |
| tls-rustls | self_signed_certs/cert.pem, key.pem |
| tls-graceful-shutdown | self_signed_certs/cert.pem, key.pem |
| low-level-rustls | self_signed_certs/cert.pem, key.pem |
| low-level-native-tls | self_signed_certs/cert.pem, key.pem |
| low-level-openssl | self_signed_certs/cert.pem, key.pem |

二进制或不可手写的文件(自签名证书、`.gz` 压缩数据)一律给生成命令,不贴内容。证书生成方法在首次出现的第 32 章(tls-rustls)讲透,后续章节链接回该章。

### 7.5 tracing_subscriber 初始化样板

收录的 54 个示例中,有 43 个在 `main` 开头有这段日志初始化:

```rust
tracing_subscriber::registry()
    .with(
        tracing_subscriber::EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| format!("{}=debug", env!("CARGO_CRATE_NAME")).into()),
    )
    .with(tracing_subscriber::fmt::layer())
    .init();
```

它第一次出现在第 03 章(form),远早于讲解日志的第 17 章。处理规则:

- **保留,不省略。** 这段代码是功能性的——去掉它,示例里的 `tracing::debug!("listening on ...")` 会静默无输出,读者会以为服务没起来。
- 首次出现的第 03 章用一小段说明它的作用(注册日志订阅器、`RUST_LOG` 环境变量如何控制级别),明确告诉读者"现在照抄即可,原理见第 17 章"。
- 首次出现章与第 17 章**之间**的各章,视为已讲过的样板,只用一句话带链接,不重复解释。第 17 章讲透之后,后续章节直接用,不再提及。
- 对应依赖:`cargo add tracing` 和 `cargo add tracing-subscriber --features env-filter`。

**日志 target 名必须按读者的项目名写,不能照抄 examples。** 样板里的 `env!("CARGO_CRATE_NAME")` 在编译期展开成读者自己的 crate 名,而第 4.1 节 (3) 规定项目名不带 `example-` 前缀。所以正文里凡是给出日志预期输出的地方,target 一律写成"示例名把连字符换成下划线"的形式:

| examples 里跑出来的 | 教程里应该写的 |
|---|---|
| `DEBUG example_reqwest_response: streaming 1 bytes` | `DEBUG reqwest_response: streaming 1 bytes` |
| `ERROR example_low_level_rustls: error during tls handshake` | `ERROR low_level_rustls: error during tls handshake` |

同理,`RUST_LOG=` 的取值也按读者的 crate 名写(如 `RUST_LOG=tracing_aka_logging=debug`)。

例外:官方源码里把 `example_*` **硬编码成字符串字面量**的地方属于源码内容,按第 7.1 节照录不改。这类硬编码本身往往就是第 7.1.2 节 C 类的示例瑕疵,按那一节指出即可。

### 7.6 示例自带的测试模块

以下 11 个示例的源码里有 `#[cfg(test)] mod tests`:

`anyhow-error-response`、`compression`、`form`、`handle-head-request`、`sse`、`static-file-server`、`templates`、`testing`、`testing-websockets`、`validator`、`versioning`

处理规则:

- **第 21 章(testing)之前的章节**:章末"完整代码"**省略**测试模块,并在完整代码段前用一句话说明"官方示例还带一个 `#[cfg(test)] mod tests`,用到的 `oneshot` 等写法见第 21 章"。相应地,`cargo add --dev` 的测试依赖(`http-body-util`、`tower`、`mime` 等)也不出现在"项目搭建"段。
- **第 21 章及之后的章节**:测试模块完整保留并讲解,测试依赖正常用 `cargo add --dev` 引入。
- 官方示例中为方便测试而抽出的 `fn app() -> Router` 这类结构**必须保留**(属于源码结构,不可改),首次出现时说明它的用途并前向引用第 21 章。

### 7.7 概念首次出现位置以实际章序为准

`State` / `with_state` 原本属于第 22 章(key-value-store)的核心内容,但 `templates-minijinja`(第 05 章)已经用到了它。按"每个概念只讲一次"的规则,**概念归属以它在章序中首次出现的位置为准**,不是以哪一章"主题上更该讲它"为准。

具体分工:

- 第 05 章讲 `State` 的基础:为什么需要共享状态、`Arc` 的作用、`with_state` 如何把 `Router<S>` 消解成 `Router<()>`
- 第 22 章在此基础上讲**可变**共享状态:`RwLock`/`Mutex` 的选择、`Clone` 语义的细节、状态设计的取舍,开头链接回第 05 章

遇到同类情况(某概念在其"主题章"之前就被用到)一律照此处理:前面的章节讲够用的部分,主题章讲深入部分并链接回去。

已识别的同类情况:

| 概念 | 首次出现 | 分工 |
|---|---|---|
| `State` / `with_state` | 05 templates-minijinja | 05 讲只读共享状态与 `Arc`;22 key-value-store 讲可变状态与锁 |
| `Json<T>`(收 JSON / 发 JSON) | 08 parse-body-based-on-content-type | 被砍掉的 `readme` 示例本是它的入门材料,砍掉后无处讲解。已在 08 章"前置知识"补一节独立讲透:提取器与响应包装两副面孔、`Deserialize`/`Serialize` 分工、`(StatusCode, Json<T>)` 返回 201 |
| 自定义提取器(`FromRequest` 手写实现) | 08 parse-body-based-on-content-type | 08 讲 trait 定义与基本实现;10 customize-extractor-error 讲三种做法的横向对比 |
| 中间件 | 09 consume-body-in-extractor-or-middleware | 09 讲 `middleware::from_fn` 与 `Next` 的洋葱结构 |
| tower `Layer` / `tower-http` 现成中间件 | 13 multipart-form | 17 之前只按 B 档"挂上即可,不解释配置";17 tracing-aka-logging 才讲透 `Layer`/`Service` 抽象与 `.layer()` 由内向外的叠加顺序 |
| `thiserror` | 10 customize-extractor-error | 10 讲 `#[error]`/`transparent`/`from` 的基本用法;之后各章再用到走 C 档回引 |

写作时若发现新的同类情况,补进上表。

### 7.8 外部服务依赖

依赖外部服务的示例(postgres、redis、mongodb),统一用 Docker 一行命令启动,命令写进"项目搭建"段。diesel 示例还需要指导 `diesel_cli` 的安装和迁移执行。

### 7.9 可运行性保证

每章的交付标准:读者**从零开始**,按"项目搭建"创建项目,逐步骤敲完代码,每一步都能按文中的验证命令看到预期结果,最终得到与官方示例逻辑一致的成品。

三条自查:

1. 每个步骤结束时,项目能编译、能运行、能按文中方式验证
2. 每条 curl 命令可直接复制执行,预期输出与实际一致
3. 章末"完整代码"是各步骤增量的累加结果,与官方示例逻辑一致,读者照它对照能发现自己写错的地方

不允许出现"读者自行补全"、遗漏 `use` 语句、遗漏 `mod` 声明这类断链。

## 8. 写作流程

全书 55 章已完成,分批推进的初稿流程(先交付 00~06 章验收、再按章号逐批推进)已经走完。以下是**新写或回改**单章时仍然适用的流程。

1. 单章写作/重写步骤:读示例全部源码和资源文件 → 规划步骤划分(确保每步可运行)→ 写文章 → 验证 `cargo add` 命令和 feature flags → 逐步骤验证编译与 curl 输出 → 校对章末完整代码与官方源码的一致性。
2. 风格类调整**先改本文档,再改章节**。改完本文档后要检查已完成章节是否需要回改,不能只改规则不改正文。
3. 回改涉及多章时,连同 `docs/README.md` 总目录一起核对。
4. 本文档的表格分两类,维护方式不同,新增表格时先想清楚属于哪一类:
   - **事实性表格**——内容能从 `examples/` 机械核对出来(多文件示例、资源文件、测试模块、样板计数)。必须与示例源码保持一致,更换示例来源时逐条重核。
   - **示范性表格**——用来示范写法或说明分工(格式实例、概念归属、主题章对照)。只列代表性条目,不追求穷举,**尤其不要登记"哪些章用到了它"**——那种清单必然过期,应该改写成让人能自己判断的条件。

## 9. 待讨论 / 未决事项

- [x] ~~`testing`(21)是否需要提前?~~ **已关闭,且不是取舍问题而是依赖问题**:21 章自身的立足点是"`Router` 是一个 `Service`",这依赖第 17 章的 tower `Service`/`Layer`;它还要用 `MockConnectInfo` 和 hyper-util 客户端。**它排不到第 17 章之前**,所以"提前到前 20 章、让读者早点用 `cargo test` 替代手工 curl"在依赖上就不成立。前 20 章靠 curl 手工验证是这个排序的必然代价,示例自带的测试模块按第 7.6 节统一后置。
- [ ] "动手练习"是否需要参考答案(单独放 `docs/solutions/`)?**待 chengzhong 决定。**
- [ ] 是否需要在仓库里同步维护一份"读者视角"的可运行工程(如 `tutorial-code/<章号>-<示例名>/`,依赖用 crates.io 版本),供读者直接 clone 运行?**待 chengzhong 决定。** 注意这一项与第 7.9 节的可运行性保证直接相关——若做,它就是自查第 1、2 条的自动化载体。
