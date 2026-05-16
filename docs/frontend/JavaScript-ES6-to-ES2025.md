# JavaScript ES6 → ES2025

## ES2015 (ES6) — 现代 JS 的起点

### 1. let / const

```js
let x = 1; // 块级作用域，可重赋值
const y = 2; // 块级作用域，不可重赋值（引用不可变，内容可变）
```

> 不再使用 `var`（存在变量提升和函数作用域问题）

---

### 2. 箭头函数

```js
const add = (a, b) => a + b;
const double = (n) => n * 2;
const getObj = () => ({ key: "value" }); // 返回对象加括号

// 不绑定自己的 this，继承外层上下文
class Timer {
  start() {
    setInterval(() => {
      this.tick(); // this 指向 Timer 实例
    }, 1000);
  }
}
```

---

### 3. 模板字符串

```js
const name = "Alice";
console.log(`Hello, ${name}!`);
console.log(`1 + 1 = ${1 + 1}`);

// 多行字符串
const html = `
  <div>
    <p>${name}</p>
  </div>
`;

// 标签模板
function highlight(strings, ...values) {
  return strings.reduce(
    (result, str, i) =>
      `${result}${str}${values[i] ? `<b>${values[i]}</b>` : ""}`,
    "",
  );
}
const msg = highlight`Hello ${name}, you are ${"awesome"}!`;
```

---

### 4. 解构赋值

```js
// 数组解构
const [a, b, ...rest] = [1, 2, 3, 4, 5];
const [, second] = [1, 2]; // 跳过元素
const [x = 10] = []; // 默认值

// 对象解构
const { name, age = 18 } = user; // 默认值
const { name: userName } = user; // 重命名
const {
  a: { b },
} = { a: { b: 1 } }; // 嵌套解构

// 函数参数解构
function greet({ name, age = 18 } = {}) {
  return `${name} is ${age}`;
}

// 交换变量
let p = 1,
  q = 2;
[p, q] = [q, p];
```

---

### 5. 默认参数

```js
function createUser(name = "Anonymous", role = "user", active = true) {
  return { name, role, active };
}

// 默认值可以是表达式或函数调用
function fn(a, b = a * 2) {
  return b;
}
```

---

### 6. Rest / Spread 运算符

```js
// Rest：收集剩余参数
function sum(...nums) {
  return nums.reduce((a, b) => a + b, 0);
}

// Spread：展开数组/对象
const arr = [...arr1, ...arr2, 4, 5];
Math.max(...[1, 2, 3]);

const obj = { ...defaults, ...overrides, extra: true };
```

---

### 7. 类 (Class)

```js
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    return `${this.name} makes a sound.`;
  }

  static create(name) {
    // 静态方法
    return new Animal(name);
  }

  get info() {
    // getter
    return `[${this.name}]`;
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name); // 必须先调用 super
    this.breed = breed;
  }

  speak() {
    return super.speak() + " Woof!";
  }
}
```

---

### 8. 模块 (Modules)

```js
// math.js — 导出
export const PI = 3.14159;
export function add(a, b) { return a + b; }
export default class Calculator { ... }

// main.js — 导入
import Calculator, { PI, add } from './math.js';
import * as MathLib from './math.js';
import './side-effect.js';           // 只执行副作用
```

---

### 9. Promise

```js
const p = new Promise((resolve, reject) => {
  setTimeout(() => resolve("done"), 1000);
});

p.then((val) => console.log(val))
  .catch((err) => console.error(err))
  .finally(() => console.log("always runs"));

// 组合
Promise.all([p1, p2, p3]); // 全部成功才 resolve
Promise.race([p1, p2]); // 第一个完成（无论成功/失败）
```

---

### 10. Symbol

```js
const id = Symbol('id');        // 唯一标识符
const obj = { [id]: 123 };     // 作为属性键，不会被枚举

// 内置 Symbol
class MyArray {
  [Symbol.iterator]() { ... }   // 自定义迭代行为
}
```

---

### 11. Map / Set / WeakMap / WeakSet

```js
// Set — 值的集合，自动去重
const set = new Set([1, 2, 2, 3]);
set.add(4);
set.has(2);      // true
set.delete(1);
set.size;        // 3

// Map — 任意类型的键值对
const map = new Map();
map.set({ key: 1 }, 'object key');
map.set('str', 42);
map.get('str');  // 42
for (const [k, v] of map) { ... }

// WeakMap / WeakSet — 弱引用，不阻止 GC，键必须是对象
const cache = new WeakMap();
cache.set(domElement, { data: '...' });
```

---

### 12. 迭代器 / Generator

```js
// 迭代器协议
const range = {
  [Symbol.iterator]() {
    let i = 0;
    return {
      next() {
        return i < 3 ? { value: i++, done: false } : { done: true };
      },
    };
  },
};
for (const n of range) console.log(n); // 0 1 2

// Generator 函数
function* gen() {
  yield 1;
  yield 2;
  return 3;
}
const g = gen();
g.next(); // { value: 1, done: false }

// 无限序列
function* fibonacci() {
  let [a, b] = [0, 1];
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}
```

---

### 13. Proxy / Reflect

```js
const handler = {
  get(target, key) {
    return key in target ? target[key] : `Key "${key}" not found`;
  },
  set(target, key, value) {
    if (typeof value !== "number") throw new TypeError("Only numbers!");
    target[key] = value;
    return true;
  },
};

const proxy = new Proxy({}, handler);
proxy.a = 42;
proxy.b; // "Key "b" not found"

// Reflect — 对应 Proxy 每个 trap 的默认行为
Reflect.get(target, key);
Reflect.set(target, key, value);
```

---

### 14. 新增数组方法

```js
Array.from("abc"); // ['a', 'b', 'c']
Array.from({ length: 3 }, (_, i) => i); // [0, 1, 2]
Array.of(1, 2, 3); // [1, 2, 3]

[1, 2, 3].find((n) => n > 1); // 2
[1, 2, 3].findIndex((n) => n > 1); // 1
[1, 2, 3].fill(0, 1, 2); // [1, 0, 3]
[1, 2, 3].includes(2); // ES2016，见下文

// 迭代
for (const [i, v] of ["a", "b"].entries()) {
}
for (const k of ["a", "b"].keys()) {
}
for (const v of ["a", "b"].values()) {
}
```

---

### 15. 新增字符串方法

```js
"hello".includes("ell"); // true
"hello".startsWith("he"); // true
"hello".endsWith("lo"); // true
"ha".repeat(3); // 'hahaha'
```

---

### 16. 对象简写 / 计算属性名

```js
const x = 1,
  y = 2;
const point = { x, y }; // 等同于 { x: x, y: y }

const key = "name";
const obj = {
  [key]: "Alice", // 计算属性名
  ["key_" + 1]: "dynamic",
  greet() {
    return "hi";
  }, // 方法简写
};
```

---

### 17. Object.assign

```js
const target = { a: 1 };
Object.assign(target, { b: 2 }, { c: 3 });
// { a: 1, b: 2, c: 3 } — 浅拷贝
```

---

## ES2016 (ES7)

### 1. Array.prototype.includes

```js
[1, 2, NaN].includes(NaN); // true（比 indexOf 更准确）
[1, 2, 3].includes(2, 1); // true，从索引 1 开始搜索
```

### 2. 指数运算符 `**`

```js
2 ** 10; // 1024
2 ** 0.5; // Math.sqrt(2)
let n = 2;
n **= 3; // 8
```

---

## ES2017 (ES8)

### 1. async / await

```js
async function fetchUser(id) {
  try {
    const res = await fetch(`/api/users/${id}`);
    const user = await res.json();
    return user;
  } catch (err) {
    console.error(err);
  }
}

// 并行执行
async function loadAll() {
  const [users, posts] = await Promise.all([fetchUsers(), fetchPosts()]);
}
```

---

### 2. Object.values / Object.entries

```js
const obj = { a: 1, b: 2, c: 3 };
Object.values(obj); // [1, 2, 3]
Object.entries(obj); // [['a',1], ['b',2], ['c',3]]

for (const [key, val] of Object.entries(obj)) {
}
```

---

### 3. Object.getOwnPropertyDescriptors

```js
const descriptors = Object.getOwnPropertyDescriptors(obj);
// 用于完整克隆（含 getter/setter）
const clone = Object.create(
  Object.getPrototypeOf(obj),
  Object.getOwnPropertyDescriptors(obj),
);
```

---

### 4. String.padStart / padEnd

```js
"5".padStart(3, "0"); // '005'
"hi".padEnd(5, "."); // 'hi...'
String(42).padStart(6); // '    42'（默认空格）
```

---

### 5. 函数参数/调用末尾逗号

```js
function fn(
  a,
  b,
  c, // ← 允许末尾逗号
) {}

fn(1, 2, 3);
```

---

### 6. SharedArrayBuffer / Atomics

```js
// 多线程间共享内存
const sab = new SharedArrayBuffer(1024);
const arr = new Int32Array(sab);

// 原子操作（线程安全）
Atomics.add(arr, 0, 5);
Atomics.load(arr, 0); // 5
Atomics.wait(arr, 0, 5); // 等待直到值不为 5
```

---

## ES2018 (ES9)

### 1. 对象 Rest / Spread

```js
const { a, b, ...rest } = { a: 1, b: 2, c: 3, d: 4 };
// rest = { c: 3, d: 4 }

const merged = { ...obj1, ...obj2, extra: true };
```

---

### 2. 异步迭代 `for await...of`

```js
async function processStream(stream) {
  for await (const chunk of stream) {
    process(chunk);
  }
}

// 自定义异步可迭代
async function* asyncRange(start, end) {
  for (let i = start; i <= end; i++) {
    await delay(100);
    yield i;
  }
}
```

---

### 3. Promise.finally

```js
fetch("/api/data")
  .then(handleData)
  .catch(handleError)
  .finally(() => hideLoadingSpinner()); // 总会执行
```

---

### 4. RegExp 增强

```js
// 命名捕获组
const re = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/;
const { year, month, day } = "2024-01-15".match(re).groups;

// lookbehind 断言
/(?<=\$)\d+/.exec("$100"); // ['100'] — 前面是 $
/(?<!\$)\d+/.exec("€100"); // ['100'] — 前面不是 $

// dotAll 模式（s 标志），. 可匹配换行
/hello.world/s.test("hello\nworld"); // true

// Unicode 属性转义
/\p{Script=Han}/u.test("汉"); // true — 汉字
/\p{Emoji}/u.test("🎉"); // true
```

---

## ES2019 (ES10)

### 1. Array.flat / flatMap

```js
[1, [2, [3, [4]]]].flat(); // [1, 2, [3, [4]]]，默认深度 1
[1, [2, [3, [4]]]].flat(Infinity); // [1, 2, 3, 4]

// flatMap = map + flat(1)，性能更好
["hello world", "foo bar"].flatMap((str) => str.split(" ")); // ['hello', 'world', 'foo', 'bar']
```

---

### 2. Object.fromEntries

```js
// entries 转回对象
const entries = [
  ["a", 1],
  ["b", 2],
];
Object.fromEntries(entries); // { a: 1, b: 2 }

// Map 转对象
Object.fromEntries(map);

// 对象变换
const doubled = Object.fromEntries(
  Object.entries(obj).map(([k, v]) => [k, v * 2]),
);
```

---

### 3. String.trimStart / trimEnd

```js
"  hello  ".trimStart(); // 'hello  '
"  hello  ".trimEnd(); // '  hello'
// trimLeft/trimRight 是别名（非标准）
```

---

### 4. 可选的 catch 绑定

```js
// 不需要捕获错误对象时，可省略参数
try {
  JSON.parse(str);
} catch {
  // 不再强制写 catch(e)
  return false;
}
```

---

### 5. Symbol.description

```js
const sym = Symbol("my symbol");
sym.description; // 'my symbol'（之前只能 toString()）
```

---

### 6. Array.sort 稳定性保证

相同元素排序后，原始顺序保持不变（V8 之前对大数组不稳定）。

---

## ES2020 (ES11)

### 1. BigInt

```js
const big = 9007199254740993n; // 字面量加 n
const fromStr = BigInt("9007199254740993");

big + 1n; // 运算两侧都必须是 BigInt
Number(big); // 转换（可能丢精度）
typeof big; // 'bigint'

// 适合场景：大整数计算、密码学、数据库 ID
```

---

### 2. 可选链 `?.`

```js
user?.profile?.avatar?.url; // 任一层为 null/undefined 则返回 undefined
arr?.[0]; // 数组元素
fn?.(); // 函数调用（fn 不存在时不报错）

// 替代
const url =
  user && user.profile && user.profile.avatar && user.profile.avatar.url;
```

---

### 3. 空值合并 `??`

```js
const name = user.name ?? "Anonymous"; // 仅 null/undefined 时取右值
const count = data.count ?? 0;

// 区别于 ||
0 || "default"; // 'default'（0 是 falsy）
0 ?? "default"; // 0（0 不是 null/undefined）
```

---

### 4. Promise.allSettled

```js
const results = await Promise.allSettled([p1, p2, p3]);
// 不管成功失败，等所有完成
results.forEach(({ status, value, reason }) => {
  if (status === "fulfilled") console.log(value);
  else console.error(reason);
});
```

---

### 5. globalThis

```js
// 跨环境获取全局对象
// 浏览器: window, Node.js: global, Worker: self
globalThis.setTimeout(() => {}, 0);
globalThis.fetch("/api");
```

---

### 6. 动态 import()

```js
// 按需加载，返回 Promise
const { add } = await import("./math.js");

// 条件加载
if (needsChart) {
  const Chart = await import("./chart.js");
}

// 路由懒加载
const routes = {
  "/home": () => import("./pages/Home.js"),
};
```

---

### 7. String.matchAll

```js
const str = "test1 test2 test3";
const re = /test(\d)/g;

// 返回迭代器，包含所有匹配及捕获组
for (const match of str.matchAll(re)) {
  console.log(match[0], match[1], match.index);
}
// test1 1 0
// test2 2 6
// test3 3 12
```

---

### 8. import.meta

```js
// 当前模块的元信息
console.log(import.meta.url); // 当前模块的 URL
console.log(import.meta.env); // 构建工具注入的环境变量（Vite 等）
```

---

## ES2021 (ES12)

### 1. String.replaceAll

```js
"a-b-c-d".replaceAll("-", "_"); // 'a_b_c_d'
// 之前需要：'a-b-c-d'.replace(/-/g, '_')
```

---

### 2. Promise.any

```js
// 只要有一个 resolve 就成功；全部 reject 才抛出 AggregateError
const first = await Promise.any([slowP, fastP, failP]);
```

---

### 3. AggregateError

```js
try {
  await Promise.any([p1, p2]);
} catch (err) {
  if (err instanceof AggregateError) {
    err.errors; // 所有失败的原因数组
  }
}
```

---

### 4. 逻辑赋值运算符

```js
// &&= 只在左值为真时赋值
user.name &&= user.name.trim();

// ||= 只在左值为假时赋值
config.timeout ||= 3000;

// ??= 只在左值为 null/undefined 时赋值
user.role ??= "guest";
```

---

### 5. 数字分隔符 `_`

```js
const billion = 1_000_000_000;
const bytes = 0xff_ec_d5_12;
const pi = 3.141_592_653;
const bits = 0b1010_0001;
```

---

### 6. WeakRef / FinalizationRegistry

```js
// WeakRef — 弱引用，不阻止 GC
const ref = new WeakRef(someObject);
const obj = ref.deref(); // 可能已被回收，返回 undefined

// FinalizationRegistry — 对象被 GC 后执行回调
const registry = new FinalizationRegistry((heldValue) => {
  console.log(`${heldValue} was collected`);
});
registry.register(obj, "myObject");
```

---

## ES2022 (ES13)

### 1. 类私有字段和方法

```js
class Counter {
  #count = 0; // 私有字段
  #step;

  constructor(step = 1) {
    this.#step = step;
  }

  #validate() {
    // 私有方法
    return this.#count >= 0;
  }

  increment() {
    this.#count += this.#step;
  }

  get value() {
    return this.#count;
  }

  static #instances = 0; // 静态私有字段
  static getInstances() {
    return Counter.#instances;
  }
}

// 检查私有字段是否存在
#count in obj; // true/false
```

---

### 2. 静态类块 (Static Class Blocks)

```js
class Config {
  static debug;
  static verbose;

  static {
    // 静态初始化块，可执行复杂逻辑
    const env = getEnvironment();
    Config.debug = env === "development";
    Config.verbose = env !== "production";
  }
}
```

---

### 3. Top-level await

```js
// 在模块顶层直接使用 await（无需 async 函数包裹）
const data = await fetch("/api/config").then((r) => r.json());
const { default: lodash } = await import("lodash");

export const config = data;
```

---

### 4. Array/String/TypedArray.at()

```js
const arr = [1, 2, 3, 4, 5];
arr.at(0); // 1
arr.at(-1); // 5（倒数第一）
arr.at(-2); // 4

"hello".at(-1); // 'o'
```

---

### 5. Object.hasOwn

```js
// 替代 Object.prototype.hasOwnProperty.call(obj, key)
Object.hasOwn({ a: 1 }, "a"); // true
Object.hasOwn({ a: 1 }, "b"); // false

// 对 Object.create(null) 创建的对象也安全
const obj = Object.create(null);
Object.hasOwn(obj, "key"); // false（不会报错）
```

---

### 6. Error.cause

```js
try {
  await fetchData();
} catch (err) {
  throw new Error('Failed to load user', { cause: err });
}

// 捕获时
try { ... } catch (err) {
  console.log(err.cause);  // 原始错误
}
```

---

### 7. RegExp d 标志（匹配下标）

```js
const re = /(\d+)/d;
const match = "2024-01-15".match(re);
match.indices[0]; // [0, 4] — 整个匹配的起止索引
match.indices[1]; // [0, 4] — 第一个捕获组的起止索引
```

---

## ES2023 (ES14)

### 1. Array findLast / findLastIndex

```js
[1, 2, 3, 4].findLast((n) => n % 2 === 0); // 4（从右往左）
[1, 2, 3, 4].findLastIndex((n) => n % 2 === 0); // 3
```

---

### 2. 不可变数组方法（Immutable Array Methods）

```js
const arr = [3, 1, 2];

arr.toSorted(); // [1, 2, 3]，原数组不变
arr.toReversed(); // [2, 1, 3]，原数组不变
arr.toSpliced(1, 1, 9); // [3, 9, 2]，原数组不变
arr.with(0, 99); // [99, 1, 2]，替换指定索引，原数组不变

// 对比破坏性方法
arr.sort(); // 修改原数组
arr.reverse(); // 修改原数组
```

---

### 3. Symbol 作为 WeakMap 键

```js
const key = Symbol("key");
const map = new WeakMap();
map.set(key, { data: "value" }); // 之前只允许对象
```

---

### 4. Hashbang 语法

```js
#!/usr/bin/env node
// 文件第一行的 #! 注释，用于 Unix 脚本
console.log("Hello");
```

---

## ES2024 (ES15)

### 1. Promise.withResolvers

```js
// 将 resolve/reject 暴露到 Promise 外部
const { promise, resolve, reject } = Promise.withResolvers();

// 之前需要
let resolve, reject;
const promise = new Promise((res, rej) => {
  resolve = res;
  reject = rej;
});

// 实际用途：事件驱动场景
function waitForClick(button) {
  const { promise, resolve } = Promise.withResolvers();
  button.addEventListener("click", resolve, { once: true });
  return promise;
}
```

---

### 2. Object.groupBy / Map.groupBy

```js
const people = [
  { name: "Alice", age: 25 },
  { name: "Bob", age: 17 },
  { name: "Carol", age: 30 },
];

// 按条件分组，返回普通对象
const grouped = Object.groupBy(people, (p) =>
  p.age >= 18 ? "adult" : "minor",
);
// { adult: [...], minor: [...] }

// Map.groupBy — 键可以是任意值
const map = Map.groupBy(people, (p) => p.age >= 18);
// Map { true => [...], false => [...] }
```

---

### 3. String.isWellFormed / toWellFormed

```js
// 检测/修复孤立代理字符（Lone Surrogate）
"\uD800".isWellFormed(); // false — 孤立代理项
"hello".isWellFormed(); // true

"\uD800".toWellFormed(); // '  ' (替换为 U+FFFD)
// 在 URL 编码、序列化场景中很有用
```

---

### 4. RegExp v 标志（集合操作）

```js
// v 标志：Unicode 集合操作，更精确的字符类
/[\p{Letter}--\p{ASCII}]/v; // 非 ASCII 字母（差集 --）
/[\p{Letter}&&\p{ASCII}]/v; // ASCII 字母（交集 &&）
/[a-z[A-Z]]/v; // 嵌套字符类（集合并集）
/^[\p{Emoji_Keycap_Sequence}]$/v.test("1️⃣"); // true
```

---

### 5. ArrayBuffer resize / transfer

```js
// 可调整大小的 ArrayBuffer
const buf = new ArrayBuffer(8, { maxByteLength: 16 });
buf.resize(12); // 调整为 12 字节
buf.byteLength; // 12

// 转移所有权（原 buffer 变为 detached）
const newBuf = buf.transfer();
const resized = buf.transfer(4); // 转移并调整大小
```

---

### 6. Atomics.waitAsync

```js
// 非阻塞的异步等待（可在主线程使用）
const result = await Atomics.waitAsync(sharedArray, 0, 0).value;
// 当 sharedArray[0] 不再是 0 时 resolve
```

---

## ES2025 (ES16)

### 1. Iterator Helpers（迭代器辅助方法）

```js
// 所有迭代器现在都有这些方法
function* range(start, end) {
  for (let i = start; i <= end; i++) yield i;
}

const iter = range(1, 10);

iter
  .map((x) => x * 2) // 映射
  .filter((x) => x > 6) // 过滤
  .take(3) // 取前 N 个
  .toArray(); // 收集为数组 → [8, 10, 12]

// 更多方法
iter.drop(2); // 跳过前 N 个
iter.flatMap((x) => [x, -x]); // 展平映射
iter.reduce((a, b) => a + b, 0);
iter.forEach((x) => console.log(x));
iter.some((x) => x > 5);
iter.every((x) => x > 0);
iter.find((x) => x > 5);
Iterator.from(arrayOrIterable); // 包装为迭代器
```

---

### 2. Set 方法

```js
const a = new Set([1, 2, 3, 4]);
const b = new Set([3, 4, 5, 6]);

a.union(b); // {1,2,3,4,5,6}  — 并集
a.intersection(b); // {3,4}           — 交集
a.difference(b); // {1,2}           — 差集 (a - b)
a.symmetricDifference(b); // {1,2,5,6}       — 对称差集

a.isSubsetOf(b); // false — a 是否是 b 的子集
a.isSupersetOf(b); // false — a 是否是 b 的超集
a.isDisjointFrom(b); // false — a 和 b 是否不相交
```

---

### 3. Import Attributes（导入属性）

```js
// 明确声明导入资源的类型
import data from "./data.json" with { type: "json" };
import sheet from "./styles.css" with { type: "css" };

// 动态导入
const mod = await import("./data.json", { with: { type: "json" } });
```

---

### 4. RegExp 重复命名捕获组

```js
// 同一个命名可以出现在多个分支中
const re = /(?<year>\d{4})-(?<month>\d{2})|(?<month>\d{2})\/(?<year>\d{4})/;
"01/2024".match(re).groups; // { year: '2024', month: '01' }
"2024-01".match(re).groups; // { year: '2024', month: '01' }
```

---

### 5. Float16Array

```js
// 16 位半精度浮点数，节省内存（机器学习、图形处理）
const f16 = new Float16Array([1.5, 2.5, 3.14]);
f16[0]; // 1.5

// 辅助函数
Math.f16round(1.337); // 四舍五入到最近的 float16
```

---

### 6. Promise.try

```js
// 统一处理同步和异步函数，总返回 Promise
Promise.try(() => JSON.parse(str))
  .then((data) => use(data))
  .catch((err) => console.error(err)); // 捕获同步和异步错误

// 替代之前的
Promise.resolve().then(() => JSON.parse(str));
```

---

## 特性速查表

| 特性                  | 版本 | 一句话               |
| --------------------- | ---- | -------------------- |
| let/const             | ES6  | 块级作用域变量       |
| 箭头函数              | ES6  | 简洁函数 + 继承 this |
| 模板字符串            | ES6  | 字符串插值           |
| 解构赋值              | ES6  | 快速提取值           |
| 默认参数              | ES6  | 函数参数默认值       |
| Rest/Spread           | ES6  | 收集/展开            |
| Class                 | ES6  | 面向对象语法         |
| Modules               | ES6  | import/export        |
| Promise               | ES6  | 异步处理             |
| Symbol                | ES6  | 唯一值类型           |
| Map/Set               | ES6  | 新数据结构           |
| Generator             | ES6  | 可暂停函数           |
| Proxy/Reflect         | ES6  | 元编程               |
| Array.includes        | ES7  | 包含检测             |
| `**`                  | ES7  | 指数运算符           |
| async/await           | ES8  | 同步风格异步         |
| Object.entries        | ES8  | 对象转 entries       |
| padStart/End          | ES8  | 字符串填充           |
| 对象 Rest/Spread      | ES9  | `{...obj}`           |
| for await...of        | ES9  | 异步迭代             |
| Promise.finally       | ES9  | 总会执行的回调       |
| RegExp 命名组         | ES9  | `(?<name>...)`       |
| Array.flat            | ES10 | 扁平化数组           |
| Object.fromEntries    | ES10 | entries 转对象       |
| trimStart/End         | ES10 | 去首/尾空格          |
| 可选 catch            | ES10 | catch 省略参数       |
| BigInt                | ES11 | 大整数               |
| 可选链 `?.`           | ES11 | 安全访问属性         |
| 空值合并 `??`         | ES11 | null 默认值          |
| Promise.allSettled    | ES11 | 等全部完成           |
| globalThis            | ES11 | 跨环境全局对象       |
| 动态 import()         | ES11 | 按需加载             |
| matchAll              | ES11 | 所有正则匹配         |
| replaceAll            | ES12 | 替换所有             |
| Promise.any           | ES12 | 任一成功             |
| 逻辑赋值              | ES12 | `&&=` `\|\|=` `??=`  |
| 数字分隔符            | ES12 | `1_000_000`          |
| 私有字段              | ES13 | `#field`             |
| 静态类块              | ES13 | `static { }`         |
| Top-level await       | ES13 | 模块顶层 await       |
| Array.at()            | ES13 | 支持负索引           |
| Object.hasOwn         | ES13 | 安全 hasOwnProperty  |
| Error.cause           | ES13 | 错误链               |
| findLast              | ES14 | 从右查找             |
| toSorted/Reversed     | ES14 | 不可变排序/反转      |
| Promise.withResolvers | ES15 | 暴露 resolve/reject  |
| Object.groupBy        | ES15 | 分组                 |
| isWellFormed          | ES15 | 检测代理字符         |
| Set 集合方法          | ES16 | 交/并/差集           |
| Iterator Helpers      | ES16 | 迭代器链式操作       |
| Import Attributes     | ES16 | `with { type }`      |
| Float16Array          | ES16 | 半精度浮点数组       |
| Promise.try           | ES16 | 统一同步/异步        |
