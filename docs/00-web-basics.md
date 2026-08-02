# 第 00 章 序章:亲手摸一次 HTTP

## 本章你将做出什么

一个不依赖任何第三方 crate 的 TCP 服务器,大约 45 行。它会把浏览器和 curl 发来的请求原文打印在终端上,再手工拼出一段 HTTP 响应发回去。

你会看到:HTTP 不是什么神秘协议,它就是一段有格式约定的纯文本。跑通这一章,你就理解了 axum 这类 Web 框架究竟在替你做什么——以及为什么它们值得存在。

学完本章你将掌握:

- HTTP 请求和响应的报文结构,以及方法、路径、状态码、头部、body 各自的位置和作用
- `Content-Type` 如何决定客户端对同一段字节的解读方式
- 用 `curl` 观察和调试 HTTP 服务的基本手法

## 前置知识

### TCP、端口与 socket

网络上两台机器之间传数据,靠的是 TCP 这类传输层协议。TCP 提供的抽象很朴素:一条**双向的字节流**。你往里写字节,对端按同样的顺序读出来,不丢不乱。它不关心你写进去的是什么,是 HTML 还是图片还是乱码,对 TCP 来说都只是字节。

一台机器上可能同时跑很多网络程序,靠**端口**(port)区分。端口是个 0~65535 的整数。服务端程序"监听"(listen)某个端口,操作系统就把发往该端口的连接交给它。本教程统一用 `3000`,和官方示例保持一致。

`127.0.0.1` 是本机回环地址,只有本机能连上。上线时通常改成 `0.0.0.0`(监听所有网卡),这个区别在第 45 章讲部署时再说。

Rust 标准库里 `std::net::TcpListener` 就是"监听一个端口"的封装,`TcpStream` 是"一条已建立的连接"。它们都实现了 `Read` / `Write`,所以你可以像读写文件一样读写网络连接。

### HTTP 是一层文本约定

TCP 只给你字节流,双方还得约定"怎么切分、怎么理解这些字节"。HTTP 就是这个约定,而且是**人类可读的文本**约定。

一个 HTTP 请求长这样(`\r\n` 表示回车换行,下面用换行显示):

```
GET /hello HTTP/1.1
Host: 127.0.0.1:3000
User-Agent: curl/8.7.1
Accept: */*

```

拆开看:

- **第一行叫请求行**(request line),三段用空格隔开:**方法**(method)`GET`、**路径**(path)`/hello`、**协议版本** `HTTP/1.1`。
- **接下来是若干头部**(header),每行一个 `名字: 值`。头部是元信息——我是谁、我能接受什么格式、body 有多长。
- **然后是一个空行**,这是分界线,表示头部结束。
- **空行之后是 body**(消息体),可以为空。`GET` 请求通常没有 body;`POST` 提交表单或 JSON 时 body 里装的就是数据。

响应的结构几乎一样,只是第一行不同:

```
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 13

Hello, World!
```

第一行叫**状态行**(status line):协议版本、**状态码**(status code)`200`、原因短语 `OK`。

请求和响应的结构完全对称,这是 HTTP 好懂的地方:头部 + 空行 + body,两边一个样。

### 方法:告诉服务器你想干什么

方法表达意图,常用的有:

- `GET` — 我要读取某个资源,不要改动它
- `POST` — 我要提交数据,通常会产生新资源
- `PUT` — 我要用请求 body 整体替换某个资源
- `PATCH` — 我要局部修改某个资源
- `DELETE` — 我要删除某个资源
- `HEAD` — 和 `GET` 一样,但只要头部不要 body(第 07 章专门讲)

方法只是约定,服务器完全可以在 `GET` 里删数据——但那样所有中间层(浏览器缓存、CDN、代理)的假设都会被打破,所以别这么干。

### 状态码:一个三位数的分类回答

状态码按首位分五类,记住这个分类比记住具体数字有用:

- `1xx` 信息性,很少直接接触(WebSocket 升级用到 `101`,第 34 章会见到)
- `2xx` 成功。`200 OK` 最常见,`201 Created` 表示创建成功,`204 No Content` 表示成功但没有 body
- `3xx` 重定向。`301`/`302` 让客户端换个地址再来一次
- `4xx` **客户端错**。`400 Bad Request` 请求格式不对,`401 Unauthorized` 没登录,`403 Forbidden` 登录了但没权限,`404 Not Found` 资源不存在,`405 Method Not Allowed` 路径对但方法不对
- `5xx` **服务端错**。`500 Internal Server Error` 服务器自己崩了

`4xx` 和 `5xx` 的分界很关键:`4xx` 是"你的请求有问题",`5xx` 是"我的代码有问题"。后面第 14 章讲错误处理时,核心就是判断该返回哪一类。

### Content-Type 与 MIME 类型

body 是一串字节,客户端怎么知道该当文本显示、当 HTML 渲染、还是当图片解码?靠 `Content-Type` 头部。它的值是一个 **MIME 类型**:

- `text/plain` — 纯文本,浏览器原样显示
- `text/html` — HTML,浏览器渲染成页面
- `application/json` — JSON 数据,给程序解析用
- `application/x-www-form-urlencoded` — HTML 表单的默认提交格式(第 03 章)
- `multipart/form-data` — 带文件上传的表单格式(第 13 章)
- `image/png`、`image/jpeg` — 图片

后面加 `; charset=utf-8` 是告诉客户端用什么字符编码解析文本,不加的话中文可能显示成乱码。

本章你会亲手改一次 `Content-Type`,看到同一段字节在浏览器里呈现出完全不同的结果。这是理解 axum 的 `IntoResponse` 的基础——axum 之所以能让你 `return Html(...)` 就自动带上 `text/html`,本质就是替你写了这个头部。

### Content-Length:body 有多长

TCP 是字节流,没有"消息边界"的概念。服务端写完响应如果不关连接,客户端怎么知道 body 读完了?靠 `Content-Length` 头部声明字节数。

这个数字必须**精确**。写少了客户端会截断 body,写多了客户端会一直等剩下的字节直到超时。本章我们手工计算它;从第 01 章开始 axum 会替你算。

注意 `Content-Length` 是**字节数**,不是字符数。`"Hello"` 是 5 字节,`"你好"` 在 UTF-8 下是 6 字节。Rust 里 `str::len()` 返回的正是字节数,刚好合用。

### curl:命令行 HTTP 客户端

`curl` 是本教程贯穿全程的验证工具,先记住这几个参数:

- `curl http://127.0.0.1:3000/` — 发个 `GET`,把 body 打到标准输出
- `curl -i ...` — 同时显示响应的状态行和头部(**看状态码就靠这个**)
- `curl -v ...` — 显示完整交互过程,包括发出去的请求头
- `curl -X POST ...` — 指定方法
- `curl -d '数据' ...` — 带 body(带了 `-d` 默认就变成 `POST`)
- `curl -H 'Content-Type: application/json' ...` — 自定义请求头

macOS 和多数 Linux 发行版自带 curl。Windows 10 之后的 PowerShell 也有,但 `curl` 在旧版 PowerShell 里是 `Invoke-WebRequest` 的别名,参数不兼容,建议装 Git Bash 或 WSL。

## 项目搭建

### 环境准备

需要 Rust 1.75 或更高版本(axum 0.8 的最低要求):

```bash
rustc --version
```

如果版本偏低,用 `rustup update stable` 升级。没装 Rust 的话去 <https://rustup.rs> 装 rustup。

还需要 curl:

```bash
curl --version
```

**关于 axum 版本**:本教程基于 **axum 0.8.9**(crates.io 上的当前稳定版)。各章一律用 `cargo add` 安装最新版,不锁死版本号。axum 0.8 之后的小版本改动很少涉及本教程内容;如果你用的是更高版本发现 API 对不上,以 <https://docs.rs/axum> 为准。

### 创建项目

本章不需要任何第三方 crate,标准库就够:

```bash
cargo new web-basics
cd web-basics
```

后面每一章都会新建一个独立项目,项目名就是那一章对应的示例名。这样每章互不干扰,你也能随时回头翻看某一章的成果。

## 分步构建

### 步骤 1:把请求原文打印出来

**要做什么**:监听 3000 端口,把客户端发来的字节原样打印到终端。先不管响应。

把 `src/main.rs` 全部替换成:

```rust
// src/main.rs
use std::io::Read;
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:3000").unwrap();
    println!("listening on 127.0.0.1:3000");

    for stream in listener.incoming() {
        let mut stream = stream.unwrap();

        let mut buf = [0u8; 4096];
        let n = stream.read(&mut buf).unwrap();

        println!("---- 收到 {n} 字节 ----");
        println!("{}", String::from_utf8_lossy(&buf[..n]));
    }
}
```

`TcpListener::bind` 占住 3000 端口。`listener.incoming()` 是个迭代器,每次有客户端连上来就产出一个 `TcpStream`,没人连就阻塞等待。所以这个 `for` 循环永远不会结束——这正是服务器该有的行为。

`stream.read(&mut buf)` 从连接里读字节。这里有个**重要的简化**:我们只读一次,读多少算多少。真实的 HTTP 服务必须循环读到头部结束的空行、再按 `Content-Length` 读完 body,因为 TCP 不保证一次 `read` 就能拿到完整请求。这一章的目的是看清报文长什么样,不是写一个健壮的服务器,所以先这么凑合。这类脏活正是 axum 会替你处理掉的。

`String::from_utf8_lossy` 把字节按 UTF-8 解释,遇到非法字节用 `�` 代替而不是 panic。HTTP 头部一定是 ASCII,所以这里很安全。

**跑起来看**:

```bash
cargo run
```

终端显示 `listening on 127.0.0.1:3000`。**另开一个终端**:

```bash
curl http://127.0.0.1:3000/hello
```

curl 会报错:

```
curl: (52) Empty reply from server
```

这是对的——我们收了请求但一个字节都没回。回到服务端的终端,你会看到:

```
---- 收到 82 字节 ----
GET /hello HTTP/1.1
Host: 127.0.0.1:3000
User-Agent: curl/8.7.1
Accept: */*

```

这就是前置知识里讲的请求报文,一字不差:请求行 + 三个头部 + 空行。字节数和 User-Agent 版本会因你的 curl 而异。

顺手试试浏览器,访问 `http://127.0.0.1:3000/hello`,你会看到浏览器发的头部比 curl 多得多——`Accept`、`Accept-Encoding`、`Accept-Language`、`Sec-Fetch-*` 等等。同一个协议,不同客户端的"自我介绍"丰俭由人。

按 `Ctrl-C` 停掉服务。

### 步骤 2:手工拼一个响应

**要做什么**:回一段合法的 HTTP 响应,让 curl 能正常显示。

改 `src/main.rs`,`use` 加上 `Write`,循环体末尾加两行:

```rust
// src/main.rs
use std::io::{Read, Write};   // 改动:加上 Write
use std::net::TcpListener;

fn main() {
    // ...(bind 和 println 不变)

    for stream in listener.incoming() {
        // ...(读取和打印不变)

        // 新增:手工拼出一段 HTTP 响应
        let body = "Hello, World!";
        let response = format!(
            "HTTP/1.1 200 OK\r\n\
             Content-Type: text/plain; charset=utf-8\r\n\
             Content-Length: {}\r\n\
             \r\n\
             {}",
            body.len(),
            body
        );
        stream.write_all(response.as_bytes()).unwrap();
    }
}
```

这段 `format!` 就是前置知识里那个响应报文,逐字对应:状态行、两个头部、空行、body。

几个容易踩的点:

**换行必须是 `\r\n`,不能只用 `\n`。** HTTP 规范要求 CRLF。宽容的客户端可能认单独的 `\n`,但别赌这个。

**头部和 body 之间是一个完整的空行**,也就是连续两个 `\r\n`。上面字符串里 `Content-Length: {}\r\n` 结尾那个 `\r\n` 结束了头部行,紧接着的 `\r\n` 才是那个空行。少写一个,客户端会把 body 的第一行当成头部继续解析。

**`Content-Length` 用 `body.len()`**,也就是字节数,和 HTTP 的要求一致。

代码里 `\` 结尾是 Rust 的续行语法,它会吃掉换行和后面的缩进,让长字符串在源码里能排整齐而不引入多余空白。

**跑起来看**:

```bash
cargo run
```

另开终端:

```bash
curl http://127.0.0.1:3000/
```

```
Hello, World!
```

加 `-i` 看完整响应:

```bash
curl -i http://127.0.0.1:3000/
```

```
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 13

Hello, World!
```

你手写的每一行都在这儿。这一刻值得停一下:你现在有一个真正的 HTTP 服务器了,没用任何框架。

试试改点东西加深印象:把 `Content-Type` 改成 `text/html`,body 改成 `<h1>Hello, World!</h1>`,重新 `cargo run`,用浏览器访问——你会看到一个大标题。把 `Content-Type` 改回 `text/plain`,浏览器就原样显示这段标签文本。**同一段字节,一个头部决定了客户端怎么理解它。** 这就是 `Content-Type` 的全部意义。

### 步骤 3:按方法和路径分发

**要做什么**:解析请求行,不同的路径返回不同内容,不认识的路径返回 `404`。

现在的服务器不管你请求什么都回同一个东西。真实服务需要**路由**(routing):按方法和路径决定交给哪段逻辑处理。这一步我们手写一个最土的路由。

先抽一个构造响应的辅助函数,加在 `main` 之后:

```rust
// src/main.rs

// 新增:按状态码、Content-Type、body 拼出完整响应报文
fn build_response(status: &str, content_type: &str, body: &str) -> String {
    format!(
        "HTTP/1.1 {}\r\n\
         Content-Type: {}\r\n\
         Content-Length: {}\r\n\
         \r\n\
         {}",
        status,
        content_type,
        body.len(),
        body
    )
}
```

然后把循环体里拼响应那段换成解析 + 分发:

```rust
// src/main.rs
    for stream in listener.incoming() {
        // ...(读取和打印不变)

        // 新增:从请求行里取出方法和路径
        let request = String::from_utf8_lossy(&buf[..n]);
        let request_line = request.lines().next().unwrap_or("");
        let mut parts = request_line.split_whitespace();
        let method = parts.next().unwrap_or("");
        let path = parts.next().unwrap_or("");

        // 改动:按方法 + 路径分发
        let response = match (method, path) {
            ("GET", "/") => build_response("200 OK", "text/plain; charset=utf-8", "Hello, World!"),
            ("GET", "/hello") => {
                build_response("200 OK", "text/html; charset=utf-8", "<h1>Hello, World!</h1>")
            }
            ("GET", "/json") => {
                build_response("200 OK", "application/json", r#"{"message":"Hello, World!"}"#)
            }
            _ => build_response("404 Not Found", "text/plain; charset=utf-8", "Not Found"),
        };
        stream.write_all(response.as_bytes()).unwrap();
    }
```

请求行 `GET /hello HTTP/1.1` 按空格切三段,取前两段就是方法和路径。`match (method, path)` 是 Rust 里表达这种分发最自然的写法。

注意 `_ =>` 这个分支返回 `404 Not Found`。这就是"路径不存在"的标准回答,不是什么错误处理的额外功夫,而是路由逻辑的自然组成部分。

`r#"..."#` 是 Rust 的原始字符串,里面的 `"` 不用转义,写 JSON 很方便。

**跑起来看**:

```bash
cargo run
```

四个路径依次试:

```bash
curl -i http://127.0.0.1:3000/
```

```
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 13

Hello, World!
```

```bash
curl -i http://127.0.0.1:3000/hello
```

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 22

<h1>Hello, World!</h1>
```

```bash
curl -i http://127.0.0.1:3000/json
```

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 27

{"message":"Hello, World!"}
```

```bash
curl -i http://127.0.0.1:3000/nope
```

```
HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=utf-8
Content-Length: 9

Not Found
```

再试一个有意思的:

```bash
curl -i -X POST http://127.0.0.1:3000/
```

```
HTTP/1.1 404 Not Found
...
```

路径 `/` 明明存在,但方法是 `POST`,我们的 `match` 匹配不上,落到了 `_` 分支返回 `404`。**严格来说这里应该返回 `405 Method Not Allowed`**——路径存在,只是不支持这个方法。要正确实现,得先按路径判断存在性、再按方法判断,分发逻辑立刻复杂一层。

记住这个不完美的地方。第 02 章你会看到 axum 自动给出正确的 `405`,一行额外代码都不用写。

## 完整代码

`src/main.rs`:

```rust
use std::io::{Read, Write};
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:3000").unwrap();
    println!("listening on 127.0.0.1:3000");

    for stream in listener.incoming() {
        let mut stream = stream.unwrap();

        let mut buf = [0u8; 4096];
        let n = stream.read(&mut buf).unwrap();

        println!("---- 收到 {n} 字节 ----");
        println!("{}", String::from_utf8_lossy(&buf[..n]));

        let request = String::from_utf8_lossy(&buf[..n]);
        let request_line = request.lines().next().unwrap_or("");
        let mut parts = request_line.split_whitespace();
        let method = parts.next().unwrap_or("");
        let path = parts.next().unwrap_or("");

        let response = match (method, path) {
            ("GET", "/") => build_response("200 OK", "text/plain; charset=utf-8", "Hello, World!"),
            ("GET", "/hello") => {
                build_response("200 OK", "text/html; charset=utf-8", "<h1>Hello, World!</h1>")
            }
            ("GET", "/json") => {
                build_response("200 OK", "application/json", r#"{"message":"Hello, World!"}"#)
            }
            _ => build_response("404 Not Found", "text/plain; charset=utf-8", "Not Found"),
        };
        stream.write_all(response.as_bytes()).unwrap();
    }
}

fn build_response(status: &str, content_type: &str, body: &str) -> String {
    format!(
        "HTTP/1.1 {}\r\n\
         Content-Type: {}\r\n\
         Content-Length: {}\r\n\
         \r\n\
         {}",
        status,
        content_type,
        body.len(),
        body
    )
}
```


## 这个服务器还差什么

把它和一个能上生产的服务对比一下,缺口是这些:

**并发**。`for stream in listener.incoming()` 一次只处理一个连接,前一个没处理完后面全排队。真实服务需要并发处理成千上万个连接。

**完整读取请求**。只 `read` 一次就当拿到了全部请求,遇到大 body 或分片到达就会读不全。

**头部解析**。我们只切了请求行,头部完全没解析。要读 `Content-Type`、`Authorization`、cookie,都得自己写解析。

**body 解析**。`POST` 提交的表单、JSON,现在完全没碰。

**路由匹配**。只支持完全相等的路径。`/users/42` 这种带参数的路径、`/static/*` 这种通配,都得自己写匹配算法。

**方法不匹配的正确响应**。上面看到的 `405` 问题。

**响应构造**。每次手拼字符串,`Content-Length` 算错一个字节客户端就挂。

**HTTP/2、TLS、压缩、keep-alive**……

这七八项每一项都是可观的工作量,而且和你的业务逻辑毫无关系。**Web 框架的价值就在这里:它把这些全部处理掉,让你只写"收到什么请求、返回什么内容"。**

axum 的做法是把"处理请求"抽象成一个普通的 Rust 异步函数——你写 `async fn handler() -> Html<&'static str>`,axum 负责把它接到 HTTP 世界里。下一章你会看到这有多轻。

## 动手练习

**1. 支持 `POST /echo`,把请求 body 原样返回。**

提示:body 在报文里位于第一个空行之后。你需要在字节数组里找到 `\r\n\r\n` 的位置,它之后的部分就是 body。`buf[..n].windows(4).position(|w| w == b"\r\n\r\n")` 能帮你定位。验证:`curl -i -d 'hello' http://127.0.0.1:3000/echo`。

**2. 正确区分 `404` 和 `405`。**

提示:把分发改成两层——先只按 `path` 匹配,确认路径存在后再检查 `method`。路径存在但方法不对时返回 `405 Method Not Allowed`,并按规范加上 `Allow: GET` 头部告诉客户端该路径支持哪些方法。写完你会体会到:为什么这件事值得让框架去做。

## 小结与下一章预告

HTTP 是建立在 TCP 字节流之上的一层文本约定:请求行/状态行 + 头部 + 空行 + body。你手写了一个能收请求、能分发路由、能返回不同 `Content-Type` 的服务器,也看清了它离生产可用还差多少——并发、解析、路由匹配、正确的状态码,全都缺。

下一章进入 [第 01 章 hello-world](01-hello-world.md)。同样的功能,用 axum 只需要 14 行,而且并发、解析、`Content-Length` 全都不用你操心。更重要的是,你会理解 axum 凭什么能让一个普通的 `async fn` 直接变成 HTTP 处理器——那是本教程后面所有内容的地基。
