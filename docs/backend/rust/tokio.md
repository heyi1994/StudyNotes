# Tokio 学习文档

## 1. 什么是 Tokio

Tokio 是 Rust 生态中最流行的异步运行时，提供：

- **异步 I/O**：基于 epoll/kqueue/IOCP 的高性能非阻塞 I/O
- **任务调度器**：多线程工作窃取调度器
- **异步原语**：通道、互斥锁、信号量等
- **定时器**：高精度时间驱动功能

```
应用代码
   │
   ▼
Tokio Runtime
   ├── 任务调度器（多线程）
   ├── I/O 事件驱动（epoll/kqueue）
   └── 定时器轮
```

---

## 2. 环境准备

### Cargo.toml

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }

# 常用搭配库
tokio-util = "0.7"
futures = "0.3"
```

常用 feature flags：

| Feature           | 说明                |
| ----------------- | ------------------- |
| `rt`              | 基础运行时          |
| `rt-multi-thread` | 多线程运行时        |
| `macros`          | `#[tokio::main]` 宏 |
| `net`             | TCP/UDP/Unix socket |
| `time`            | 定时器功能          |
| `sync`            | 同步原语            |
| `fs`              | 异步文件系统        |
| `io-util`         | 异步 I/O 工具       |
| `full`            | 以上全部            |

### 最小示例

```rust
#[tokio::main]
async fn main() {
    println!("Hello, Tokio!");
}
```

`#[tokio::main]` 展开后等价于：

```rust
fn main() {
    tokio::runtime::Runtime::new()
        .unwrap()
        .block_on(async {
            println!("Hello, Tokio!");
        });
}
```

---

## 3. 异步编程基础

### async / await

```rust
use tokio::time::{sleep, Duration};

async fn fetch_data() -> String {
    sleep(Duration::from_millis(100)).await;
    "data".to_string()
}

#[tokio::main]
async fn main() {
    let result = fetch_data().await;
    println!("{}", result);
}
```

**核心概念：**

- `async fn` 返回一个实现了 `Future` trait 的类型
- `.await` 暂停当前任务，让出执行权，等待 Future 完成
- Future 是惰性的：不 `.await` 就不会执行

### 并发执行：tokio::join!

```rust
use tokio::time::{sleep, Duration};

async fn task_a() -> &'static str {
    sleep(Duration::from_millis(200)).await;
    "A"
}

async fn task_b() -> &'static str {
    sleep(Duration::from_millis(100)).await;
    "B"
}

#[tokio::main]
async fn main() {
    // 并发执行，总耗时约 200ms（不是 300ms）
    let (a, b) = tokio::join!(task_a(), task_b());
    println!("{} {}", a, b); // A B
}
```

### 竞速执行：tokio::select!

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    tokio::select! {
        _ = sleep(Duration::from_millis(100)) => {
            println!("100ms 定时器先触发");
        }
        _ = sleep(Duration::from_millis(200)) => {
            println!("200ms 定时器先触发");
        }
    }
}
```

`select!` 只执行第一个完成的分支，其余分支被取消。

---

## 4. 任务（Tasks）

### spawn：创建独立任务

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let handle = task::spawn(async {
        println!("在独立任务中运行");
        42
    });

    let result = handle.await.unwrap();
    println!("任务返回: {}", result);
}
```

### 任务与线程的区别

| 维度     | 线程              | Tokio 任务         |
| -------- | ----------------- | ------------------ |
| 创建开销 | 高（MB 级栈）     | 低（KB 级）        |
| 切换方式 | 抢占式（OS 调度） | 协作式（await 点） |
| 并发量   | 数千              | 数百万             |
| 阻塞影响 | 只阻塞当前线程    | 阻塞整个线程       |

### 在任务中共享数据

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

#[tokio::main]
async fn main() {
    let counter = Arc::new(Mutex::new(0u32));

    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        handles.push(tokio::spawn(async move {
            let mut lock = counter.lock().await;
            *lock += 1;
        }));
    }

    for h in handles {
        h.await.unwrap();
    }

    println!("最终计数: {}", *counter.lock().await); // 10
}
```

### spawn_blocking：运行阻塞代码

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    // 将阻塞操作放到专用线程池，避免阻塞异步运行时
    let result = task::spawn_blocking(|| {
        // 可以放耗时的 CPU 计算或同步 I/O
        std::fs::read_to_string("/etc/hosts").unwrap()
    })
    .await
    .unwrap();

    println!("文件内容长度: {}", result.len());
}
```

---

## 5. 通道（Channels）

### mpsc：多生产者单消费者

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<i32>(32); // 缓冲区大小 32

    // 克隆 sender 实现多生产者
    let tx2 = tx.clone();

    tokio::spawn(async move {
        tx.send(1).await.unwrap();
        tx.send(2).await.unwrap();
    });

    tokio::spawn(async move {
        tx2.send(3).await.unwrap();
    });

    drop(tx2); // 显式 drop（此处 tx2 已 move，仅示意）

    // 当所有 sender drop 后，recv() 返回 None
    while let Some(val) = rx.recv().await {
        println!("收到: {}", val);
    }
}
```

### oneshot：一次性通道

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel::<String>();

    tokio::spawn(async move {
        let response = "任务完成".to_string();
        tx.send(response).unwrap();
    });

    match rx.await {
        Ok(msg) => println!("收到: {}", msg),
        Err(_) => println!("发送端已关闭"),
    }
}
```

### broadcast：广播通道

```rust
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    let (tx, _) = broadcast::channel::<String>(16);

    let mut rx1 = tx.subscribe();
    let mut rx2 = tx.subscribe();

    tx.send("广播消息".to_string()).unwrap();

    println!("rx1: {}", rx1.recv().await.unwrap());
    println!("rx2: {}", rx2.recv().await.unwrap());
}
```

### watch：状态监听通道

```rust
use tokio::sync::watch;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = watch::channel("初始状态");

    tokio::spawn(async move {
        tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
        tx.send("更新状态").unwrap();
    });

    // 等待值变化
    rx.changed().await.unwrap();
    println!("状态变为: {}", *rx.borrow());
}
```

---

## 6. I/O 操作

### TCP 客户端

```rust
use tokio::net::TcpStream;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut stream = TcpStream::connect("127.0.0.1:8080").await?;

    stream.write_all(b"Hello, Server!").await?;

    let mut buf = vec![0u8; 1024];
    let n = stream.read(&mut buf).await?;
    println!("收到: {}", String::from_utf8_lossy(&buf[..n]));

    Ok(())
}
```

### 异步文件操作

```rust
use tokio::fs;
use tokio::io::AsyncWriteExt;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 写文件
    let mut file = fs::File::create("hello.txt").await?;
    file.write_all(b"Hello, File!").await?;

    // 读文件
    let content = fs::read_to_string("hello.txt").await?;
    println!("{}", content);

    // 删除文件
    fs::remove_file("hello.txt").await?;

    Ok(())
}
```

### 使用 BufReader 提升性能

```rust
use tokio::io::{BufReader, AsyncBufReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let file = File::open("large_file.txt").await?;
    let reader = BufReader::new(file);
    let mut lines = reader.lines();

    while let Some(line) = lines.next_line().await? {
        println!("{}", line);
    }

    Ok(())
}
```

---

## 7. 定时器

```rust
use tokio::time::{sleep, timeout, interval, Duration, Instant};

#[tokio::main]
async fn main() {
    // 延迟
    sleep(Duration::from_secs(1)).await;

    // 超时控制
    let result = timeout(Duration::from_millis(100), async {
        sleep(Duration::from_millis(200)).await;
        "完成"
    })
    .await;

    match result {
        Ok(val) => println!("成功: {}", val),
        Err(_) => println!("超时！"),
    }

    // 定期执行（类似 ticker）
    let mut ticker = interval(Duration::from_millis(500));
    for i in 0..3 {
        ticker.tick().await;
        println!("第 {} 次 tick，时间: {:?}", i, Instant::now());
    }
}
```

---

## 8. 同步原语

### Mutex（异步互斥锁）

```rust
use tokio::sync::Mutex;
use std::sync::Arc;

// 注意：持有锁时不要 .await 其他 Future（可能导致死锁）
// 如需跨 await 持有锁，使用 tokio::sync::Mutex（而非 std::sync::Mutex）
let mutex = Arc::new(Mutex::new(vec![]));
let m = Arc::clone(&mutex);

tokio::spawn(async move {
    let mut lock = m.lock().await;
    lock.push(1);
    // lock 在此 drop
});
```

### RwLock（读写锁）

```rust
use tokio::sync::RwLock;

let lock = RwLock::new(5u32);

// 多个读锁可以同时持有
let r1 = lock.read().await;
let r2 = lock.read().await;
println!("r1={} r2={}", *r1, *r2);
drop((r1, r2));

// 写锁是独占的
let mut w = lock.write().await;
*w += 1;
```

### Semaphore（信号量）

```rust
use tokio::sync::Semaphore;
use std::sync::Arc;

// 限制并发请求数量
let sem = Arc::new(Semaphore::new(3)); // 最多 3 个并发

let mut handles = vec![];
for i in 0..10 {
    let sem = Arc::clone(&sem);
    handles.push(tokio::spawn(async move {
        let _permit = sem.acquire().await.unwrap();
        println!("任务 {} 正在执行", i);
        tokio::time::sleep(Duration::from_millis(100)).await;
        // _permit drop 后，释放信号量
    }));
}
```

### Notify（通知）

```rust
use tokio::sync::Notify;
use std::sync::Arc;

let notify = Arc::new(Notify::new());
let n = Arc::clone(&notify);

tokio::spawn(async move {
    tokio::time::sleep(Duration::from_millis(100)).await;
    n.notify_one();
});

notify.notified().await;
println!("收到通知！");
```

---

## 9. 错误处理

### 使用 anyhow

```toml
[dependencies]
anyhow = "1"
```

```rust
use anyhow::{Context, Result};

async fn read_config(path: &str) -> Result<String> {
    tokio::fs::read_to_string(path)
        .await
        .with_context(|| format!("读取配置文件失败: {}", path))
}

#[tokio::main]
async fn main() -> Result<()> {
    let config = read_config("config.toml").await?;
    println!("{}", config);
    Ok(())
}
```

### JoinHandle 错误处理

```rust
use tokio::task::JoinError;

let handle = tokio::spawn(async {
    panic!("任务崩溃了！");
});

match handle.await {
    Ok(result) => println!("任务成功: {:?}", result),
    Err(e) if e.is_panic() => println!("任务 panic: {:?}", e),
    Err(e) if e.is_cancelled() => println!("任务被取消"),
    Err(e) => println!("其他错误: {:?}", e),
}
```

---

## 10. 实战：构建 TCP 服务器

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

async fn handle_client(mut stream: TcpStream) {
    let peer = stream.peer_addr().unwrap();
    println!("新连接: {}", peer);

    let mut buf = vec![0u8; 4096];

    loop {
        match stream.read(&mut buf).await {
            Ok(0) => {
                println!("连接断开: {}", peer);
                return;
            }
            Ok(n) => {
                // 回显服务
                if stream.write_all(&buf[..n]).await.is_err() {
                    return;
                }
            }
            Err(e) => {
                eprintln!("读取错误 {}: {}", peer, e);
                return;
            }
        }
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("服务器监听 127.0.0.1:8080");

    loop {
        let (stream, _) = listener.accept().await?;
        // 每个连接一个任务
        tokio::spawn(handle_client(stream));
    }
}
```

---

## 11. 常见陷阱

### 陷阱 1：在异步代码中使用阻塞操作

```rust
// 错误：会阻塞整个线程
async fn bad() {
    std::thread::sleep(std::time::Duration::from_secs(1)); // 阻塞！
    std::fs::read_to_string("file.txt").unwrap();          // 阻塞！
}

// 正确：使用异步版本
async fn good() {
    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
    tokio::fs::read_to_string("file.txt").await.unwrap();
}

// 正确：用 spawn_blocking 包装无法避免的阻塞操作
async fn also_good() {
    tokio::task::spawn_blocking(|| {
        std::fs::read_to_string("file.txt").unwrap()
    })
    .await
    .unwrap();
}
```

### 陷阱 2：跨 await 持有 std::sync::MutexGuard

```rust
use std::sync::Mutex;

// 错误：std::sync::MutexGuard 跨 await 不是 Send
async fn bad(mutex: &Mutex<u32>) {
    let mut guard = mutex.lock().unwrap();
    some_async_fn().await; // 编译错误：MutexGuard 不是 Send
    *guard += 1;
}

// 正确：使用 tokio::sync::Mutex
use tokio::sync::Mutex;

async fn good(mutex: &Mutex<u32>) {
    let mut guard = mutex.lock().await;
    some_async_fn().await; // OK
    *guard += 1;
}

// 或者：在 await 前 drop
async fn also_good(mutex: &std::sync::Mutex<u32>) {
    {
        let mut guard = mutex.lock().unwrap();
        *guard += 1;
    } // guard 在这里 drop
    some_async_fn().await;
}
```

### 陷阱 3：select! 的公平性

```rust
// select! 随机选择就绪的分支，不是按顺序
// 如果某个分支持续就绪，其他分支可能饥饿
// 使用 tokio::select! 的 biased 修饰符强制顺序检查
tokio::select! {
    biased;                    // 按声明顺序检查
    _ = high_priority() => {} // 优先检查
    _ = low_priority() => {}
}
```

### 陷阱 4：任务泄露

```rust
// tokio::spawn 返回的 JoinHandle 如果被 drop，任务会继续运行（detached）
// 如果需要在函数退出时取消任务，使用 JoinHandle 并在 Drop 时 abort

struct TaskGuard(tokio::task::JoinHandle<()>);

impl Drop for TaskGuard {
    fn drop(&mut self) {
        self.0.abort(); // 函数退出时自动取消任务
    }
}
```

---

## 快速参考

| 需求                | API                           |
| ------------------- | ----------------------------- |
| 并发运行多个 Future | `tokio::join!`                |
| 竞速，取第一个完成  | `tokio::select!`              |
| 创建独立任务        | `tokio::spawn`                |
| 运行阻塞代码        | `tokio::task::spawn_blocking` |
| 延迟                | `tokio::time::sleep`          |
| 超时                | `tokio::time::timeout`        |
| 定期执行            | `tokio::time::interval`       |
| 多生产者单消费者    | `tokio::sync::mpsc`           |
| 一次性消息          | `tokio::sync::oneshot`        |
| 广播                | `tokio::sync::broadcast`      |
| 状态订阅            | `tokio::sync::watch`          |
| 异步互斥锁          | `tokio::sync::Mutex`          |
| 限制并发            | `tokio::sync::Semaphore`      |

