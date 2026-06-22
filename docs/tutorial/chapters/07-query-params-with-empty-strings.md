# 07. query-params-with-empty-strings

对应示例：`examples/query-params-with-empty-strings`

上一章用 `Form<T>` 处理表单。这章用 `Query<T>` 处理 URL query string,并解决一个常见问题:**当 query 参数值为空字符串时,怎么把它当 `None` 而不是报错**。

## Cargo.toml

````toml
[package]
name = "example-query-params-with-empty-strings"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]
axum = "0.8"
http-body-util = "0.1.0"
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
tower = { version = "0.5.2", features = ["util"] }
````

> 本地 `axum` 依赖如何配置见 [项目 README](../../../README.md#运行前提)。

## src/main.rs

````rust
use axum::{extract::Query, routing::get, Router};
use serde::{de, Deserialize, Deserializer};
use std::{fmt, str::FromStr};

#[tokio::main]
async fn main() {
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app()).await.unwrap();
}

fn app() -> Router {
    Router::new().route("/", get(handler))
}

async fn handler(Query(params): Query<Params>) -> String {
    format!("{params:?}")
}

/// See the tests below for which combinations of `foo` and `bar` result in
/// which deserializations.
///
/// This example only shows one possible way to do this. [`serde_with`] provides
/// another way. Use which ever method works best for you.
///
/// [`serde_with`]: https://docs.rs/serde_with/1.11.0/serde_with/rust/string_empty_as_none/index.html
#[derive(Debug, Deserialize)]
#[allow(dead_code)]
struct Params {
    #[serde(default, deserialize_with = "empty_string_as_none")]
    foo: Option<i32>,
    bar: Option<String>,
}

/// Serde deserialization decorator to map empty Strings to None,
fn empty_string_as_none<'de, D, T>(de: D) -> Result<Option<T>, D::Error>
where
    D: Deserializer<'de>,
    T: FromStr,
    T::Err: fmt::Display,
{
    let opt = Option::<String>::deserialize(de)?;
    match opt.as_deref() {
        None | Some("") => Ok(None),
        Some(s) => FromStr::from_str(s).map_err(de::Error::custom).map(Some),
    }
}
````

## 运行

````bash
cd examples
cargo run -p example-query-params-with-empty-strings
````

用不同 query string 测试:

````bash
curl 'http://127.0.0.1:3000/?foo=1&bar=bar'
# Params { foo: Some(1), bar: Some("bar") }

curl 'http://127.0.0.1:3000/?foo=&bar=bar'
# Params { foo: None, bar: Some("bar") }

curl 'http://127.0.0.1:3000/?foo=1'
# Params { foo: Some(1), bar: None }

curl 'http://127.0.0.1:3000/?bar='
# Params { foo: None, bar: Some("") }

curl 'http://127.0.0.1:3000/'
# Params { foo: None, bar: None }
````

## 解读

### 问题:空字符串 vs 缺失参数

`Query<T>` 内部用 serde 从 URL query string 反序列化。对 `Option<T>` 字段:

```text
参数缺失(如 ?bar=1 但没 foo)   → None
参数存在但为空(如 ?foo=&bar=1) → serde 默认尝试解析 "" 为 T
```

问题在第二种情况。`foo` 类型是 `Option<i32>`,如果 query 是 `?foo=`,serde 尝试把空字符串 `""` 解析成 `i32`,失败,整个请求返回 400 错误。但用户的意图往往是"这个参数我没填值,当成没有"。

`bar: Option<String>` 不受影响——空字符串 `""` 是合法的 `String`,所以 `?bar=` 得到 `Some("")`。

### `empty_string_as_none` deserializer

````rust
#[serde(default, deserialize_with = "empty_string_as_none")]
foo: Option<i32>,
````

两个 serde 属性配合:

- `deserialize_with = "empty_string_as_none"`:告诉 serde 用自定义函数反序列化这个字段
- `default`:参数缺失时用 `Default::default()`(对 `Option` 就是 `None`),而不是报错

自定义函数逻辑:

````rust
fn empty_string_as_none<'de, D, T>(de: D) -> Result<Option<T>, D::Error>
where
    D: Deserializer<'de>,
    T: FromStr,
    T::Err: fmt::Display,
{
    let opt = Option::<String>::deserialize(de)?;     // 先按 Option<String> 反序列化
    match opt.as_deref() {
        None | Some("") => Ok(None),                    // 缺失或空字符串 → None
        Some(s) => FromStr::from_str(s)                 // 非空 → 解析成 T
            .map_err(de::Error::custom).map(Some),
    }
}
````

先把输入当 `Option<String>` 读出来(这样 `?foo=` 得到 `Some("")`,`foo` 缺失得到 `None`),然后:
- `None` 或 `Some("")` → 返回 `None`
- `Some(非空)` → 用 `FromStr` 解析成目标类型(如 `i32`),失败则 `de::Error::custom`

泛型约束 `T: FromStr` + `T::Err: fmt::Display` 让这个函数可用于任何能从字符串解析的类型(`i32`、`u64`、`bool` 等)。

### 各种 query 组合的行为

| query | foo | bar | 说明 |
| --- | --- | --- | --- |
| `foo=1&bar=bar` | `Some(1)` | `Some("bar")` | 两个参数都有值 |
| `foo=&bar=bar` | `None` | `Some("bar")` | foo 空字符串→None(自定义 deserializer) |
| `foo=&bar=` | `None` | `Some("")` | foo→None,bar→空字符串(合法 String) |
| `foo=1` | `Some(1)` | `None` | bar 缺失→None(default) |
| `bar=bar` | `None` | `Some("bar")` | foo 缺失→None(default) |
| `foo=` | `None` | `None` | 两者都→None |
| `bar=` | `None` | `Some("")` | foo 缺失,bar 空字符串 |
| ``(空) | `None` | `None` | 都缺失 |

### 其他方案

除了自定义 deserializer,还有别的方式处理空字符串:

- **`serde_with` crate**:`#[serde_with::serde_as)] #[serde_as(as = "serde_with::NoneAsEmptyString")]` — 更声明式但引入额外依赖
- **改用 `String` 然后 handler 里手动判断**:`foo: Option<String>`,handler 里 `foo.and_then(|s| if s.is_empty() { None } else { s.parse().ok() })` — 更直白但每个字段都要写

本 example 的 `empty_string_as_none` 是**无需额外依赖**的通用方案,适合多个字段复用。

## 手写任务

跑通后做几个小改动:

1. 新增 `baz: Option<u64>` 字段,用 `empty_string_as_none`,测试 `?baz=` 和 `?baz=42`。
2. 把 `foo` 的 deserializer 改成:空字符串当 `Some(0)` 而不是 `None`(理解 `empty_string_as_none` 的灵活性)。
3. 不用 `#[serde(default)]`,只保留 `deserialize_with`,观察参数缺失时是否报错。

## 小结

- `Query<T>` 用 serde 从 URL query string 反序列化;`Option<T>` 字段参数缺失得到 `None`,但参数为空字符串时 serde 尝试解析 `""` 为 `T` 可能失败。
- `#[serde(default, deserialize_with = "empty_string_as_none")]` 让空字符串变 `None`:先用 `Option::<String>::deserialize` 读出,再判断空或缺失。
- `default` 让参数缺失时不报错而是用 `None`;`deserialize_with` 自定义反序列化逻辑。
- 泛型 `T: FromStr` 让 `empty_string_as_none` 可复用于 `i32`/`u64`/`bool` 等类型。

## 源码对照

- `examples/query-params-with-empty-strings/Cargo.toml`
- `examples/query-params-with-empty-strings/src/main.rs`
