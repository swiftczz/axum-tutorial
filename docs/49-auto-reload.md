# 第 49 章 auto-reload:改代码自动重启,还不断连

## 本章你将做出什么

一个**开发时改一行代码就自动重编译重启**的服务,而且——

**重启的那几秒里,发过来的请求不会被拒绝,只是稍微等一下。**

对比一下两种体验:

```
普通的自动重启:  200 200 200 000 000 000 200 200 200
                                ↑ 重启窗口,curl 报 Connection refused

本章的做法:      200 200 200 ......等待...... 200 200 200
                                ↑ 重启窗口,请求排队,最终正常返回
```

学完本章你将掌握:

- **套接字激活(socket activation)**:一个从 1980 年代传下来的机制
- `systemfd` + `listenfd`:在开发环境用上它
- **为什么 `set_nonblocking(true)` 不加会出大问题**
- `--no-pid` 那个参数到底在关掉什么
- 同一个二进制**既能在 systemfd 下跑、也能独立跑**的写法
- 这套机制在生产环境的对应物:systemd 的 socket unit

## 前置知识

### 开发循环的成本

写 Web 服务时最高频的动作是:

```
改一行代码  →  Ctrl-C  →  cargo run  →  等编译  →  切到终端重新 curl
```

其中"Ctrl-C + `cargo run`"是纯粹的仪式,毫无价值。**自动化掉它,每天能省几百次操作。**

但这件事其实是**两个独立的问题**,很多人只解决了第一个:

| 问题 | 解法 |
|---|---|
| ① 谁检测文件变化并重启进程 | 文件监视工具(`cargo watch` / `watchexec` / `bacon`) |
| ② **重启的那几秒,进来的请求怎么办** | **套接字激活** |

**只解决 ①,你会得到一个每次重启都短暂"服务器不存在"的开发环境。** 手工 curl 时问题不大(重试一下),但如果你正在调试一个前端页面、一个 WebSocket 连接、或者一个自动化测试,**那几秒的连接拒绝会让你怀疑是自己代码写错了**。

本章两个问题都解决。

### 套接字激活是什么

正常情况下,是**应用自己**创建监听套接字:

```
应用启动  →  socket()  →  bind(3000)  →  listen()  →  accept()...
应用退出  →  套接字关闭  →  端口空出来  →  这期间连接被拒
```

套接字激活把顺序反过来:**由另一个进程(父进程)创建套接字,然后把它交给应用**。

```
systemfd/systemd  →  socket() → bind(3000) → listen()     ← 套接字在这里,一直开着
                  →  fork/exec 应用,并把这个 fd 传下去
                             应用  →  accept()...
                             应用退出
                             应用重启  →  拿到同一个 fd  →  继续 accept()
```

**关键在于:套接字从头到尾没有关闭过。**

应用挂掉的那几秒,内核仍然在**监听这个端口**,新来的连接被放进 **accept 队列(backlog)** 排着。等新进程起来调用 `accept()`,这些排队的连接就被依次处理。

**客户端的感受:请求慢了一点,但成功了。** 而不是 `Connection refused`。

这个机制的历史很长:最早是 Unix 的 `inetd`(1986),现在最常见的载体是 **systemd 的 socket unit**。Docker 和 k8s 生态里用得少(它们靠多副本 + 负载均衡解决同一问题),但在单机部署里,**systemd 的 socket activation 是实现"重启服务不断连"最省事的办法**。

### 传递协议:`LISTEN_FDS` 和 `LISTEN_PID`

父进程怎么告诉子进程"我给你留了个套接字"?靠**两个环境变量 + 文件描述符继承**:

| 环境变量 | 含义 |
|---|---|
| `LISTEN_FDS` | 传下来几个 fd。fd 编号从 **3** 开始(0/1/2 是 stdin/stdout/stderr) |
| `LISTEN_PID` | 这些 fd 是给**哪个进程**的 |

fd 本身通过 `exec` 时的继承传下来,不需要序列化任何东西——**文件描述符本来就是跨 `exec` 继承的**。

**`LISTEN_PID` 存在的理由**:环境变量会被子进程的子进程继承。如果不校验 PID,一个孙子进程可能误以为 fd 3 是给它的。所以协议规定:**只有 `LISTEN_PID` 等于自己 PID 时才认领这些 fd**。

 **而这正是本章要关掉它的原因**,下面说。

### `systemfd`:把这套机制搬到开发环境

systemd 只在 Linux 上有,而且配 unit 文件太重。`systemfd` 是一个小工具,**干的是同样的事**:

```bash
systemfd --no-pid -s http::3000 -- cargo watch -x run
```

拆开看:

- **`-s http::3000`** —— 创建一个监听 3000 端口的 TCP 套接字。`http::3000` 是 systemfd 的地址语法(`http` 表示协议提示,双冒号后是 `host:port` 里的 port,host 省略)
- **`--`** 后面是要运行的命令
- **`--no-pid`** —— **不设置 `LISTEN_PID`**

**为什么要 `--no-pid`?** 看进程链:

```
systemfd  →  cargo watch  →  cargo run  →  target/debug/auto-reload   ← 真正 accept 的是它
```

`systemfd` 只知道自己直接启动的是 `cargo watch` 的 PID。而 fd 一路继承到了最下面的二进制,**它的 PID 和 `LISTEN_PID` 对不上**,严格按协议就会拒绝认领。

`--no-pid` 让 `listenfd` 跳过 PID 校验。**代价是理论上不够严谨,但在开发环境完全没问题。**

### `listenfd`:应用侧读取

```rust
let mut listenfd = ListenFd::from_env();
match listenfd.take_tcp_listener(0).unwrap() {
    Some(listener) => { /* 用父进程给的 */ }
    None => { /* 自己 bind */ }
}
```

- **`from_env()`** 读 `LISTEN_FDS` / `LISTEN_PID`
- **`take_tcp_listener(0)`** 取第 0 个(也就是 fd 3)。返回 `Option`:没有套接字激活时是 `None`
- **`take`** 这个词表示所有权转移,**同一个 fd 只能取一次**

**那个 `None` 分支非常重要**:它让同一个二进制**在没有 systemfd 的环境下也能正常跑**。`cargo run`、`docker run`、生产部署——全都走 `None` 分支自己 bind。**不需要为开发和生产准备两份代码。**

### 为什么必须 `set_nonblocking(true)`

```rust
listener.set_nonblocking(true).unwrap();
TcpListener::from_std(listener).unwrap()
```

`listenfd` 给的是 **`std::net::TcpListener`**(标准库的,阻塞的),而 tokio 要的是 `tokio::net::TcpListener`。`from_std` 做转换,但**它有一个前提:套接字必须已经处于非阻塞模式**。

**tokio 的整个 IO 模型建立在"系统调用永远不阻塞"之上**:调用 `accept()`,要么立刻返回一个连接,要么立刻返回 `EWOULDBLOCK`,然后 tokio 把这个任务挂起、去跑别的任务,等 epoll 通知再回来。

**如果套接字是阻塞的**,`accept()` 会**卡住整个工作线程**直到有连接进来。那条线程上排队的所有任务全部停摆——这和[第 22 章](22-key-value-store.md#阻塞与-await)、[第 26 章](26-diesel-postgres.md)讲的"在异步运行时里做阻塞操作"是同一类错误,只是这次发生在最底层。

**`from_std` 不会替你检查这件事**(检查需要额外的系统调用)。步骤 5 会亲手演示后果。

## 项目搭建

```bash
cargo new auto-reload
cd auto-reload
```

```bash
cargo add axum
cargo add listenfd
cargo add tokio --features full
```

还要装两个命令行工具:

```bash
cargo install systemfd cargo-watch
```

>  **关于 `cargo-watch`**:它的作者已经宣布**停止功能更新,只做维护**,并推荐改用 [`watchexec`](https://github.com/watchexec/watchexec) 或 [`bacon`](https://github.com/Canop/bacon)。官方示例的 README 写的是 `cargo-watch`,本章沿用它以便对照,但**新项目建议用 `watchexec`**:
>
> ```bash
> cargo install watchexec-cli
> systemfd --no-pid -s http::3000 -- watchexec -r -e rs -- cargo run
> ```
>
> `bacon` 更偏向"在 TUI 里持续跑 `cargo check`/`clippy`",适合边写边看错误,但**不适合本章这种"重启服务进程"的场景**。三者的分工值得记住。

## 分步构建

### 步骤 1:普通服务,先感受一下手动重启

**要做什么**:写一个最简单的服务,不做任何自动化。

```rust
// src/main.rs
use axum::{response::Html, routing::get, Router};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    // build our application with a route
    let app = Router::new().route("/", get(handler));

    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();

    // run it
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
```

**验证**:

```bash
cargo run
curl http://127.0.0.1:3000/
```

```
<h1>Hello, World!</h1>
```

现在把 `Hello, World!` 改成 `Hello, axum!`,再 `curl` —— **还是旧的**。得先 Ctrl-C、再 `cargo run`、等编译、再 curl。

**这就是要自动化掉的循环。**

### 步骤 2:加上文件监视,解决第一个问题

**要做什么**:用 `cargo watch` 自动重编译重启。**这一步不改代码。**

```bash
cargo watch -x run
```

`-x run` 的意思是"文件变化时执行 `cargo run`"。

**验证**:

保持它开着,改一下 `handler` 的返回内容并保存。终端里会看到重新编译,然后服务重启。

```bash
curl http://127.0.0.1:3000/
```

**新内容出来了,你什么都没做。**

**但现在暴露出第二个问题。** 开一个终端跑这个循环:

```bash
while true; do
  curl -s -o /dev/null -w "%{http_code} " http://127.0.0.1:3000/
  sleep 0.2
done
```

然后触发一次重启(改一下代码保存,或者 `touch src/main.rs`)。

**你会看到一串 `000`:**

```
200 200 200 200 000 000 000 000 000 000 000 200 200 200
                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                重启窗口,curl 连不上
```

`000` 是 curl 在连接失败时给的状态码。真实的错误是:

```
curl: (7) Failed to connect to 127.0.0.1 port 3000: Connection refused
```

**旧进程退出时把监听套接字关了,新进程还没起来 bind,这几秒钟端口上什么都没有。**

编译越慢,这个窗口越长。一个中等规模的项目,增量编译几秒到几十秒都很正常。

**先 Ctrl-C 停掉 `cargo watch`,下一步解决它。**

### 步骤 3:接上 listenfd

**要做什么**:让应用**优先使用父进程传下来的套接字**,拿不到才自己 bind。

```rust
use listenfd::ListenFd;
```

```rust
#[tokio::main]
async fn main() {
    // build our application with a route
    let app = Router::new().route("/", get(handler));

    let mut listenfd = ListenFd::from_env();
    let listener = match listenfd.take_tcp_listener(0).unwrap() {
        // if we are given a tcp listener on listen fd 0, we use that one
        Some(listener) => {
            listener.set_nonblocking(true).unwrap();
            TcpListener::from_std(listener).unwrap()
        }
        // otherwise fall back to local listening
        None => TcpListener::bind("127.0.0.1:3000").await.unwrap(),
    };

    // run it
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

**整个改动只有 `listener` 的获取方式,`app` 和 `serve` 一行没动。**

**`take_tcp_listener(0).unwrap()` 里的 `unwrap` 处理的是"环境变量格式不对"这类错误**,不是"没有套接字"——后者返回的是 `Ok(None)`,走 `None` 分支。这两层的区分容易看混。

**`None` 分支保证了向后兼容**:没有 systemfd 时行为和步骤 1 完全一样。

**验证(先确认没坏)**:

```bash
cargo run
curl http://127.0.0.1:3000/
```

正常返回。**说明 `None` 分支工作正常。**

### 步骤 4:套上 systemfd,解决第二个问题

**要做什么**:让 `systemfd` 来持有套接字。**这一步不改代码,只改启动命令。**

```bash
systemfd --no-pid -s http::3000 -- cargo watch -x run
```

启动后你会看到 systemfd 打印一行类似:

```
~> socket http://127.0.0.1:3000/ -> fd #3
```

**注意应用自己打印的 `listening on ...`** —— 它现在报告的是从 fd 拿到的那个地址,而不是自己 bind 的。

**验证**:

再跑一次那个循环:

```bash
while true; do
  curl -s -o /dev/null -w "%{http_code} " http://127.0.0.1:3000/
  sleep 0.2
done
```

改一下 `handler` 的内容并保存,触发重启。

**这次的输出:**

```
200 200 200 200 ...(停顿几秒,没有输出)... 200 200 200
```

**一个 `000` 都没有。**

停顿期间那几个 curl 并没有失败,它们**连接成功了,但一直在等响应** —— 连接被内核放进 accept 队列,等新进程起来才被处理。所以 `curl` 阻塞在那里,直到拿到 200。

**改动的内容也立刻生效了**,你会看到停顿之后的响应已经是新内容。

**这就是本章的目标达成了。** 再对比一下两次的区别:

| | 步骤 2(只有 cargo watch) | 步骤 4(+ systemfd) |
|---|---|---|
| 谁持有套接字 | 应用进程 | **systemfd 进程** |
| 重启时套接字 | **关闭** | **保持打开** |
| 客户端体验 | `Connection refused` | **请求变慢,但成功** |
| 应用代码改动 | 无 | 十行 |

**注意 systemfd 只需要在最外层启动一次。** 它不参与重启循环,`cargo watch` 反复杀掉和拉起的是它的孙子进程。

### 步骤 5:故意改坏,理解 `set_nonblocking`

**要做什么**:把 `set_nonblocking(true)` 那行注释掉,看看会怎样。

```rust
        Some(listener) => {
            // listener.set_nonblocking(true).unwrap();
            TcpListener::from_std(listener).unwrap()
        }
```

```bash
systemfd --no-pid -s http::3000 -- cargo watch -x run
```

**现象**:程序**能编译、能启动、能打印 `listening on ...`,`from_std` 也没报错。**

然后 curl:

```bash
curl http://127.0.0.1:3000/
```

第一个请求**可能是正常的**。但接着做这个测试:

```bash
for i in $(seq 1 20); do curl -s -o /dev/null -w "%{http_code} " http://127.0.0.1:3000/ & done; wait
```

或者干脆:

```bash
curl http://127.0.0.1:3000/ &   # 第一个
sleep 1
curl http://127.0.0.1:3000/     # 第二个
```

**你会看到请求变得断断续续、时快时慢,或者干脆挂住。**

**原因**:阻塞模式的套接字上,`accept()` **会卡住整个 tokio 工作线程**,直到有新连接进来。这条线程上排队的所有任务(包括正在处理响应的那些)全部停摆。

多线程运行时有多条工作线程,所以症状**不是彻底死机,而是随机的卡顿** —— 这恰恰是最难查的一类问题。如果换成单线程运行时(`#[tokio::main(flavor = "current_thread")]`),整个服务会**彻底冻住**。

**这类 bug 的可怕之处在于:**

- **编译通过,没有 warning**
- **`from_std` 不报错** —— 检查套接字模式需要额外的系统调用,tokio 选择把责任交给调用方
- **启动日志一切正常**
- 症状是"偶尔有点卡",很容易被归因到别的地方

**tokio 的文档明确要求调用方保证非阻塞模式**,这是一个典型的"契约在文档里、不在类型里"的 API。碰到 `from_std`、`from_raw_fd` 这类"接管外部资源"的函数,**一定要读文档确认前置条件**。

**把那行加回来。**

## 完整代码

`src/main.rs`:

```rust
use axum::{response::Html, routing::get, Router};
use listenfd::ListenFd;
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    // build our application with a route
    let app = Router::new().route("/", get(handler));

    let mut listenfd = ListenFd::from_env();
    let listener = match listenfd.take_tcp_listener(0).unwrap() {
        // if we are given a tcp listener on listen fd 0, we use that one
        Some(listener) => {
            listener.set_nonblocking(true).unwrap();
            TcpListener::from_std(listener).unwrap()
        }
        // otherwise fall back to local listening
        None => TcpListener::bind("127.0.0.1:3000").await.unwrap(),
    };

    // run it
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
```

## 代价与边界

**这只是"重启得快一点",不是真正的热重载。** 进程还是被杀掉重建的,所以:

- **内存里的状态全丢** —— [第 22 章](22-key-value-store.md)那个 `Arc<RwLock<HashMap>>` 里的数据、任何缓存、任何计数器,每次改代码都清零
- **数据库连接池重建** —— [第 24 章](24-sqlx-postgres.md)的 `PgPool` 要重新建立连接,慢的话开发体验反而变差
- **WebSocket 和 SSE 连接全断** —— [第 34 章](34-websockets.md)那种长连接的调试会很痛苦,浏览器需要重连逻辑
- **正在处理的请求会被打断** —— 除非配上[第 45 章](45-graceful-shutdown.md)的优雅停机(见下)

Rust 目前没有成熟的、生产可用的代码热替换方案。**"重启快"就是当前的最优解。**

**编译时间才是真正的瓶颈,这个方案没解决它。** 套接字激活只是让重启窗口不再"拒绝连接",窗口本身还是那么长。想缩短它:

- **换链接器**:`mold`(Linux)或 `lld`,链接时间常常能砍掉一半以上,是性价比最高的一招
- **`[profile.dev] debug = "line-tables-only"`** —— 少生成调试信息,链接更快
- **拆 workspace** —— 让改动只触发一个小 crate 的重编译
- **`cargo check` 而不是 `cargo run`** —— 只想看错误时用它快得多(这正是 `bacon` 的定位)

**backlog 有上限。** 内核的 accept 队列不是无限的(常见默认值 128 到 4096)。如果重启耗时很长、请求速率很高,**队列满了之后新连接照样被拒**。开发环境不会碰到,但这说明套接字激活**不能替代真正的高可用方案**。

**可能陷入无限重编译循环。** 如果构建过程会写文件到被监视的目录里,保存 → 编译 → 写文件 → 触发监视 → 再编译……`cargo watch` 默认忽略 `target/` 和 `.gitignore` 里的路径,但**生成代码的项目(build script 写文件、`sqlx prepare`、protobuf 生成)容易踩到**。用 `-i`/`--ignore` 排除掉。

**没有优雅停机。** `cargo watch` 默认给旧进程发 SIGTERM,**这意味着[第 45 章](45-graceful-shutdown.md)的优雅停机在这里能直接派上用场**:加上 `with_graceful_shutdown` 之后,重启时在途请求会被正常处理完,而不是被切断。**两章配合起来,开发环境的重启就几乎完全无感了。** 本章示例没做。

**用 `println!` 而不是 `tracing`。** 这个示例连 `tracing` 依赖都没加。[第 17 章](17-tracing-aka-logging.md)讲过为什么结构化日志更好。

**端口在两处出现。** 代码里写死 `127.0.0.1:3000`,systemfd 命令里写 `http::3000`。**两处不一致的话,没有 systemfd 时听 3000、有 systemfd 时听 systemfd 指定的端口**,而代码里那个数字变成了误导。生产代码应该从环境变量读端口。

**systemfd 只适合本机开发。** 容器环境里没有意义(容器重启本来就是整个重建),k8s 里靠多副本 + 滚动更新解决同一问题。

**生产环境的对应物是 systemd 的 socket unit。** 同样的机制,配置在 systemd 里:

```ini
# myapp.socket
[Socket]
ListenStream=3000

# myapp.service
[Service]
ExecStart=/usr/local/bin/myapp
```

`systemctl restart myapp.service` 时套接字由 systemd 持有,**连接不会被拒**。这是单机部署里实现"重启不断连"最省事的办法,而且**你的应用代码不用改**——`listenfd` 那段在两种环境下都工作。

**模板和静态资源的热更新是另一回事。** [第 05 章](05-templates-minijinja.md)的 minijinja 是**运行时**加载模板的,改模板不需要重编译(配上文件监视甚至不用重启);[第 04 章](04-templates.md)的 askama 把模板编译进二进制,改模板必须重编译。**选型时这一点值得考虑进去。** 前端资源的热更新则完全属于前端工具链(Vite 之类)的范畴。

## 动手练习

**1. 配上优雅停机,让重启完全无感。**

提示:把[第 45 章](45-graceful-shutdown.md)的 `shutdown_signal()` 抄过来,`axum::serve(...).with_graceful_shutdown(shutdown_signal())`。加一条 `/slow` 路由(睡 3 秒)。验证:请求 `/slow` 的同时触发重启,**改造前请求被切断(`Empty reply`),改造后请求正常返回 200**。**这个组合是本教程里两章配合最直接的一次。**

**2. 把端口做成可配置的。**

提示:`std::env::var("PORT").unwrap_or_else(|_| "3000".into())`,消除"代码里 3000、命令里 3000"的重复。**顺便想一想:有 systemfd 时这个环境变量其实是无效的,该不该在日志里提示一下?**

**3. 换成 `watchexec`,并排除掉不该触发重建的文件。**

提示:

```bash
systemfd --no-pid -s http::3000 -- watchexec -r -e rs -- cargo run
```

`-e rs` 限定只监视 `.rs` 文件,`-r` 表示重启而不是并行启动。**对比一下 `cargo watch` 和 `watchexec` 的启动速度和输出**。

**4. 亲眼确认套接字确实是同一个。**

提示:在 handler 里返回 `std::process::id()`(进程 ID)。触发一次重启,你会看到 PID 变了,**但连接从来没有被拒绝过**。再用 `lsof -i :3000`(macOS/Linux)看看是哪个进程持有着监听套接字 —— **你会看到 systemfd 和应用同时持有**。

**5.(进阶)写一个 systemd socket unit,在 Linux 上验证生产版本。**

提示:按「代价与边界」里的模板写 `myapp.socket` 和 `myapp.service`,`systemctl enable --now myapp.socket`。用一个循环 curl,期间 `systemctl restart myapp.service`,确认没有连接被拒。**做完你就理解了为什么这段 `listenfd` 代码在生产环境也值得保留。**

## 小结与下一章预告

自动重载是**两个独立问题**:谁检测变化并重启(文件监视工具),以及**重启窗口里的请求怎么办**(套接字激活)。只解决前者,你会得到一个每次重启都短暂"服务器不存在"的开发环境——步骤 2 那一串 `000` 就是它。

**套接字激活的核心是:套接字由父进程持有,应用重启期间从不关闭。** 内核继续监听,新连接进 accept 队列排着,等新进程起来处理。客户端只是等一下,不会 `Connection refused`。这个机制从 1986 年的 `inetd` 传到今天的 systemd,协议就是两个环境变量 `LISTEN_FDS` / `LISTEN_PID` 加上文件描述符继承。

`--no-pid` 关掉的是 PID 校验,因为进程链是 `systemfd → cargo watch → cargo → 二进制`,最下面那个的 PID 和 `LISTEN_PID` 对不上。

应用侧的代码只有十行,而且**那个 `None` 分支很关键**:拿不到父进程的套接字就自己 bind,所以**同一个二进制在 `cargo run`、Docker、systemd 下都能跑**,不需要两份代码。

步骤 5 的故意破坏值得记住:**去掉 `set_nonblocking(true)` 之后,程序照样编译、照样启动、`from_std` 也不报错**,但阻塞的 `accept()` 会卡住整个 tokio 工作线程,表现为随机卡顿。**契约写在文档里而不是类型里**——遇到 `from_std`、`from_raw_fd` 这类接管外部资源的函数,务必读文档确认前置条件。

要认清它的边界:**这不是热重载**,进程仍然是重建的,内存状态、连接池、WebSocket 全部丢失;**编译时间才是真瓶颈**,套接字激活没解决它(换 `mold` 链接器往往比什么都管用)。配上[第 45 章](45-graceful-shutdown.md)的优雅停机,重启才算真正无感——练习 1 会让你做出这个组合。

---

**「生产部署」阶段(45~49)到此结束。** 服务现在能优雅地起、优雅地停、被监控看见、给 API 划分版本、并且在本地开发时改一行就自动重启还不断连。

接下来是最后的「专题」阶段(50~54),内容偏底层和特殊场景。[第 50 章 low-level-rustls](50-low-level-rustls.md) 回到 TLS,但这次不用 `axum-server` 了 —— 你会**自己完成 TLS 握手**,把 `rustls` 的 `TlsAcceptor` 接进[第 44 章](44-serve-with-hyper.md)那个手写的 accept 循环里。这么做的回报是:能拿到客户端证书、能按 SNI 选证书、能在握手层面做定制——都是 `axum-server` 那层封装拿不到的东西。
