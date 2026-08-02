# 第 53 章 unix-domain-socket:不走网络的 HTTP

## 本章你将做出什么

一个**没有端口**的 HTTP 服务:

```
/tmp/axum/helloworld    ← 服务监听在这个文件上
```

```bash
curl --unix-socket /tmp/axum/helloworld http://localhost/
```

```
Hello, World!
```

而且 handler 能拿到一个 TCP 永远给不了的东西——**内核直接告诉你对端进程的用户 ID、组 ID 和进程 ID**:

```
new connection from `UdsConnectInfo { peer_addr: ..., peer_cred: UCred { uid: 501, gid: 20, pid: Some(48213) } }`
```

**这是连接层面的身份认证,业务代码一行不用写,而且伪造不了。**

学完本章你将掌握:

- Unix 域套接字是什么,和 TCP 回环比有什么不同
- `axum::serve` 为什么能直接吃 `UnixListener`
- **`Connected` trait:自定义 `ConnectInfo` 的内容**——[第 44 章](44-serve-with-hyper.md)的续集
- **`SO_PEERCRED`:内核背书的对端身份**,以及它能替代什么
- `hyper::client::conn::http1::handshake`:最底层的 HTTP 客户端
- 一个会让人一头雾水的报错:**"地址已被占用",但没有任何进程在监听**
- 套接字文件的权限问题,以及为什么不该放在 `/tmp`

## 前置知识

### Unix 域套接字是什么

它和 TCP 套接字用同一套 API(`socket`/`bind`/`listen`/`accept`/`read`/`write`),**但不经过网络协议栈**:

| | TCP 回环(`127.0.0.1`) | **Unix 域套接字** |
|---|---|---|
| 地址形式 | IP + 端口 | **文件系统路径** |
| 经过协议栈 | 是(IP、TCP 分段、校验和、拥塞控制) | **否**,内核直接搬数据 |
| 性能 | 快 | **更快**(少一层封装,吞吐通常高 20%~50%) |
| 能否跨机器 | 能(改个 IP 就行) | **不能**,只能本机 |
| 访问控制 | 靠防火墙 / 绑定地址 | **文件权限** |
| 端口冲突 | 会 | 不会(但有文件冲突) |
| 对端身份 | 只知道 IP:端口 | **内核提供 uid / gid / pid** |

**最后两行才是真正的卖点。** 性能差别在多数应用里不是瓶颈,但"用文件权限做访问控制"和"内核背书的对端身份"是 TCP 给不了的。

**真实世界里到处都是它:**

- **Nginx → 应用**:`proxy_pass http://unix:/run/myapp.sock;` 是标准部署方式
- **Docker**:`/var/run/docker.sock`,`docker` 命令就是通过它跟守护进程讲 HTTP
- **PostgreSQL / MySQL / Redis**:本机连接默认走 UDS,比 TCP 快
- **systemd**:`journald`、`dbus` 全靠它

**[第 43 章](43-reverse-proxy.md)那个反向代理,在生产里连上游时用的往往就是 UDS 而不是 TCP。**

### `SO_PEERCRED`:伪造不了的身份

这是本章最值得学的一点。

TCP 连接上,服务端只知道对端的 IP 和端口。**"你是谁"必须靠应用层自己解决**——[第 30 章](30-jwt.md)的 JWT、[第 31 章](31-oauth.md)的 OAuth、密码、API key,都是在补这一课。

**Unix 域套接字上不需要。** 内核知道连接是哪个进程发起的,直接告诉你:

```rust
let cred: UCred = stream.peer_cred()?;
cred.uid();    // 对端进程的用户 ID
cred.gid();    // 组 ID
cred.pid();    // 进程 ID(Option)
```

**这些值是内核填的,进程无法伪造。** 一个以 `www-data` 身份运行的进程,`uid` 就一定是 `www-data` 的。

**这带来一种非常干净的授权模型:**

```rust
if cred.uid() != EXPECTED_UID {
    return StatusCode::FORBIDDEN;
}
```

不需要密钥、不需要令牌、不会过期、不会泄露。**同机的服务间调用用这个,比任何 token 方案都简单可靠。**

**两个注意点:**

1. **`pid` 不可用于安全判断。** 进程可以退出,PID 会被复用。做审计日志可以,做授权不行。**`uid` / `gid` 才是可靠的。**
2. **这是"连接建立时"的凭据。** 长连接期间对端如果换了身份(极少见),你拿到的还是建连时的值。

### `axum::serve` 不只吃 `TcpListener`

前 52 章一直是:

```rust
axum::serve(tcp_listener, app)
```

axum 0.8 把 `serve` 做成了**对监听器泛型**的:任何实现了 `axum::serve::Listener` 的类型都能传进去,`TcpListener` 和 **`UnixListener`** 都实现了它。

```rust
axum::serve(unix_listener, app)     // ← 一样能用
```

**所以本章不需要像[第 50 章](50-low-level-rustls.md)那样手写 accept 循环。** 换个监听器就完事了。

### `Connected`:自定义 `ConnectInfo` 装什么

[第 44 章](44-serve-with-hyper.md#service-与-makeservice多出来的那一层)讲过 `ConnectInfo<SocketAddr>` 的完整链路:

```
accept 拿到连接  →  MakeService 把连接信息塞进请求扩展  →  提取器读出来
```

当时用的是现成的 `SocketAddr`。**这一章要自己定义"连接信息"里装什么。**

关键是 `Connected` trait:

```rust
impl connect_info::Connected<IncomingStream<'_, UnixListener>> for UdsConnectInfo {
    fn connect_info(stream: IncomingStream<'_, UnixListener>) -> Self {
        // 从刚 accept 到的连接里提取你想要的东西
    }
}
```

**它回答的问题是:"给你一条刚建立的连接,你要从里面记下什么?"**

- `SocketAddr` 的实现记的是对端地址
- 这个 `UdsConnectInfo` 记的是**对端地址 + 对端凭据**

**`IncomingStream<'_, L>` 的那个 `L` 就是监听器类型。** 因为不同监听器给出的连接类型不同(`TcpStream` vs `UnixStream`),能提取的信息也不同——`peer_cred()` 只有 `UnixStream` 才有。

实现了这个 trait 之后:

```rust
app.into_make_service_with_connect_info::<UdsConnectInfo>()
```

handler 里就能 `ConnectInfo(info): ConnectInfo<UdsConnectInfo>`。**整条链路和[第 44 章](44-serve-with-hyper.md)一模一样,只是换了个装载的类型。**

### 套接字文件的两个陷阱

**① 绑定前必须删掉旧文件。**

`bind` 一个已存在的路径会失败,报 **`Address already in use`** —— 即使**没有任何进程在监听**。因为套接字文件是**普通的文件系统条目**,进程退出时不会自动删除。

所以标准写法是:

```rust
let _ = tokio::fs::remove_file(&path).await;   // 忽略"文件不存在"的错误
let uds = UnixListener::bind(path)?;
```

步骤 4 会演示忘了这一步的后果。

**这个 `remove_file` 本身也有风险**:如果真的有另一个进程正在用这个套接字,你会**把它的套接字文件删掉**(它还在跑,但没人能连上它了)。生产环境应该用文件锁或 systemd 的套接字激活([第 49 章](49-auto-reload.md#套接字激活是什么))来避免。

**② 路径长度有硬限制。**

`sockaddr_un.sun_path` 在 Linux 上是 **108 字节**,macOS/BSD 上是 **104 字节**。超了就 bind 失败。

**这个限制比想象中容易撞到**:`/home/someuser/projects/my-long-project-name/target/debug/run/app.sock` 就已经不短了。**用 `/run/` 或 `/tmp/` 这类短路径,别用项目目录。**

### `#[cfg(unix)]`:整个模块的条件编译

```rust
#[cfg(unix)]
#[tokio::main]
async fn main() { unix::server().await; }

#[cfg(not(unix))]
fn main() { println!("This example requires unix") }

#[cfg(unix)]
mod unix { /* 全部实现 */ }
```

[第 45 章](45-graceful-shutdown.md#tokiosignal)用 `#[cfg(unix)]` 处理过单个语句,这里是**整个模块**。

**为什么要包成模块?** 因为 `use tokio::net::UnixListener;` 这样的导入在 Windows 上根本不存在,**放在文件顶层会直接编译失败**。包进 `#[cfg(unix)] mod unix` 里,非 Unix 平台上整块代码连解析都不会做。

**注意两个 `main` 只有一个会被编译**,而且非 Unix 那个连 `#[tokio::main]` 都不需要——用不上异步运行时。

## 项目搭建

```bash
cargo new unix-domain-socket
cd unix-domain-socket
```

```bash
cargo add axum
cargo add http-body-util
cargo add hyper --features full
cargo add hyper-util --features tokio,server-auto,http1
cargo add tokio --features full
cargo add tracing-subscriber --features env-filter
```

**本章需要 Linux 或 macOS。** Windows 虽然从 Windows 10 起支持 AF_UNIX,但 `tokio::net::UnixListener` 目前只在 Unix 上提供。

## 分步构建

### 步骤 1:监听一个文件

**要做什么**:先把服务跑起来,用 `curl --unix-socket` 验证。

```rust
// src/main.rs
#[cfg(unix)]
#[tokio::main]
async fn main() {
    unix::server().await;
}

#[cfg(not(unix))]
fn main() {
    println!("This example requires unix")
}

#[cfg(unix)]
mod unix {
    use axum::{routing::get, Router};
    use std::path::PathBuf;
    use tokio::net::UnixListener;
    use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

    pub async fn server() {
        tracing_subscriber::registry()
            .with(
                tracing_subscriber::EnvFilter::try_from_default_env()
                    .unwrap_or_else(|_| "debug".into()),
            )
            .with(tracing_subscriber::fmt::layer())
            .init();

        let path = PathBuf::from("/tmp/axum/helloworld");

        let _ = tokio::fs::remove_file(&path).await;
        tokio::fs::create_dir_all(path.parent().unwrap())
            .await
            .unwrap();

        let uds = UnixListener::bind(path.clone()).unwrap();

        let app = Router::new().route("/", get(handler));

        axum::serve(uds, app).await.unwrap();
    }

    async fn handler() -> &'static str {
        "Hello, World!"
    }
}
```

**四行准备工作:**

```rust
let path = PathBuf::from("/tmp/axum/helloworld");
let _ = tokio::fs::remove_file(&path).await;              // ① 删掉可能残留的旧套接字
tokio::fs::create_dir_all(path.parent().unwrap()).await   // ② 确保目录存在
    .unwrap();
let uds = UnixListener::bind(path.clone()).unwrap();      // ③ 绑定
```

**`let _ =` 是刻意忽略错误** —— 文件不存在时 `remove_file` 会返回 `NotFound`,那是正常情况。前置知识说过这一步的必要性和风险。

**`create_dir_all` 是幂等的**(目录已存在也返回 `Ok`),这一点比[第 40 章](40-stream-to-file.md#代价与边界)那个 `create_dir` 强——那一章批评过它第二次启动就 panic。

**`axum::serve(uds, app)` —— 和 TCP 版本一字之差。** 前置知识说过,`serve` 对监听器是泛型的。

**注意日志过滤器是 `"debug"`,一个全局级别**,不是[第 03 章](03-form.md)以来那种 `{crate}=debug`。这意味着**所有 crate 的 debug 日志都会打出来**,包括 hyper 内部的。信息量很大,调试时有用,生产环境会淹没你。

**跑起来看**:

```bash
cargo run
```

另开一个终端:

```bash
curl --unix-socket /tmp/axum/helloworld http://localhost/
```

```
Hello, World!
```

**`--unix-socket` 后面是套接字路径,URL 里的主机名完全没用** —— 写 `http://localhost/` 也好,`http://whatever/` 也好,curl 只用它来取路径和构造 `Host` 头。**UDS 上没有"主机"这个概念。**

看看套接字文件长什么样:

```bash
ls -l /tmp/axum/
```

```
srwxr-xr-x  1 you  staff  0 Jul 30 10:00 helloworld
```

**开头的 `s` 表示这是一个套接字文件**(普通文件是 `-`,目录是 `d`)。大小是 0——它不存数据,只是一个内核对象的入口。

再确认一下端口:

```bash
lsof -U -a -p $(pgrep unix-domain-socket) 2>/dev/null | head
```

**没有任何 TCP 端口被占用。** 这个服务在 `netstat -an | grep 3000` 里是查不到的。

### 步骤 2:拿到对端身份

**要做什么**:定义 `UdsConnectInfo`,实现 `Connected`,让 handler 能看到对端的 uid/gid/pid。

```rust
    use axum::{
        extract::connect_info::{self, ConnectInfo},
        routing::get,
        serve::IncomingStream,
        Router,
    };
    use std::sync::Arc;
    use tokio::net::{unix::UCred, UnixListener};
```

```rust
    #[derive(Clone, Debug)]
    #[allow(dead_code)]
    struct UdsConnectInfo {
        peer_addr: Arc<tokio::net::unix::SocketAddr>,
        peer_cred: UCred,
    }

    impl connect_info::Connected<IncomingStream<'_, UnixListener>> for UdsConnectInfo {
        fn connect_info(stream: IncomingStream<'_, UnixListener>) -> Self {
            let peer_addr = stream.io().peer_addr().unwrap();
            let peer_cred = stream.io().peer_cred().unwrap();
            Self {
                peer_addr: Arc::new(peer_addr),
                peer_cred,
            }
        }
    }
```

```rust
    async fn handler(ConnectInfo(info): ConnectInfo<UdsConnectInfo>) -> &'static str {
        println!("new connection from `{info:?}`");

        "Hello, World!"
    }
```

```rust
        let app = Router::new()
            .route("/", get(handler))
            .into_make_service_with_connect_info::<UdsConnectInfo>();

        axum::serve(uds, app).await.unwrap();
```

**逐块看:**

**① `#[derive(Clone)]` 是必需的。** [第 44 章](44-serve-with-hyper.md#service-与-makeservice多出来的那一层)讲过,MakeService 会把这个值塞进**每个请求**的扩展里,所以要能克隆。

**② `Arc<tokio::net::unix::SocketAddr>` 为什么要包 `Arc`?**

因为 **`tokio::net::unix::SocketAddr` 没有实现 `Clone`**。用 `Arc` 包一层就有了(克隆 `Arc` 只是引用计数 +1)。

**这是一个很实用的小技巧**:遇到"我需要 `Clone` 但这个类型没有"时,`Arc` 往往是最省事的答案——前提是你只需要读。

**`UCred` 本身是 `Copy` 的**(就是几个整数),所以不用包。

**③ `stream.io()`** 拿到底层的 `&UnixStream`,然后调它的两个方法:

- `peer_addr()` —— 对端的套接字地址。**对 UDS 来说通常是"未命名"的**(客户端一般不 bind 自己的路径),所以打印出来往往是空的
- `peer_cred()` —— **本章的主角**,返回 `UCred`

**④ `#[allow(dead_code)]`** 是因为这两个字段只在 `Debug` 输出里用到,没有代码直接读它们。示例加这个注解是为了消除警告。

> **更好的写法**:字段永远只被 `Debug` 打印,说明**这个能力拿到了但没用起来**。真实项目里应该用它做点事:
>
> ```rust
> async fn handler(ConnectInfo(info): ConnectInfo<UdsConnectInfo>) -> Result<&'static str, StatusCode> {
>     const ALLOWED_UID: u32 = 33;   // 比如 www-data
>     if info.peer_cred.uid() != ALLOWED_UID {
>         return Err(StatusCode::FORBIDDEN);
>     }
>     Ok("Hello, World!")
> }
> ```
>
> **这就是前置知识说的那种"不需要令牌的授权"。** 更好的做法是把它做成一个中间件([第 09 章](09-consume-body-in-extractor-or-middleware.md))或提取器([第 48 章](48-versioning.md)),而不是散在每个 handler 里。

**跑起来看**:

```bash
cargo run
```

```bash
curl --unix-socket /tmp/axum/helloworld http://localhost/
```

服务端打印:

```
new connection from `UdsConnectInfo { peer_addr: (unnamed), peer_cred: UCred { uid: 501, gid: 20, pid: Some(48213) } }`
```

**对一下你自己的身份:**

```bash
id -u    # 应该等于日志里的 uid
id -g    # 应该等于 gid
```

**内核直接告诉了服务端"发起这个请求的是谁"** —— 没有登录、没有令牌、没有任何应用层协议参与。

**再验证一下这个身份确实来自内核而非客户端自称**:用 `sudo` 发一次请求:

```bash
sudo curl --unix-socket /tmp/axum/helloworld http://localhost/
```

```
new connection from `UdsConnectInfo { ..., peer_cred: UCred { uid: 0, gid: 0, pid: Some(48277) } }`
```

**`uid: 0`** —— curl 什么都没多发,内核如实报告了它是 root 跑的。**这个值伪造不了。**

### 步骤 3:内置一个客户端

**要做什么**:官方示例不用 curl,它自己**既是服务端又是客户端**。这一步把客户端部分补上。

```rust
    use axum::{body::Body, http::{Request, StatusCode}};
    use http_body_util::BodyExt;
    use hyper_util::rt::TokioIo;
    use tokio::net::UnixStream;
```

```rust
        let uds = UnixListener::bind(path.clone()).unwrap();
        tokio::spawn(async move {
            let app = Router::new()
                .route("/", get(handler))
                .into_make_service_with_connect_info::<UdsConnectInfo>();

            axum::serve(uds, app).await.unwrap();
        });

        let stream = TokioIo::new(UnixStream::connect(path).await.unwrap());
        let (mut sender, conn) = hyper::client::conn::http1::handshake(stream).await.unwrap();
        tokio::task::spawn(async move {
            if let Err(err) = conn.await {
                println!("Connection failed: {err:?}");
            }
        });

        let request = Request::get("http://uri-doesnt-matter.com")
            .body(Body::empty())
            .unwrap();

        let response = sender.send_request(request).await.unwrap();

        assert_eq!(response.status(), StatusCode::OK);

        let body = response.collect().await.unwrap().to_bytes();
        let body = String::from_utf8(body.to_vec()).unwrap();
        assert_eq!(body, "Hello, World!");
```

**服务端被 `tokio::spawn` 到后台**,主流程继续当客户端。[第 43 章](43-reverse-proxy.md#tokiospawn一个进程跑两个服务)用过同样的手法。

**注意程序会在断言跑完后退出** —— 主流程结束,spawn 出去的服务端任务被丢弃。**所以这一步之后,curl 验证要赶在退出前,或者干脆注释掉客户端部分。**

**`hyper::client::conn::http1::handshake` 是第三种 HTTP 客户端了**,和前面两种对比一下:

| | 出现在 | 层次 | 你要管什么 |
|---|---|---|---|
| `reqwest` | [第 41 章](41-reqwest-response.md) | 最高 | 什么都不用管 |
| `hyper-util` 的 `legacy::Client` | [第 43 章](43-reverse-proxy.md) | 中 | 连接池它管,连接器你选 |
| **`hyper::client::conn`** | **本章** | **最低** | **连接是你自己建的,连接任务也要你自己 spawn** |

**为什么这里要用最低层的?** 因为前两种都假设"给我一个 URL,我去建连接"。**而你已经有一条现成的 `UnixStream` 了**,需要的是"在这条已有的流上讲 HTTP"。这正是 `conn` 模块的定位。

**`handshake` 返回一对东西:**

```rust
let (mut sender, conn) = hyper::client::conn::http1::handshake(stream).await?;
```

- **`sender`** —— 用来发请求(`send_request`)
- **`conn`** —— **连接的驱动任务**,必须被 spawn 或 await,否则**什么都不会发生**

**第二点是这个 API 最容易踩的坑。** 如果你忘了 `tokio::spawn(conn)`,`sender.send_request(...)` 会永远挂着——**因为没有人在真正读写那条流**。这和[第 46 章](46-tls-graceful-shutdown.md#步骤-5故意改坏暴露那个optional的隐患)那个"future 创建了但没人轮询"是同一类问题。

**`Request::get("http://uri-doesnt-matter.com")`** —— 这个 URL 是示例作者的幽默,但确实说明了问题:**UDS 上没有主机的概念**,这个字符串只用来生成请求行和 `Host` 头,连到哪由 `UnixStream::connect` 决定。

**`response.collect().await.to_bytes()`** —— `http_body_util::BodyExt` 提供,把流式 body 收集成 `Bytes`。[第 21 章](21-testing.md)、[第 48 章](48-versioning.md)测试里用的是同一个方法。

**跑起来看**:

```bash
cargo run
```

程序跑完直接退出,**没有 panic 就说明两个 `assert_eq!` 都过了**。日志里能看到:

```
new connection from `UdsConnectInfo { ... }`
```

**这个程序本质上是一个自测**:自己起服务、自己发请求、自己断言。

### 步骤 4:故意改坏,理解那个莫名其妙的报错

**要做什么**:把 `remove_file` 那行注释掉,连续运行两次。

```rust
        // let _ = tokio::fs::remove_file(&path).await;
```

**第一次运行**:

```bash
cargo run
```

一切正常。

**第二次运行**:

```bash
cargo run
```

```
thread 'main' panicked at src/main.rs:...:
called `Result::unwrap()` on an `Err` value: Os { code: 48, kind: AddrInUse, message: "Address already in use" }
```

**"地址已被占用"。**

**但是——**

```bash
lsof /tmp/axum/helloworld
```

**没有任何进程在用它。** 第一次运行的程序早就退出了。

```bash
ps aux | grep unix-domain-socket
```

**什么都没有。**

**这就是本章最容易让人一头雾水的地方:**

- 报错说"地址被占用",但**没有任何进程占用**
- 用 TCP 的经验会让你去找"是不是有个僵尸进程",而**这条路完全是死胡同**
- 换个端口能解决 TCP 的问题,但这里没有端口可换

**真相是:套接字文件是普通的文件系统条目,进程退出时不会自动删除。** `bind` 拒绝绑定一个已存在的路径,不管那个路径背后还有没有活着的进程。

**解决方法就是 `rm` 掉:**

```bash
rm /tmp/axum/helloworld
cargo run     # 又能跑了
```

**这也是为什么标准写法里一定有那一行 `remove_file`。**

>  **但这个"标准写法"本身是有代价的**:如果**真的**有另一个实例在跑,你会把它的套接字文件删掉。那个实例还活着、还在 accept,但**再也没有客户端能连上它了**——因为路径已经指向你新建的套接字。
>
> **两个实例都以为自己在服务,实际上只有一个收得到请求。** 这在滚动重启时会造成难以察觉的流量丢失。
>
> 生产环境的正确做法是:
> - **用 systemd 的套接字激活**([第 49 章](49-auto-reload.md#套接字激活是什么))—— 套接字由 systemd 持有,应用根本不 bind
> - 或者**用文件锁(`flock`)先抢占**,抢不到就说明有实例在跑,直接退出

**把那行加回来。**

### 步骤 5:看看谁能连上来

**要做什么**:检查套接字文件的权限,理解 UDS 的访问控制模型。

```bash
cargo run &      # 后台跑
sleep 1
ls -l /tmp/axum/helloworld
```

```
srwxr-xr-x  1 you  staff  0 Jul 30 10:00 /tmp/axum/helloworld
```

**`srwxr-xr-x` = 0755。** 拆开看:

| | 权限 | 能连上吗 |
|---|---|---|
| 所有者(你) | `rwx` |  **有 w** |
| 同组用户 | `r-x` |  没有 w |
| 其他人 | `r-x` |  没有 w |

**在 Linux 和 macOS 上,连接一个 UDS 需要对套接字文件有写权限。** 所以默认的 0755 实际上只允许你自己连——**这个默认值比看起来安全**。

**但它取决于你的 umask。** 如果某台机器的 umask 是 `000`,创建出来就是 **0777**,**任何用户都能连上你的服务**。

```bash
umask          # 看看你的
```

> **更好的写法**:**别依赖 umask,绑定后显式设置权限**:
>
> ```rust
> use std::os::unix::fs::PermissionsExt;
>
> let uds = UnixListener::bind(&path)?;
> tokio::fs::set_permissions(&path, std::fs::Permissions::from_mode(0o600)).await?;
> ```
>
>  注意这里有一个**竞态窗口**:从 `bind` 到 `set_permissions` 之间,套接字是按 umask 的权限存在的。要彻底消除,应该**把套接字放进一个权限受限的目录里**(比如先 `mkdir -m 700 /run/myapp`),这样即使套接字文件本身权限宽松,别人也进不了那个目录。
>
> 如果要允许特定的用户或组连接(比如 Nginx 以 `www-data` 运行),用 `0660` 加上 `chown :www-data`。

**另一个问题是路径选在了 `/tmp`。** `/tmp` 是**所有用户可写的共享目录**。虽然有 sticky bit 保护(别人删不掉你的文件),但:

- 任何人都能创建 `/tmp/axum/` —— **如果攻击者抢先创建了这个目录并设成自己可写,你的 `create_dir_all` 会直接成功用它**
- `/tmp` 在很多系统上会被定期清理,套接字文件可能凭空消失

**生产环境应该用 `/run/<appname>/`(需要 root 或 systemd 的 `RuntimeDirectory=` 帮你建)或者 `$XDG_RUNTIME_DIR`(用户级服务)。**

**别忘了收拾后台进程:**

```bash
kill %1
```

## 完整代码

`src/main.rs`:

```rust
#[cfg(unix)]
#[tokio::main]
async fn main() {
    unix::server().await;
}

#[cfg(not(unix))]
fn main() {
    println!("This example requires unix")
}

#[cfg(unix)]
mod unix {
    use axum::{
        body::Body,
        extract::connect_info::{self, ConnectInfo},
        http::{Request, StatusCode},
        routing::get,
        serve::IncomingStream,
        Router,
    };
    use http_body_util::BodyExt;
    use hyper_util::rt::TokioIo;
    use std::{path::PathBuf, sync::Arc};
    use tokio::net::{unix::UCred, UnixListener, UnixStream};
    use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

    pub async fn server() {
        tracing_subscriber::registry()
            .with(
                tracing_subscriber::EnvFilter::try_from_default_env()
                    .unwrap_or_else(|_| "debug".into()),
            )
            .with(tracing_subscriber::fmt::layer())
            .init();

        let path = PathBuf::from("/tmp/axum/helloworld");

        let _ = tokio::fs::remove_file(&path).await;
        tokio::fs::create_dir_all(path.parent().unwrap())
            .await
            .unwrap();

        let uds = UnixListener::bind(path.clone()).unwrap();
        tokio::spawn(async move {
            let app = Router::new()
                .route("/", get(handler))
                .into_make_service_with_connect_info::<UdsConnectInfo>();

            axum::serve(uds, app).await.unwrap();
        });

        let stream = TokioIo::new(UnixStream::connect(path).await.unwrap());
        let (mut sender, conn) = hyper::client::conn::http1::handshake(stream).await.unwrap();
        tokio::task::spawn(async move {
            if let Err(err) = conn.await {
                println!("Connection failed: {err:?}");
            }
        });

        let request = Request::get("http://uri-doesnt-matter.com")
            .body(Body::empty())
            .unwrap();

        let response = sender.send_request(request).await.unwrap();

        assert_eq!(response.status(), StatusCode::OK);

        let body = response.collect().await.unwrap().to_bytes();
        let body = String::from_utf8(body.to_vec()).unwrap();
        assert_eq!(body, "Hello, World!");
    }

    async fn handler(ConnectInfo(info): ConnectInfo<UdsConnectInfo>) -> &'static str {
        println!("new connection from `{info:?}`");

        "Hello, World!"
    }

    #[derive(Clone, Debug)]
    #[allow(dead_code)]
    struct UdsConnectInfo {
        peer_addr: Arc<tokio::net::unix::SocketAddr>,
        peer_cred: UCred,
    }

    impl connect_info::Connected<IncomingStream<'_, UnixListener>> for UdsConnectInfo {
        fn connect_info(stream: IncomingStream<'_, UnixListener>) -> Self {
            let peer_addr = stream.io().peer_addr().unwrap();
            let peer_cred = stream.io().peer_cred().unwrap();
            Self {
                peer_addr: Arc::new(peer_addr),
                peer_cred,
            }
        }
    }
}
```

## 代价与边界

**套接字文件权限依赖 umask,而且路径在 `/tmp`。** 步骤 5 详细说了。这是本章最实际的安全问题。

**`remove_file` 可能删掉另一个活着的实例的套接字。** 步骤 4 的警告框说了。滚动重启时会造成难以察觉的流量丢失。

**拿到了 `peer_cred` 却没用。** 字段带着 `#[allow(dead_code)]`,只在 `Debug` 里出现。**这个示例展示了能力,但没展示怎么用**——步骤 2 的「更好的写法」给了方向。

**退出时不清理套接字文件。** 进程结束后 `/tmp/axum/helloworld` 还在。下次启动靠 `remove_file` 兜底,但如果换了个路径,垃圾文件就永久留下了。加上[第 45 章](45-graceful-shutdown.md)的优雅停机后,可以在退出前 `remove_file`。

**没有优雅停机。** 同上。

**路径长度限制。** 前置知识说过,Linux 108 字节、macOS/BSD 104 字节。**把套接字放在深层项目目录里很容易撞上**,而错误信息通常只是一句 `invalid argument`,极难联想到长度问题。

**日志过滤器是全局 `"debug"`。** 步骤 1 提过,会把 hyper 内部的日志全打出来。应该像其他章一样用 `format!("{}=debug", env!("CARGO_CRATE_NAME"))`。

**用 `println!` 而不是 `tracing`。** 明明初始化了 `tracing_subscriber`,handler 里却用 `println!`。[第 17 章](17-tracing-aka-logging.md)讲过为什么这不好。

**程序跑完就退出。** 它是个自测程序,不是一个真的服务。想当服务用,把客户端那段删掉,或者把 `axum::serve` 放回主流程。

**跨机器就完全不适用。** UDS 只能本机。**如果你的架构可能从"单机多进程"演进到"多机",一开始就该考虑清楚**——切换到 TCP 不只是改一行地址,`peer_cred` 那套授权模型会整个失效,得换成 mTLS([第 50 章](50-low-level-rustls.md#动手练习))或令牌。

**没有连接数限制、没有超时。** 和前面几章一样。

**`peer_addr` 对 UDS 基本没用。** 客户端通常不 bind 自己的路径,所以拿到的是"未命名"。**真正有用的是 `peer_cred`。** 示例把两个都存了,但只有一个有意义。

## 动手练习

**1. 用 `peer_cred` 做真正的授权。**

提示:按步骤 2 的「更好的写法」,只允许你自己的 uid 访问。验证:普通 `curl --unix-socket ...` 通过,`sudo curl --unix-socket ...` 返回 403。**做完把它抽成一个中间件**,这样所有路由都受保护。

**2. 显式设置套接字权限,并换到安全的目录。**

提示:按步骤 5 的「更好的写法」设成 `0o600`。再试着把路径改成 `$XDG_RUNTIME_DIR/myapp.sock`(macOS 上没有这个变量,可以用 `/tmp/myapp-$(id -u)/`,并把目录设成 700)。验证:`ls -l` 确认权限,再用另一个用户(或 `sudo -u nobody curl ...`)确认连不上。

**3. 加上优雅停机,退出时清理套接字文件。**

提示:结合[第 45 章](45-graceful-shutdown.md):`axum::serve(uds, app).with_graceful_shutdown(shutdown_signal())`,`await` 之后 `tokio::fs::remove_file(&path).await`。验证:Ctrl-C 之后 `ls /tmp/axum/` 应该是空的。

**4. 把它变成一个真正的服务,再用 Nginx 代理。**

提示:删掉客户端那段,让服务常驻。然后配 Nginx:

```nginx
upstream app { server unix:/tmp/axum/helloworld; }
server {
    listen 8080;
    location / { proxy_pass http://app; }
}
```

访问 `http://localhost:8080/`,看服务端日志里的 uid —— **是 Nginx 的运行用户,不是你。** 这正好说明了 `peer_cred` 在代理后面的含义变化:**它告诉你的是"直接连上来的那个进程"是谁**,和[第 43 章](43-reverse-proxy.md#步骤-7看看上游到底收到了什么)讲的 `X-Forwarded-For` 是同一个问题的两面。

**5. 撞一次路径长度限制。**

提示:把路径改成一个很长的(比如在项目目录下嵌套十几层),看 `bind` 报什么错。**记住那个错误信息的样子**,以后撞上能立刻反应过来。

**6.(进阶)对比 UDS 和 TCP 回环的吞吐。**

提示:同一份 handler,分别用 `UnixListener` 和 `TcpListener` 起,用 `wrk` 或 `oha` 压测。 注意 curl 和多数压测工具对 UDS 的支持不如 TCP,可能需要用 `wrk` 的 Lua 脚本或专门的工具。**看看前置知识里说的"20%~50%"在你的机器上是多少。**

## 小结与下一章预告

Unix 域套接字用同一套 socket API,**但不经过网络协议栈**:地址是文件系统路径,访问控制靠文件权限,只能本机。真实世界里到处都是——Nginx 到应用、Docker 守护进程、PostgreSQL 的本机连接。

`axum::serve` 在 0.8 里对监听器是泛型的,**`UnixListener` 直接传进去就行**,不需要像[第 50 章](50-low-level-rustls.md)那样手写 accept 循环。

**本章真正的收获是 `SO_PEERCRED`**:内核直接告诉你对端进程的 uid、gid、pid,**进程无法伪造**。步骤 2 用 `sudo curl` 验证了这一点——客户端什么都没多发,服务端就看到了 `uid: 0`。这带来一种非常干净的授权模型:同机服务间调用不需要令牌、不需要密钥、不会过期。 但 **pid 会被复用,不能用于安全判断,只有 uid/gid 可靠**。

实现上,`Connected` trait 回答的是"给你一条刚建立的连接,你要记下什么"。整条 `ConnectInfo` 链路和[第 44 章](44-serve-with-hyper.md)完全一样,只是换了装载的类型。`Arc<SocketAddr>` 那个包装是因为 `tokio::net::unix::SocketAddr` 没有 `Clone` —— **"需要 Clone 但类型没有"时,`Arc` 往往是最省事的答案。**

步骤 4 那个坑值得单独记住:**套接字文件是普通的文件系统条目,进程退出不会删除**,于是第二次启动会报 `Address already in use`,而 `lsof` 和 `ps` 什么都查不到。用 TCP 的经验在这里完全帮不上忙。标准写法是 bind 前 `remove_file`,但**这本身有风险**——真有另一个实例在跑的话,你会把它的套接字删掉,它还活着却再也收不到请求。

安全上要记住两条:**套接字权限依赖 umask,应该显式设成 0600 并放进权限受限的目录**;**别用 `/tmp`**,那是所有用户可写的共享目录,应该用 `/run/<app>/` 或 `$XDG_RUNTIME_DIR`。

本章还顺带见到了第三种 HTTP 客户端:`hyper::client::conn::http1::handshake`,它假设"连接你已经建好了",只负责在这条流上讲 HTTP。**记住它返回的 `conn` 必须被 spawn,否则 `send_request` 会永远挂着** —— 又一个"创建了但没人驱动"的坑。

[第 54 章 simple-router-wasm](54-simple-router-wasm.md) 是本教程的最后一章,也是最反直觉的一章:**没有服务器,没有监听,连 tokio 都没有。** 你会看到 axum 的 `Router` 被当作一个纯粹的函数使用——喂一个请求进去,拿一个响应出来——以及这为什么正好是 serverless 和 WebAssembly 需要的形状。
