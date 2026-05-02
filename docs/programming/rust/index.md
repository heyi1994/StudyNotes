# Rust

Rust 是一门系统级编程语言，专注于内存安全、并发安全和高性能。它在编译期通过所有权（Ownership）系统消除内存错误和数据竞争，无需垃圾回收器，性能与 C/C++ 相当，是 WebAssembly、嵌入式、操作系统、高性能服务等场景的首选语言。

**核心特性：**

| 特性 | 说明 |
|------|------|
| **所有权（Ownership）** | 每个值有且只有一个所有者，离开作用域自动释放，无需 GC |
| **借用（Borrowing）** | 通过引用访问数据，编译期检查引用有效性，防止悬垂指针 |
| **生命周期（Lifetime）** | 编译器追踪引用的存活范围，确保引用永不比数据活得更长 |
| **零成本抽象** | 泛型、trait、迭代器等高级特性编译后无运行时开销 |
| **无数据竞争** | 借用规则保证同一时刻只有一个可变引用或多个不可变引用 |
| **模式匹配** | 穷举性 match 表达式，配合 enum 处理所有情况 |
| **Cargo** | 官方包管理器 + 构建系统，集成测试、文档、发布 |

---

## 安装

```shell
# 通过 rustup 安装（官方推荐）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 安装后刷新环境
source $HOME/.cargo/env

# 验证安装
rustc --version
cargo --version
rustup --version

# 更新到最新稳定版
rustup update

# 安装 nightly（部分实验特性需要）
rustup toolchain install nightly
rustup default nightly        # 切换为默认
rustup default stable         # 切回稳定版

# 添加编译目标（交叉编译）
rustup target add wasm32-unknown-unknown      # WebAssembly
rustup target add aarch64-unknown-linux-gnu   # ARM64 Linux
```

### 推荐工具

```shell
# rust-analyzer（LSP，IDE 智能提示）
rustup component add rust-analyzer

# rustfmt（代码格式化）
rustup component add rustfmt
cargo fmt

# clippy（代码检查/lint）
rustup component add clippy
cargo clippy

# cargo-watch（文件变化自动重编译）
cargo install cargo-watch
cargo watch -x run

# cargo-expand（展开宏）
cargo install cargo-expand
cargo expand

# cargo-audit（安全漏洞检查）
cargo install cargo-audit
cargo audit
```

---

## Cargo 项目管理

```shell
# 创建新项目
cargo new hello_world          # 二进制项目（bin）
cargo new my_lib --lib         # 库项目（lib）

# 项目结构
# hello_world/
# ├── Cargo.toml    # 包元数据 + 依赖声明
# ├── Cargo.lock    # 依赖版本锁文件（bin 项目提交，lib 项目不提交）
# └── src/
#     └── main.rs   # 入口（bin）/ lib.rs（lib）

# 构建与运行
cargo build                    # Debug 构建（target/debug/）
cargo build --release          # Release 构建（target/release/），开启优化
cargo run                      # 构建并运行
cargo run -- arg1 arg2         # 传递命令行参数
cargo check                    # 只检查语法，不生成产物（比 build 快）

# 测试
cargo test                     # 运行所有测试
cargo test test_name           # 运行名称包含关键字的测试
cargo test -- --nocapture      # 显示 println! 输出

# 文档
cargo doc --open               # 生成并打开文档

# 依赖管理
cargo add serde                # 添加依赖
cargo add serde --features derive
cargo add tokio --features full
cargo remove serde             # 移除依赖
cargo update                   # 更新所有依赖到兼容的最新版
```

### Cargo.toml

```toml
[package]
name    = "my_app"
version = "0.1.0"
edition = "2021"               # Rust Edition，目前最新是 2021

[dependencies]
serde       = { version = "1", features = ["derive"] }
serde_json  = "1"
tokio       = { version = "1", features = ["full"] }
anyhow      = "1"
thiserror   = "1"
clap        = { version = "4", features = ["derive"] }
reqwest     = { version = "0.12", features = ["json"] }

[dev-dependencies]             # 只在测试时使用
mockall = "0.13"

[build-dependencies]           # 构建脚本依赖
cc = "1"

[profile.release]
opt-level     = 3              # 最高优化
lto           = true           # 链接时优化
codegen-units = 1              # 减少并行代码生成（更慢但更小）
strip         = true           # 去除符号表

[profile.dev]
opt-level = 0                  # 不优化，加快编译速度
```

---

## 基础语法

### 变量与数据类型

```rust
fn main() {
    // 变量默认不可变
    let x = 5;
    // x = 6; // 编译错误

    // mut 使变量可变
    let mut y = 5;
    y = 6;

    // 常量（必须标注类型，编译期求值）
    const MAX_POINTS: u32 = 100_000;

    // 遮蔽（Shadowing）：同名新变量覆盖旧变量
    let x = x + 1;     // x = 6
    let x = x * 2;     // x = 12
    let x = "hello";   // 可以改变类型

    // 解构赋值
    let (a, b, c) = (1, 2, 3);
    let [first, .., last] = [1, 2, 3, 4, 5];
}
```

**标量类型：**

```rust
// 整数
let a: i8   = -128;          // 有符号 8 位
let b: u8   = 255;           // 无符号 8 位
let c: i32  = 2_147_483_647; // 有符号 32 位（默认整数类型）
let d: i64  = 9_223_372_036_854_775_807;
let e: u64  = 18_446_744_073_709_551_615;
let f: isize = -1;           // 指针大小（平台相关）
let g: usize = 42;           // 索引/长度常用

// 进制字面量
let hex   = 0xFF;
let octal = 0o77;
let bin   = 0b1111_0000;
let byte  = b'A';            // u8 字节

// 浮点数
let h: f32 = 3.14;           // 32 位浮点
let i: f64 = 2.718_281_828;  // 64 位浮点（默认）

// 布尔
let j: bool = true;

// 字符（Unicode 标量，4 字节）
let k: char = '中';
let l: char = '😀';
```

**复合类型：**

```rust
// 元组（固定长度，各元素类型可不同）
let tup: (i32, f64, bool) = (500, 6.4, true);
let (x, y, z) = tup;            // 解构
println!("{}", tup.0);          // 索引访问

// 数组（固定长度，元素类型相同）
let arr: [i32; 5] = [1, 2, 3, 4, 5];
let zeros = [0; 5];             // [0, 0, 0, 0, 0]
println!("{}", arr[0]);

// 切片（对连续内存的引用，不拥有数据）
let slice: &[i32] = &arr[1..3]; // &[2, 3]
```

### 函数

```rust
// 基本函数
fn add(x: i32, y: i32) -> i32 {
    x + y       // 最后一个表达式作为返回值（不加分号）
}

// 多返回值（元组）
fn min_max(arr: &[i32]) -> (i32, i32) {
    let min = *arr.iter().min().unwrap();
    let max = *arr.iter().max().unwrap();
    (min, max)
}

// 发散函数（永不返回）
fn crash(msg: &str) -> ! {
    panic!("{}", msg);
}

// 闭包（匿名函数，捕获环境）
let factor = 3;
let multiply = |x: i32| x * factor;   // 捕获 factor
println!("{}", multiply(5));           // 15

// 闭包作为参数
fn apply<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
    f(x)
}

// 高阶函数
let numbers = vec![1, 2, 3, 4, 5];
let sum: i32 = numbers.iter()
    .filter(|&&x| x % 2 == 0)
    .map(|&x| x * x)
    .sum();  // 4 + 16 = 20
```

### 控制流

```rust
// if 表达式（有返回值）
let number = 7;
let description = if number % 2 == 0 { "偶数" } else { "奇数" };

// loop（无限循环，可返回值）
let mut counter = 0;
let result = loop {
    counter += 1;
    if counter == 10 {
        break counter * 2;  // break 携带返回值
    }
};

// while
while counter < 10 {
    counter += 1;
}

// for（迭代器）
for i in 0..5 {        // 0, 1, 2, 3, 4
    println!("{i}");
}
for i in 0..=5 {       // 0, 1, 2, 3, 4, 5（包含右边界）
    println!("{i}");
}
for (i, val) in arr.iter().enumerate() {
    println!("{i}: {val}");
}

// match 表达式（穷举匹配）
let x = 5;
match x {
    1 => println!("一"),
    2 | 3 => println!("二或三"),
    4..=6 => println!("四到六"),
    n if n > 6 => println!("大于六：{n}"),
    _ => println!("其他"),
}

// if let（匹配单个模式，简化 match）
if let Some(value) = some_option {
    println!("{value}");
}

// while let
while let Some(top) = stack.pop() {
    println!("{top}");
}
```

---

## 所有权系统

Rust 最核心的特性，通过三条规则在编译期管理内存：

1. 每个值有且只有一个所有者（变量）
2. 所有者离开作用域，值被自动释放（调用 `drop`）
3. 任意时刻，要么只有一个可变引用，要么有任意多个不可变引用

### 移动（Move）

```rust
// 堆上数据：赋值 = 移动所有权
let s1 = String::from("hello");
let s2 = s1;          // s1 的所有权移动到 s2
// println!("{s1}"); // 编译错误：s1 已失效

// 栈上数据（实现了 Copy trait）：赋值 = 拷贝
let x = 5;
let y = x;            // x 和 y 都有效
println!("{x} {y}"); // 正常

// 实现了 Copy 的类型：所有整数、浮点、bool、char、元组（元素都是 Copy）

// 函数调用也是移动
fn takes_ownership(s: String) { /* s 被移动进来 */ }
let s = String::from("hello");
takes_ownership(s);
// println!("{s}"); // 编译错误

// 函数返回可以把所有权移出
fn gives_ownership() -> String {
    String::from("hello")  // 移动到调用方
}
```

### 借用（Borrowing）

```rust
// 不可变引用（&T）：可以同时存在多个
let s = String::from("hello");
let len = calculate_length(&s);  // 借用，不移动
println!("{s}");                  // s 仍然有效

fn calculate_length(s: &String) -> usize {
    s.len()
}

// 可变引用（&mut T）：同一时刻只能有一个
let mut s = String::from("hello");
let r = &mut s;
r.push_str(", world");
// let r2 = &mut s; // 编译错误：已有一个可变引用

// 不可变引用和可变引用不能同时存在
let mut s = String::from("hello");
let r1 = &s;
let r2 = &s;
// let r3 = &mut s; // 编译错误
println!("{r1} {r2}");
// r1 r2 最后一次使用后，可变引用可以创建
let r3 = &mut s;   // ✅
```

### 生命周期（Lifetime）

```rust
// 生命周期标注：告诉编译器引用间的关系
// 'a 是生命周期参数名（约定用小写字母）
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
    // 返回引用的生命周期 ≤ x 和 y 中较短的那个
}

// 结构体中的引用必须标注生命周期
struct Excerpt<'a> {
    part: &'a str,   // Excerpt 不能比 part 引用的数据活得更长
}

// 静态生命周期：整个程序运行期间有效
let s: &'static str = "I have a static lifetime.";
```

---

## 字符串

Rust 有两种字符串类型，初学者容易混淆：

| 类型 | 存储 | 可变性 | 说明 |
|------|------|--------|------|
| `&str` | 栈（指针+长度）| 不可变 | 字符串切片，借用的视图 |
| `String` | 堆 | 可变 | 拥有所有权的字符串 |

```rust
// &str：字符串字面量，编译期确定
let s1: &str = "hello";

// String：运行时创建
let s2 = String::from("hello");
let s3 = "hello".to_string();
let s4 = "hello".to_owned();

// 字符串拼接
let s5 = s2 + &s3;           // s2 所有权被移动，s3 被借用
let s6 = format!("{s1} world"); // 不移动所有权

// 常用操作
let s = String::from("Hello, 世界！");
println!("{}", s.len());               // 字节长度（非字符数！）
println!("{}", s.chars().count());     // 字符数（正确方式）
println!("{}", s.contains("世界"));    // true
println!("{}", s.replace("世界", "World"));
println!("{}", s.to_uppercase());
println!("{}", s.trim());

// 切片（按字节索引，中文字符占 3 字节，需小心）
let hello = &s[0..5];   // "Hello"

// 按字符遍历（正确方式）
for c in s.chars() {
    print!("{c} ");
}

// 按字节遍历
for b in s.bytes() {
    print!("{b} ");
}

// 字符串转数字
let n: i32 = "42".parse().unwrap();
let n: i32 = "42".parse::<i32>().unwrap();

// 数字转字符串
let s = 42.to_string();
let s = format!("{}", 42);

// 分割与收集
let csv = "a,b,c,d";
let parts: Vec<&str> = csv.split(',').collect();
let joined = parts.join(" | ");   // "a | b | c | d"
```

---

## 集合类型

### Vec（动态数组）

```rust
// 创建
let mut v: Vec<i32> = Vec::new();
let mut v = vec![1, 2, 3, 4, 5];

// 增删
v.push(6);
v.pop();                  // 返回 Option<i32>
v.insert(2, 99);          // 在索引 2 处插入
v.remove(2);              // 删除索引 2 的元素
v.retain(|&x| x % 2 == 0); // 只保留偶数

// 访问
let third = v[2];             // 越界会 panic
let third = v.get(2);         // 返回 Option<&i32>，推荐
let first = v.first();        // Option<&i32>
let last  = v.last();         // Option<&i32>

// 切片
let slice = &v[1..3];         // &[i32]

// 迭代
for val in &v { }             // 不可变迭代
for val in &mut v { *val *= 2; } // 可变迭代
for val in v { }              // 消费迭代（v 移动）

// 常用方法
v.len()
v.is_empty()
v.contains(&3)
v.sort()                      // 原地排序
v.sort_by(|a, b| b.cmp(a))   // 逆序
v.sort_by_key(|x| x.abs())   // 按绝对值排序
v.dedup()                     // 去除连续重复元素（排序后使用效果好）
v.reverse()
v.extend([7, 8, 9])
v.truncate(3)                 // 只保留前 3 个
v.clear()

// 函数式操作
let doubled: Vec<i32> = v.iter().map(|&x| x * 2).collect();
let evens: Vec<&i32> = v.iter().filter(|&&x| x % 2 == 0).collect();
let sum: i32 = v.iter().sum();
let product: i32 = v.iter().product();
let max = v.iter().max();
let min = v.iter().min();

// 枚举（同时获取索引和值）
let indexed: Vec<(usize, &i32)> = v.iter().enumerate().collect();

// 扁平化
let nested = vec![vec![1, 2], vec![3, 4]];
let flat: Vec<i32> = nested.into_iter().flatten().collect();
```

### HashMap

```rust
use std::collections::HashMap;

// 创建
let mut scores: HashMap<String, i32> = HashMap::new();

// 插入
scores.insert(String::from("Alice"), 100);
scores.insert(String::from("Bob"), 80);

// 从迭代器构建
let map: HashMap<&str, i32> = vec![("a", 1), ("b", 2)].into_iter().collect();

// 访问
let score = scores.get("Alice");      // Option<&i32>
let score = scores["Alice"];          // &i32，键不存在会 panic

// 条件插入（键不存在时才插入）
scores.entry(String::from("Charlie")).or_insert(60);

// 基于旧值更新
let text = "hello world hello rust";
let mut word_count: HashMap<&str, i32> = HashMap::new();
for word in text.split_whitespace() {
    let count = word_count.entry(word).or_insert(0);
    *count += 1;
}

// 删除
scores.remove("Bob");

// 遍历（无序）
for (key, value) in &scores {
    println!("{key}: {value}");
}

// 判断存在
scores.contains_key("Alice");
scores.len()
scores.is_empty()
```

### HashSet / BTreeMap / BTreeSet

```rust
use std::collections::{HashSet, BTreeMap, BTreeSet};

// HashSet：无序集合，元素唯一
let mut set: HashSet<i32> = HashSet::new();
set.insert(1);
set.insert(2);
set.insert(1);       // 重复不插入
set.contains(&1);    // true

let a: HashSet<i32> = [1, 2, 3].into();
let b: HashSet<i32> = [2, 3, 4].into();
let union: HashSet<_>        = a.union(&b).collect();         // 并集
let intersection: HashSet<_> = a.intersection(&b).collect();  // 交集
let difference: HashSet<_>   = a.difference(&b).collect();    // 差集

// BTreeMap：有序 HashMap（按 key 排序）
let mut map: BTreeMap<&str, i32> = BTreeMap::new();
map.insert("b", 2);
map.insert("a", 1);
// 遍历时按 key 字母序

// VecDeque：双端队列
use std::collections::VecDeque;
let mut deque: VecDeque<i32> = VecDeque::new();
deque.push_back(1);
deque.push_front(0);
deque.pop_front();
deque.pop_back();
```

---

## 枚举与模式匹配

```rust
// 基本枚举
#[derive(Debug)]
enum Direction {
    North,
    South,
    East,
    West,
}

// 带数据的枚举（每个变体可携带不同类型的数据）
#[derive(Debug)]
enum Message {
    Quit,                        // 无数据
    Move { x: i32, y: i32 },    // 命名字段
    Write(String),               // 单个值
    ChangeColor(u8, u8, u8),     // 元组
}

// 使用
let msg = Message::Move { x: 10, y: 20 };
let msg = Message::Write(String::from("hello"));

// match 解构
match msg {
    Message::Quit => println!("退出"),
    Message::Move { x, y } => println!("移动到 ({x}, {y})"),
    Message::Write(text) => println!("写入：{text}"),
    Message::ChangeColor(r, g, b) => println!("颜色：({r}, {g}, {b})"),
}

// Option<T>：避免 null 的标准方式
let some_value: Option<i32> = Some(42);
let no_value: Option<i32> = None;

let doubled = match some_value {
    Some(v) => Some(v * 2),
    None    => None,
};
// 等价于：
let doubled = some_value.map(|v| v * 2);

// 常用 Option 方法
some_value.unwrap()               // 取值，None 时 panic
some_value.unwrap_or(0)           // 取值，None 时用默认值
some_value.unwrap_or_else(|| 0)   // 取值，None 时执行闭包
some_value.map(|v| v * 2)         // 转换内部值
some_value.and_then(|v| if v > 0 { Some(v) } else { None })
some_value.or(Some(0))            // None 时替换
some_value.is_some()
some_value.is_none()
some_value?                       // 在返回 Option 的函数中：None 则提前返回

// Result<T, E>：错误处理的标准方式
let result: Result<i32, String> = Ok(42);
let error:  Result<i32, String> = Err(String::from("出错了"));

result.unwrap()
result.unwrap_or(0)
result.map(|v| v * 2)
result.map_err(|e| format!("错误：{e}"))
result.is_ok()
result.is_err()
result?                           // 在返回 Result 的函数中：Err 则提前返回
```

---

## 结构体

```rust
// 定义
#[derive(Debug, Clone, PartialEq)]
struct User {
    username: String,
    email:    String,
    age:      u32,
    active:   bool,
}

// 创建实例
let user = User {
    username: String::from("alice"),
    email:    String::from("alice@example.com"),
    age:      30,
    active:   true,
};

// 字段访问
println!("{}", user.username);

// 结构体更新语法（复用字段）
let user2 = User {
    email: String::from("bob@example.com"),
    ..user    // user 的其他字段被移动（username 是 String，会被移动）
};

// 元组结构体
struct Point(f64, f64);
struct Color(u8, u8, u8);
let p = Point(1.0, 2.0);
println!("{}", p.0);

// 单元结构体（不含字段，常用于实现 trait）
struct AlwaysEqual;

// 方法
impl User {
    // 关联函数（类似静态方法，用 :: 调用）
    fn new(username: &str, email: &str, age: u32) -> Self {
        User {
            username: username.to_string(),
            email:    email.to_string(),
            age,
            active: true,
        }
    }

    // 方法（第一个参数是 self）
    fn greet(&self) -> String {
        format!("Hi, I'm {}!", self.username)
    }

    fn deactivate(&mut self) {
        self.active = false;
    }

    fn into_username(self) -> String {  // 消费 self
        self.username
    }
}

let mut u = User::new("alice", "alice@example.com", 30);
println!("{}", u.greet());
u.deactivate();
```

---

## Trait（特征）

Trait 类似其他语言的接口，定义共享行为。

```rust
// 定义 trait
trait Animal {
    // 必须实现的方法
    fn name(&self) -> &str;
    fn sound(&self) -> &str;

    // 默认实现（可被覆盖）
    fn describe(&self) -> String {
        format!("{} 叫声是：{}", self.name(), self.sound())
    }
}

struct Dog { name: String }
struct Cat { name: String }

impl Animal for Dog {
    fn name(&self) -> &str { &self.name }
    fn sound(&self) -> &str { "汪汪" }
}

impl Animal for Cat {
    fn name(&self) -> &str { &self.name }
    fn sound(&self) -> &str { "喵喵" }
    fn describe(&self) -> String {   // 覆盖默认实现
        format!("猫咪 {} 优雅地叫：{}", self.name(), self.sound())
    }
}

// Trait 作为参数（impl Trait 语法）
fn make_sound(animal: &impl Animal) {
    println!("{}", animal.describe());
}

// Trait bound（泛型约束语法，更灵活）
fn make_sound<T: Animal>(animal: &T) {
    println!("{}", animal.describe());
}

// 多个 Trait bound
fn process<T: Animal + std::fmt::Debug>(item: &T) { }
// 等价写法（where 子句，更清晰）
fn process<T>(item: &T) where T: Animal + std::fmt::Debug { }

// 返回实现了 Trait 的类型
fn make_animal(is_dog: bool) -> impl Animal {
    if is_dog { Dog { name: "旺财".to_string() } }
    else       { Cat { name: "咪咪".to_string() } }
}

// dyn Trait（动态分发，运行时多态）
fn make_sounds(animals: &[Box<dyn Animal>]) {
    for animal in animals {
        println!("{}", animal.describe());
    }
}
let animals: Vec<Box<dyn Animal>> = vec![
    Box::new(Dog { name: "旺财".to_string() }),
    Box::new(Cat { name: "咪咪".to_string() }),
];
```

### 常用标准库 Trait

```rust
// Display：格式化输出（用于 {}）
use std::fmt;
impl fmt::Display for User {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "User({})", self.username)
    }
}

// From / Into：类型转换
impl From<&str> for User {
    fn from(s: &str) -> Self {
        User::new(s, "", 0)
    }
}
let u: User = "alice".into();   // Into 自动由 From 实现

// Clone / Copy
#[derive(Clone, Copy)]    // derive 宏自动实现
struct Point { x: f64, y: f64 }

// Iterator：自定义迭代器
struct Counter { count: u32 }
impl Iterator for Counter {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        self.count += 1;
        if self.count <= 5 { Some(self.count) } else { None }
    }
}
```

---

## 泛型

```rust
// 泛型函数
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest { largest = item; }
    }
    largest
}

// 泛型结构体
struct Pair<T> {
    first:  T,
    second: T,
}

impl<T: std::fmt::Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.first >= self.second {
            println!("最大值是 {}", self.first);
        } else {
            println!("最大值是 {}", self.second);
        }
    }
}

// 泛型枚举（标准库的 Option 和 Result 就是这样实现的）
enum MyOption<T> {
    Some(T),
    None,
}

// 关联类型（比泛型更简洁）
trait Container {
    type Item;              // 关联类型
    fn get(&self) -> &Self::Item;
}
```

---

## 错误处理

```rust
use std::fs;
use std::io;
use std::num::ParseIntError;

// ? 运算符：自动传播错误
fn read_number_from_file(path: &str) -> Result<i32, Box<dyn std::error::Error>> {
    let content = fs::read_to_string(path)?;  // io::Error 自动转换
    let n: i32 = content.trim().parse()?;     // ParseIntError 自动转换
    Ok(n)
}

// 自定义错误类型（使用 thiserror 库）
use thiserror::Error;

#[derive(Debug, Error)]
enum AppError {
    #[error("IO 错误：{0}")]
    Io(#[from] io::Error),

    #[error("解析错误：{0}")]
    Parse(#[from] ParseIntError),

    #[error("业务错误：{message}")]
    Business { message: String },

    #[error("找不到 ID 为 {id} 的用户")]
    UserNotFound { id: u64 },
}

fn process(path: &str) -> Result<i32, AppError> {
    let content = fs::read_to_string(path)?;  // io::Error 通过 #[from] 自动转换
    let n: i32 = content.trim().parse()?;     // ParseIntError 自动转换
    if n < 0 {
        return Err(AppError::Business { message: "数字不能为负".to_string() });
    }
    Ok(n * 2)
}

// anyhow：快速错误处理（适合应用层，不适合库）
use anyhow::{anyhow, bail, Context, Result};

fn process_anyhow(path: &str) -> Result<i32> {
    let content = fs::read_to_string(path)
        .with_context(|| format!("无法读取文件：{path}"))?;
    let n: i32 = content.trim().parse()
        .context("文件内容不是有效数字")?;
    if n < 0 {
        bail!("数字不能为负：{n}");  // 等价于 return Err(anyhow!(...))
    }
    Ok(n)
}
```

---

## 迭代器与函数式编程

```rust
// 迭代器是惰性的，只有在消费时才执行
let v = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

// 链式操作
let result: Vec<String> = v.iter()
    .filter(|&&x| x % 2 == 0)       // 过滤偶数
    .map(|&x| x * x)                 // 平方
    .take(3)                         // 只取前 3 个
    .map(|x| x.to_string())          // 转 String
    .collect();                      // 收集为 Vec
// result = ["4", "16", "36"]

// 常用迭代器方法
v.iter().sum::<i32>()                // 求和
v.iter().product::<i32>()           // 乘积
v.iter().count()                     // 计数
v.iter().max()                       // 最大值 Option
v.iter().min()                       // 最小值 Option
v.iter().any(|&x| x > 5)            // 任意满足
v.iter().all(|&x| x > 0)            // 全部满足
v.iter().find(|&&x| x > 5)          // 找到第一个满足的元素
v.iter().position(|&x| x == 5)      // 找到位置
v.iter().enumerate()                 // 添加索引
v.iter().zip([1, 2, 3].iter())       // 配对
v.iter().flat_map(|&x| [x, x * 10]) // 映射并展平
v.iter().fold(0, |acc, &x| acc + x) // 归约
v.iter().scan(0, |state, &x| {       // 有状态映射
    *state += x;
    Some(*state)
})
v.iter().peekable()                  // 可以窥视下一个元素
v.iter().skip(2).take(3)            // 跳过再取
v.iter().step_by(2)                  // 步长迭代
v.iter().chain(v.iter())             // 连接两个迭代器
v.iter().rev()                       // 反转（需 DoubleEndedIterator）

// 迭代器类型
v.iter()          // -> Iterator<Item = &i32>（借用）
v.iter_mut()      // -> Iterator<Item = &mut i32>（可变借用）
v.into_iter()     // -> Iterator<Item = i32>（消费，移动所有权）

// collect 到不同集合
use std::collections::{HashMap, HashSet};
let set: HashSet<i32> = v.iter().cloned().collect();
let map: HashMap<i32, i32> = v.iter().map(|&x| (x, x * x)).collect();
```

---

## 智能指针

```rust
use std::rc::Rc;
use std::cell::RefCell;
use std::sync::{Arc, Mutex};

// Box<T>：堆分配，单一所有权
// 用途：递归类型、trait 对象、大数据转移所有权
let b = Box::new(5);
println!("{}", *b);

// 递归类型（编译器无法推断大小，必须用 Box）
enum List {
    Cons(i32, Box<List>),
    Nil,
}

// Rc<T>：引用计数，单线程多所有权
let a = Rc::new(5);
let b = Rc::clone(&a);    // 引用计数 +1，不是深拷贝
println!("引用计数：{}", Rc::strong_count(&a));  // 2
// 两个变量离开作用域后自动释放

// RefCell<T>：运行时借用检查（内部可变性）
// 用途：在不可变引用场景中实现可变性
let data = RefCell::new(vec![1, 2, 3]);
data.borrow().len();           // 不可变借用
data.borrow_mut().push(4);    // 可变借用（运行时检查，违规则 panic）

// Rc<RefCell<T>>：多所有者 + 内部可变性（单线程）
let shared = Rc::new(RefCell::new(vec![1, 2]));
let clone1 = Rc::clone(&shared);
let clone2 = Rc::clone(&shared);
clone1.borrow_mut().push(3);
println!("{:?}", clone2.borrow()); // [1, 2, 3]

// Arc<T>：原子引用计数（多线程安全版 Rc）
let arc = Arc::new(5);
let arc2 = Arc::clone(&arc);
std::thread::spawn(move || println!("{}", arc2));

// Mutex<T>：互斥锁（多线程可变共享）
let counter = Arc::new(Mutex::new(0));
let c = Arc::clone(&counter);
std::thread::spawn(move || {
    let mut num = c.lock().unwrap();  // 获取锁
    *num += 1;
});                                   // 离开作用域自动释放锁

// RwLock<T>：读写锁（允许多读一写）
use std::sync::RwLock;
let lock = RwLock::new(5);
let r1 = lock.read().unwrap();
let r2 = lock.read().unwrap();    // 多个读锁同时存在
// let w = lock.write().unwrap(); // 有读锁时不能写
```

---

## 并发

### 线程

```rust
use std::thread;
use std::sync::{Arc, Mutex};
use std::time::Duration;

// 创建线程
let handle = thread::spawn(|| {
    for i in 1..=5 {
        println!("子线程：{i}");
        thread::sleep(Duration::from_millis(100));
    }
});

// 等待线程结束
handle.join().unwrap();

// move 闭包：将所有权移入线程
let v = vec![1, 2, 3];
let handle = thread::spawn(move || {
    println!("{:?}", v);  // v 被移入线程
});
handle.join().unwrap();

// 线程间共享数据
let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let c = Arc::clone(&counter);
    let h = thread::spawn(move || {
        let mut num = c.lock().unwrap();
        *num += 1;
    });
    handles.push(h);
}

for h in handles { h.join().unwrap(); }
println!("最终计数：{}", *counter.lock().unwrap());
```

### 消息传递（Channel）

```rust
use std::sync::mpsc;  // multiple producer, single consumer

// 单发送者
let (tx, rx) = mpsc::channel();

thread::spawn(move || {
    tx.send(String::from("hello")).unwrap();
    tx.send(String::from("world")).unwrap();
    // tx 离开作用域，channel 关闭
});

// 接收
let msg = rx.recv().unwrap();  // 阻塞等待
println!("{msg}");

// 遍历接收所有消息（直到 channel 关闭）
for msg in rx {
    println!("{msg}");
}

// 多发送者
let (tx, rx) = mpsc::channel::<String>();
let tx2 = tx.clone();

thread::spawn(move || tx.send("from tx".to_string()).unwrap());
thread::spawn(move || tx2.send("from tx2".to_string()).unwrap());

for msg in rx { println!("{msg}"); }
```

---

## 异步编程（Tokio）

Rust 的异步通过 `async/await` + 运行时（Runtime）实现，最主流的运行时是 Tokio。

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

### 基础用法

```rust
use tokio::time::{sleep, Duration};

// async fn 返回 Future，需要 .await 执行
async fn say_hello() {
    println!("Hello");
    sleep(Duration::from_millis(100)).await;
    println!("World");
}

// #[tokio::main] 启动运行时
#[tokio::main]
async fn main() {
    say_hello().await;
}

// 并发执行多个任务
#[tokio::main]
async fn main() {
    // tokio::join!：并发等待多个 Future
    let (r1, r2) = tokio::join!(
        async_task_1(),
        async_task_2(),
    );

    // tokio::spawn：后台任务（不等待）
    let handle = tokio::spawn(async {
        sleep(Duration::from_secs(1)).await;
        42
    });
    let result = handle.await.unwrap();

    // select!：等待最先完成的 Future
    tokio::select! {
        val = async_task_1() => println!("task1 先完成：{val}"),
        val = async_task_2() => println!("task2 先完成：{val}"),
    }
}
```

### 异步网络请求（reqwest）

```rust
use reqwest;
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize)]
struct User {
    id:    u32,
    name:  String,
    email: String,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let client = reqwest::Client::new();

    // GET 请求
    let user: User = client
        .get("https://jsonplaceholder.typicode.com/users/1")
        .send()
        .await?
        .error_for_status()?       // 非 2xx 返回错误
        .json()
        .await?;

    println!("{user:?}");

    // POST 请求
    #[derive(Serialize)]
    struct NewUser { name: String, email: String }

    let new_user = NewUser {
        name:  "Alice".to_string(),
        email: "alice@example.com".to_string(),
    };

    let response = client
        .post("https://jsonplaceholder.typicode.com/users")
        .json(&new_user)
        .header("Authorization", "Bearer token")
        .send()
        .await?
        .error_for_status()?;

    println!("状态码：{}", response.status());
    Ok(())
}
```

### 异步文件 I/O

```rust
use tokio::fs;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // 读文件
    let content = fs::read_to_string("input.txt").await?;

    // 写文件
    fs::write("output.txt", "Hello, Async!").await?;

    // 流式读取
    let mut file = fs::File::open("large.txt").await?;
    let mut buffer = Vec::new();
    file.read_to_end(&mut buffer).await?;

    // 流式写入
    let mut file = fs::File::create("result.txt").await?;
    file.write_all(b"Hello").await?;
    file.flush().await?;

    Ok(())
}
```

---

## 宏

```rust
// 常用内置宏
println!("Hello, {name}!");          // 打印并换行
print!("无换行");
eprintln!("错误输出");               // 输出到 stderr
format!("{} + {} = {}", 1, 2, 3)     // 格式化为 String
vec![1, 2, 3]                        // 创建 Vec
panic!("不可恢复的错误")              // 程序崩溃
assert!(1 + 1 == 2)                  // 断言
assert_eq!(1 + 1, 2)                 // 相等断言（失败时显示两个值）
assert_ne!(1, 2)                     // 不等断言
todo!()                              // 标记未实现（运行时 panic）
unimplemented!()                     // 标记不支持（运行时 panic）
unreachable!()                       // 标记不可达代码
dbg!(&value)                        // 调试输出（含文件名和行号）
include_str!("file.txt")            // 编译期嵌入文件内容为 &str
include_bytes!("file.bin")          // 编译期嵌入文件内容为 &[u8]
env!("CARGO_PKG_VERSION")           // 编译期读取环境变量

// derive 宏：自动实现 trait
#[derive(
    Debug,          // {:?} 格式化
    Clone,          // .clone() 方法
    Copy,           // 栈上复制语义
    PartialEq,      // == 运算符
    Eq,             // 完全相等（配合 PartialEq）
    PartialOrd,     // < > <= >= 运算符
    Ord,            // 完全排序
    Hash,           // 用于 HashMap key
    Default,        // Default::default()
    Serialize,      // serde JSON 序列化
    Deserialize,    // serde JSON 反序列化
)]
struct MyStruct {
    field: i32,
}
```

---

## 序列化（serde）

```toml
[dependencies]
serde      = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use serde::{Deserialize, Serialize};
use serde_json;

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id:       u32,
    name:     String,
    email:    String,
    #[serde(skip_serializing_if = "Option::is_none")]
    avatar:   Option<String>,         // None 时不输出字段
    #[serde(rename = "created_at")]
    created:  String,                 // JSON key 重命名
    #[serde(skip)]
    password: String,                 // 序列化/反序列化时跳过
}

// 序列化为 JSON
let user = User {
    id: 1, name: "Alice".to_string(), email: "a@b.com".to_string(),
    avatar: None, created: "2024-01-01".to_string(), password: "secret".to_string(),
};
let json = serde_json::to_string(&user)?;           // 紧凑格式
let json = serde_json::to_string_pretty(&user)?;    // 美化格式

// 反序列化
let user: User = serde_json::from_str(&json)?;

// 动态 JSON（Value 类型）
use serde_json::{json, Value};
let v: Value = json!({
    "name": "Alice",
    "scores": [100, 95, 88],
    "active": true
});
println!("{}", v["name"]);           // "Alice"
println!("{}", v["scores"][0]);      // 100

// 解析为动态 Value
let v: Value = serde_json::from_str(r#"{"x": 1}"#)?;
let x = v["x"].as_i64().unwrap();
```

---

## 测试

```rust
// src/lib.rs 或 src/main.rs
pub fn add(a: i32, b: i32) -> i32 { a + b }
pub fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 { Err("除数不能为零".to_string()) }
    else         { Ok(a / b) }
}

#[cfg(test)]                  // 只在 cargo test 时编译
mod tests {
    use super::*;             // 引入父模块的所有内容

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
        assert_ne!(add(2, 3), 6);
    }

    #[test]
    fn test_divide_ok() {
        let result = divide(10.0, 2.0).unwrap();
        assert!((result - 5.0).abs() < f64::EPSILON);
    }

    #[test]
    fn test_divide_by_zero() {
        assert!(divide(10.0, 0.0).is_err());
    }

    #[test]
    #[should_panic(expected = "越界")]   // 预期 panic
    fn test_panic() {
        let v = vec![1, 2, 3];
        v[10];   // 实际上不含"越界"字样，此测试会失败
    }

    #[test]
    #[ignore]   // 跳过（cargo test -- --ignored 可单独运行）
    fn expensive_test() {
        // 耗时测试
    }
}

// 集成测试（tests/ 目录）
// tests/integration_test.rs
use my_crate::add;
#[test]
fn test_add_integration() {
    assert_eq!(add(2, 3), 5);
}
```

```shell
cargo test                          # 运行所有测试
cargo test test_add                 # 运行名称包含 test_add 的测试
cargo test -- --nocapture           # 显示 println! 输出
cargo test -- --test-threads=1      # 单线程运行（避免并发影响）
cargo test -- --ignored             # 运行被 ignore 的测试
```

---

## 常用标准库

```rust
// 文件 I/O
use std::fs;
use std::io::{self, BufRead, Write};

let content = fs::read_to_string("file.txt")?;
fs::write("file.txt", "content")?;
fs::copy("src.txt", "dst.txt")?;
fs::rename("old.txt", "new.txt")?;
fs::remove_file("file.txt")?;
fs::create_dir_all("a/b/c")?;
fs::remove_dir_all("dir")?;

// 逐行读取（大文件推荐）
let file = fs::File::open("file.txt")?;
for line in io::BufReader::new(file).lines() {
    println!("{}", line?);
}

// 路径操作
use std::path::{Path, PathBuf};
let path = Path::new("/home/user/file.txt");
path.exists()
path.is_file()
path.is_dir()
path.extension()              // Some("txt")
path.file_name()              // Some("file.txt")
path.file_stem()              // Some("file")
path.parent()                 // Some("/home/user")
let mut buf = PathBuf::from("/home/user");
buf.push("file.txt");

// 环境变量 & 命令行
use std::env;
let args: Vec<String> = env::args().collect();
let home = env::var("HOME")?;
env::set_var("KEY", "value");
let cwd = env::current_dir()?;

// 进程
use std::process::{Command, exit};
let output = Command::new("ls")
    .args(["-la", "/tmp"])
    .output()?;
println!("{}", String::from_utf8_lossy(&output.stdout));
exit(0);

// 时间
use std::time::{Duration, Instant, SystemTime, UNIX_EPOCH};
let start = Instant::now();
// ... 执行代码 ...
println!("耗时：{:?}", start.elapsed());

let timestamp = SystemTime::now()
    .duration_since(UNIX_EPOCH)
    .unwrap()
    .as_secs();
```

---

## 常用第三方库

| 库 | 用途 |
|------|------|
| `tokio` | 异步运行时（标配） |
| `reqwest` | HTTP 客户端（基于 tokio） |
| `axum` | Web 框架（轻量、高性能，tokio 原生） |
| `actix-web` | Web 框架（高性能） |
| `serde` + `serde_json` | 序列化/反序列化 |
| `anyhow` | 应用层错误处理 |
| `thiserror` | 库层自定义错误类型 |
| `clap` | 命令行参数解析 |
| `tracing` | 结构化日志 + 追踪 |
| `sqlx` | 异步数据库（编译期 SQL 检查） |
| `diesel` | ORM（同步，强类型） |
| `redis` | Redis 客户端 |
| `uuid` | UUID 生成 |
| `chrono` | 时间日期处理 |
| `regex` | 正则表达式 |
| `rayon` | 数据并行（并行迭代器） |
| `crossbeam` | 并发原语（channel、无锁数据结构）|
| `dashmap` | 并发 HashMap |
| `bytes` | 高效字节缓冲区 |
| `flate2` | gzip/zlib 压缩 |
| `base64` | Base64 编解码 |

---

## Axum Web 服务示例

```toml
[dependencies]
axum        = "0.7"
tokio       = { version = "1", features = ["full"] }
serde       = { version = "1", features = ["derive"] }
serde_json  = "1"
tower-http  = { version = "0.5", features = ["cors", "trace"] }
```

```rust
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    response::Json,
    routing::{delete, get, post, put},
    Router,
};
use serde::{Deserialize, Serialize};
use std::sync::{Arc, Mutex};
use std::collections::HashMap;

#[derive(Debug, Clone, Serialize, Deserialize)]
struct User {
    id:    u64,
    name:  String,
    email: String,
}

// 应用状态
type Db = Arc<Mutex<HashMap<u64, User>>>;

#[tokio::main]
async fn main() {
    let db: Db = Arc::new(Mutex::new(HashMap::new()));

    let app = Router::new()
        .route("/users",      get(list_users).post(create_user))
        .route("/users/:id",  get(get_user).put(update_user).delete(delete_user))
        .with_state(db);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("监听 http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}

async fn list_users(State(db): State<Db>) -> Json<Vec<User>> {
    let db = db.lock().unwrap();
    Json(db.values().cloned().collect())
}

async fn get_user(
    State(db): State<Db>,
    Path(id): Path<u64>,
) -> Result<Json<User>, StatusCode> {
    let db = db.lock().unwrap();
    db.get(&id)
        .cloned()
        .map(Json)
        .ok_or(StatusCode::NOT_FOUND)
}

#[derive(Deserialize)]
struct CreateUser { name: String, email: String }

async fn create_user(
    State(db): State<Db>,
    Json(payload): Json<CreateUser>,
) -> (StatusCode, Json<User>) {
    let mut db = db.lock().unwrap();
    let id = db.len() as u64 + 1;
    let user = User { id, name: payload.name, email: payload.email };
    db.insert(id, user.clone());
    (StatusCode::CREATED, Json(user))
}

async fn update_user(
    State(db): State<Db>,
    Path(id): Path<u64>,
    Json(payload): Json<CreateUser>,
) -> Result<Json<User>, StatusCode> {
    let mut db = db.lock().unwrap();
    if let Some(user) = db.get_mut(&id) {
        user.name  = payload.name;
        user.email = payload.email;
        Ok(Json(user.clone()))
    } else {
        Err(StatusCode::NOT_FOUND)
    }
}

async fn delete_user(
    State(db): State<Db>,
    Path(id): Path<u64>,
) -> StatusCode {
    let mut db = db.lock().unwrap();
    if db.remove(&id).is_some() { StatusCode::NO_CONTENT }
    else                         { StatusCode::NOT_FOUND  }
}
```

---

## 最佳实践

1. **拥抱编译器**：Rust 的编译错误信息极为详细，认真阅读错误提示往往直接给出修复建议。遇到借用检查器报错时，优先思考数据的生命周期和所有权归属。

2. **优先使用 `?` 传播错误**：避免 `.unwrap()` 在生产代码中使用，除非你确定该值一定存在（如程序初始化阶段）。

3. **库用 `thiserror`，应用用 `anyhow`**：库应定义具体错误类型方便调用方处理，应用层用 `anyhow` 快速汇聚所有错误。

4. **结构体派生常用 Trait**：养成给数据结构加 `#[derive(Debug, Clone)]` 的习惯，`Debug` 方便调试，`Clone` 减少借用冲突。

5. **多用迭代器链式调用**：比手写循环更安全、更惯用，编译器通常能优化得和循环一样快甚至更快。

6. **`Arc<Mutex<T>>` 是多线程共享状态的标准模式**：单线程用 `Rc<RefCell<T>>`，多线程用 `Arc<Mutex<T>>` 或 `Arc<RwLock<T>>`（读多写少时）。

7. **避免过早优化**：先用 `clone()` 让代码跑通，再根据性能分析决定是否消除克隆。编写正确代码比编写"零克隆"代码更重要。

8. **善用 `clippy`**：`cargo clippy` 会指出大量不惯用写法和潜在问题，是学习 Rust 惯用法的最佳工具之一。

9. **使用 `cargo fmt` 统一格式**：在 CI 中加入 `cargo fmt --check` 保证代码风格一致。

10. **从已有代码学习**：阅读 `std` 标准库源码、`tokio`、`axum` 等优秀库的源码是提升 Rust 水平最快的方式。
