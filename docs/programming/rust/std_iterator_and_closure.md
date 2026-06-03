# std:Iterator & Closure

> 迭代器和闭包是写出"地道 Rust"的两大关键。它们配合在一起，构成了 Rust 中**零成本抽象（zero-cost abstraction）**的典范——写起来像高级语言的链式表达，编译后却和手写循环一样快。

---

## 目录

- [第一部分：闭包（Closure）](#第一部分闭包closure)
  - [一、什么是闭包](#一什么是闭包)
  - [二、闭包如何捕获环境](#二闭包如何捕获环境)
  - [三、三个闭包 trait：Fn / FnMut / FnOnce](#三三个闭包-trait-fn--fnmut--fnonce)
  - [四、move 关键字](#四move-关键字)
  - [五、闭包作为参数与返回值](#五闭包作为参数与返回值)
- [第二部分：迭代器（Iterator）](#第二部分迭代器iterator)
  - [六、Iterator trait 的本质](#六iterator-trait-的本质)
  - [七、惰性求值与三类方法](#七惰性求值与三类方法)
  - [八、适配器方法大全（lazy）](#八适配器方法大全lazy)
  - [九、消费者方法大全（eager）](#九消费者方法大全eager)
  - [十、IntoIterator 与 for 循环](#十intoiterator-与-for-循环)
  - [十一、三种迭代方式：iter / iter_mut / into_iter](#十一三种迭代方式iter--iter_mut--into_iter)
  - [十二、为自定义类型实现 Iterator](#十二为自定义类型实现-iterator)
- [第三部分：实战与性能](#第三部分实战与性能)
  - [十三、常见组合套路](#十三常见组合套路)
  - [十四、性能：零成本抽象](#十四性能零成本抽象)
  - [十五、常见陷阱](#十五常见陷阱)

---

# 第一部分：闭包（Closure）

## 一、什么是闭包

闭包是**可以捕获其所在环境变量的匿名函数**。语法用 `|参数| 表达式`：

```rust
let x = 5;

// 普通函数无法捕获 x；闭包可以
let add_x = |n| n + x;   // 捕获了环境中的 x
println!("{}", add_x(3)); // 8
```

与普通函数的区别：

| 特性 | 普通函数 `fn` | 闭包 |
|------|-------------|-----|
| 捕获环境变量 |  不能 |  能 |
| 类型标注 | 必须写 | 通常可省略（编译器推断） |
| 是否匿名 | 有名字 | 匿名（可绑定到变量） |

完整 vs 简写：

```rust
// 完整写法
let f = |x: i32| -> i32 { x + 1 };
// 省略类型 + 单表达式省略大括号
let f = |x| x + 1;
```

> 注意：闭包的参数类型一旦被首次调用推断确定，就固定了，不能再用别的类型调用。

---

## 二、闭包如何捕获环境

闭包根据**对变量的使用方式**，自动选择三种捕获方式之一（按"侵入性从小到大"）：

| 捕获方式 | 何时发生 | 对应的借用 |
|---------|---------|-----------|
| 不可变借用 `&T` | 只读取变量 | `&` |
| 可变借用 `&mut T` | 修改变量 | `&mut` |
| 取得所有权 `T` | 需要移动/消耗变量，或用 `move` | 移动 |

编译器**优先选择侵入性最小**的方式（能借用就不移动）。

```rust
let list = vec![1, 2, 3];

// 1. 不可变借用：只是读
let only_read = || println!("{:?}", list);
only_read();
println!("{:?}", list); // 仍可用

let mut list = vec![1, 2, 3];

// 2. 可变借用：修改了 list
let mut push_one = || list.push(4);
push_one();
println!("{:?}", list); // [1, 2, 3, 4]
```

---

## 三、三个闭包 trait：Fn / FnMut / FnOnce

这是闭包最核心、也最容易混淆的知识点。每个闭包会**自动实现**下面一到三个 trait，取决于它**如何使用捕获的变量**：

| Trait | 含义 | 调用方式 | 能调用几次 |
|-------|------|---------|-----------|
| `FnOnce` | 会**消耗**捕获的值（移动出去） | `self` | **至少一次**（可能只能一次） |
| `FnMut` | 会**修改**捕获的值 | `&mut self` | 多次 |
| `Fn` | 只**读取**捕获的值 | `&self` | 多次 |

**包含关系（重点）**：

```
Fn  ⊂  FnMut  ⊂  FnOnce
```

- 所有闭包都实现 `FnOnce`（任何东西至少能被调用一次）。
- 只读 / 可变的闭包还额外实现 `FnMut`。
- 只读闭包再额外实现 `Fn`。

也就是说：一个 `Fn` 闭包，可以传给要求 `Fn`、`FnMut` 或 `FnOnce` 的任何地方；而一个 `FnOnce` 闭包只能传给要求 `FnOnce` 的地方。

```rust
// FnOnce：把捕获的 String 移动出去（消耗掉）
let s = String::from("hello");
let consume = move || s;     // 返回 s，把所有权交出去
let owned = consume();        // 只能调用一次
// consume();                 // ❌ 第二次调用报错：s 已被移动

// FnMut：修改捕获变量
let mut count = 0;
let mut inc = || { count += 1; };
inc(); inc();                 // 可多次调用
println!("{}", count);        // 2

// Fn：只读
let factor = 10;
let multiply = |x| x * factor;
println!("{}", multiply(3));  // 30，可多次
println!("{}", multiply(4));  // 40
```

**如何判断一个闭包是哪种？** 看它对捕获变量做了什么：
- 把变量移动走 / `drop` 掉 → `FnOnce`
- 修改了变量 → `FnMut`
- 只读取 → `Fn`

---

## 四、move 关键字

`move` 强制闭包**取得所有捕获变量的所有权**（而非借用），即使它本来只需要借用。

```rust
let data = vec![1, 2, 3];

// 不加 move：闭包借用 data，但下面 spawn 要求 'static，会报错
// 加 move：data 的所有权移入闭包，可安全跨线程
let handle = std::thread::spawn(move || {
    println!("{:?}", data);
});
handle.join().unwrap();
// 此处 data 已被移动，不能再用
```

**最常见用途**：
1. 把闭包传给线程（`thread::spawn`）——要求闭包是 `'static`，必须拥有数据。
2. 把闭包作为返回值——闭包活得比当前函数久，不能借用局部变量。

> 注意：`move` 只改变"如何捕获"（借用→拥有），不改变"是哪种 Fn trait"。一个 `move` 闭包若只读取数据，仍然是 `Fn`。

---

## 五、闭包作为参数与返回值

### 作为参数

两种写法：泛型（静态分发，零开销，推荐）或 trait 对象（动态分发）：

```rust
// 1. 泛型 + trait bound（静态分发，最常用）
fn apply<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
    f(x)
}

// 1'. impl Trait 简写，效果相同
fn apply2(f: impl Fn(i32) -> i32, x: i32) -> i32 {
    f(x)
}

// 2. trait 对象（动态分发，需要 Box，运行时有一点开销）
fn apply3(f: &dyn Fn(i32) -> i32, x: i32) -> i32 {
    f(x)
}

apply(|n| n * 2, 5); // 10
```

选 bound 的原则：**能不能多次调用、要不要改环境**，对应选 `Fn` / `FnMut` / `FnOnce`。一般优先写 `Fn`，需要时再放宽。

### 作为返回值

返回闭包用 `impl Fn...`（编译期已知具体类型）或 `Box<dyn Fn...>`（需要返回不同闭包时）：

```rust
// impl Trait：返回单一具体闭包类型
fn make_adder(x: i32) -> impl Fn(i32) -> i32 {
    move |n| n + x   // 必须 move，否则 x 被借用却已离开作用域
}

// Box<dyn>：可在不同分支返回不同闭包
fn make_op(add: bool) -> Box<dyn Fn(i32) -> i32> {
    if add {
        Box::new(|x| x + 1)
    } else {
        Box::new(|x| x - 1)
    }
}

let add5 = make_adder(5);
println!("{}", add5(10)); // 15
```

---

# 第二部分：迭代器（Iterator）

## 六、Iterator trait 的本质

迭代器的核心是一个出奇简单的 trait——只需实现一个方法 `next`：

```rust
pub trait Iterator {
    type Item;                          // 关联类型：每次产出的元素类型

    fn next(&mut self) -> Option<Self::Item>;  // 唯一必须实现的方法

    // ……标准库基于 next 默认实现了几十个方法（map/filter/fold...）
}
```

- `next()` 每次返回 `Some(元素)`，迭代结束返回 `None`。
- 只要实现了 `next`，就**自动获得**几十个适配器和消费方法。

手动驱动一个迭代器：

```rust
let v = vec![1, 2, 3];
let mut it = v.iter();
assert_eq!(it.next(), Some(&1));
assert_eq!(it.next(), Some(&2));
assert_eq!(it.next(), Some(&3));
assert_eq!(it.next(), None);
```

---

## 七、惰性求值与三类方法

迭代器是**惰性的（lazy）**：只构建链条不会做任何实际工作，直到一个"消费者"来拉取元素。

```rust
let v = vec![1, 2, 3];

// 只是构建了一个迭代器，map 里的闭包一次都没执行！
let it = v.iter().map(|x| {
    println!("processing {}", x);
    x * 2
});
// 这里什么都不会打印

// 直到 collect 来消费，才真正执行
let doubled: Vec<i32> = it.collect(); // 此时才打印三行
```

把迭代器方法分成三类来记：

| 类别 | 作用 | 返回 | 例子 |
|------|------|------|------|
| **生产者** | 创建迭代器 | `Iterator` | `iter()` / `0..10` / `into_iter()` |
| **适配器（lazy）** | 迭代器 → 新迭代器 | `Iterator` | `map` / `filter` / `take` / `zip` |
| **消费者（eager）** | 迭代器 → 最终结果 | 具体值 | `collect` / `sum` / `fold` / `for_each` |

> 口诀：**适配器返回迭代器（链下去），消费者返回结果（终结链）。** 一条链通常是「生产者 → 若干适配器 → 一个消费者」。

---

## 八、适配器方法大全（lazy）

这些方法返回新的迭代器，可以继续链式调用：

| 方法 | 作用 | 示例 |
|------|------|------|
| `map(f)` | 对每个元素变换 | `.map(\|x\| x * 2)` |
| `filter(p)` | 保留满足条件的元素 | `.filter(\|x\| x % 2 == 0)` |
| `filter_map(f)` | map + 过滤 None，一步完成 | `.filter_map(\|s\| s.parse().ok())` |
| `flat_map(f)` | map 后把嵌套迭代器摊平 | `.flat_map(\|x\| vec![x, x])` |
| `flatten()` | 摊平一层嵌套结构 | `.flatten()` |
| `enumerate()` | 配上索引，产出 `(i, item)` | `.enumerate()` |
| `zip(other)` | 与另一个迭代器配对 | `a.iter().zip(b.iter())` |
| `take(n)` | 只取前 n 个 | `.take(3)` |
| `skip(n)` | 跳过前 n 个 | `.skip(2)` |
| `take_while(p)` | 取到不满足条件为止 | `.take_while(\|x\| *x < 5)` |
| `skip_while(p)` | 跳过开头满足条件的 | `.skip_while(\|x\| *x < 5)` |
| `step_by(n)` | 每隔 n 个取一个 | `.step_by(2)` |
| `chain(other)` | 把两个迭代器首尾相接 | `a.chain(b)` |
| `rev()` | 反向迭代（需双端迭代器） | `.rev()` |
| `peekable()` | 可 `peek()` 预览下一个 | `.peekable()` |
| `scan(state, f)` | 带状态的 map | 累计、运行总和 |
| `inspect(f)` | 偷看每个元素（调试用），不改变 | `.inspect(\|x\| println!("{x}"))` |
| `cloned()` / `copied()` | `&T` → `T`（克隆/复制） | `.copied()` |
| `cycle()` | 无限循环重复 | `.cycle().take(10)` |

链式示例：

```rust
let result: Vec<i32> = (1..=10)
    .filter(|x| x % 2 == 0)   // 2,4,6,8,10
    .map(|x| x * x)           // 4,16,36,64,100
    .take(3)                  // 4,16,36
    .collect();
assert_eq!(result, vec![4, 16, 36]);
```

---

## 九、消费者方法大全（eager）

这些方法会"拉完"迭代器，返回最终结果：

| 方法 | 作用 | 返回 |
|------|------|------|
| `collect()` | 收集成集合（Vec/HashMap/String...） | 集合 |
| `sum()` / `product()` | 求和 / 求积 | 数值 |
| `count()` | 计数 | `usize` |
| `min()` / `max()` | 最小 / 最大 | `Option<T>` |
| `min_by_key(f)` / `max_by_key(f)` | 按 key 求最值 | `Option<T>` |
| `fold(init, f)` | 累积折叠（万能消费者） | 累积值 |
| `reduce(f)` | 用首元素作初值的 fold | `Option<T>` |
| `for_each(f)` | 对每个元素执行副作用 | `()` |
| `find(p)` | 找第一个满足条件的 | `Option<T>` |
| `position(p)` | 找第一个满足条件的索引 | `Option<usize>` |
| `any(p)` | 是否存在满足条件的 | `bool` |
| `all(p)` | 是否全部满足 | `bool` |
| `last()` | 最后一个元素 | `Option<T>` |
| `nth(n)` | 第 n 个元素 | `Option<T>` |
| `partition(p)` | 按条件分成两个集合 | `(A, B)` |
| `unzip()` | `(A,B)` 迭代器 → 两个集合 | `(Vec, Vec)` |

### `fold` —— 万能消费者

很多消费者本质都是 `fold` 的特例，理解它就理解了一大半：

```rust
// fold(初始值, |累加器, 当前元素| 新累加器)
let sum = (1..=5).fold(0, |acc, x| acc + x);     // 15
let product = (1..=5).fold(1, |acc, x| acc * x); // 120

// 用 fold 拼字符串
let s = ["a", "b", "c"]
    .iter()
    .fold(String::new(), |mut acc, x| {
        acc.push_str(x);
        acc
    });
assert_eq!(s, "abc");
```

### `collect` 的强大之处

`collect` 能根据**目标类型**收集成不同集合（涡轮鱼 `::<>` 或类型标注指定）：

```rust
use std::collections::HashMap;

// 收集成 Vec
let v: Vec<i32> = (1..=3).collect();

// 收集成 String
let s: String = vec!['h', 'i'].into_iter().collect();

// 收集成 HashMap（从元组迭代器）
let map: HashMap<&str, i32> = vec![("a", 1), ("b", 2)].into_iter().collect();

// 收集 Result：任一失败则整体 Err（非常实用）
let nums: Result<Vec<i32>, _> = vec!["1", "2", "3"]
    .iter()
    .map(|s| s.parse::<i32>())
    .collect();   // Ok(vec![1, 2, 3])
```

> `Result`/`Option` 的 `collect` 是个隐藏神技：`Iterator<Item = Result<T, E>>` 可以直接 `collect` 成 `Result<Vec<T>, E>`，遇到第一个 `Err` 就短路返回。

---

## 十、IntoIterator 与 for 循环

`for` 循环本质是 `IntoIterator` 的语法糖：

```rust
let v = vec![1, 2, 3];

// 你写的
for x in &v {
    println!("{}", x);
}

// 编译器展开成（约等于）
let mut it = (&v).into_iter();
while let Some(x) = it.next() {
    println!("{}", x);
}
```

`IntoIterator` 定义了"如何把某个类型变成迭代器"。任何实现了它的类型都能直接用在 `for` 中。

---

## 十一、三种迭代方式：iter / iter_mut / into_iter

对集合迭代有三种方式，区别在于**产出元素的所有权**：

| 方法 | 产出 `Item` | 对原集合 | 用途 |
|------|------------|---------|------|
| `iter()` | `&T`（不可变引用） | 借用，之后还能用 | 只读遍历 |
| `iter_mut()` | `&mut T`（可变引用） | 可变借用 | 原地修改 |
| `into_iter()` | `T`（值本身） | **消耗**集合 | 拿走所有权 |

```rust
let v = vec![1, 2, 3];

// iter()：&i32
for x in v.iter() { print!("{} ", x); }   // 1 2 3
println!("{:?}", v);  // v 还在

let mut v = vec![1, 2, 3];

// iter_mut()：&mut i32，可原地改
for x in v.iter_mut() { *x *= 10; }
println!("{:?}", v);  // [10, 20, 30]

// into_iter()：i32，消耗 v
for x in v.into_iter() { print!("{} ", x); }  // 10 20 30
// println!("{:?}", v); // ❌ v 已被消耗
```

**`for` 循环的简写对应关系**：
- `for x in &v` ≡ `v.iter()` → `&T`
- `for x in &mut v` ≡ `v.iter_mut()` → `&mut T`
- `for x in v` ≡ `v.into_iter()` → `T`（消耗 v）

---

## 十二、为自定义类型实现 Iterator

只需实现 `next`，就能享受全部适配器/消费者方法。下面实现一个斐波那契迭代器：

```rust
struct Fibonacci {
    curr: u64,
    next: u64,
}

impl Iterator for Fibonacci {
    type Item = u64;

    fn next(&mut self) -> Option<u64> {
        let new_next = self.curr + self.next;
        self.curr = self.next;
        self.next = new_next;
        Some(self.curr)   // 永不返回 None → 无限迭代器
    }
}

fn fib() -> Fibonacci {
    Fibonacci { curr: 0, next: 1 }
}

fn main() {
    // 因为实现了 Iterator，所有适配器/消费者都能用！
    let first10: Vec<u64> = fib().take(10).collect();
    println!("{:?}", first10); // [1, 1, 2, 3, 5, 8, 13, 21, 34, 55]

    let sum: u64 = fib().take_while(|&x| x < 100).sum();
    println!("{}", sum);
}
```

让你的集合类型支持 `for`，则实现 `IntoIterator`（通常委托给内部 `Vec` 的迭代器即可）。

---

# 第三部分：实战与性能

## 十三、常见组合套路

```rust
use std::collections::HashMap;

// 1. 解析一批字符串为数字，跳过非法的
let nums: Vec<i32> = ["1", "x", "3", "4"]
    .iter()
    .filter_map(|s| s.parse().ok())   // 非法的变 None 被过滤
    .collect();                        // [1, 3, 4]

// 2. 词频统计
let text = "a b a c b a";
let mut freq: HashMap<&str, i32> = HashMap::new();
for word in text.split_whitespace() {
    *freq.entry(word).or_insert(0) += 1;
}
// {"a": 3, "b": 2, "c": 1}

// 3. 找最大值及其位置
let data = vec![3, 7, 2, 9, 4];
let (idx, max) = data.iter()
    .enumerate()
    .max_by_key(|(_, &v)| v)
    .unwrap();
println!("max {} at index {}", max, idx); // max 9 at index 3

// 4. 两个序列逐元素相加
let a = vec![1, 2, 3];
let b = vec![10, 20, 30];
let sums: Vec<i32> = a.iter().zip(b.iter()).map(|(x, y)| x + y).collect();
// [11, 22, 33]

// 5. 分组：奇偶分流
let (evens, odds): (Vec<i32>, Vec<i32>) =
    (1..=10).partition(|x| x % 2 == 0);
```

---

## 十四、性能：零成本抽象

迭代器链**不比手写循环慢**。编译器会把适配器链内联、优化成等价的循环，没有中间集合、没有额外分配：

```rust
// 这段链式代码
let sum: u64 = (1..=1000)
    .filter(|x| x % 2 == 0)
    .map(|x| x * x)
    .sum();

// 编译优化后 ≈ 手写的
let mut sum: u64 = 0;
for x in 1..=1000 {
    if x % 2 == 0 {
        sum += x * x;
    }
}
```

所以在 Rust 里**优先用迭代器链**，既安全、可读，又没有性能损失。不必为了"性能"退回手写 index 循环。

---

## 十五、常见陷阱

1. **忘了消费 → 适配器不执行**
   只写 `v.iter().map(...)` 而没有 `collect`/`for_each`/`sum` 等，闭包根本不会运行。编译器通常会给 `unused_must_use` 警告。

2. **`iter()` 给的是引用，注意解引用**
   `v.iter()` 产出 `&i32`，在闭包里常要写 `|&x|` 或 `*x`：
   ```rust
   v.iter().filter(|&&x| x > 0)   // 双重引用，模式 &&x 解开
   v.iter().map(|x| *x * 2)        // 或显式解引用
   ```

3. **`into_iter` 会消耗集合**
   用完后原集合不可再用。只读遍历请用 `iter()`。

4. **无限迭代器必须配 `take`/`take_while`**
   像 `(0..)` 或自定义无限迭代器，忘了限制会死循环。

5. **`collect` 需要类型标注**
   编译器不知道你要收集成什么，用 `let x: Vec<_> =` 或 `.collect::<Vec<_>>()` 指明。

6. **闭包 trait 选太严**
   函数参数写 `Fn` 但传入的闭包需要修改环境（`FnMut`）会报错；按需放宽到 `FnMut`/`FnOnce`。

---

## 附：速查总结

```
闭包 trait：  Fn ⊂ FnMut ⊂ FnOnce
            只读   改环境    消耗环境
            &self  &mut self  self

迭代器三段式：生产者 → 适配器（lazy） → 消费者（eager）
             iter()   map/filter...     collect/sum/fold

三种迭代：   iter()      → &T    （借用，只读）
            iter_mut()  → &mut T （借用，可改）
            into_iter() → T      （消耗，拿走所有权）

核心心法：迭代器是惰性的，不消费就不执行；
        迭代器链是零成本抽象，放心用。
```
