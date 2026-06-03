# std:OnceCell / OnceLock / LazyCell / LazyLock

> 适用版本：
> - `OnceCell` / `OnceLock`：Rust **1.70+** 稳定
> - `LazyCell` / `LazyLock`：Rust **1.80+** 稳定
>
> 在此之前，社区普遍使用 [`once_cell`](https://crates.io/crates/once_cell) 和 [`lazy_static`](https://crates.io/crates/lazy_static) 这两个第三方 crate。如今标准库已内置同等能力，新项目推荐直接用标准库。

---

## 目录

- [一、为什么需要这些类型](#一为什么需要这些类型)
- [二、四种类型总览](#二四种类型总览)
- [三、Cell 后缀 vs Lock 后缀（单线程 vs 多线程）](#三cell-后缀-vs-lock-后缀单线程-vs-多线程)
- [四、Once 前缀 vs Lazy 前缀（手动 vs 自动）](#四once-前缀-vs-lazy-前缀手动-vs-自动)
- [五、OnceCell 详解](#五oncecell-详解)
- [六、OnceLock 详解](#六oncelock-详解)
- [七、LazyCell 详解](#七lazycell-详解)
- [八、LazyLock 详解](#八lazylock-详解)
- [九、选型决策表](#九选型决策表)
- [十、常见陷阱与注意事项](#十常见陷阱与注意事项)
- [十一、与第三方 crate 的对应关系](#十一与第三方-crate-的对应关系)

---

## 一、为什么需要这些类型

Rust 中有两类常见需求：

1. **延迟初始化（lazy initialization）**：一个值的计算开销较大，或依赖运行时信息，希望**第一次用到时才计算**，而不是程序启动就计算。
2. **"只初始化一次"的全局/共享状态**：例如全局配置、全局连接池、编译好的正则表达式等。

普通的 `static` 变量要求初始值是**编译期常量**，无法执行运行时逻辑：

```rust
//  编译错误：static 的初始值必须是常量表达式
static CONFIG: String = String::from("hello");
```

而普通的 `Cell` / `RefCell` / `Mutex` 虽然能存可变值，但无法表达"**只能写一次、写完即只读**"的语义，也不能保证多线程下只初始化一次。

`OnceCell`、`OnceLock`、`LazyCell`、`LazyLock` 就是为解决这些问题而生的。

---

## 二、四种类型总览

可以用一个 **2×2 矩阵** 来理解它们。两个维度：

- **横轴：单线程 vs 多线程**
  - `*Cell`：单线程，无同步开销，**不是 `Sync`**
  - `*Lock`：多线程，内部带同步，**是 `Sync`**，可用于 `static`
- **纵轴：手动初始化 vs 自动初始化**
  - `Once*`：你手动调用 `set` / `get_or_init`，初始化逻辑在"使用点"
  - `Lazy*`：初始化闭包在"声明点"写好，首次解引用时自动触发

|                | 单线程（`Cell`）  | 多线程（`Lock`）   |
|----------------|------------------|-------------------|
| **手动（`Once`）** | `OnceCell<T>`    | `OnceLock<T>`     |
| **自动（`Lazy`）** | `LazyCell<T, F>` | `LazyLock<T, F>`  |

所在模块：

| 类型         | 路径                       |
|-------------|----------------------------|
| `OnceCell`  | `std::cell::OnceCell`      |
| `OnceLock`  | `std::sync::OnceLock`      |
| `LazyCell`  | `std::cell::LazyCell`      |
| `LazyLock`  | `std::sync::LazyLock`      |

> 记忆口诀：`cell` 模块里的都是**单线程**版本；`sync` 模块里的都是**线程安全**版本。

---

## 三、Cell 后缀 vs Lock 后缀（单线程 vs 多线程）

这是这四个类型最核心的区别。

### `OnceCell` / `LazyCell`（单线程）

- 内部**没有任何同步机制**（没有锁、没有原子操作）。
- 性能最好，零额外开销。
- **没有实现 `Sync`**，因此**不能跨线程共享**，也**不能直接用于 `static`**（`static` 要求类型是 `Sync`）。
- 适合：函数内部、结构体字段、线程局部（`thread_local!`）等单线程场景。

### `OnceLock` / `LazyLock`（多线程）

- 内部使用同步原语（原子状态 + 必要时阻塞），保证**并发初始化时只有一个线程执行初始化逻辑**，其余线程等待结果。
- 实现了 `Sync`（在 `T: Sync + Send` 等约束下），**可以用作 `static` 全局变量**，也可以放进 `Arc` 跨线程共享。
- 有轻微同步开销（初始化后通常只是一次原子读取，开销极小）。
- 适合：全局单例、跨线程共享的延迟初始化。

```rust
use std::sync::OnceLock;

//  可以作为全局 static
static GLOBAL: OnceLock<String> = OnceLock::new();

//  OnceCell 不行：
// static GLOBAL: std::cell::OnceCell<String> = OnceCell::new();
//   error: `OnceCell<String>` cannot be shared between threads safely
```

---

## 四、Once 前缀 vs Lazy 前缀（手动 vs 自动）

### `Once*`：手动初始化

- 创建时是**空的**，里面还没有值。
- 你需要在运行时的某个时机**主动**写入：用 `set()` 直接放值，或 `get_or_init(closure)` 在首次访问时初始化。
- 初始化逻辑写在**使用的地方**，灵活——可以在运行到某个条件时才决定用什么值初始化。

```rust
use std::sync::OnceLock;

static LOGGER: OnceLock<String> = OnceLock::new();

fn init_logger(path: String) {
    LOGGER.set(path).expect("logger 已经初始化过了");
}

fn logger() -> &'static String {
    LOGGER.get().expect("logger 尚未初始化")
}
```

### `Lazy*`：自动初始化

- 创建时就把**初始化闭包**一起存进去。
- 第一次通过 `Deref`（解引用）访问内部值时，**自动调用闭包**完成初始化。
- 初始化逻辑写在**声明的地方**，使用处可以像普通引用一样直接用，代码更简洁。
- 本质上 `LazyLock` ≈ `OnceLock` + 一个保存好的初始化闭包。

```rust
use std::sync::LazyLock;

static CONFIG: LazyLock<String> = LazyLock::new(|| {
    // 第一次访问 CONFIG 时才执行
    std::env::var("CONFIG").unwrap_or_else(|_| "default".to_string())
});

fn main() {
    // 直接当作 &String 用，无需手动 init
    println!("{}", *CONFIG);
    println!("{}", CONFIG.len());
}
```

**一句话区分**：
- 不知道初始值、或想在运行时某个时机手动决定 → 用 `Once*`。
- 初始化逻辑固定、希望"首次访问自动算" → 用 `Lazy*`（更常用、更省事）。

---

## 五、OnceCell 详解

`std::cell::OnceCell<T>` —— 单线程、手动、只能写一次的容器。

### 常用 API

```rust
use std::cell::OnceCell;

let cell: OnceCell<u32> = OnceCell::new();

// 1. get：未初始化返回 None
assert_eq!(cell.get(), None);

// 2. set：写入值；若已有值则返回 Err(传入的值)
assert_eq!(cell.set(42), Ok(()));
assert_eq!(cell.set(99), Err(99)); // 已经有值了

// 3. get：现在能拿到
assert_eq!(cell.get(), Some(&42));

// 4. get_or_init：有值则返回，无值则用闭包初始化
let v = cell.get_or_init(|| 100); // 已有 42，闭包不执行
assert_eq!(*v, 42);

// 5. into_inner：消耗 cell，取出内部 Option<T>
assert_eq!(cell.into_inner(), Some(42));
```

| 方法                       | 作用                                                |
|----------------------------|-----------------------------------------------------|
| `new()`                    | 创建空 cell                                          |
| `get(&self)`               | `Option<&T>`，未初始化为 `None`                      |
| `get_mut(&mut self)`       | `Option<&mut T>`，可变借用时拿可变引用               |
| `set(&self, value)`        | `Result<(), T>`，已初始化则返回 `Err(value)`         |
| `get_or_init(&self, f)`    | `&T`，无值时执行 `f` 初始化                          |
| `take(&mut self)`          | `Option<T>`，取出值并重置为空（需要 `&mut`）         |
| `into_inner(self)`         | `Option<T>`，消耗自身                                |

### 典型场景：结构体内部的延迟缓存

```rust
use std::cell::OnceCell;

struct Widget {
    // 某个昂贵的派生值，只在需要时计算一次
    cached_size: OnceCell<usize>,
    data: Vec<u8>,
}

impl Widget {
    fn size(&self) -> usize {
        *self.cached_size.get_or_init(|| {
            // 假设这是个昂贵计算
            self.data.iter().map(|b| *b as usize).sum()
        })
    }
}
```

注意：`get_or_init` 只需要 `&self`（不可变借用），却能完成内部初始化——这正是 `Cell` 系列"内部可变性"的体现。

---

## 六、OnceLock 详解

`std::sync::OnceLock<T>` —— 多线程、手动、只能写一次的容器。是 `OnceCell` 的线程安全版本。

### 与 OnceCell 的关系

API 几乎一一对应，主要差异：

- `OnceLock` 是 `Sync`，可用于 `static`、可放进 `Arc` 跨线程。
- 并发调用 `get_or_init` 时，**保证初始化闭包只执行一次**；其他线程阻塞等待，最终拿到同一个值。

### 典型场景：全局单例

```rust
use std::sync::OnceLock;

#[derive(Debug)]
struct Config {
    name: String,
    workers: usize,
}

fn config() -> &'static Config {
    static CONFIG: OnceLock<Config> = OnceLock::new();
    CONFIG.get_or_init(|| {
        // 即使多个线程同时首次调用，这里也只执行一次
        Config {
            name: "Wardrobe".to_string(),
            workers: 4,
        }
    })
}

fn main() {
    println!("{:?}", config());      // 首次：初始化
    println!("{}", config().workers); // 之后：直接返回引用
}
```

### 典型场景：运行时一次性设置（set 模式）

适合"在 `main` 启动时根据命令行/环境变量决定值"的情况：

```rust
use std::sync::OnceLock;

static APP_NAME: OnceLock<String> = OnceLock::new();

fn main() {
    let name = std::env::args().nth(1).unwrap_or_else(|| "default".into());
    APP_NAME.set(name).expect("APP_NAME 只能设置一次");

    println!("App: {}", APP_NAME.get().unwrap());
}
```

### 处理"初始化可能失败"

`get_or_init` 的闭包不能返回错误。如果初始化可能失败，可用尚未稳定的 `get_or_try_init`（nightly），或自己用 `get` + `set` 组合：

```rust
use std::sync::OnceLock;

static DATA: OnceLock<String> = OnceLock::new();

fn load() -> Result<&'static String, std::io::Error> {
    if let Some(v) = DATA.get() {
        return Ok(v);
    }
    let value = std::fs::read_to_string("config.txt")?; // 可能失败
    // 若并发下别的线程已先 set，这里的 set 会失败，忽略即可
    let _ = DATA.set(value);
    Ok(DATA.get().unwrap())
}
```

---

## 七、LazyCell 详解

`std::cell::LazyCell<T, F>` —— 单线程、自动、首次解引用时初始化。

### 关键点

- 创建时传入初始化闭包 `F: FnOnce() -> T`。
- 通过 `Deref` 访问时，若尚未初始化则自动执行闭包。
- 不是 `Sync`，**不能用于 `static`**，但可用于 `thread_local!` 或局部变量、结构体字段。

```rust
use std::cell::LazyCell;

let lazy: LazyCell<Vec<u32>> = LazyCell::new(|| {
    println!("初始化中……");
    (1..=5).collect()
});

println!("声明完成，还没初始化");
// 第一次解引用，触发闭包
println!("{:?}", *lazy);   // 打印 "初始化中……" 然后 [1, 2, 3, 4, 5]
// 第二次直接用缓存
println!("{}", lazy.len()); // 5，不再打印 "初始化中……"
```

### 配合 `thread_local!`

`LazyCell` 在线程局部存储里很合适——每个线程一份，无需同步：

```rust
use std::cell::LazyCell;

thread_local! {
    static BUFFER: LazyCell<Vec<u8>> = LazyCell::new(|| {
        Vec::with_capacity(1024)
    });
}
```

---

## 八、LazyLock 详解

`std::sync::LazyLock<T, F>` —— 多线程、自动、首次解引用时初始化。**最常用、最方便的全局延迟初始化方案**，等价于旧的 `lazy_static!`。

### 关键点

- 创建时传入初始化闭包。
- 首次解引用时自动初始化，并发安全（只执行一次）。
- 是 `Sync`，**可用于 `static`**。

### 典型场景：全局只读数据

```rust
use std::sync::LazyLock;
use std::collections::HashMap;

static LOOKUP: LazyLock<HashMap<&'static str, u32>> = LazyLock::new(|| {
    let mut m = HashMap::new();
    m.insert("one", 1);
    m.insert("two", 2);
    m.insert("three", 3);
    m
});

fn main() {
    // 像普通引用一样使用，首次访问自动构建 HashMap
    println!("{:?}", LOOKUP.get("two")); // Some(2)
    println!("{}", LOOKUP.len());        // 3
}
```

### 典型场景：编译一次的正则（实战常见）

```rust
use std::sync::LazyLock;
use regex::Regex; // 需要 regex crate

static EMAIL_RE: LazyLock<Regex> = LazyLock::new(|| {
    Regex::new(r"^[\w.+-]+@[\w-]+\.[\w.-]+$").unwrap()
});

fn is_email(s: &str) -> bool {
    EMAIL_RE.is_match(s)
}
```

### 配合 `Mutex` 实现可变全局状态

`LazyLock` 本身存的是不可变值；若需要全局可变状态，把 `Mutex`/`RwLock` 放进去：

```rust
use std::sync::{LazyLock, Mutex};

static COUNTER: LazyLock<Mutex<u32>> = LazyLock::new(|| Mutex::new(0));

fn increment() {
    let mut n = COUNTER.lock().unwrap();
    *n += 1;
}
```

---

## 九、选型决策表

| 需求                                              | 推荐类型      |
|--------------------------------------------------|--------------|
| 全局常量，初始化逻辑固定，首次访问自动算           | **`LazyLock`** |
| 全局值，需在运行时某时机手动 `set`                 | **`OnceLock`** |
| 全局可变状态                                      | `LazyLock<Mutex<T>>` / `OnceLock<Mutex<T>>` |
| 单线程局部缓存，初始化逻辑固定                     | **`LazyCell`** |
| 单线程局部缓存，手动控制初始化时机                 | **`OnceCell`** |
| 结构体字段中的延迟派生值（单线程）                 | `OnceCell` / `LazyCell` |
| `thread_local!` 中的延迟初始化                     | `LazyCell` / `OnceCell` |

**快速决策流程：**

1. 需要跨线程 / 用作 `static`？
   - 是 → `*Lock`
   - 否 → `*Cell`
2. 初始化逻辑固定、想首次访问自动触发？
   - 是 → `Lazy*`
   - 否（要手动 `set` 或运行时决定）→ `Once*`

---

## 十、常见陷阱与注意事项

1. **`*Cell` 不能用于 `static`**
   `static` 要求类型是 `Sync`，而 `OnceCell` / `LazyCell` 不是。全局变量请用 `OnceLock` / `LazyLock`。

2. **`set` 在已初始化时会失败**
   `OnceCell::set` / `OnceLock::set` 返回 `Result`，第二次调用返回 `Err(传入的值)`。需要的话记得处理。

3. **初始化闭包里不要再去访问同一个 cell（重入死锁/panic）**
   在 `get_or_init` 或 `Lazy*` 的闭包中再次访问同一个实例会导致 panic（甚至 `*Lock` 下可能死锁）。初始化逻辑应是自包含的。

4. **`LazyLock` 初始化时 panic 会"毒化"该实例**
   若初始化闭包 panic，后续访问会再次 panic。确保初始化逻辑稳健（必要时在闭包里用 `unwrap_or_else` 提供兜底）。

5. **初始化的执行时机**
   `Lazy*` 是"首次解引用"才初始化，不是声明时。如果某个全局值从未被访问，它的初始化闭包永远不会执行。

6. **闭包返回错误怎么办**
   `get_or_init` 的闭包必须返回 `T`，不能返回 `Result`。初始化可能失败时，用 `get` + `set` 手动组合，或 `unwrap`/兜底，或等待 `get_or_try_init` 稳定。

---

## 十一、与第三方 crate 的对应关系

标准库这四个类型，基本覆盖了过去两个流行 crate 的功能：

| 旧方案                              | 标准库等价物                  |
|------------------------------------|------------------------------|
| `once_cell::unsync::OnceCell`      | `std::cell::OnceCell`        |
| `once_cell::sync::OnceCell`        | `std::sync::OnceLock`        |
| `once_cell::unsync::Lazy`          | `std::cell::LazyCell`        |
| `once_cell::sync::Lazy`            | `std::sync::LazyLock`        |
| `lazy_static!`（宏）               | `std::sync::LazyLock`        |

**迁移建议**：
- 新项目优先用标准库的四个类型，无需额外依赖。
- 旧项目里的 `lazy_static!` 可平滑替换为 `static X: LazyLock<T> = LazyLock::new(|| ...);`。
- 如果还需要 `once_cell` 的一些尚未进标准库的高级特性（如 `get_or_try_init` 的稳定版），可继续保留该 crate。

---

## 附：一段汇总对比代码

```rust
use std::cell::{OnceCell, LazyCell};
use std::sync::{OnceLock, LazyLock};

// 1. OnceLock：全局 + 手动
static A: OnceLock<u32> = OnceLock::new();

// 2. LazyLock：全局 + 自动
static B: LazyLock<u32> = LazyLock::new(|| 2 + 2);

fn main() {
    // OnceLock 手动初始化
    A.set(10).unwrap();
    println!("A = {}", A.get().unwrap()); // 10

    // LazyLock 首次访问自动初始化
    println!("B = {}", *B); // 4

    // OnceCell：单线程、手动
    let c: OnceCell<String> = OnceCell::new();
    println!("C = {}", c.get_or_init(|| "lazy".to_string()));

    // LazyCell：单线程、自动
    let d: LazyCell<Vec<i32>> = LazyCell::new(|| vec![1, 2, 3]);
    println!("D = {:?}", *d);
}
```
