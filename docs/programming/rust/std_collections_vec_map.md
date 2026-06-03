# std:Vec / HashMap / BTreeMap / HashSet

> `Vec` 和 `HashMap` 是日常 Rust 代码里出现频率最高的两种集合。它们都是**拥有数据、可增长的堆分配容器**。掌握它们的常用方法、所有权细节（尤其是 `entry` API 和借用规则），能让你写出既地道又高效的代码。本篇也顺带覆盖 `BTreeMap`、`HashSet`、`VecDeque` 等亲戚。

---

## 目录

- [一、Vec：动态数组](#一vec动态数组)
  - [1.1 创建与基本操作](#11-创建与基本操作)
  - [1.2 访问元素：index vs get](#12-访问元素index-vs-get)
  - [1.3 遍历与所有权](#13-遍历与所有权)
  - [1.4 增删改与常用方法](#14-增删改与常用方法)
  - [1.5 容量与性能](#15-容量与性能)
- [二、HashMap：哈希表](#二hashmap哈希表)
  - [2.1 创建与基本操作](#21-创建与基本操作)
  - [2.2 entry API（重点）](#22-entry-api重点)
  - [2.3 遍历与所有权](#23-遍历与所有权)
- [三、BTreeMap：有序映射](#三btreemap有序映射)
- [四、HashSet 与 BTreeSet](#四hashset-与-btreeset)
- [五、VecDeque / BinaryHeap 等](#五vecdeque--binaryheap-等)
- [六、集合与迭代器：collect 的威力](#六集合与迭代器collect-的威力)
- [七、选型决策表](#七选型决策表)
- [八、常见陷阱](#八常见陷阱)

---

## 一、Vec：动态数组

`Vec<T>` 是可增长的数组，元素在堆上连续存储。它是 Rust 用得最多的集合。

### 1.1 创建与基本操作

```rust
// 多种创建方式
let mut v: Vec<i32> = Vec::new();      // 空，需类型标注
let mut v = vec![1, 2, 3];             // vec! 宏，最常用
let v = vec![0; 5];                    // [0, 0, 0, 0, 0]，重复 5 个
let v: Vec<i32> = (1..=5).collect();   // 从迭代器收集

// 基本操作
let mut v = vec![1, 2, 3];
v.push(4);          // 尾部添加：[1,2,3,4]
let last = v.pop(); // 尾部弹出：Some(4)，v 变 [1,2,3]
println!("长度 {}, 是否空 {}", v.len(), v.is_empty());
```

### 1.2 访问元素：index vs get

两种方式，**越界行为不同**：

```rust
let v = vec![10, 20, 30];

// 1. 索引语法：越界直接 panic
let x = v[1];          // 20
// let y = v[10];      // ❌ panic: index out of bounds

// 2. get：返回 Option，越界得 None（安全）
let x: Option<&i32> = v.get(1);   // Some(&20)
let y: Option<&i32> = v.get(10);  // None

// 安全访问的惯用写法
if let Some(val) = v.get(10) {
    println!("{}", val);
} else {
    println!("越界了");
}
```

> 经验：能确定不越界、越界即 bug 时用 `v[i]`；索引来自外部/不确定时用 `v.get(i)`。

### 1.3 遍历与所有权

```rust
let v = vec![1, 2, 3];

// 不可变借用：&i32
for x in &v {
    println!("{}", x);
}
println!("{:?}", v);   // v 还能用

let mut v = vec![1, 2, 3];

// 可变借用：&mut i32，原地修改
for x in &mut v {
    *x *= 10;
}
println!("{:?}", v);   // [10, 20, 30]

// 消耗：i32（拿走所有权）
for x in v {
    println!("{}", x);
}
// v 已被消耗，不能再用
```

> ⚠️ **遍历时不能修改长度**：在 `for x in &v` 期间调用 `v.push(...)` 会触发借用冲突（编译错误）。需要边遍历边改结构时，见[陷阱](#八常见陷阱)。

### 1.4 增删改与常用方法

| 方法 | 作用 |
|------|------|
| `push(x)` / `pop()` | 尾部加 / 弹出（`pop` 返回 `Option`） |
| `insert(i, x)` | 在索引 i 处插入（后面元素后移，O(n)） |
| `remove(i)` | 移除索引 i 的元素并返回（后面前移，O(n)） |
| `swap_remove(i)` | 用末尾元素替换 i 并移除（O(1)，但打乱顺序） |
| `extend(iter)` | 把另一个迭代器的元素全加进来 |
| `truncate(n)` | 截断到 n 个元素 |
| `clear()` | 清空 |
| `contains(&x)` | 是否包含某值 |
| `sort()` / `sort_by` / `sort_by_key` | 排序 |
| `dedup()` | 去除**连续**重复（常配合 `sort` 先排序） |
| `retain(p)` | 原地保留满足条件的元素（高效过滤） |
| `reverse()` | 反转 |
| `iter().position(p)` | 找第一个满足条件的索引 |

```rust
let mut v = vec![3, 1, 4, 1, 5, 9, 2, 6];

v.sort();                      // [1,1,2,3,4,5,6,9]
v.dedup();                     // [1,2,3,4,5,6,9]（去连续重复）
v.retain(|&x| x % 2 == 0);     // 只保留偶数：[2,4,6]

// 按 key 排序（常用）
let mut words = vec!["apple", "hi", "banana"];
words.sort_by_key(|s| s.len()); // 按长度：["hi","apple","banana"]

// 原地过滤比 collect 新 Vec 更高效
let mut nums = vec![1, 2, 3, 4, 5, 6];
nums.retain(|&x| x > 3);       // [4, 5, 6]
```

### 1.5 容量与性能

`Vec` 有 **长度（len）** 和 **容量（capacity）** 两个概念：容量是已分配的空间，长度是实际元素数。容量不够时会**重新分配**（通常翻倍）并搬移数据。

```rust
let mut v = Vec::with_capacity(10);  // 预分配 10 的容量，避免多次扩容
println!("len={}, cap={}", v.len(), v.capacity()); // len=0, cap=10
for i in 0..10 {
    v.push(i);   // 不触发重新分配
}
```

性能要点：
- 已知大致元素数量时，用 `Vec::with_capacity(n)` **预分配**，避免反复扩容搬移。
- 尾部 `push`/`pop` 是 O(1)（摊还）；中间 `insert`/`remove` 是 O(n)。
- 不在乎顺序时，用 `swap_remove`（O(1)）代替 `remove`（O(n)）。

---

## 二、HashMap：哈希表

`HashMap<K, V>` 存储键值对，通过哈希实现平均 O(1) 的查找/插入。键 `K` 必须实现 `Eq + Hash`。

> 注意：`HashMap` 不在 prelude 里，要 `use std::collections::HashMap;`。

### 2.1 创建与基本操作

```rust
use std::collections::HashMap;

let mut scores: HashMap<String, i32> = HashMap::new();

// 插入
scores.insert(String::from("Alice"), 90);
scores.insert(String::from("Bob"), 85);
scores.insert(String::from("Alice"), 95); // 同 key 覆盖：Alice 变 95

// 查询：get 返回 Option<&V>
if let Some(score) = scores.get("Alice") {
    println!("Alice: {}", score);  // 95
}

// 取值带默认
let carol = scores.get("Carol").copied().unwrap_or(0); // 0

// 是否包含 key
println!("{}", scores.contains_key("Bob")); // true

// 删除，返回被删的值
let removed = scores.remove("Bob"); // Some(85)

// 数量
println!("{}", scores.len());
```

从迭代器构建：

```rust
use std::collections::HashMap;

// 从元组数组收集
let map: HashMap<&str, i32> =
    [("a", 1), ("b", 2), ("c", 3)].into_iter().collect();

// 两个 Vec zip 成 map
let keys = vec!["x", "y"];
let vals = vec![10, 20];
let map: HashMap<_, _> = keys.into_iter().zip(vals).collect();
```

### 2.2 entry API（重点）

`entry` 是 `HashMap` 最强大、最该掌握的 API：**一次查找完成"有则取、无则插"**，避免重复哈希和借用冲突。

```rust
use std::collections::HashMap;

let mut map: HashMap<String, i32> = HashMap::new();

// or_insert：key 不存在则插入默认值，返回对值的可变引用
map.entry("a".to_string()).or_insert(0);

// 经典：计数器 / 词频统计
let text = "the quick brown the lazy the";
let mut freq: HashMap<&str, i32> = HashMap::new();
for word in text.split_whitespace() {
    *freq.entry(word).or_insert(0) += 1;   // 不存在初始化为 0，再 +1
}
println!("{:?}", freq); // {"the": 3, "quick": 1, ...}

// or_insert_with：默认值需要计算时用闭包（惰性，只在缺失时执行）
let mut groups: HashMap<bool, Vec<i32>> = HashMap::new();
for n in [1, 2, 3, 4, 5] {
    groups.entry(n % 2 == 0).or_insert_with(Vec::new).push(n);
}
// {false: [1,3,5], true: [2,4]}

// and_modify + or_insert：存在则改、不存在则插
let mut m: HashMap<&str, i32> = HashMap::new();
m.entry("x").and_modify(|v| *v += 1).or_insert(1); // 不存在 → 插 1
m.entry("x").and_modify(|v| *v += 1).or_insert(1); // 存在 → 变 2
```

| entry 方法 | 作用 |
|-----------|------|
| `or_insert(v)` | 缺失则插入 v，返回 `&mut V` |
| `or_insert_with(f)` | 缺失则插入 `f()`（惰性计算），返回 `&mut V` |
| `or_default()` | 缺失则插入 `V::default()` |
| `and_modify(f)` | 存在则用 `f` 修改，可与上面链式组合 |

> **为什么要用 entry？** 不用它的话，"先 `get` 判断再 `insert`"会查两次哈希，而且容易踩借用冲突。`entry` 一次搞定，更快更安全。

### 2.3 遍历与所有权

```rust
use std::collections::HashMap;

let map: HashMap<&str, i32> = [("a", 1), ("b", 2)].into_iter().collect();

// 遍历键值对（顺序随机！）
for (k, v) in &map {
    println!("{} = {}", k, v);
}

// 只遍历键 / 值
for k in map.keys() { println!("{}", k); }
for v in map.values() { println!("{}", v); }

// 可变遍历值
let mut map = map;
for v in map.values_mut() {
    *v *= 10;
}
```

> ⚠️ `HashMap` 的遍历顺序是**不确定的**（每次运行可能不同，出于防 HashDoS 的随机化）。需要有序遍历用 `BTreeMap`。

---

## 三、BTreeMap：有序映射

`BTreeMap<K, V>` 用 B 树实现，**按键有序**存储。查找/插入是 O(log n)，比 HashMap 略慢，但换来有序遍历和范围查询。键 `K` 需 `Ord`。

```rust
use std::collections::BTreeMap;

let mut map = BTreeMap::new();
map.insert(3, "three");
map.insert(1, "one");
map.insert(2, "two");

// 遍历总是按 key 升序（这是和 HashMap 的关键区别）
for (k, v) in &map {
    println!("{} => {}", k, v);  // 1, 2, 3 顺序
}

// 范围查询（HashMap 做不到）
for (k, v) in map.range(1..=2) {
    println!("范围内: {} => {}", k, v);  // 1, 2
}

// 最小/最大键
println!("{:?}", map.first_key_value()); // Some((1, "one"))
println!("{:?}", map.last_key_value());  // Some((3, "three"))
```

`HashMap` vs `BTreeMap`：

| | `HashMap` | `BTreeMap` |
|---|-----------|------------|
| 底层 | 哈希表 | B 树 |
| 复杂度 | 平均 O(1) | O(log n) |
| 顺序 | 无序（随机） | 按 key 升序 |
| 范围查询 | ❌ | ✅ `range()` |
| 键约束 | `Eq + Hash` | `Ord` |
| 何时用 | 默认选择，纯查找 | 需要有序/范围/最值 |

---

## 四、HashSet 与 BTreeSet

`HashSet<T>` 是只有键没有值的 `HashMap`，用于**去重和集合运算**。`BTreeSet` 是其有序版本。

```rust
use std::collections::HashSet;

let mut set = HashSet::new();
set.insert(1);
set.insert(2);
set.insert(2);            // 重复无效
println!("{}", set.len()); // 2

println!("{}", set.contains(&1)); // true

// 快速去重：Vec → HashSet → Vec
let v = vec![1, 2, 2, 3, 3, 3];
let unique: Vec<i32> = v.into_iter().collect::<HashSet<_>>().into_iter().collect();

// 集合运算
let a: HashSet<i32> = [1, 2, 3].into_iter().collect();
let b: HashSet<i32> = [2, 3, 4].into_iter().collect();

let inter: HashSet<_> = a.intersection(&b).copied().collect(); // 交集 {2,3}
let union: HashSet<_> = a.union(&b).copied().collect();        // 并集 {1,2,3,4}
let diff: HashSet<_> = a.difference(&b).copied().collect();    // 差集 {1}
```

| 方法 | 作用 |
|------|------|
| `insert` / `remove` / `contains` | 增 / 删 / 查 |
| `intersection(&other)` | 交集 |
| `union(&other)` | 并集 |
| `difference(&other)` | 差集（在 a 不在 b） |
| `symmetric_difference` | 对称差（只在其中一个） |
| `is_subset` / `is_superset` | 子集 / 超集判断 |

---

## 五、VecDeque / BinaryHeap 等

标准库 `std::collections` 还有几个常用容器：

### VecDeque：双端队列

两端都能高效 O(1) 增删，适合队列（FIFO）/双端队列：

```rust
use std::collections::VecDeque;

let mut q: VecDeque<i32> = VecDeque::new();
q.push_back(1);     // 尾部加
q.push_back(2);
q.push_front(0);    // 头部加：[0, 1, 2]
let front = q.pop_front(); // 头部弹：Some(0)，常用于队列
```

### BinaryHeap：优先队列（最大堆）

每次取出最大元素，O(log n) 插入/弹出：

```rust
use std::collections::BinaryHeap;

let mut heap = BinaryHeap::new();
heap.push(3);
heap.push(1);
heap.push(5);
println!("{:?}", heap.peek()); // Some(5)，堆顶最大
while let Some(max) = heap.pop() {
    print!("{} ", max);  // 5 3 1（从大到小）
}

// 想要最小堆？用 std::cmp::Reverse 包裹
use std::cmp::Reverse;
let mut min_heap = BinaryHeap::new();
min_heap.push(Reverse(3));
min_heap.push(Reverse(1));
println!("{:?}", min_heap.peek()); // Some(Reverse(1))，最小在顶
```

### 集合一览

| 集合 | 用途 |
|------|------|
| `Vec<T>` | 动态数组，默认首选 |
| `VecDeque<T>` | 双端队列、FIFO |
| `HashMap<K,V>` | 键值映射，默认首选 |
| `BTreeMap<K,V>` | 有序映射、范围查询 |
| `HashSet<T>` | 去重、集合运算 |
| `BTreeSet<T>` | 有序集合 |
| `BinaryHeap<T>` | 优先队列 |
| `LinkedList<T>` | 双向链表（极少用，几乎总有更好的选择） |

---

## 六、集合与迭代器：collect 的威力

```rust
use std::collections::{HashMap, HashSet, BTreeMap};

// → Vec
let squares: Vec<i32> = (1..=5).map(|x| x * x).collect();

// → HashMap（从键值元组）
let map: HashMap<i32, i32> = (1..=3).map(|x| (x, x * x)).collect();

// → HashSet（自动去重）
let set: HashSet<i32> = vec![1, 2, 2, 3].into_iter().collect();

// → BTreeMap（有序）
let sorted: BTreeMap<i32, &str> =
    [(3, "c"), (1, "a"), (2, "b")].into_iter().collect();

// → String（字符迭代器）
let s: String = vec!['r', 'u', 's', 't'].into_iter().collect();

// 分组：把 Vec 按条件分进 HashMap
let words = vec!["apple", "banana", "avocado", "cherry"];
let mut by_first: HashMap<char, Vec<&str>> = HashMap::new();
for w in words {
    by_first.entry(w.chars().next().unwrap()).or_default().push(w);
}
// {'a': ["apple","avocado"], 'b': ["banana"], 'c': ["cherry"]}
```

常见迭代器 → 集合操作：

```rust
let v = vec![1, 2, 3, 4, 5];

let sum: i32 = v.iter().sum();                            // 求和 15
let max = v.iter().max();                                 // Some(&5)
let evens: Vec<&i32> = v.iter().filter(|&&x| x%2==0).collect(); // [2,4]
let doubled: Vec<i32> = v.iter().map(|x| x*2).collect();  // [2,4,6,8,10]
let count = v.iter().filter(|&&x| x > 2).count();         // 3
```

---

## 七、选型决策表

| 需求 | 选择 |
|------|------|
| 有序列表、随机访问、尾部增删 | `Vec<T>` |
| 频繁头尾两端增删（队列） | `VecDeque<T>` |
| 键值映射、纯查找 | `HashMap<K,V>` |
| 键值映射、要有序/范围查询 | `BTreeMap<K,V>` |
| 去重、判断成员、集合运算 | `HashSet<T>` |
| 去重且要有序 | `BTreeSet<T>` |
| 每次取最大/最小（优先队列） | `BinaryHeap<T>` |
| 统计计数 / 分组 | `HashMap` + `entry` API |

**默认选择**：列表用 `Vec`，映射用 `HashMap`，集合用 `HashSet`。只有明确需要"有序/范围/优先级/双端"时才换其他。

---

## 八、常见陷阱

1. **遍历集合时修改它（借用冲突）**
   `for x in &v { v.push(...) }` 编译不过。对策：先收集要改的内容，循环后再改；或用 `retain`（删）、`iter_mut`（改值）、索引循环（小心边界）。

2. **`HashMap`/`HashSet` 顺序不可依赖**
   遍历顺序随机且可能每次运行不同。需要确定顺序用 `BTreeMap`/`BTreeSet`，或把键收集后 `sort`。

3. **该用 `entry` 却用 get + insert**
   "先查再插"查两次哈希、易借用冲突。计数/分组/有则改无则插，统一用 `entry`。

4. **`v[i]` 越界 panic**
   索引不确定时用 `v.get(i)` 拿 `Option`。

5. **`remove` vs `swap_remove` 性能**
   `Vec::remove(i)` 是 O(n)（要搬移）；不在乎顺序用 `swap_remove(i)`（O(1)）。

6. **忘记预分配导致反复扩容**
   已知大小时 `Vec::with_capacity(n)` / `HashMap::with_capacity(n)` 一次到位。

7. **`get` 返回引用，注意借用期**
   持有 `map.get(k)` 的引用期间不能再可变借用该 map。需要修改时先取出值（`copied()`/`cloned()`）或用 `entry`。

8. **`String` 作 key 时的所有权**
   `HashMap<String, _>` 插入需要拥有的 `String`；查询时可用 `&str`（`map.get("key")`），得益于 `Borrow` 机制，无需构造 `String`。

---

## 附：速查总结

```
Vec<T>      vec![..] / Vec::new() / with_capacity(n)
           push/pop（尾部 O(1)）  insert/remove（中间 O(n)）
           swap_remove（O(1) 乱序）  retain(过滤)  sort/dedup
           v[i] 越界 panic   v.get(i) → Option（安全）

HashMap     use std::collections::HashMap;  平均 O(1)，无序
           insert/get/remove/contains_key
           ★ entry API：or_insert / or_insert_with / or_default / and_modify
             *map.entry(k).or_insert(0) += 1   计数/分组神器

BTreeMap    有序（按 key 升序），O(log n)，支持 range() 范围查询、最值

HashSet     去重 + 集合运算：intersection/union/difference
BTreeSet    有序集合

VecDeque    双端队列 push/pop_front/back（队列 FIFO）
BinaryHeap  优先队列，peek/pop 取最大；Reverse 包裹得最小堆

collect     迭代器 → Vec / HashMap / HashSet / BTreeMap / String

默认选择：列表 Vec，映射 HashMap，集合 HashSet；
        需要有序/范围/优先级/双端时才换。
心法：计数分组用 entry；越界用 get；遍历别改结构；已知大小先预分配。
```
