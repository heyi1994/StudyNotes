# std:async / await / Future / Tokio

> async/await 是 Rust 应对**高并发 IO**的利器：用同步般的写法，跑出异步的并发量。但它也是 Rust 公认较难的一块——因为标准库只提供"语言机制"，真正的"运行时"在第三方（tokio 等）。理解"`async` 只是生成状态机、必须有运行时来驱动"这件事，是入门的关键。

---

## 目录

- [一、为什么需要异步](#一为什么需要异步)
- [二、async / await 基础](#二async--await-基础)
- [三、Future：异步的核心抽象](#三future异步的核心抽象)
- [四、运行时与执行器（Runtime / Executor）](#四运行时与执行器runtime--executor)
- [五、Tokio 快速上手](#五tokio-快速上手)
- [六、并发执行：join / select / spawn](#六并发执行join--select--spawn)
- [七、异步中的共享状态与通道](#七异步中的共享状态与通道)
- [八、async 与生命周期 / Send](#八async-与生命周期--send)
- [九、阻塞操作的处理](#九阻塞操作的处理)
- [十、async 多线程 vs 同步多线程](#十async-多线程-vs-同步多线程)
- [十一、常见陷阱](#十一常见陷阱)

---

## 一、为什么需要异步

考虑一个服务器要同时处理一万个网络连接，大部分时间都在**等待**（等数据库、等网络响应）。两种做法：

| 做法 | 问题 |
|------|------|
| 每个连接开一个 OS 线程 | 一万个线程 = 一万个栈（每个几 MB）+ 大量上下文切换，扛不住 |
| 异步：少量线程 + 大量轻量任务 | 一个线程在某任务等待时切去执行别的任务，几十万任务也能跑 |

核心区别：

- **多线程（`std::thread`）**：适合 **CPU 密集**（并行计算）。阻塞 = 线程睡着。
- **异步（async）**：适合 **IO 密集**（大量等待）。等待时**不占线程**，任务被挂起，线程转去做别的。

> 一句话：异步用**协作式调度**把"等待"的时间利用起来，用极少的线程支撑海量并发连接。

---

## 二、async / await 基础

### async：定义异步函数

`async fn` 不会立即执行函数体，而是返回一个 **`Future`**（一个"将来会产生值的计算"）：

```rust
// 普通函数：调用即执行，立刻返回 u32
fn normal() -> u32 { 5 }

// async 函数：调用返回一个 Future，函数体还没运行！
async fn async_fn() -> u32 { 5 }
//  实际返回类型 ≈ impl Future<Output = u32>

fn main() {
    let fut = async_fn();   // 此刻函数体一行都没执行，fut 只是个"计划"
    // fut 必须被 .await 或交给运行时，才会真正执行
}
```

`async` 还能用于代码块：

```rust
let fut = async {
    let x = compute().await;
    x + 1
};
```

### await：等待 Future 完成

`.await` 用在 `Future` 上，**挂起当前任务直到结果就绪**，期间把执行权让给运行时去跑别的任务：

```rust
async fn fetch_user() -> String {
    let id = get_id().await;          // 等待 get_id 完成，期间线程可去做别的
    let name = get_name(id).await;    // 再等 get_name
    name
}
```

**关键规则**：
- `.await` **只能在 `async` 函数 / async 块内使用**。
- `.await` 一个 Future 才会推进它；不 `.await` 的 Future 什么都不做（"惰性"）。
- `async fn` 内部是**顺序**执行的——上面的 `get_id` 和 `get_name` 是一前一后，不是并发（要并发见第六节）。

### "惰性"是重点

Rust 的 Future 是**惰性的（lazy）**：创建出来不会自动跑，必须被 `.await` 或被运行时 `spawn` 才会执行。这和 JavaScript 的 Promise（创建即开始执行）截然不同：

```rust
async fn say(msg: &str) { println!("{}", msg); }

#[tokio::main]
async fn main() {
    let fut = say("hello"); // 什么都不打印！Future 没被驱动
    fut.await;              // 现在才打印 hello
}
```

---

## 三、Future：异步的核心抽象

`async fn` / `async {}` 在底层会被编译器转换成一个实现了 `Future` trait 的**状态机**。理解这个 trait 能让你看穿异步的本质：

```rust
pub trait Future {
    type Output;   // 完成时产生的值的类型

    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),    // 完成了，这是结果
    Pending,     // 还没好，稍后再来问
}
```

工作机制：

1. 运行时（executor）调用 Future 的 `poll`。
2. 如果数据没准备好，返回 `Pending`，并通过 `Context` 里的 **Waker** 登记"好了请叫我"。
3. 当 IO 就绪，Waker 通知运行时，运行时再次 `poll` 这个 Future。
4. 直到返回 `Ready(value)`，任务完成。

### 编译器把 async 变成状态机

每个 `.await` 是一个**可能暂停的点**。编译器把 `async fn` 改写成一个枚举状态机，每个状态对应一个 `.await` 暂停处，保存当时需要的局部变量：

```rust
async fn example() {
    let a = step1().await;  // 状态 0 → 1
    let b = step2(a).await;  // 状态 1 → 2
    // 完成
}
// 编译器大致生成（概念示意）：
// enum ExampleStateMachine {
//     Start,
//     WaitingStep1 { fut: Step1Future },
//     WaitingStep2 { a: ..., fut: Step2Future },
//     Done,
// }
// 每次 poll 推进一个状态
```

> 这解释了一些后面会遇到的现象：为什么跨 `.await` 持有 `MutexGuard` 不好（它被存进状态机、跨越暂停点）、为什么自引用需要 `Pin`。
> 多数情况你不用手写 `poll`——直接用 `async/await` 即可。手写 `Future` 只在实现底层原语时才需要。

---

## 四、运行时与执行器（Runtime / Executor）

**这是 Rust 异步最容易绊倒新手的地方**：标准库**只提供 `Future` trait、`async`/`await` 语法**，但**不提供运行时**。没有运行时，Future 永远不会被 `poll`，等于不会执行。

运行时（runtime）负责：
- **Executor（执行器）**：不断 `poll` 各个任务，推进它们。
- **Reactor（反应器）**：对接操作系统的 IO 事件（epoll/kqueue/IOCP），就绪时唤醒对应任务。
- **Timer / 任务调度** 等。

主流运行时：

| 运行时 | 特点 |
|--------|------|
| [`tokio`](https://crates.io/crates/tokio) | 生态最大、功能最全，事实标准。多线程调度器、完整异步 IO/网络/定时器 |
| [`async-std`](https://crates.io/crates/async-std) | API 贴近标准库风格 |
| [`smol`](https://crates.io/crates/smol) | 小巧轻量 |

> 没有运行时时，最小的"驱动一个 Future"工具是 [`futures::executor::block_on`](https://docs.rs/futures)。但实际项目几乎都用 tokio。

---

## 五、Tokio 快速上手

### 入口：#[tokio::main]

这个宏把你的 `async main` 包进一个运行时并 `block_on` 它：

```rust
#[tokio::main]
async fn main() {
    println!("在异步运行时里跑");
    let result = compute().await;
    println!("{}", result);
}

async fn compute() -> u32 { 42 }

// #[tokio::main] 展开后约等于：
// fn main() {
//     tokio::runtime::Runtime::new().unwrap().block_on(async {
//         ... 你的 async main 函数体 ...
//     });
// }
```

`Cargo.toml` 依赖（`full` 打开全部特性，入门方便）：

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

### 异步版的常用操作

tokio 提供了标准库阻塞 API 的异步对应物——注意都要 `.await`：

```rust
use tokio::time::{sleep, Duration};
use tokio::fs;

#[tokio::main]
async fn main() {
    // 异步睡眠：不阻塞线程，线程可去跑别的任务
    sleep(Duration::from_secs(1)).await;

    // 异步文件读写
    let content = fs::read_to_string("config.toml").await.unwrap();
    fs::write("out.txt", "hello").await.unwrap();
}
```

> ⚠️ 千万别在异步代码里用 `std::thread::sleep`（同步阻塞）——它会卡住整个线程，所有任务都跟着停。要用 `tokio::time::sleep`。

---

## 六、并发执行：join / select / spawn

前面说过 `async fn` 内部是顺序的。要真正**并发**，需要这些工具。

### join!：同时等多个，全部完成

并发地推进多个 Future，等它们**都**完成。比顺序 `.await` 快：

```rust
use tokio::time::{sleep, Duration};

async fn task(name: &str, secs: u64) -> String {
    sleep(Duration::from_secs(secs)).await;
    format!("{} 完成", name)
}

#[tokio::main]
async fn main() {
    // ❌ 顺序：耗时 2 + 3 = 5 秒
    // let a = task("A", 2).await;
    // let b = task("B", 3).await;

    // ✅ 并发：两者同时跑，耗时 max(2,3) = 3 秒
    let (a, b) = tokio::join!(task("A", 2), task("B", 3));
    println!("{}, {}", a, b);
}
```

> 注意：`join!` 是在**同一个任务里**并发推进多个 Future（单任务内的并发），不一定用到多线程。

### select!：等最先完成的那个

多个 Future 竞争，**哪个先就绪就用哪个**，其余取消。适合超时、二选一等：

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    tokio::select! {
        _ = sleep(Duration::from_secs(5)) => {
            println!("任务完成");
        }
        _ = sleep(Duration::from_secs(2)) => {
            println!("超时了！");   // 这个先到，被选中
        }
    }
}
```

### tokio::spawn：派生独立任务

把一个 Future 交给运行时作为**独立任务**后台运行，立即返回 `JoinHandle`（类似线程的 spawn，但是轻量任务）。这才是真正利用多线程调度器并行的方式：

```rust
#[tokio::main]
async fn main() {
    let mut handles = vec![];
    for i in 0..5 {
        let h = tokio::spawn(async move {
            // 每个任务独立运行，可能分布在线程池的不同线程
            format!("任务 {} 的结果", i)
        });
        handles.push(h);
    }
    for h in handles {
        println!("{}", h.await.unwrap()); // 等任务完成取结果
    }
}
```

| 工具 | 作用 |
|------|------|
| `join!` | 并发等多个 Future，全部完成 |
| `try_join!` | 同上，但任一 `Err` 立即返回 |
| `select!` | 等最先完成的一个，其余取消 |
| `tokio::spawn` | 派生后台独立任务，返回 `JoinHandle` |

---

## 七、异步中的共享状态与通道

### 锁：用 tokio::sync::Mutex（跨 .await 时）

如果持锁期间**不跨 `.await`**，用标准库 `std::sync::Mutex` 即可（更快）。但如果需要**持锁跨越 `.await`**，必须用 `tokio::sync::Mutex`（它的 guard 是 `Send`，且不会在持锁时阻塞线程）：

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

#[tokio::main]
async fn main() {
    let data = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..5 {
        let data = Arc::clone(&data);
        handles.push(tokio::spawn(async move {
            let mut num = data.lock().await;  // 异步加锁，注意 .await
            *num += 1;
        }));
    }
    for h in handles { h.await.unwrap(); }
    println!("{}", *data.lock().await); // 5
}
```

> 经验：**优先用 `std::sync::Mutex`，只在锁要跨 `.await` 时才换 `tokio::sync::Mutex`**。

### 异步通道

tokio 提供多种异步通道，`send`/`recv` 都是 `.await`：

| 通道 | 用途 |
|------|------|
| `mpsc` | 多生产者单消费者（最常用） |
| `oneshot` | 一次性，单个值（任务间传一个结果） |
| `broadcast` | 多生产者多消费者，每个消费者都收到 |
| `watch` | 单值广播，只关心最新值（如配置热更新） |

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(32); // 容量 32

    tokio::spawn(async move {
        for i in 0..5 {
            tx.send(i).await.unwrap();
        }
    });

    while let Some(v) = rx.recv().await {
        println!("收到 {}", v);
    }
}
```

---

## 八、async 与生命周期 / Send

异步代码有两个常见的"编译器拦路"，理解原理就不慌。

### 1. spawn 的任务要求 'static + Send

`tokio::spawn` 的 Future 可能被调度到任意线程、活得很久，所以要求 `'static`（不借用外部短命数据）和 `Send`（能跨线程）：

```rust
// ❌ 借用了外部 data，不是 'static
// let data = vec![1, 2, 3];
// tokio::spawn(async { println!("{:?}", data); });

// ✅ move 进去，获得所有权
let data = vec![1, 2, 3];
tokio::spawn(async move { println!("{:?}", data); });
```

### 2. 跨 .await 不要持有非 Send 的东西

因为 Future 是状态机，跨 `.await` 持有的变量会被**存进状态机**。如果持有 `Rc`、`RefCell` 的 guard、或 `std::sync::MutexGuard` 跨越 `.await`，整个 Future 就变成非 `Send`，无法 `spawn`：

```rust
use std::sync::Mutex;

// ❌ std MutexGuard 跨 .await，Future 变非 Send，spawn 会报错
// async fn bad(m: &Mutex<i32>) {
//     let guard = m.lock().unwrap();
//     some_async().await;   // guard 跨越了 await！
//     println!("{}", *guard);
// }

// ✅ 要么用 tokio::sync::Mutex，要么在 await 前释放锁
async fn good(m: &Mutex<i32>) {
    let value = {
        let guard = m.lock().unwrap();
        *guard                 // 取出值，guard 在块结束即释放
    };                         // 锁已释放
    some_async().await;        // 此时没持锁，安全
    println!("{}", value);
}
async fn some_async() {}
```

> 记住这条线索：**报错说 Future 不是 `Send`** → 多半是跨 `.await` 持有了非 Send 的临时量（常见是锁的 guard）。缩小作用域让它在 `.await` 前 drop。

---

## 九、阻塞操作的处理

异步运行时的线程很宝贵——**一个任务里跑同步阻塞代码（重计算、`std::fs`、`std::thread::sleep`），会卡住该线程上的所有任务**。

对策：把阻塞活儿挪到专门的线程：

```rust
#[tokio::main]
async fn main() {
    // 1. CPU 密集 / 阻塞 IO：用 spawn_blocking 丢到专用阻塞线程池
    let result = tokio::task::spawn_blocking(|| {
        // 这里可以放重计算或同步阻塞调用
        heavy_computation()
    }).await.unwrap();
    println!("{}", result);
}

fn heavy_computation() -> u64 {
    (0..1_000_000).sum()
}
```

| 任务类型 | 怎么跑 |
|---------|--------|
| 异步 IO（网络/异步文件） | 直接 `.await` |
| CPU 密集计算 | `tokio::task::spawn_blocking` 或交给 rayon |
| 同步阻塞库（老的 DB 驱动等） | `spawn_blocking` |

---

## 十、async 多线程 vs 同步多线程

| | 同步多线程（`std::thread`） | 异步（async + tokio） |
|---|---------------------------|----------------------|
| 适合 | CPU 密集、并行计算 | IO 密集、海量并发连接 |
| 并发单位 | OS 线程（重，几 MB 栈） | 任务 Future（轻，几十万个没问题） |
| 阻塞代价 | 线程睡着，浪费一个线程 | 任务挂起，线程转去跑别的 |
| 运行时 | 标准库自带 | 需 tokio 等第三方 |
| 心智负担 | 较低 | 较高（惰性、Pin、Send、染色） |

**"函数染色"问题**：`async` 有传染性——`async fn` 只能被 `async` 上下文 `.await`，于是异步会顺着调用链蔓延。同步和异步两套生态不能无缝混用，这是 async 复杂度的一大来源。

选型：
- 纯 CPU 并行 → 直接多线程 / rayon，别引入 async。
- 大量网络 IO / 高并发服务 → async + tokio。
- 混合 → tokio 跑异步主体，重活用 `spawn_blocking`。

---

## 十一、常见陷阱

1. **Future 不 `.await` 就不执行**
   `let f = do_something();` 不会运行任何东西。必须 `.await` 或 `spawn`。

2. **在异步里调用同步阻塞**
   `std::thread::sleep`、`std::fs`、大循环计算会卡住运行时线程。用异步版本或 `spawn_blocking`。

3. **持锁跨 `.await` → Future 非 Send**
   `std::sync::MutexGuard` 别跨越 `.await`。缩小作用域，或换 `tokio::sync::Mutex`。

4. **顺序 `.await` 误当并发**
   `a().await; b().await;` 是串行。要并发用 `join!` / `spawn`。

5. **忘了运行时**
   没有 `#[tokio::main]` 或 `block_on`，`async main` / Future 根本跑不起来。

6. **spawn 的任务借用了局部变量**
   要求 `'static`，用 `move` 转移所有权，或用 `Arc` 共享。

7. **`select!` 取消导致状态丢失**
   `select!` 中未完成的分支会被取消（drop）。如果该 Future 有副作用或需保留进度，注意"取消安全性（cancellation safety）"。

8. **CPU 密集任务塞进异步**
   async 不会让计算变快，只擅长"等待"。重计算该用线程/rayon。

---

## 附：速查总结

```
定义异步    async fn / async {}   → 返回 Future（惰性，不自动跑）
等待        .await                → 挂起当前任务直到就绪，期间让出线程
                                   只能在 async 上下文里用

核心抽象    Future::poll → Poll::Ready(v) | Poll::Pending
           编译器把 async 改写成状态机，每个 .await 是一个暂停点

运行时      标准库只给语法，不给运行时！必须用 tokio 等
           #[tokio::main] 包装 async main 并 block_on

并发        join!(a, b)     并发等全部完成（单任务内）
           select!{...}    等最先就绪的一个，其余取消
           tokio::spawn    派生独立后台任务（用上多线程调度）→ JoinHandle

共享        Arc + std::sync::Mutex     不跨 await 时（更快）
           Arc + tokio::sync::Mutex   需跨 await 时
           tokio::sync::{mpsc,oneshot,broadcast,watch}  异步通道

阻塞活儿    spawn_blocking(|| 重计算/同步阻塞)

拦路规律    "Future is not Send" → 多半跨 .await 持有了锁 guard / Rc
           缩小作用域让它在 .await 前 drop

心法：异步擅长"等"（IO 密集），不擅长"算"（CPU 密集）。
     Future 惰性，必须被驱动；async 会染色，会蔓延。
```
