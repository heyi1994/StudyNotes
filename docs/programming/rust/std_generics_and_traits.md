# std:Generics / Trait / Bound / 关联类型 / Trait 对象

> 泛型和 trait 是 Rust 抽象能力的两大支柱。泛型回答"对不同类型复用同一段逻辑"，trait 回答"不同类型共享同一种行为"。两者结合，构成 Rust **零成本、强约束**的抽象体系。

---

## 目录

- [一、泛型：参数化类型](#一泛型参数化类型)
- [二、Trait：定义共享行为](#二trait定义共享行为)
- [三、Trait Bound：给泛型加约束](#三trait-bound给泛型加约束)
- [四、where 子句](#四where-子句)
- [五、默认实现与孤儿规则](#五默认实现与孤儿规则)
- [六、关联类型 vs 泛型参数](#六关联类型-vs-泛型参数)
- [七、静态分发 vs 动态分发（impl Trait vs dyn Trait）](#七静态分发-vs-动态分发impl-trait-vs-dyn-trait)
- [八、Trait 对象与对象安全](#八trait-对象与对象安全)
- [九、运算符重载与常用标准 Trait](#九运算符重载与常用标准-trait)
- [十、高级特性：super trait / 泛型 trait / blanket impl](#十高级特性super-trait--泛型-trait--blanket-impl)
- [十一、常见陷阱](#十一常见陷阱)

---

## 一、泛型：参数化类型

泛型让你写**一份代码、适配多种类型**，避免为每个类型重复实现。类型参数通常用 `T`、`U`、`E` 等。

### 泛型函数

```rust
// 对任意类型 T 都能用的函数（但这里需要 T 可比较，见第三节）
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn main() {
    println!("{}", largest(&[1, 5, 3, 2]));       // i32
    println!("{}", largest(&['y', 'a', 'q']));    // char
}
```

### 泛型结构体与枚举

```rust
struct Point<T> {
    x: T,
    y: T,
}

// 两个不同类型参数
struct Pair<T, U> {
    first: T,
    second: U,
}

// 标准库里的 Option/Result 就是泛型枚举
enum MyOption<T> {
    Some(T),
    None,
}
```

### 泛型方法（impl 块也要声明 T）

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {              // impl<T> 声明泛型，再用于 Point<T>
    fn x(&self) -> &T {
        &self.x
    }
}

// 还能只给特定类型实现额外方法
impl Point<f64> {              // 仅当 T = f64 时才有这个方法
    fn distance_from_origin(&self) -> f64 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

### 零成本：单态化（monomorphization）

Rust 的泛型在编译期通过**单态化**实现：编译器为每个实际用到的具体类型**生成一份专门的代码**。所以泛型**没有运行时开销**，和手写具体类型一样快：

```rust
let int = largest(&[1, 2, 3]);    // 编译器生成 largest_i32
let ch = largest(&['a', 'b']);    // 编译器生成 largest_char
// 运行时没有类型判断、没有间接调用，零成本
```

> 代价是编译产物可能变大（代码膨胀），编译变慢——这是静态分发的固有权衡。

---

## 二、Trait：定义共享行为

trait 定义一组**方法签名**，是"某类型能做什么"的契约（类似其他语言的接口 interface，但更强大）。

```rust
// 定义 trait：实现它的类型必须提供 summarize
trait Summary {
    fn summarize(&self) -> String;
}

struct Article {
    title: String,
    content: String,
}

struct Tweet {
    username: String,
    text: String,
}

// 为不同类型实现同一 trait
impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}: {}...", self.title, &self.content[..10.min(self.content.len())])
    }
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("@{}: {}", self.username, self.text)
    }
}
```

调用：

```rust
let a = Article { title: "Rust".into(), content: "泛型与 trait 详解".into() };
let t = Tweet { username: "rustlang".into(), text: "1.80 发布".into() };
println!("{}", a.summarize());
println!("{}", t.summarize());
```

---

## 三、Trait Bound：给泛型加约束

光有泛型 `T` 还不够——编译器不知道 `T` 能做什么。**trait bound** 用来声明"`T` 必须实现某些 trait",从而获得对应能力：

```rust
// T 必须实现 Summary，函数体里才能调用 .summarize()
fn notify<T: Summary>(item: &T) {
    println!("快讯！{}", item.summarize());
}

// 多个约束用 + 连接
fn process<T: Summary + Clone>(item: &T) {
    let copy = item.clone();      // 因为有 Clone
    println!("{}", copy.summarize()); // 因为有 Summary
}
```

### 三种等价写法

```rust
// 1. 尖括号写法
fn f1<T: Summary>(item: &T) {}

// 2. impl Trait 参数（语法糖，简洁，适合简单情况）
fn f2(item: &impl Summary) {}

// 3. where 子句（约束多时更清晰，见下节）
fn f3<T>(item: &T) where T: Summary {}
```

> `&impl Summary` 和 `<T: Summary>` 在多数情况等价；区别是后者能在多个参数间强制"同一个 T",而 `impl Trait` 每个都是独立类型。

### 为什么需要 bound

回看第一节的 `largest`：比较 `item > largest` 需要 `T: PartialOrd`；如果还要把元素复制出来，需要 `T: Copy`。没有这些 bound，编译器会报错"`T` 可能不支持 `>` 运算"。**bound 是泛型代码能调用方法的前提**。

### 有条件地实现方法

可以"仅当 T 满足某些 bound 时,才给泛型类型实现某方法":

```rust
struct Wrapper<T> {
    value: T,
}

impl<T: std::fmt::Display> Wrapper<T> {
    // 只有当 T 可打印时，Wrapper<T> 才有 show 方法
    fn show(&self) {
        println!("值是: {}", self.value);
    }
}
```

---

## 四、where 子句

当约束变多、变复杂时，挤在尖括号里可读性差。`where` 把约束移到签名后面，更清晰：

```rust
use std::fmt::{Debug, Display};

// 难读：约束全堆在尖括号
fn complex<T: Display + Clone, U: Clone + Debug>(t: &T, u: &U) -> i32 { 0 }

// 清晰：用 where
fn complex2<T, U>(t: &T, u: &U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{
    0
}
```

`where` 还能表达**更复杂的约束**（比如对关联类型、对 `&T` 的约束），这是尖括号写法做不到的：

```rust
fn process<I>(iter: I)
where
    I: Iterator,
    I::Item: Display,        // 约束关联类型，尖括号里写不了
{
    for x in iter {
        println!("{}", x);
    }
}
```

---

## 五、默认实现与孤儿规则

### 默认实现

trait 方法可以提供**默认实现**，实现者可选择沿用或覆盖：

```rust
trait Greet {
    fn name(&self) -> String;

    // 默认实现，可调用 trait 里的其他方法
    fn greet(&self) -> String {
        format!("你好, {}!", self.name())
    }
}

struct Person { name: String }

impl Greet for Person {
    fn name(&self) -> String { self.name.clone() }
    // 不实现 greet，自动使用默认版本
}
```

### 孤儿规则（orphan rule）

实现 trait 有一条重要限制:**trait 或类型至少有一个是在当前 crate 定义的**,才能写 `impl Trait for Type`。即不能"为外部类型实现外部 trait"。

```rust
//  自己的 trait 给外部类型实现
trait MyTrait { fn hello(&self); }
impl MyTrait for String { fn hello(&self) {} }  // String 是外部的，但 MyTrait 是我的

//  外部 trait 给自己的类型实现
struct MyType;
impl std::fmt::Display for MyType {              // Display 外部，MyType 我的
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result { Ok(()) }
}

//  外部 trait 给外部类型实现：编译错误
// impl std::fmt::Display for Vec<i32> { ... }
```

> 目的：防止两个 crate 给同一组合各写一份实现导致冲突。绕过的常用手法是 **newtype 模式**——用自己的元组结构体包一层（`struct MyVec(Vec<i32>)`），就能为它实现外部 trait。

---

## 六、关联类型 vs 泛型参数

trait 里"占位的类型"有两种表达方式,新手常混淆。

### 关联类型（associated type）

用 `type` 在 trait 里声明一个占位类型,**每个实现只能指定一种**:

```rust
trait Container {
    type Item;                       // 关联类型
    fn get(&self, i: usize) -> Option<&Self::Item>;
}

struct IntList(Vec<i32>);
impl Container for IntList {
    type Item = i32;                 // 为这个实现确定 Item = i32
    fn get(&self, i: usize) -> Option<&i32> { self.0.get(i) }
}
```

`Iterator` 就是典型:

```rust
trait Iterator {
    type Item;                       // 每个迭代器产出一种确定的元素类型
    fn next(&mut self) -> Option<Self::Item>;
}
```

### 泛型 trait（泛型参数）

把类型作为 trait 的泛型参数,**同一个类型可以实现多次**(每个类型参数一次):

```rust
trait Convert<T> {
    fn convert(&self) -> T;
}

struct Celsius(f64);
impl Convert<f64> for Celsius {   // 转成 f64（华氏）
    fn convert(&self) -> f64 { self.0 * 9.0 / 5.0 + 32.0 }
}
impl Convert<String> for Celsius { // 也能转成 String
    fn convert(&self) -> String { format!("{}°C", self.0) }
}
```

`From<T>` 就是泛型 trait——所以一个类型能 `From` 多种来源。

### 怎么选

| | 关联类型 | 泛型参数 |
|---|---------|---------|
| 一个类型能实现几次 | **一次**（Item 唯一） | 多次（每个 T 一次） |
| 何时用 | 类型间是"函数关系"（一个输入对应一个输出类型） | 需要对同一类型的多种参数都实现 |
| 例子 | `Iterator::Item`、`Deref::Target` | `From<T>`、`Add<Rhs>` |

> 经验：**如果"每个实现只会有一种对应类型",用关联类型**(更简洁、调用方不用指定);需要一个类型对多种参数实现,才用泛型参数。

---

## 七、静态分发 vs 动态分发（impl Trait vs dyn Trait）

"用 trait 当类型"有两条路,这是性能与灵活性的核心权衡。

### 静态分发：impl Trait / 泛型（编译期确定）

`impl Trait` 作返回值,或泛型参数,都是**静态分发**:编译期就知道具体类型,单态化生成专门代码,**零运行时开销**,但每个返回点/调用必须是同一具体类型。

```rust
// 返回 impl Trait：调用方不知道具体类型，但编译器知道，零开销
fn make_summary() -> impl Summary {
    Tweet { username: "a".into(), text: "b".into() }
    // ⚠️ 所有返回分支必须返回同一具体类型，不能一会儿 Tweet 一会儿 Article
}

// 泛型参数也是静态分发
fn notify<T: Summary>(item: &T) {
    println!("{}", item.summarize());  // 编译期定向到 T 的具体实现
}
```

### 动态分发：dyn Trait（运行期确定）

`dyn Trait` 是 **trait 对象**,通过**虚函数表(vtable)**在运行期决定调用哪个实现。灵活(能在容器里混装不同类型),但有一次指针跳转的开销,且需要间接(`&dyn`/`Box<dyn>`):

```rust
// Vec 里混装不同的具体类型，靠 dyn 动态分发
let items: Vec<Box<dyn Summary>> = vec![
    Box::new(Tweet { username: "a".into(), text: "b".into() }),
    Box::new(Article { title: "t".into(), content: "ccccccccccc".into() }),
];
for item in &items {
    println!("{}", item.summarize()); // 运行期查 vtable 调用对应实现
}

// 函数参数也可用 &dyn
fn notify_dyn(item: &dyn Summary) {
    println!("{}", item.summarize());
}
```

### 对比

| | 静态分发（泛型/impl Trait） | 动态分发（dyn Trait） |
|---|---------------------------|----------------------|
| 决定时机 | 编译期 | 运行期（vtable） |
| 性能 | 零开销、可内联 | 一次间接跳转，不能内联 |
| 代码体积 | 可能膨胀（单态化多份） | 单份代码，体积小 |
| 灵活性 | 每处一种具体类型 | 可混装多种类型 |
| 形式 | `T: Trait` / `impl Trait` | `&dyn Trait` / `Box<dyn Trait>` |

> 默认优先**静态分发**(性能好);当你需要"一个集合里放多种类型""返回不同的具体类型""减少代码膨胀"时,才用 **dyn**。

---

## 八、Trait 对象与对象安全

要把 trait 做成 `dyn Trait`(trait 对象),该 trait 必须是 **对象安全(object-safe)** 的。否则编译器报错"the trait cannot be made into an object"。

主要规则（直觉版）:

1. **方法不能返回 `Self`**——因为 trait 对象擦除了具体类型,不知道 `Self` 多大。
2. **方法不能有泛型参数**——vtable 无法为无限多种泛型实例都存一个入口。
3. 方法的 receiver 得是 `&self`/`&mut self`/`self`/`Box<Self>` 等(能通过指针调用)。

```rust
//  非对象安全：返回 Self
trait Cloneable {
    fn duplicate(&self) -> Self;   // 不知道 Self 大小 → 不能 dyn
}
// let x: Box<dyn Cloneable> = ...; // 编译错误

//  对象安全
trait Drawable {
    fn draw(&self);                // 返回 ()，接收 &self，OK
}
let shapes: Vec<Box<dyn Drawable>> = vec![/* ... */];
```

> `Clone` 不是对象安全的(它返回 `Self`),所以不能直接有 `Box<dyn Clone>`——这是常见的困惑来源。需要时用 `dyn-clone` 等技巧绕过。

---

## 九、运算符重载与常用标准 Trait

很多语言特性其实是"实现某个标准 trait"。掌握这些能让自定义类型融入语言。

### 运算符重载（std::ops）

运算符都对应一个 trait:

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy)]
struct Point { x: i32, y: i32 }

impl Add for Point {
    type Output = Point;                 // 关联类型：相加结果类型
    fn add(self, other: Point) -> Point {
        Point { x: self.x + other.x, y: self.y + other.y }
    }
}

let p = Point { x: 1, y: 2 } + Point { x: 3, y: 4 };
println!("{:?}", p); // Point { x: 4, y: 6 }
```

| 运算符 | trait | | 运算符 | trait |
|--------|-------|---|--------|-------|
| `+` | `Add` | | `[]` | `Index`/`IndexMut` |
| `-` | `Sub` | | `*`(解引用) | `Deref`/`DerefMut` |
| `*` | `Mul` | | `==` | `PartialEq` |
| `/` | `Div` | | `<` `>` | `PartialOrd` |
| `-`(取负) | `Neg` | | 函数调用 | `Fn`/`FnMut`/`FnOnce` |

### 常用可派生 trait（#[derive]）

很多标准 trait 能用 `#[derive(...)]` 自动实现,省去样板:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, Default)]
struct Config {
    retries: u32,
    name: String,  // 注意：含 String 时不能 derive Copy
}
```

| Trait | 作用 |
|-------|------|
| `Debug` | `{:?}` 调试打印 |
| `Clone` / `Copy` | 复制语义 |
| `PartialEq` / `Eq` | `==` 比较 |
| `PartialOrd` / `Ord` | 排序比较 |
| `Hash` | 作 HashMap 的 key |
| `Default` | 提供默认值 `Config::default()` |

> 这些 trait 串起了前面的笔记:`Iterator`(关联类型)、`From`/`Into`(泛型 trait)、`Deref`/`Drop`(智能指针)、`Send`/`Sync`(并发)、`Fn` 家族(闭包)、`Error`(错误处理)——全是 trait 系统的应用。

---

## 十、高级特性：super trait / 泛型 trait / blanket impl

### Super trait：trait 继承

要求实现某 trait 前必须先实现另一个 trait:

```rust
use std::fmt::Display;

// 实现 Summary 的类型必须也实现 Display
trait Summary: Display {
    fn summarize(&self) -> String {
        format!("摘要 of {}", self)   // 因为有 Display 约束，可以用 {}
    }
}
```

### Blanket implementation：一揽子实现

为"所有满足某约束的类型"统一实现 trait——标准库大量使用:

```rust
// 标准库里的真实例子：为所有实现了 Display 的类型自动实现 ToString
// impl<T: Display> ToString for T { ... }
// 所以任何能 Display 的类型，都自动有了 .to_string()

// 自定义一个 blanket impl
trait Printable {
    fn print(&self);
}
impl<T: std::fmt::Display> Printable for T {   // 对所有可 Display 的 T 实现
    fn print(&self) {
        println!("{}", self);
    }
}
// 现在 i32、String、f64... 全都自动有了 .print()
42.print();
"hello".print();
```

> blanket impl 是 Rust 抽象的"魔法"来源之一:`Into` 由 `From` 自动得来、`ToString` 由 `Display` 自动得来,都是这个机制。

---

## 十一、常见陷阱

1. **忘了加 trait bound**
   泛型函数里调 `.method()` 报错"method not found",通常是缺了对应的 `T: SomeTrait` 约束。

2. **`impl Trait` 返回类型必须单一**
   `fn f() -> impl Trait` 的所有返回分支必须是**同一个**具体类型。要返回不同类型,改用 `Box<dyn Trait>`。

3. **trait 对象的对象安全限制**
   返回 `Self`、带泛型方法的 trait 不能做 `dyn`。报"cannot be made into an object"就往这查。

4. **混淆关联类型与泛型参数**
   "一个实现只有一种对应类型"用关联类型;"一个类型要对多种参数实现"用泛型参数。选错会导致实现冲突或调用啰嗦。

5. **孤儿规则挡路**
   不能为外部类型实现外部 trait。用 newtype(`struct My(ExternalType)`)包一层。

6. **`dyn` 需要间接**
   `dyn Trait` 是不定大小类型(unsized),不能直接当值,必须放在 `&`、`Box`、`Rc` 等后面。

7. **过度泛型化**
   不是所有代码都该泛型。只有一种类型时,具体类型更简单。泛型是为"真的需要复用多类型"准备的。

---

## 附：速查总结

```
泛型        fn f<T>() / struct S<T> / impl<T>   一份代码适配多类型
           单态化 → 编译期生成具体版本，零运行时开销

Trait      trait T { fn m(&self); }            定义共享行为（接口）
           impl T for Type { ... }             为类型实现

Trait Bound <T: Trait>  /  &impl Trait  /  where T: Trait
           给泛型加能力约束；多个用 +；复杂用 where

关联类型 vs 泛型参数
           type Item;     一个实现一种类型（Iterator/Deref）
           Trait<T>       一个类型可多次实现（From/Add）

静态分发    <T: Trait> / impl Trait   编译期定，零开销，单一类型
动态分发    &dyn Trait / Box<dyn>     运行期 vtable，可混装多类型
           → 默认用静态；需要混装/返回不同类型才用 dyn

对象安全    dyn 要求：不返回 Self、方法无泛型参数

派生        #[derive(Debug, Clone, PartialEq, Hash, Default, ...)]
运算符      Add/Sub/Mul/Index/Deref... 都是 trait

高级        super trait（trait: 依赖）  blanket impl（impl<T: A> B for T）
孤儿规则    trait 或类型至少一个是本 crate 的；绕过用 newtype

心法：泛型 = 对类型复用逻辑；trait = 对类型共享行为。
     前面所有笔记的 Iterator/Future/From/Deref/Send 全是 trait 系统的应用。
```
