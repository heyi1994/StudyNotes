# std:Box / Rc / Arc / Cell / RefCell / Weak / Cow

> 智能指针是"行为像指针、但还附带额外能力（自动释放、引用计数、内部可变性……）"的类型。它们是理解 Rust 内存模型、所有权与共享的关键。掌握这一章，很多"借用检查器为什么不让我这么写"的困惑都会迎刃而解。

---

## 目录

- [一、什么是智能指针](#一什么是智能指针)
- [二、全家桶总览](#二全家桶总览)
- [三、Box：堆分配](#三box堆分配)
- [四、Rc：单线程引用计数](#四rc单线程引用计数)
- [五、Arc：多线程引用计数](#五arc多线程引用计数)
- [六、内部可变性：Cell 与 RefCell](#六内部可变性cell-与-refcell)
- [七、组合拳：Rc<RefCell<T>> 与 Arc<Mutex<T>>](#七组合拳rcrefcellt-与-arcmutext)
- [八、Weak：打破循环引用](#八weak打破循环引用)
- [九、Cow：写时克隆](#九cow写时克隆)
- [十、Deref 与 Drop：智能指针的两大基石](#十deref-与-drop智能指针的两大基石)
- [十一、选型决策表](#十一选型决策表)
- [十二、常见陷阱](#十二常见陷阱)

---

## 一、什么是智能指针

普通引用 `&T` 只是借用一个值；**智能指针**则是拥有数据、且实现了特定 trait 从而"表现得像指针"的结构体。它们的共性：

- 实现 `Deref`：可以用 `*` 解引用，像普通指针一样访问内部数据。
- 实现 `Drop`：离开作用域时自动执行清理（释放内存等），这就是 RAII。

为什么需要它们？Rust 的所有权规则很严格——**一个值只能有一个所有者**、**要么多个不可变借用、要么一个可变借用**。但现实中我们需要：

- 在堆上放数据（递归类型、大对象、trait 对象）→ `Box`
- 让多处共享同一份数据 → `Rc` / `Arc`
- 在只有不可变引用时也能改内部 → `Cell` / `RefCell`
- 跨线程共享可变状态 → `Arc<Mutex<T>>`

智能指针就是在**遵守所有权规则的前提下**，把这些能力安全地封装出来。

---

## 二、全家桶总览

| 类型 | 一句话 | 所有权模型 | 线程安全 | 检查时机 |
|------|--------|-----------|----|---------|
| `Box<T>` | 把值放到堆上，独占 | 单一所有者 | 跟随 T | 编译期 |
| `Rc<T>` | 引用计数共享（单线程） | 多所有者 |  否 | 编译期 |
| `Arc<T>` | 引用计数共享（多线程） | 多所有者 |  是 | 编译期 |
| `Cell<T>` | 内部可变（整体存取） | 单一所有者 |  否 | 编译期 |
| `RefCell<T>` | 内部可变（借用检查移到运行时） | 单一所有者 |  否 | **运行期** |
| `Mutex<T>` | 内部可变 + 互斥锁（多线程） | 共享 |  是 | 运行期 |
| `RwLock<T>` | 内部可变 + 读写锁（多线程） | 共享 |  是 | 运行期 |
| `Weak<T>` | 弱引用，不增加强计数 | 不拥有 | 跟随 Rc/Arc | 运行期 |
| `Cow<'a, T>` | 读时借用、写时克隆 | 借用或拥有 | 跟随 T | 编译期 |

两条记忆主线：
- **共享所有权**：`Rc`（单线程）↔ `Arc`（多线程）
- **内部可变性**：`Cell`/`RefCell`（单线程）↔ `Mutex`/`RwLock`（多线程）

---

## 三、Box：堆分配

`Box<T>` 是最简单的智能指针：把一个值放到**堆**上，栈上只留一个指向它的指针。独占所有权，离开作用域时自动释放堆内存。

```rust
let b = Box::new(5);   // 5 被放到堆上
println!("{}", *b);     // 解引用访问：5
// b 离开作用域，堆内存自动释放
```

### Box 的三大用途

**1. 递归类型（编译期必须知道大小）**

链表、树这类递归结构，如果直接嵌套自身，编译器无法计算大小（无限大）。`Box` 是一个固定大小的指针，打破递归：

```rust
//  无法编译：List 的大小无限递归
// enum List { Cons(i32, List), Nil }

//  用 Box 间接持有，大小固定为一个指针
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use List::{Cons, Nil};
let list = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
```

**2. trait 对象（动态分发）**

把不同的具体类型统一成同一个接口，需要在堆上存"某个实现了 trait 的东西"：

```rust
trait Animal {
    fn speak(&self) -> String;
}
struct Dog;
struct Cat;
impl Animal for Dog { fn speak(&self) -> String { "Woof".into() } }
impl Animal for Cat { fn speak(&self) -> String { "Meow".into() } }

// Vec 里装不同类型，用 Box<dyn Animal> 统一
let animals: Vec<Box<dyn Animal>> = vec![Box::new(Dog), Box::new(Cat)];
for a in &animals {
    println!("{}", a.speak());
}
```

**3. 把大对象移到堆上**，避免在栈上移动时大量复制（次要用途）。

---

## 四、Rc：单线程引用计数

`Rc<T>`（Reference Counted）允许**多个所有者共享同一份数据**。它维护一个引用计数，每 `clone` 一次计数 +1，每 drop 一个计数 -1，归零时才释放数据。

适用场景：单线程下，一份数据需要被多个地方持有，且无法确定谁该负责释放（如图、树的共享节点）。

```rust
use std::rc::Rc;

let a = Rc::new(String::from("shared data"));
println!("计数 = {}", Rc::strong_count(&a)); // 1

let b = Rc::clone(&a);   // 计数 +1，注意：只复制指针，不复制数据
println!("计数 = {}", Rc::strong_count(&a)); // 2

{
    let c = Rc::clone(&a);
    println!("计数 = {}", Rc::strong_count(&a)); // 3
} // c 离开作用域，计数 -1

println!("计数 = {}", Rc::strong_count(&a)); // 2
// a、b 共享同一份 "shared data"
```

要点：
- **`Rc::clone` 是浅拷贝**（只增计数、复制指针），开销极小。习惯写 `Rc::clone(&a)` 而非 `a.clone()`，以示区别。
- `Rc<T>` 只给**不可变**访问。要改内部，需配合 `RefCell`（见第七节）。
- **`Rc` 不是线程安全的**——它的计数不是原子操作。跨线程要用 `Arc`。

---

## 五、Arc：多线程引用计数

`Arc<T>`（Atomically Reference Counted）是 `Rc` 的线程安全版本，计数用**原子操作**维护，可以跨线程共享。

```rust
use std::sync::Arc;
use std::thread;

let data = Arc::new(vec![1, 2, 3]);
let mut handles = vec![];

for i in 0..3 {
    let data = Arc::clone(&data);  // 每个线程持有一份 Arc
    let h = thread::spawn(move || {
        println!("线程 {} 看到 {:?}", i, data);
    });
    handles.push(h);
}
for h in handles {
    h.join().unwrap();
}
```

`Rc` vs `Arc`：

| | `Rc<T>` | `Arc<T>` |
|---|---------|----------|
| 计数操作 | 普通整数 | 原子操作 |
| 线程安全 | ❌ | ✅ |
| 性能 | 略快 | 略慢（原子开销） |
| 何时用 | 单线程共享 | 多线程共享 |

> 原则：**能用 `Rc` 就别用 `Arc`**（单线程下 `Rc` 更快）；需要跨线程时才升级到 `Arc`。
> 注意：`Arc<T>` 本身也只给不可变访问，要改内部得配 `Mutex`/`RwLock`。

---

## 六、内部可变性：Cell 与 RefCell

**内部可变性（interior mutability）** 是 Rust 的一个核心模式：在只持有**不可变引用 `&T`** 的情况下，依然能修改内部数据。它通过 `Cell` / `RefCell` 这类类型，把借用规则的检查从编译期挪到运行期（或用整体替换规避）。

为什么需要？有时你的对象逻辑上"对外不可变"，但内部需要缓存、计数等可变状态；或者像 `Rc` 只给 `&T`，你却需要改。

### Cell：适合 Copy 类型，整体存取

`Cell<T>` 不返回内部引用，而是**整体地 get/set 值**，因此没有借用冲突的问题。适合小的 `Copy` 类型。

```rust
use std::cell::Cell;

struct Counter {
    count: Cell<u32>,   // 即使 self 是 &self，也能改 count
}

impl Counter {
    fn increment(&self) {        // 注意是 &self，不可变！
        self.count.set(self.count.get() + 1);
    }
}

let c = Counter { count: Cell::new(0) };
c.increment();
c.increment();
println!("{}", c.count.get()); // 2
```

| 方法 | 作用 |
|------|------|
| `get()` | 返回内部值的拷贝（需 `T: Copy`） |
| `set(v)` | 设置新值 |
| `replace(v)` | 设新值并返回旧值 |
| `take()` | 取出值并留下 `Default` |

### RefCell：适合任意类型，运行期借用检查

`RefCell<T>` 允许你拿到内部数据的**引用**（`&T` 或 `&mut T`），但借用规则改为**运行期检查**：违反"同时只能有一个可变借用或多个不可变借用"会 **panic**，而非编译错误。

```rust
use std::cell::RefCell;

let cell = RefCell::new(vec![1, 2, 3]);

// borrow()：拿不可变引用 Ref<T>，可多个
{
    let r1 = cell.borrow();
    let r2 = cell.borrow();
    println!("{:?} {:?}", r1, r2); // 同时多个不可变借用，OK
}

// borrow_mut()：拿可变引用 RefMut<T>，独占
{
    let mut m = cell.borrow_mut();
    m.push(4);
}

println!("{:?}", cell.borrow()); // [1, 2, 3, 4]

// ⚠️ 运行期 panic 示例：
// let r = cell.borrow();
// let m = cell.borrow_mut(); // panic: already borrowed
```

| 方法 | 作用 | 冲突时 |
|------|------|--------|
| `borrow()` | 返回 `Ref<T>`（不可变） | 已有可变借用则 panic |
| `borrow_mut()` | 返回 `RefMut<T>`（可变） | 已有任何借用则 panic |
| `try_borrow()` / `try_borrow_mut()` | 返回 `Result`，不 panic | 返回 `Err` |

### Cell vs RefCell

| | `Cell<T>` | `RefCell<T>` |
|---|-----------|--------------|
| 访问方式 | 整体 get/set（拷贝/替换） | 借出引用 `Ref`/`RefMut` |
| 类型要求 | get 需 `T: Copy` | 任意类型 |
| 借用检查 | 无（不存在引用冲突） | 运行期检查，违反则 panic |
| 适用 | 小的 Copy 值、标志位、计数 | 复杂数据、需要引用的场景 |

> 二者都**不是线程安全**的。多线程下的内部可变性请用 `Mutex` / `RwLock`。

---

## 七、组合拳：Rc<RefCell<T>> 与 Arc<Mutex<T>>

单个智能指针能力有限，真正的威力来自组合。两个最经典的组合：

### `Rc<RefCell<T>>`：单线程"多所有者 + 可变"

`Rc` 提供多所有者共享，但只给不可变访问；`RefCell` 补上内部可变性。合起来 = **多个所有者都能修改的共享数据**（单线程）。常用于图、树、观察者等结构。

```rust
use std::rc::Rc;
use std::cell::RefCell;

// 多个变量共享、且都能修改同一个 Vec
let shared = Rc::new(RefCell::new(vec![1, 2, 3]));

let a = Rc::clone(&shared);
let b = Rc::clone(&shared);

a.borrow_mut().push(4);   // 通过 a 修改
b.borrow_mut().push(5);   // 通过 b 修改

println!("{:?}", shared.borrow()); // [1, 2, 3, 4, 5]
```

### `Arc<Mutex<T>>`：多线程"共享 + 可变"

跨线程版本：`Arc` 让多个线程共享所有权，`Mutex` 保证同一时刻只有一个线程能改，避免数据竞争。这是 Rust 多线程共享可变状态的**最标准组合**。

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    let h = thread::spawn(move || {
        let mut num = counter.lock().unwrap(); // 加锁
        *num += 1;
    }); // 锁在 num 离开作用域时自动释放
    handles.push(h);
}
for h in handles {
    h.join().unwrap();
}
println!("结果 = {}", *counter.lock().unwrap()); // 10
```

**对应关系一目了然**：

| 单线程 | 多线程 |
|--------|--------|
| `Rc<T>` | `Arc<T>` |
| `RefCell<T>` | `Mutex<T>` / `RwLock<T>` |
| `Rc<RefCell<T>>` | `Arc<Mutex<T>>` |

---

## 八、Weak：打破循环引用

`Rc`/`Arc` 有个隐患：**循环引用会导致内存泄漏**。如果 A 持有 B 的 `Rc`，B 又持有 A 的 `Rc`，二者的强计数永远不会归零，内存永不释放。

`Weak<T>`（弱引用）解决这个问题：它指向数据但**不增加强引用计数**，因此不影响数据是否被释放。访问时需 `upgrade()` 尝试升级成 `Rc`（数据可能已被释放，返回 `Option`）。

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,   // 指向父节点：用 Weak，避免循环
    children: RefCell<Vec<Rc<Node>>>, // 指向子节点：用 Rc，拥有它们
}

let leaf = Rc::new(Node {
    value: 3,
    parent: RefCell::new(Weak::new()),
    children: RefCell::new(vec![]),
});

let branch = Rc::new(Node {
    value: 5,
    parent: RefCell::new(Weak::new()),
    children: RefCell::new(vec![Rc::clone(&leaf)]),
});

// leaf 的 parent 用 Weak 指向 branch（不构成循环）
*leaf.parent.borrow_mut() = Rc::downgrade(&branch);

// 访问父节点：upgrade 返回 Option<Rc<Node>>
if let Some(parent) = leaf.parent.borrow().upgrade() {
    println!("leaf 的父节点 value = {}", parent.value); // 5
}
```

要点：
- **拥有关系用 `Rc`/`Arc`，反向/非拥有关系用 `Weak`**（如子→父、观察者→被观察者）。
- `Rc::downgrade(&rc)` 得到 `Weak`；`weak.upgrade()` 得到 `Option<Rc<T>>`。
- `Rc::weak_count` / `Rc::strong_count` 可查看两种计数。

---

## 九、Cow：写时克隆

`Cow<'a, T>`（Clone on Write，写时克隆）是个智能枚举：要么**借用**数据，要么**拥有**数据。读的时候用借用（零拷贝），只有真正需要修改时才克隆出一份拥有的副本。

```rust
pub enum Cow<'a, B: ?Sized + ToOwned> {
    Borrowed(&'a B),     // 借用，没有分配
    Owned(<B as ToOwned>::Owned), // 拥有，已分配
}
```

经典场景：一个函数大多数情况下不改输入、可以直接返回借用；少数情况才需要修改、返回新分配。用 `Cow` 避免"为了少数情况而总是克隆"的浪费。

```rust
use std::borrow::Cow;

// 把字符串里的空格替换为下划线；没有空格时不分配新内存
fn sanitize(input: &str) -> Cow<str> {
    if input.contains(' ') {
        Cow::Owned(input.replace(' ', "_")) // 需要修改：克隆出新 String
    } else {
        Cow::Borrowed(input)                 // 无需修改：直接借用，零分配
    }
}

let a = sanitize("hello");        // 无空格 → Borrowed，零拷贝
let b = sanitize("hello world");  // 有空格 → Owned，分配一次
println!("{} {}", a, b);          // hello hello_world
```

要点：
- 适合**读多写少**、且"是否需要修改取决于运行时"的场景。
- `Cow` 实现了 `Deref`，可以直接当 `&T` 用（读取时无需关心是哪种变体）。
- `.to_mut()` 会在必要时克隆并返回可变引用；`.into_owned()` 取得拥有的值。

---

## 十、Deref 与 Drop：智能指针的两大基石

所有智能指针之所以"像指针、能自动清理"，靠的是这两个 trait。

### Deref：让类型表现得像引用

实现 `Deref` 后，`*x` 会被转成 `*(x.deref())`，并且支持**自动解引用强制转换（deref coercion）**——`&Box<String>` 能自动当 `&str` 用：

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> { MyBox(x) }
}

impl<T> Deref for MyBox<T> {
    type Target = T;
    fn deref(&self) -> &T { &self.0 }
}

let b = MyBox::new(5);
println!("{}", *b); // 5，等价于 *(b.deref())

// deref coercion：&MyBox<String> 自动转成 &str
fn hello(name: &str) { println!("Hello, {}!", name); }
let m = MyBox::new(String::from("Rust"));
hello(&m); // &MyBox<String> → &String → &str，自动完成
```

这就是为什么你能直接在 `Box<T>`、`Rc<T>` 上调用 `T` 的方法——deref coercion 自动帮你解开层层包装。

### Drop：离开作用域时自动清理

实现 `Drop` 的 `drop` 方法，会在值离开作用域时**自动调用**，用来释放资源（内存、文件句柄、锁等）。这就是 Rust 的 RAII。

```rust
struct Resource {
    name: String,
}

impl Drop for Resource {
    fn drop(&mut self) {
        println!("释放资源: {}", self.name);
    }
}

fn main() {
    let _a = Resource { name: "A".into() };
    let _b = Resource { name: "B".into() };
    println!("main 结束");
} // 输出顺序：main 结束 → 释放资源: B → 释放资源: A（后创建的先释放）
```

要点：
- **不能手动调用 `.drop()`**；需要提前释放用 `std::mem::drop(x)`（即 `drop(x)`）。
- 释放顺序是**后进先出**（与创建相反）。
- `Box`/`Rc`/`Arc` 等正是在自己的 `Drop` 里释放堆内存 / 减少计数。

---

## 十一、选型决策表

按需求快速定位：

| 你的需求 | 选择 |
|---------|------|
| 在堆上放一个值（递归类型/trait 对象） | `Box<T>` |
| 单线程，多处共享同一份**只读**数据 | `Rc<T>` |
| 多线程，多处共享同一份**只读**数据 | `Arc<T>` |
| 单线程，`&self` 下改一个 Copy 小值/计数 | `Cell<T>` |
| 单线程，`&self` 下改复杂数据（要引用） | `RefCell<T>` |
| 单线程，多所有者**且都要改** | `Rc<RefCell<T>>` |
| 多线程，共享**且要改**（互斥） | `Arc<Mutex<T>>` |
| 多线程，共享且要改（读多写少） | `Arc<RwLock<T>>` |
| 避免 Rc/Arc 循环引用（反向指针） | `Weak<T>` |
| 读多写少、按需才克隆 | `Cow<'a, T>` |

**两条主线**再强调一遍：
- 共享所有权：`Rc`（单线程） / `Arc`（多线程）
- 内部可变性：`Cell`·`RefCell`（单线程） / `Mutex`·`RwLock`（多线程）

---

## 十二、常见陷阱

1. **`Rc`/`Arc` 循环引用导致内存泄漏**
   双向关系中，一个方向用 `Weak` 打破循环。

2. **`RefCell` 借用冲突在运行期 panic**
   `borrow` 和 `borrow_mut` 同时存在会 panic（而非编译错误）。把内部可变性的借用检查从编译期推迟到了运行期，需自己小心，必要时用 `try_borrow`。

3. **在多线程里误用 `Rc` / `RefCell`**
   它们不是 `Send`/`Sync`，编译器会拦住你。跨线程换 `Arc` / `Mutex`。

4. **`Mutex` 死锁**
   同一线程重复锁同一个 `Mutex`、或多个锁加锁顺序不一致，会死锁。固定加锁顺序、缩小锁的作用域、用 `{}` 让锁尽早释放。

5. **`Mutex` 的"中毒"（poisoning）**
   持锁线程 panic 后，锁被标记为 poisoned，之后 `lock()` 返回 `Err`。通常 `.unwrap()` 让其传播，或用 `.into_inner()` 恢复。

6. **`Rc::clone` 不是深拷贝**
   它只增加计数、共享同一份数据。想要独立副本要克隆内部数据本身。

7. **过度包装**
   不要无脑套 `Rc<RefCell<...>>`。先想能否用普通所有权/借用解决——很多时候重构数据结构比堆叠智能指针更好。

---

## 附：速查总结

```
堆分配         Box<T>            独占所有权，递归类型 / trait 对象

共享所有权     Rc<T>   单线程     Rc::clone 增计数（浅拷贝）
              Arc<T>  多线程     原子计数，可跨线程

内部可变性     Cell<T>           整体 get/set，Copy 小值
              RefCell<T>        借出 Ref/RefMut，运行期检查（违规 panic）
              Mutex<T>          多线程互斥锁
              RwLock<T>         多线程读写锁

经典组合       Rc<RefCell<T>>    单线程：多所有者 + 可变
              Arc<Mutex<T>>     多线程：共享 + 可变

弱引用         Weak<T>           不增强计数，打破循环；upgrade() → Option<Rc>
写时克隆       Cow<'a,T>         读借用、写才克隆，读多写少

基石 trait     Deref             像指针一样解引用 + 自动强制转换
              Drop              离开作用域自动清理（RAII，后进先出）

两条主线：共享 = Rc/Arc；可变 = Cell·RefCell / Mutex·RwLock
        单线程用左边，多线程用右边。
```
