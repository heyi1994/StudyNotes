# std:Option / Result / ? / Error

> Rust 没有异常（exception）机制，而是把"可能没有值"和"可能失败"做成了**类型系统里的一等公民**：`Option<T>` 和 `Result<T, E>`。错误成了必须显式处理的返回值，编译器逼你面对它——这正是 Rust 健壮性的来源。

---

## 目录

- [一、Rust 错误处理的总体哲学](#一rust-错误处理的总体哲学)
- [二、Option：可能没有值](#二option可能没有值)
- [三、Result：可能失败](#三result可能失败)
- [四、? 运算符：错误传播神器](#四-运算符错误传播神器)
- [五、panic! 与不可恢复错误](#五panic-与不可恢复错误)
- [六、Option / Result 方法大全](#六option--result-方法大全)
- [七、Error trait 与自定义错误](#七error-trait-与自定义错误)
- [八、错误类型转换：From 与 ?](#八错误类型转换from-与-)
- [九、实战：库 vs 应用的错误策略](#九实战库-vs-应用的错误策略)
- [十、常用第三方 crate](#十常用第三方-crate)
- [十一、常见陷阱](#十一常见陷阱)

---

## 一、Rust 错误处理的总体哲学

Rust 把错误分成两大类，应对方式完全不同：

| 类别 | 含义 | 处理方式 | 例子 |
|------|------|---------|------|
| **可恢复错误** | 预期内、可应对 | 返回 `Result<T, E>` / `Option<T>` | 文件不存在、解析失败、网络超时 |
| **不可恢复错误** | bug、违反前提条件 | `panic!` 直接终止 | 数组越界、除零、断言失败 |

核心原则：

1. **用返回值表达错误，而非抛异常。** 错误是数据，跟着函数签名走，调用方一眼可见。
2. **编译器强制处理。** `Result`/`Option` 带 `#[must_use]`，不处理会警告。
3. **`?` 让传播变轻松。** 不必层层手写 match。

```
没有值？     → Option<T>     （Some / None）
可能失败？   → Result<T, E>  （Ok / Err）
程序 bug？   → panic!        （直接崩溃）
```

---

## 二、Option：可能没有值

`Option<T>` 表达"**也许有一个 T，也许什么都没有**"，取代了其他语言的 `null`：

```rust
pub enum Option<T> {
    Some(T),  // 有值
    None,     // 无值
}
```

什么时候返回 `Option`：查找可能找不到、可选字段、可能为空的计算等。

```rust
// 查找：找不到返回 None
let v = vec![1, 2, 3];
let found: Option<&i32> = v.iter().find(|&&x| x == 2); // Some(&2)
let missing: Option<&i32> = v.iter().find(|&&x| x == 9); // None

// 解析、首元素、HashMap 取值等都返回 Option
let first: Option<&i32> = v.first();   // Some(&1)
```

### 取出 Option 里的值

```rust
let x: Option<i32> = Some(5);

// 1. match —— 最基础、最清晰
match x {
    Some(n) => println!("有值 {}", n),
    None => println!("没值"),
}

// 2. if let —— 只关心一种情况
if let Some(n) = x {
    println!("有值 {}", n);
}

// 3. let else —— 不满足就提前返回（很实用）
let Some(n) = x else {
    return; // 或 continue/break/panic
};
println!("n = {}", n); // 此后 n 直接可用

// 4. 各种方法（见第六节）
let n = x.unwrap_or(0);          // 没值给默认
let n = x.unwrap_or_else(|| 0);  // 没值用闭包算默认
```

> **不要轻易 `unwrap()`**：`None.unwrap()` 会 panic。生产代码中优先用 `unwrap_or` / `?` / `match`。

---

## 三、Result：可能失败

`Result<T, E>` 表达"**要么成功得到 T，要么失败得到错误 E**"：

```rust
pub enum Result<T, E> {
    Ok(T),   // 成功，携带结果
    Err(E),  // 失败，携带错误信息
}
```

凡是可能失败的操作（IO、解析、网络、转换）都返回 `Result`：

```rust
use std::fs::File;

// 打开文件可能失败
let f: Result<File, std::io::Error> = File::open("hello.txt");

match f {
    Ok(file) => println!("打开成功"),
    Err(e) => println!("打开失败: {}", e),
}

// 字符串解析
let n: Result<i32, _> = "42".parse::<i32>();   // Ok(42)
let bad: Result<i32, _> = "abc".parse::<i32>(); // Err(ParseIntError)
```

### 处理 Result

```rust
// 1. match
match "42".parse::<i32>() {
    Ok(n) => println!("解析得到 {}", n),
    Err(e) => println!("解析失败: {}", e),
}

// 2. 方法
let n = "42".parse::<i32>().unwrap_or(0);
let n = "42".parse::<i32>().unwrap_or_else(|_| -1);

// 3. 出错就 panic（带自定义信息），适合原型/测试
let n = "42".parse::<i32>().expect("应该是合法数字");
```

---

## 四、? 运算符：错误传播神器

`?` 是 Rust 错误处理的精髓，用来**把错误向上传播**，避免层层手写 match。

`?` 作用在 `Result` 上：
- 如果是 `Ok(v)`，取出 `v` 继续执行；
- 如果是 `Err(e)`，**立即从当前函数 return `Err(e)`**。

### 对比：有无 `?`

```rust
use std::fs::File;
use std::io::{self, Read};

//  不用 ?：又长又啰嗦
fn read_username_verbose() -> Result<String, io::Error> {
    let f = File::open("user.txt");
    let mut f = match f {
        Ok(file) => file,
        Err(e) => return Err(e),
    };
    let mut s = String::new();
    match f.read_to_string(&mut s) {
        Ok(_) => Ok(s),
        Err(e) => Err(e),
    }
}

//  用 ?：简洁明了
fn read_username() -> Result<String, io::Error> {
    let mut s = String::new();
    File::open("user.txt")?.read_to_string(&mut s)?;
    Ok(s)
}

//  还能链式
fn read_username_chain() -> Result<String, io::Error> {
    let mut s = String::new();
    File::open("user.txt")?.read_to_string(&mut s)?;
    Ok(s)
}
```

### `?` 也能用于 Option

在返回 `Option` 的函数里，`?` 遇到 `None` 会提前 return `None`：

```rust
fn first_char_upper(s: &str) -> Option<char> {
    let c = s.chars().next()?;       // None 则直接返回 None
    Some(c.to_ascii_uppercase())
}
```

### `?` 的使用前提

- 只能用在**返回 `Result` 或 `Option`（或实现了 `Try`）的函数**里。
- `main` 函数也可以返回 `Result<(), E>`，从而在 `main` 里用 `?`：

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let content = std::fs::read_to_string("config.txt")?;
    println!("{}", content);
    Ok(())
}
```

---

## 五、panic! 与不可恢复错误

`panic!` 用于**程序无法继续、属于 bug 的情况**。它会展开栈（unwind）并终止线程。

```rust
panic!("出现了不该出现的情况");

// 隐式 panic 的常见来源：
let v = vec![1, 2, 3];
// v[10];                  // 越界 panic
// let x: Option<i32> = None; x.unwrap(); // None.unwrap() panic
// 1 / 0;                  // 除零（编译期常量会直接报错）
```

### 什么时候该 panic？什么时候该返回 Result？

| 用 `panic!` | 用 `Result` |
|------------|------------|
| 违反函数前置条件（调用方的 bug） | 预期内的失败 |
| 继续下去会导致内存不安全/数据损坏 | 外部输入、IO、网络等不可控因素 |
| 测试、原型、示例代码 | 库的公共 API |
| 不变量被破坏（"绝不可能发生"） | 调用方有合理的应对方式 |

> 经验法则：**库代码尽量返回 `Result` 让调用方决定；只有"这是个 bug"时才 panic。** 应用顶层可以选择 `unwrap`/`expect` 把错误转成崩溃。

### `unwrap` vs `expect`

```rust
let n: i32 = "42".parse().unwrap();           // panic 信息较少
let n: i32 = "42".parse().expect("配置里的端口号必须是数字"); // panic 带上下文，推荐
```

> 用 `expect` 而非 `unwrap`，并写明"为什么这里不该失败"，对排查问题非常有帮助。

---

## 六、Option / Result 方法大全

熟练这些方法，能让代码摆脱大量 match，变得流畅。

### 提取值

| 方法 | 作用 | 失败/空时 |
|------|------|----------|
| `unwrap()` | 取值 | panic |
| `expect(msg)` | 取值 | panic（带 msg） |
| `unwrap_or(default)` | 取值 | 返回 default |
| `unwrap_or_else(f)` | 取值 | 返回 `f()` 计算的默认 |
| `unwrap_or_default()` | 取值 | 返回类型的 `Default` |

### 变换（map 家族）

| 方法 | 适用 | 作用 |
|------|------|------|
| `map(f)` | 两者 | 对内部值变换：`Some(x)→Some(f(x))` / `Ok(x)→Ok(f(x))` |
| `map_err(f)` | Result | 只变换错误：`Err(e)→Err(f(e))` |
| `map_or(default, f)` | 两者 | 有值则 `f`，无值给 default |
| `map_or_else(d, f)` | 两者 | 有值 `f`，无值 `d()` |
| `and_then(f)` | 两者 | 链式，`f` 自身返回 Option/Result（避免嵌套） |
| `or_else(f)` | 两者 | 失败时用 `f` 提供备选 |

```rust
// map：变换成功值
let len: Option<usize> = Some("hello").map(|s| s.len()); // Some(5)

// and_then：链式可能失败的操作（扁平化，避免 Option<Option<_>>）
let r: Option<i32> = Some("42")
    .and_then(|s| s.parse::<i32>().ok())  // 解析失败则 None
    .and_then(|n| if n > 0 { Some(n * 2) } else { None });
// Some(84)

// map_err：统一错误类型
let n: Result<i32, String> = "x"
    .parse::<i32>()
    .map_err(|e| format!("解析失败: {}", e));
```

### Option ↔ Result 互转

| 方法 | 方向 | 说明 |
|------|------|------|
| `ok_or(err)` | Option → Result | `None` 变成 `Err(err)` |
| `ok_or_else(f)` | Option → Result | `None` 变成 `Err(f())` |
| `ok()` | Result → Option | `Err` 丢弃，变 `None` |
| `err()` | Result → Option | 取出错误部分 |

```rust
// Option → Result：把"没找到"变成具体错误，方便配合 ?
fn get_config(key: &str) -> Result<String, String> {
    std::env::var(key).ok()         // Result → Option
        .ok_or_else(|| format!("缺少配置 {}", key))  // Option → Result
}

// Result → Option：只关心成功值时丢弃错误
let n: Option<i32> = "42".parse().ok(); // Some(42)
```

### 判断与过滤

| 方法 | 作用 |
|------|------|
| `is_some()` / `is_none()` | Option 判断 |
| `is_ok()` / `is_err()` | Result 判断 |
| `filter(p)` | Option：不满足条件则变 None |
| `as_ref()` / `as_mut()` | `&Option<T>` → `Option<&T>`，避免移动 |

---

## 七、Error trait 与自定义错误

标准库的 `std::error::Error` trait 是错误类型的统一约定：

```rust
pub trait Error: Debug + Display {
    // 返回底层错误（错误链），默认 None
    fn source(&self) -> Option<&(dyn Error + 'static)> { None }
}
```

要成为"标准错误"，类型需要：
1. 实现 `Debug`（通常 `#[derive(Debug)]`）；
2. 实现 `Display`（给人看的错误信息）；
3. 实现 `Error`（可选实现 `source` 提供错误链）。

### 手写一个自定义错误（enum 形式，最常见）

```rust
use std::fmt;

#[derive(Debug)]
enum ConfigError {
    NotFound(String),
    ParseError(String),
    Io(std::io::Error),  // 包裹底层错误
}

// 1. Display：面向用户的描述
impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            ConfigError::NotFound(k) => write!(f, "找不到配置项: {}", k),
            ConfigError::ParseError(s) => write!(f, "解析失败: {}", s),
            ConfigError::Io(e) => write!(f, "IO 错误: {}", e),
        }
    }
}

// 2. Error：可提供错误链
impl std::error::Error for ConfigError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            ConfigError::Io(e) => Some(e),  // 指向底层错误
            _ => None,
        }
    }
}
```

---

## 八、错误类型转换：From 与 ?

`?` 有个强大特性：**当错误类型不一致时，会自动调用 `From::from` 转换**。

也就是说，只要为你的错误类型实现了 `From<其他错误>`，`?` 就能自动把底层错误转换成你的错误类型：

```rust
// 为 ConfigError 实现 From<io::Error>
impl From<std::io::Error> for ConfigError {
    fn from(e: std::io::Error) -> Self {
        ConfigError::Io(e)
    }
}

impl From<std::num::ParseIntError> for ConfigError {
    fn from(e: std::num::ParseIntError) -> Self {
        ConfigError::ParseError(e.to_string())
    }
}

// 现在函数返回 ConfigError，但内部可以对不同错误用 ?
fn load_port() -> Result<u16, ConfigError> {
    // read_to_string 返回 io::Error，? 自动转成 ConfigError::Io
    let content = std::fs::read_to_string("port.txt")?;
    // parse 返回 ParseIntError，? 自动转成 ConfigError::ParseError
    let port: u16 = content.trim().parse()?;
    Ok(port)
}
```

这就是 `?` + `From` 的黄金组合：**一个函数里可以混用多种来源的错误，`?` 自动统一成你的错误类型。**

### 偷懒方案：Box<dyn Error>

如果不想定义具体错误类型，可以用 trait 对象兜住任何错误：

```rust
use std::error::Error;

// Box<dyn Error> 能容纳任何实现了 Error 的错误
fn run() -> Result<(), Box<dyn Error>> {
    let content = std::fs::read_to_string("config.txt")?; // io::Error 自动装箱
    let n: i32 = content.trim().parse()?;                  // ParseIntError 自动装箱
    println!("{}", n);
    Ok(())
}
```

> `Box<dyn Error>` 之所以好用，是因为标准库为它实现了 `From<E: Error>`，所以任何错误都能被 `?` 装箱进去。代价是丢失了具体类型信息（无法 match 区分错误种类）。

---

## 九、实战：库 vs 应用的错误策略

错误处理策略要分场景：

### 写**库（library）**：错误类型要精确

- 定义**具体的错误类型**（enum），让调用方能 `match` 区分、做不同处理。
- 实现 `Error` + `Display` + `From`。
- 不要随便 `panic`（除非是调用方的 bug）。
- 推荐用 [`thiserror`](https://crates.io/crates/thiserror) 减少样板代码。

```rust
// 用 thiserror（推荐给库）
use thiserror::Error;

#[derive(Error, Debug)]
enum DataError {
    #[error("找不到配置项: {0}")]
    NotFound(String),

    #[error("IO 错误")]
    Io(#[from] std::io::Error),   // 自动生成 From + source

    #[error("解析失败: {0}")]
    Parse(#[from] std::num::ParseIntError),
}
// thiserror 自动帮你实现 Display / Error / From，非常省事
```

### 写**应用（application）**：方便优先

- 通常不在乎错误的具体类型，只想"出错就报告并退出/记录"。
- 用 `Box<dyn Error>` 或 [`anyhow`](https://crates.io/crates/anyhow) 兜底。
- 在错误上附加上下文信息（`anyhow` 的 `.context()`）。

```rust
// 用 anyhow（推荐给应用）
use anyhow::{Context, Result};

fn load_config() -> Result<String> {
    let path = "config.toml";
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("读取配置文件失败: {}", path))?;
    Ok(content)
}
```

**一句话**：
- **库用 `thiserror`**（精确、可被调用方区分）
- **应用用 `anyhow`**（方便、带上下文、不在乎具体类型）

---

## 十、常用第三方 crate

| Crate | 用途 | 适用 |
|-------|------|------|
| [`thiserror`](https://crates.io/crates/thiserror) | 派生宏自动实现自定义错误的样板代码 | 库 |
| [`anyhow`](https://crates.io/crates/anyhow) | 通用错误类型 + 上下文，类似增强版 `Box<dyn Error>` | 应用 |
| [`eyre`](https://crates.io/crates/eyre) | anyhow 的变体，可定制错误报告格式 | 应用 |

> 标准库够用，但这两个 crate（`thiserror` + `anyhow`）几乎是 Rust 生态的事实标准，强烈建议掌握。

---

## 十一、常见陷阱

1. **滥用 `unwrap()`**
   原型阶段可以，生产代码里一个 `None.unwrap()` 就是一次崩溃。优先 `?` / `unwrap_or` / `match` / `expect("原因")`。

2. **忽略 `Result`（不处理返回值）**
   `Result` 带 `#[must_use]`，丢弃会有警告。确实想忽略时显式写 `let _ = ...;`。

3. **错误类型不匹配导致 `?` 编译失败**
   函数返回的错误类型必须能从 `?` 处的错误 `From` 转换而来。要么实现 `From`，要么用 `Box<dyn Error>` / `anyhow`，要么 `.map_err(...)` 手动转。

4. **在不返回 Result/Option 的函数里用 `?`**
   `?` 需要当前函数返回 `Result`/`Option`。普通函数里用不了，得改签名或换 `match`。

5. **`Box<dyn Error>` 丢失类型信息**
   方便但无法对错误种类做 `match` 区分。需要精细处理错误时，定义具体 enum。

6. **`panic` 跨 FFI 边界是未定义行为**
   通过 C 接口暴露的函数里不要让 panic 逃逸出去，用 `catch_unwind` 兜住。

---

## 附：速查总结

```
没有值      → Option<T>     Some(x) / None
可能失败    → Result<T, E>  Ok(x) / Err(e)
是 bug      → panic!        崩溃

传播错误    → ?             Err/None 时提前 return（自动 From 转换）
取值兜底    → unwrap_or / unwrap_or_else / unwrap_or_default
变换        → map / map_err / and_then / or_else
互转        → ok_or / ok()

自定义错误  → 实现 Debug + Display + Error，配 From 让 ? 自动转换
库          → thiserror（精确错误，可被 match 区分）
应用        → anyhow（方便兜底，带 .context() 上下文）

心法：库返回 Result 让调用方决定；只有"这是 bug"才 panic。
     生产代码慎用 unwrap，多用 ? 和 expect("原因")。
```
