# std:Thread / Channel / Mutex / Atomic / Send & Sync

> Rust 的并发口号是 **"无畏并发（Fearless Concurrency）"**：靠所有权和类型系统，把绝大多数数据竞争（data race）挡在**编译期**。在别的语言里跑起来才暴露的并发 bug，在 Rust 里很多直接编译不过。本篇梳理标准库的并发原语与心法。

---

## 目录

- [一、Rust 并发的核心思想](#一rust-并发的核心思想)
- [二、线程：thread::spawn](#二线程threadspawn)
- [三、Send 与 Sync：并发安全的根基](#三send-与-sync并发安全的根基)
- [四、共享状态并发：Arc + Mutex / RwLock](#四共享状态并发arc--mutex--rwlock)
- [五、消息传递并发：channel](#五消息传递并发channel)
- [六、原子类型：Atomic](#六原子类型atomic)
- [七、其他同步原语：Once / Barrier / Condvar](#七其他同步原语once--barrier--condvar)
- [八、作用域线程：thread::scope](#八作用域线程threadscope)
- [九、两种并发范式的选择](#九两种并发范式的选择)
- [十、与 async 异步的关系](#十与-async-异步的关系)
- [十一、常见陷阱](#十一常见陷阱)

---

## 一、Rust 并发的核心思想

Rust 不靠程序员"小心翼翼"，而是用**所有权规则**把并发安全编码进类型系统：

1. **数据竞争的定义**：两个及以上线程同时访问同一数据，至少一个在写，且没有同步 → 未定义行为。
2. **Rust 的破解之道**：
   - 要么数据**只有一个所有者**（移动给某个线程，别人碰不到）；
   - 要么共享时**强制同步**（`Mutex`/原子操作），由类型系统（`Send`/`Sync`）保证。
3. **结果**：大多数数据竞争在编译期就被拒绝。

两大并发范式（本篇都会讲）：

| 范式 | 思路 | 标准库工具 | 一句话 |
|------|------|-----------|--------|
| **共享状态** | 多线程访问同一块加锁的数据 | `Arc<Mutex<T>>` | "通过共享内存来通信" |
| **消息传递** | 线程间通过通道发送数据，不共享 | `mpsc::channel` | "通过通信来共享内存" |

> Rust 两者都支持，但官方文档倾向推荐**消息传递**（更不容易出错）。

---

## 二、线程：thread::spawn

`std::thread::spawn` 创建一个新的操作系统线程（1:1 模型），返回 `JoinHandle`：

```rust
use std::thread;

let handle = thread::spawn(|| {
    for i in 1..=5 {
        println!("子线程: {}", i);
    }
});

// 主线程做自己的事
for i in 1..=3 {
    println!("主线程: {}", i);
}

// join：等待子线程结束（否则主线程结束时子线程可能被强行终止）
handle.join().unwrap();
```

### move 闭包：把数据交给线程

子线程可能比当前函数活得久，所以闭包里用到的外部变量必须**移动所有权**进去（`move`）：

```rust
use std::thread;

let data = vec![1, 2, 3];

let handle = thread::spawn(move || {     // move：data 所有权进入线程
    println!("线程拥有 {:?}", data);
});
// 此处 data 不可再用，已被移动
handle.join().unwrap();
```

### 从线程取返回值

`join()` 返回 `Result`，其中 `Ok` 携带闭包的返回值：

```rust
let handle = thread::spawn(|| {
    (1..=100).sum::<i32>()    // 线程的计算结果
});
let result: i32 = handle.join().unwrap();
println!("总和 = {}", result); // 5050
```

> `join()` 返回 `Err` 表示该线程 **panic 了**——这是检测子线程崩溃的方式。

---

## 三、Send 与 Sync：并发安全的根基

这两个 **marker trait**（标记 trait，没有方法）是 Rust 并发安全的理论基础。它们由编译器自动推导，你几乎不用手动实现，但理解它们能看懂大量并发相关的编译错误。

| Trait | 含义 | 直觉 |
|-------|------|------|
| `Send` | 类型的**所有权可以安全地转移到另一个线程** | "能不能把它搬给别的线程" |
| `Sync` | 类型的**引用 `&T` 可以安全地被多个线程共享** | "能不能多个线程同时只读它"（`T: Sync` ⟺ `&T: Send`） |

### 谁是 / 谁不是

- **绝大多数类型都是 `Send + Sync`**（基本类型、`String`、`Vec`、`Arc<T>`……）。
- **典型的非 `Send`/`Sync`**：
  - `Rc<T>`：计数非原子，多线程会数据竞争 → 既非 `Send` 也非 `Sync`。
  - `RefCell<T>`：运行期借用检查非线程安全 → `Send` 但**非 `Sync`**。
  - 裸指针 `*const T` / `*mut T`：非 `Send` 非 `Sync`。

### 它们如何保护你

`thread::spawn` 的签名要求闭包是 `Send`，闭包捕获的所有数据也必须 `Send`。于是：

```rust
use std::rc::Rc;
use std::thread;

let data = Rc::new(5);
//  编译错误：Rc<i32> 不是 Send，不能进线程
// thread::spawn(move || { println!("{}", data); });
//   error: `Rc<i32>` cannot be sent between threads safely

//  换成 Arc 就行
use std::sync::Arc;
let data = Arc::new(5);
thread::spawn(move || { println!("{}", data); }).join().unwrap();
```

> 这就是"无畏并发"的精髓：**用 `Rc` 跨线程根本编译不过**，错误在写代码时就被拦下，而不是上线后偶发崩溃。

---

## 四、共享状态并发：Arc + Mutex / RwLock

多个线程要访问同一份**可变**数据时，标准组合是 `Arc<Mutex<T>>`：
- `Arc`：让多个线程共享所有权（原子引用计数）。
- `Mutex`：保证同一时刻只有一个线程能访问内部数据。

### Mutex：互斥锁

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    let h = thread::spawn(move || {
        let mut num = counter.lock().unwrap(); // 加锁，返回 MutexGuard
        *num += 1;
    }); // num（MutexGuard）离开作用域 → 自动解锁
    handles.push(h);
}
for h in handles {
    h.join().unwrap();
}
println!("结果 = {}", *counter.lock().unwrap()); // 10
```

要点：
- `lock()` 返回 `Result<MutexGuard<T>>`，`MutexGuard` 实现了 `Deref`/`DerefMut`，可以直接当 `&mut T` 用。
- **锁会在 `MutexGuard` 离开作用域时自动释放**（RAII），不用手动 unlock。
- 想提前释放，用 `drop(guard)` 或加 `{}` 缩小作用域。

### RwLock：读写锁

`RwLock` 区分读和写：**允许多个读者同时持有，或一个写者独占**。读多写少时比 `Mutex` 并发度更高。

```rust
use std::sync::{Arc, RwLock};
use std::thread;

let data = Arc::new(RwLock::new(vec![1, 2, 3]));

// 多个读者可同时进入
{
    let r1 = data.read().unwrap();
    let r2 = data.read().unwrap();   // OK，多个读锁共存
    println!("{:?} {:?}", *r1, *r2);
}

// 写者独占
{
    let mut w = data.write().unwrap();
    w.push(4);
}
```

| | `Mutex<T>` | `RwLock<T>` |
|---|----------|------------|
| 并发读 |  一次一个 |  多读者共存 |
| 写 | 独占 | 独占 |
| 适用 | 读写都频繁 / 临界区短 | 读远多于写 |

---

## 五、消息传递并发：channel

消息传递让线程间**通过发送数据通信，而不共享内存**。标准库提供 `std::sync::mpsc`（multi-producer, single-consumer：多生产者、单消费者）。

```rust
use std::sync::mpsc;
use std::thread;

let (tx, rx) = mpsc::channel(); // tx 发送端，rx 接收端

thread::spawn(move || {
    let vals = vec!["hi", "from", "the", "thread"];
    for v in vals {
        tx.send(v).unwrap();   // 发送（所有权转移给通道）
        thread::sleep(std::time::Duration::from_millis(100));
    }
}); // tx 离开作用域 → 通道关闭

// rx 当迭代器用：通道关闭且取完后自动结束
for received in rx {
    println!("收到: {}", received);
}
```

### 多生产者

`clone` 发送端即可有多个生产者；只有一个接收端：

```rust
use std::sync::mpsc;
use std::thread;

let (tx, rx) = mpsc::channel();

for i in 0..3 {
    let tx = tx.clone();    // 每个线程一个发送端
    thread::spawn(move || {
        tx.send(format!("来自线程 {}", i)).unwrap();
    });
}
drop(tx); // 丢弃原始 tx，否则 rx 永远等不到关闭

for msg in rx {
    println!("{}", msg);
}
```

### 同步通道 vs 异步通道

| 函数 | 行为 |
|------|------|
| `mpsc::channel()` | **异步**（无界）：`send` 立即返回，消息进缓冲队列 |
| `mpsc::sync_channel(n)` | **同步**（有界，容量 n）：缓冲满时 `send` 阻塞，形成背压 |

要点：
- `send` 把数据的**所有权**转移进通道——接收方拿到后发送方就没了，天然无数据竞争。
- `recv()` 阻塞等待；`try_recv()` 非阻塞；把 `rx` 当迭代器是最常见写法。
- 所有发送端都 drop 后，通道关闭，接收端迭代结束。

> 需要多消费者（MPMC）或更高性能，社区常用 [`crossbeam-channel`](https://crates.io/crates/crossbeam-channel) 或 `flume`。

---

## 六、原子类型：Atomic

`std::sync::atomic` 提供无锁的原子类型（`AtomicBool`、`AtomicUsize`、`AtomicI32` 等）。对于简单的计数器、标志位，原子操作比 `Mutex` 更轻量（无锁、无阻塞）。

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use std::thread;

let counter = Arc::new(AtomicUsize::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    let h = thread::spawn(move || {
        // fetch_add：原子地加 1，无需锁
        counter.fetch_add(1, Ordering::SeqCst);
    });
    handles.push(h);
}
for h in handles {
    h.join().unwrap();
}
println!("{}", counter.load(Ordering::SeqCst)); // 10
```

常用方法：`load`（读）、`store`（写）、`fetch_add`/`fetch_sub`（加减）、`compare_exchange`（CAS 比较交换）、`swap`（交换）。

### 内存序（Ordering）

每个原子操作要指定**内存序**，决定编译器/CPU 可以怎样重排内存访问：

| Ordering | 强度 | 说明 |
|----------|------|------|
| `Relaxed` | 最弱 | 只保证操作本身原子，不约束顺序。适合纯计数器 |
| `Acquire` / `Release` | 中 | 配对使用：构建"获取-释放"同步关系（锁的典型实现） |
| `AcqRel` | 中 | 读改写操作同时具备 Acquire + Release |
| `SeqCst` | 最强 | 全局顺序一致，最安全也最慢。**拿不准就用它** |

> 入门建议：**不确定就用 `SeqCst`**，正确性优先。等理解了 happens-before 关系，再针对性优化成 `Relaxed`/`Acquire`/`Release`。
> 原子只适合**单个值**的同步；多个值的一致性还是要用 `Mutex`。

---

## 七、其他同步原语：Once / Barrier / Condvar

### Once / OnceLock：只执行一次

保证某段初始化逻辑在多线程下**只运行一次**（详见 `once_and_lazy.md`）：

```rust
use std::sync::Once;

static INIT: Once = Once::new();

fn setup() {
    INIT.call_once(|| {
        println!("只会打印一次，无论多少线程调用 setup");
    });
}
```

> 现代代码里更推荐 `OnceLock` / `LazyLock`（能直接存值），`Once` 是更底层的原语。

### Barrier：线程集合点

让一组线程在某个点**互相等待**，全部到齐后再一起继续：

```rust
use std::sync::{Arc, Barrier};
use std::thread;

let barrier = Arc::new(Barrier::new(3)); // 等 3 个线程
let mut handles = vec![];

for i in 0..3 {
    let b = Arc::clone(&barrier);
    handles.push(thread::spawn(move || {
        println!("线程 {} 到达屏障前", i);
        b.wait();   // 在此等待，直到 3 个线程都调用了 wait
        println!("线程 {} 越过屏障", i);
    }));
}
for h in handles { h.join().unwrap(); }
```

### Condvar：条件变量

配合 `Mutex` 使用，让线程**等待某个条件成立**时被唤醒（避免忙等轮询）。典型用于生产者-消费者：

```rust
use std::sync::{Arc, Mutex, Condvar};
use std::thread;

let pair = Arc::new((Mutex::new(false), Condvar::new()));
let pair2 = Arc::clone(&pair);

thread::spawn(move || {
    let (lock, cvar) = &*pair2;
    let mut ready = lock.lock().unwrap();
    *ready = true;
    cvar.notify_one();   // 通知等待者：条件已满足
});

let (lock, cvar) = &*pair;
let mut ready = lock.lock().unwrap();
while !*ready {
    ready = cvar.wait(ready).unwrap();  // 释放锁并睡眠，被唤醒后重新拿锁
}
println!("条件满足，继续执行");
```

---

## 八、作用域线程：thread::scope

`thread::spawn` 要求 `'static`（线程可能比当前栈帧活得久），所以不能借用局部变量，只能 `move`。**`thread::scope`（Rust 1.63+）** 解决了这个痛点：它保证所有子线程在作用域结束前 join 完，因此**可以安全地借用栈上的局部变量**，无需 `Arc`/`move` 克隆：

```rust
use std::thread;

let mut data = vec![1, 2, 3];

thread::scope(|s| {
    // 多个线程可以借用 data，无需 Arc
    s.spawn(|| {
        println!("只读借用: {:?}", &data);
    });
    s.spawn(|| {
        println!("另一个线程也能读: {}", data.len());
    });
}); // scope 结束前，所有子线程已自动 join

// 子线程都结束了，data 可以继续可变使用
data.push(4);
println!("{:?}", data);
```

> 当你只是想"并行处理本地数据、用完就汇合"时，`thread::scope` 比 `Arc` + `move` 简洁得多。需要数据并行还可以直接上 [`rayon`](https://crates.io/crates/rayon)（`par_iter()` 一行并行化）。

---

## 九、两种并发范式的选择

| | 共享状态（`Arc<Mutex<T>>`） | 消息传递（`channel`） |
|---|---------------------------|----------------------|
| 思路 | 多线程读写同一加锁数据 | 数据所有权在线程间传递 |
| 优点 | 直接、适合共享大状态 | 不易出错、无锁竞争、解耦 |
| 缺点 | 易死锁、需小心锁粒度 | 数据要移动、可能有拷贝 |
| 适合 | 共享缓存、计数器、状态表 | 任务分发、流水线、actor 模型 |

经验法则：
- **优先考虑消息传递**（`channel`），把可变状态局限在单个线程里。
- 状态确实需要被多个线程共享读写时，才用 `Arc<Mutex<T>>`。
- 只是简单计数 / 标志位 → `Atomic`，比锁更轻。
- 只读共享 → `Arc<T>`（无需锁）。

---

## 十、与 async 异步的关系

本篇讲的是**多线程并发**（OS 线程，`std::thread`），适合 **CPU 密集**型任务。

另一条线是 **async/await 异步**，适合 **IO 密集**型（大量网络/磁盘等待）：
- `async fn` / `.await`、`Future` trait 是语言级支持；
- 但标准库**不含运行时**，需要第三方 executor，如 [`tokio`](https://crates.io/crates/tokio)、[`async-std`](https://crates.io/crates/async-std)。

简单区分：

| | 多线程（thread） | 异步（async） |
|---|----------------|---------------|
| 适合 | CPU 密集、并行计算 | IO 密集、高并发连接 |
| 开销 | 每线程有栈，较重 | 任务很轻，可开几十万个 |
| 运行时 | 标准库自带 | 需 tokio 等 |

> 二者不互斥，实际项目常混用（tokio 内部也用线程池）。async 是一个大主题，可单独成章。

---

## 十一、常见陷阱

1. **死锁（deadlock）**
   同一线程重复锁同一 `Mutex`，或多个锁的加锁顺序不一致。对策：固定加锁顺序、缩小临界区、避免持锁时调用可能再加锁的代码。

2. **锁中毒（poisoning）**
   持锁线程 panic 后，`Mutex`/`RwLock` 被标记 poisoned，后续 `lock()` 返回 `Err`。通常 `.unwrap()` 让其传播，确需恢复用 `.into_inner()` 或 `e.get_ref()`。

3. **忘记 join，主线程提前退出**
   `main` 结束时未 join 的子线程会被直接终止，可能没跑完。需要结果就 `join()`。

4. **channel 接收端永久阻塞**
   忘记 drop 多余的发送端，`rx` 等不到通道关闭而一直阻塞。把多余 `tx` 显式 `drop`。

5. **临界区过大，并发退化成串行**
   锁住的代码块太大，线程都在排队等锁。让临界区尽量短，只在真正访问共享数据时持锁。

6. **误用 `Relaxed` 内存序**
   用 `Relaxed` 去同步多个变量的依赖关系会出错。拿不准就 `SeqCst`。

7. **跨线程用了非 Send/Sync 类型**
   `Rc`、`RefCell` 等会被编译器拒绝。换 `Arc`、`Mutex`/`RwLock`。这是编译器在帮你，别用 `unsafe` 硬绕。

---

## 附：速查总结

```
创建线程    thread::spawn(move || {...})   → JoinHandle，join() 等待/取结果
借用本地    thread::scope(|s| s.spawn(...))→ 可借用栈变量，自动 join

并发安全    Send  能把所有权搬到别的线程
           Sync  能让 &T 被多线程共享（&T: Send）
           Rc/RefCell 非 Sync → 编译期拦截跨线程误用

共享状态    Arc<Mutex<T>>    多线程共享 + 可变（互斥）
           Arc<RwLock<T>>   读多写少（多读单写）
           Arc<T>           只读共享，无需锁

消息传递    mpsc::channel()      异步无界
           mpsc::sync_channel(n) 同步有界（背压）
           多生产者 clone(tx)；单消费者 rx；send 转移所有权

无锁原语    AtomicUsize 等 + Ordering（拿不准用 SeqCst）

其他同步    Once/OnceLock 一次性  Barrier 集合点  Condvar 条件等待

心法：优先消息传递；共享可变才上 Arc<Mutex>；简单计数用 Atomic。
     数据竞争大多在编译期就被 Send/Sync 拦下——这就是无畏并发。
```
