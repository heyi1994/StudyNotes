# TypeScript

TypeScript 是 JavaScript 的超集，添加了静态类型系统。所有合法的 JavaScript 代码都是合法的 TypeScript 代码。TS 代码最终编译为普通 JS 运行。

对熟悉 Kotlin/Java/Dart 的开发者来说，TypeScript 的类型系统与它们类似，但更灵活（结构化类型，而非名义类型）。

---

## 安装与配置

```shell
npm install -D typescript
npx tsc --init     # 生成 tsconfig.json
```

### 常用 tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020", // 编译目标 JS 版本
    "module": "ESNext", // 模块系统
    "moduleResolution": "bundler", // 配合 Vite/webpack
    "strict": true, // 开启所有严格检查（强烈推荐）
    "noUncheckedIndexedAccess": true, // 数组访问返回 T | undefined
    "exactOptionalPropertyTypes": true, // 严格区分 undefined 和可选属性
    "lib": ["ES2020", "DOM"],
    "jsx": "react-jsx", // React JSX 支持
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] },
    "outDir": "dist",
    "declaration": true // 生成 .d.ts 类型声明文件
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

---

## 基础类型

TypeScript 的原始类型对应关系：

| TypeScript  | Kotlin           | Java               | Dart               |
| ----------- | ---------------- | ------------------ | ------------------ |
| `string`    | `String`         | `String`           | `String`           |
| `number`    | `Int` / `Double` | `int` / `double`   | `int` / `double`   |
| `boolean`   | `Boolean`        | `boolean`          | `bool`             |
| `null`      | `null`           | `null`             | `null`             |
| `undefined` | —                | —                  | —                  |
| `any`       | `Any`            | `Object`           | `dynamic`          |
| `unknown`   | `Any`（需检查）  | `Object`（需检查） | `Object`（需检查） |
| `never`     | `Nothing`        | —                  | `Never`            |
| `void`      | `Unit`           | `void`             | `void`             |
| `bigint`    | `Long`           | `long`             | —                  |

```typescript
let name: string = "Alice";
let age: number = 25;
let isDone: boolean = false;
let nothing: null = null;
let notDefined: undefined = undefined;

// 数组（两种写法等价）
let nums: number[] = [1, 2, 3];
let strs: Array<string> = ["a", "b"];

// 元组（固定长度和类型，类似 Kotlin 的 Pair/Triple）
let point: [number, number] = [10, 20];
let entry: [string, number] = ["age", 25];

// 元组可命名
let rgb: [red: number, green: number, blue: number] = [255, 0, 128];
```

---

## 类型推断

TypeScript 能从赋值自动推断类型，无需总是显式标注（类似 Kotlin 的类型推断）：

```typescript
let message = "hello"; // 推断为 string
let count = 42; // 推断为 number
let flag = true; // 推断为 boolean
let items = [1, 2, 3]; // 推断为 number[]
let mixed = [1, "two", true]; // 推断为 (number | string | boolean)[]

// 函数返回值也会推断
function double(x: number) {
  return x * 2; // 推断返回 number
}
```

---

## 函数

```typescript
// 参数和返回值类型
function add(a: number, b: number): number {
  return a + b;
}

// 可选参数（? 等同于 Kotlin 的默认值参数，但只能为 undefined）
function greet(name: string, greeting?: string): string {
  return `${greeting ?? "Hello"}, ${name}!`;
}

// 默认参数（同 Kotlin）
function createUser(name: string, role: string = "user") {
  return { name, role };
}

// 剩余参数（类似 Kotlin vararg）
function sum(...nums: number[]): number {
  return nums.reduce((a, b) => a + b, 0);
}

// 函数类型
type Transformer = (input: string) => string;
const toUpper: Transformer = (s) => s.toUpperCase();

// 函数重载（类似 Java/Kotlin 的方法重载）
function format(value: string): string;
function format(value: number): string;
function format(value: string | number): string {
  return typeof value === "string" ? value.trim() : value.toFixed(2);
}
```

---

## 接口（Interface）

接口描述对象的结构，类似 Kotlin 的 `interface` + `data class`，或 Dart 的抽象类：

```typescript
interface User {
  id: number;
  name: string;
  email?: string; // 可选属性（类似 Kotlin 可空类型 String?）
  readonly createdAt: Date; // 只读属性（类似 Kotlin val）
}

// 使用接口
const user: User = {
  id: 1,
  name: "Alice",
  createdAt: new Date(),
};

// 接口可以描述函数
interface Comparator {
  (a: number, b: number): number;
}

// 接口可以描述索引签名（类似 Map<String, Any>）
interface Dictionary {
  [key: string]: string;
}
```

### 接口继承

```typescript
interface Animal {
  name: string;
  sound(): string;
}

interface Pet extends Animal {
  owner: string;
}

interface ServiceDog extends Pet {
  certificationId: string;
}

// 多继承（Java 接口/Kotlin 接口都支持）
interface Flyable {
  fly(): void;
}
interface Swimmable {
  swim(): void;
}
interface Duck extends Animal, Flyable, Swimmable {}
```

---

## 类型别名（Type Alias）

`type` 比 `interface` 更灵活，可以表达联合类型、交叉类型等：

```typescript
type ID = string | number; // 联合类型
type Point = { x: number; y: number };
type Callback = () => void;
type AsyncResult<T> = Promise<T | null>;

// interface 和 type 的主要区别：
// - interface 可以被重复声明并自动合并（声明合并）
// - type 不能重复声明
// - type 可以表达联合/交叉/条件类型，interface 不行
// - 扩展类时，类只能 implements interface 或 type（对象形状）
```

---

## 联合类型与交叉类型

### 联合类型（`|`）

类似 Kotlin 的 `sealed class`，表示"或"关系：

```typescript
type StringOrNumber = string | number;
type Status = "loading" | "success" | "error"; // 字面量联合类型
type ID = string | number;

function printId(id: ID) {
  if (typeof id === "string") {
    console.log(id.toUpperCase()); // 这里 TS 知道 id 是 string
  } else {
    console.log(id.toFixed(0)); // 这里 TS 知道 id 是 number
  }
}
```

### 交叉类型（`&`）

类似 Kotlin 的接口实现，将多个类型合并为一个，表示"且"关系：

```typescript
type Serializable = { serialize(): string };
type Loggable = { log(): void };
type Service = Serializable & Loggable; // 必须同时具备两者

// 常用于混入（Mixin）模式
type WithTimestamp<T> = T & { createdAt: Date; updatedAt: Date };
type UserWithTimestamp = WithTimestamp<User>;
```

---

## 字面量类型

字面量可以作为类型，比枚举更轻量：

```typescript
type Direction = "up" | "down" | "left" | "right"
type Digit = 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
type BooleanLike = true | false | 0 | 1

function move(direction: Direction, steps: number) { ... }
move("up", 3)    // ✓
move("UP", 3)    // ✗ 类型错误

// 模板字面量类型（TS 4.1+）
type EventName = `on${Capitalize<string>}`  // onCLick, onChange...
type CssValue = `${number}px` | `${number}rem` | `${number}%`
type Getter<T extends string> = `get${Capitalize<T>}`
type Setter<T extends string> = `set${Capitalize<T>}`
```

---

## 枚举（Enum）

```typescript
// 数字枚举（类似 Java/Kotlin enum，默认从 0 开始）
enum Direction {
  Up, // 0
  Down, // 1
  Left, // 2
  Right, // 3
}

// 字符串枚举（推荐，调试友好）
enum Status {
  Loading = "LOADING",
  Success = "SUCCESS",
  Error = "ERROR",
}

console.log(Status.Loading); // "LOADING"
console.log(Direction.Up); // 0

// const enum：编译时内联，不生成 JS 对象（性能更好）
const enum Color {
  Red,
  Green,
  Blue,
}
const c = Color.Red; // 编译为 const c = 0
```

> 很多 TS 项目更倾向用**字面量联合类型**替代枚举，因为枚举会生成额外的 JS 运行时代码，而字面量类型是纯类型检查，零运行时开销。

---

## 类

TypeScript 的类与 Kotlin/Java 非常相似：

```typescript
class Animal {
  // 属性声明（Kotlin 中在构造函数参数里）
  private name: string;
  protected age: number;
  readonly species: string;

  // 构造函数
  constructor(name: string, age: number, species: string) {
    this.name = name;
    this.age = age;
    this.species = species;
  }

  // 简写：参数直接声明为属性（类似 Kotlin 主构造函数）
  // constructor(private name: string, protected age: number) {}

  speak(): string {
    return `${this.name} makes a sound`;
  }

  // getter / setter（类似 Kotlin 属性访问器）
  get displayName(): string {
    return this.name.toUpperCase();
  }

  set displayName(value: string) {
    this.name = value.trim();
  }

  // 静态方法
  static create(name: string): Animal {
    return new Animal(name, 0, "unknown");
  }
}

class Dog extends Animal {
  constructor(
    name: number,
    private breed: string,
  ) {
    super(name, age, "Canis lupus"); // 必须先调用 super
  }

  // 方法重写
  override speak(): string {
    return `${super.speak()}: Woof!`;
  }
}
```

### 参数属性简写

```typescript
// 普通写法
class Point {
  public x: number;
  public y: number;
  constructor(x: number, y: number) {
    this.x = x;
    this.y = y;
  }
}

// 简写（类似 Kotlin 主构造函数 val/var）
class Point {
  constructor(
    public x: number,
    public y: number,
  ) {}
}
```

### 抽象类

```typescript
abstract class Shape {
  abstract area(): number; // 子类必须实现（类似 Kotlin abstract fun）
  abstract perimeter(): number;

  describe(): string {
    return `面积：${this.area()}, 周长：${this.perimeter()}`;
  }
}

class Circle extends Shape {
  constructor(private radius: number) {
    super();
  }

  area(): number {
    return Math.PI * this.radius ** 2;
  }
  perimeter(): number {
    return 2 * Math.PI * this.radius;
  }
}
```

### 实现接口

```typescript
interface Printable {
  print(): void;
}

interface Serializable {
  serialize(): string;
}

// 类可以实现多个接口（同 Java/Kotlin）
class Document implements Printable, Serializable {
  constructor(private content: string) {}

  print(): void {
    console.log(this.content);
  }
  serialize(): string {
    return JSON.stringify({ content: this.content });
  }
}
```

---

## 泛型

TypeScript 泛型与 Kotlin/Java/Dart 泛型概念基本一致：

```typescript
// 泛型函数（类似 Kotlin <T> fun）
function identity<T>(value: T): T {
  return value;
}

function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

// 多类型参数
function zip<A, B>(a: A[], b: B[]): [A, B][] {
  return a.map((item, i) => [item, b[i]]);
}

// 泛型接口
interface Repository<T, ID = number> {
  findById(id: ID): Promise<T | null>;
  findAll(): Promise<T[]>;
  save(entity: T): Promise<T>;
  delete(id: ID): Promise<void>;
}

// 泛型类
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }
  pop(): T | undefined {
    return this.items.pop();
  }
  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }
  get size(): number {
    return this.items.length;
  }
}

const stack = new Stack<number>();
stack.push(1);
stack.push(2);
```

### 泛型约束（`extends`）

类似 Kotlin 的泛型上界 `<T : Comparable<T>>`：

```typescript
// 约束 T 必须有 length 属性
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
}

longest("hello", "hi"); // ✓ string 有 length
longest([1, 2], [3]); // ✓ array 有 length
longest(10, 20); // ✗ number 没有 length

// 约束键名必须是对象的属性（类似 Kotlin 的 reified + 反射）
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "Alice", age: 25 };
getProperty(user, "name"); // ✓ 返回类型为 string
getProperty(user, "age"); // ✓ 返回类型为 number
getProperty(user, "email"); // ✗ 不存在的属性
```

### 默认类型参数

```typescript
// 类似 Kotlin 的默认参数，但用于泛型
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
  message: string;
}

type UserResponse = ApiResponse<User>; // 显式指定
type RawResponse = ApiResponse; // 使用默认 unknown
```

---

## 类型守卫（Type Guards）

类型守卫用于在运行时缩窄类型范围，类似 Kotlin 的 `is` + 智能转换：

### typeof

```typescript
function process(value: string | number) {
  if (typeof value === "string") {
    return value.toUpperCase(); // TS 知道这里是 string
  }
  return value.toFixed(2); // TS 知道这里是 number
}
```

### instanceof

```typescript
class Cat {
  meow() {}
}
class Dog {
  bark() {}
}

function makeSound(animal: Cat | Dog) {
  if (animal instanceof Cat) {
    animal.meow(); // 智能转换为 Cat
  } else {
    animal.bark(); // 智能转换为 Dog
  }
}
```

### in 操作符

```typescript
interface Fish {
  swim(): void;
}
interface Bird {
  fly(): void;
}

function move(animal: Fish | Bird) {
  if ("swim" in animal) {
    animal.swim(); // 类型缩窄为 Fish
  } else {
    animal.fly(); // 类型缩窄为 Bird
  }
}
```

### 自定义类型守卫（`is`）

类似 Kotlin 的 `is` 检查，但需要手动编写断言函数：

```typescript
interface Dog {
  bark(): void;
  breed: string;
}
interface Cat {
  meow(): void;
  indoor: boolean;
}

// 返回类型 "value is Dog" 即类型谓词
function isDog(animal: Dog | Cat): animal is Dog {
  return "bark" in animal;
}

function handlePet(pet: Dog | Cat) {
  if (isDog(pet)) {
    console.log(pet.breed); // 已知是 Dog
  } else {
    console.log(pet.indoor); // 已知是 Cat
  }
}
```

### 可辨识联合（Discriminated Union）

类似 Kotlin 的 `sealed class`，通过公共字段区分联合成员：

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rectangle"; width: number; height: number }
  | { kind: "triangle"; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "rectangle":
      return shape.width * shape.height;
    case "triangle":
      return (shape.base * shape.height) / 2;
    // TS 会检查是否覆盖了所有 case（穷举检查）
  }
}
```

穷举检查——利用 `never` 确保没有漏掉 case：

```typescript
function assertNever(value: never): never {
  throw new Error(`未处理的值：${JSON.stringify(value)}`);
}

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "rectangle":
      return shape.width * shape.height;
    default:
      return assertNever(shape); // 如果忘写 triangle，编译报错
  }
}
```

---

## 工具类型

TypeScript 内置了大量工具类型，类似 Kotlin 的扩展函数/标准库：

### Partial / Required / Readonly

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

// Partial<T>：所有属性变可选（类似 Kotlin copy() 的参数）
type PartialUser = Partial<User>;
// { id?: number; name?: string; email?: string }

// Required<T>：所有属性变必填（去掉所有 ?）
type RequiredUser = Required<PartialUser>;

// Readonly<T>：所有属性变只读（类似 Kotlin data class）
type FrozenUser = Readonly<User>;
const u: FrozenUser = { id: 1, name: "Alice", email: "a@b.com" };
u.name = "Bob"; // ✗ 编译错误
```

### Pick / Omit

```typescript
// Pick<T, K>：只保留指定属性（类似 Kotlin 的 select/filter）
type UserPreview = Pick<User, "id" | "name">;
// { id: number; name: string }

// Omit<T, K>：排除指定属性
type UserWithoutId = Omit<User, "id">;
// { name: string; email: string }
```

### Record

```typescript
// Record<K, V>：构造键为 K、值为 V 的对象类型（类似 Map<K, V>）
type RolePermissions = Record<string, string[]>;
type CountByStatus = Record<"active" | "inactive" | "pending", number>;

const counts: CountByStatus = {
  active: 5,
  inactive: 2,
  pending: 1,
};
```

### Extract / Exclude

```typescript
type T1 = "a" | "b" | "c";
type T2 = "b" | "c" | "d";

// Extract：取交集
type Common = Extract<T1, T2>; // "b" | "c"

// Exclude：取差集（从 T1 中排除 T2 中有的）
type OnlyInT1 = Exclude<T1, T2>; // "a"

// 常用场景：从联合类型中剔除 null/undefined
type NonNullString = Exclude<string | null | undefined, null | undefined>;
// 等价于 NonNullable<string | null | undefined>
```

### NonNullable

```typescript
type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>; // string
```

### ReturnType / Parameters

```typescript
function fetchUser(id: number, token: string): Promise<User> { ... }

type FetchReturn = ReturnType<typeof fetchUser>
// Promise<User>

type FetchParams = Parameters<typeof fetchUser>
// [id: number, token: string]

// 常用于推断第三方库函数的类型
type RouterConfig = Parameters<typeof createBrowserRouter>[0]
```

### Awaited

```typescript
type Result = Awaited<Promise<User>>; // User
type Nested = Awaited<Promise<Promise<number>>>; // number
```

### 条件类型

```typescript
// 类似三元运算符，但在类型层面
type IsString<T> = T extends string ? true : false;

type A = IsString<string>; // true
type B = IsString<number>; // false

// infer：在条件类型中推断类型变量
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;
type Value = UnwrapPromise<Promise<string>>; // string
type Same = UnwrapPromise<number>; // number

type ElementType<T> = T extends (infer U)[] ? U : never;
type Item = ElementType<string[]>; // string
```

### 映射类型

```typescript
// 遍历对象属性，逐一转换
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

type Optional<T> = {
  [K in keyof T]?: T[K];
};

type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface Person {
  name: string;
  age: number;
}
type PersonGetters = Getters<Person>;
// { getName: () => string; getAge: () => number }
```

---

## 可空性处理

TypeScript 的可空处理与 Kotlin 的可空类型（`?`）非常相似：

```typescript
// 开启 strictNullChecks 后，null 和 undefined 不能赋给其他类型
let name: string = null; // ✗ 错误
let name: string | null = null; // ✓

// 可选链（?. ）—— 等同于 Kotlin 的 ?.
const city = user?.address?.city; // user 或 address 为 null/undefined 时返回 undefined

// 空值合并（?? ）—— 等同于 Kotlin 的 ?:（Elvis 运算符）
const displayName = user?.name ?? "匿名"; // 为 null/undefined 时用右值

// 非空断言（!）—— 等同于 Kotlin 的 !!
const el = document.getElementById("app")!; // 告诉 TS 此值不为 null

// 可选参数 vs 可空参数
function f(x?: string) {} // x 类型为 string | undefined
function g(x: string | null) {} // x 必须传，但可以是 null

// 空值合并赋值 ??=
let cache: string | null = null;
cache ??= computeValue(); // 仅在 cache 为 null/undefined 时赋值
```

---

## 类型断言

类型断言告诉编译器"我比你更了解这个值的类型"，类似 Kotlin/Java 的强制类型转换，但**不做运行时检查**：

```typescript
// as 语法（推荐）
const input = document.getElementById("name") as HTMLInputElement
input.value = "hello"

// 尖括号语法（JSX 中不可用）
const input = <HTMLInputElement>document.getElementById("name")

// 双重断言（绕过类型检查，尽量避免）
const value = someValue as unknown as string

// satisfies 操作符（TS 4.9+，兼顾推断精度和类型检查）
const palette = {
  red: [255, 0, 0],
  green: "#00ff00",
} satisfies Record<string, string | number[]>

palette.red.forEach(...)   // TS 知道 red 是 number[]，不是 string
palette.green.toUpperCase() // TS 知道 green 是 string
```

---

## 声明文件（`.d.ts`）

为没有类型的 JS 库编写类型声明：

```typescript
// global.d.ts
declare global {
  interface Window {
    analytics: AnalyticsInstance;
  }
}

// 为模块补充类型
declare module "*.svg" {
  const content: string;
  export default content;
}

declare module "*.png" {
  const src: string;
  export default src;
}

// 为 JS 库编写声明（通常由 @types/xxx 包提供）
declare module "some-js-lib" {
  export function doSomething(value: string): number;
  export class SomeClass {
    constructor(options: SomeOptions);
    method(): void;
  }
}
```

安装社区维护的类型声明：

```shell
npm install -D @types/node
npm install -D @types/lodash
```

---

## 命名空间 vs 模块

现代 TypeScript 推荐使用 **ES 模块**（`import/export`），命名空间（`namespace`）主要用于类型声明文件中避免命名冲突：

```typescript
// ✓ 现代写法：ES 模块
// utils/math.ts
export function add(a: number, b: number) {
  return a + b;
}

// main.ts
import { add } from "./utils/math";

// 命名空间（主要在 .d.ts 中使用）
namespace Validation {
  export interface StringValidator {
    isAcceptable(s: string): boolean;
  }
}
```

---

## 装饰器（Decorators）

装饰器是实验性特性（需 `"experimentalDecorators": true`），类似 Java 注解 / Kotlin 注解：

```typescript
// 类装饰器
function Singleton<T extends { new (...args: any[]): {} }>(constructor: T) {
  let instance: T;
  return class extends constructor {
    constructor(...args: any[]) {
      if (!instance) {
        instance = new constructor(...args) as T;
        super(...args);
      }
      return instance;
    }
  };
}

// 方法装饰器
function Log(target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  descriptor.value = function (...args: any[]) {
    console.log(`调用 ${key}，参数：`, args);
    const result = original.apply(this, args);
    console.log(`${key} 返回：`, result);
    return result;
  };
  return descriptor;
}

class UserService {
  @Log
  findUser(id: number) {
    return { id, name: "Alice" };
  }
}
```

---

## 常用模式

### 类型安全的事件系统

```typescript
type EventMap = {
  login: { userId: string; timestamp: Date };
  logout: { userId: string };
  error: { message: string; code: number };
};

class EventEmitter<T extends Record<string, unknown>> {
  private handlers = new Map<keyof T, Set<(data: any) => void>>();

  on<K extends keyof T>(event: K, handler: (data: T[K]) => void) {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, new Set());
    }
    this.handlers.get(event)!.add(handler);
  }

  emit<K extends keyof T>(event: K, data: T[K]) {
    this.handlers.get(event)?.forEach((h) => h(data));
  }
}

const emitter = new EventEmitter<EventMap>();
emitter.on("login", ({ userId }) => console.log(userId)); // 类型安全
emitter.emit("login", { userId: "1", timestamp: new Date() });
```

### Builder 模式

```typescript
class QueryBuilder<T> {
  private filters: Partial<T> = {};
  private limitValue?: number;
  private offsetValue?: number;

  where(filter: Partial<T>): this {
    this.filters = { ...this.filters, ...filter };
    return this;
  }

  limit(n: number): this {
    this.limitValue = n;
    return this;
  }

  offset(n: number): this {
    this.offsetValue = n;
    return this;
  }

  build() {
    return {
      filters: this.filters,
      limit: this.limitValue,
      offset: this.offsetValue,
    };
  }
}

const query = new QueryBuilder<User>()
  .where({ role: "admin" })
  .limit(10)
  .offset(20)
  .build();
```

### 类型安全的本地存储

```typescript
class TypedStorage<T extends Record<string, unknown>> {
  get<K extends keyof T>(key: K): T[K] | null {
    const raw = localStorage.getItem(String(key));
    return raw ? JSON.parse(raw) : null;
  }

  set<K extends keyof T>(key: K, value: T[K]): void {
    localStorage.setItem(String(key), JSON.stringify(value));
  }

  remove<K extends keyof T>(key: K): void {
    localStorage.removeItem(String(key));
  }
}

type AppStorage = {
  token: string;
  userId: number;
  preferences: { theme: "light" | "dark"; language: string };
};

const storage = new TypedStorage<AppStorage>();
storage.set("token", "abc123"); // ✓
storage.set("userId", 42); // ✓
storage.set("userId", "hello"); // ✗ 类型错误
storage.get("preferences")?.theme; // 类型为 "light" | "dark" | undefined
```

---

## 与 Kotlin/Java/Dart 对比速查

| 概念         | TypeScript               | Kotlin              | Java               | Dart                 |
| ------------ | ------------------------ | ------------------- | ------------------ | -------------------- |
| 可空类型     | `string \| null`         | `String?`           | `@Nullable String` | `String?`            |
| 可选链       | `a?.b`                   | `a?.b`              | —                  | `a?.b`               |
| Elvis/空合并 | `a ?? b`                 | `a ?: b`            | —                  | `a ?? b`             |
| 非空断言     | `a!`                     | `a!!`               | —                  | `a!`                 |
| 类型判断     | `typeof` / `instanceof`  | `is`                | `instanceof`       | `is`                 |
| 智能转换     | 类型守卫后自动           | `is` 后自动         | 需手动转换         | `is` 后自动          |
| 密封类       | 可辨识联合               | `sealed class`      | `sealed class`     | `sealed class`       |
| 数据类       | `interface` + `Readonly` | `data class`        | Record (Java 16)   | —                    |
| 扩展函数     | 无（用模块函数代替）     | 扩展函数            | 无                 | 扩展方法             |
| 泛型约束     | `<T extends U>`          | `<T : U>`           | `<T extends U>`    | `<T extends U>`      |
| 协变/逆变    | 结构化类型，自动         | `out T` / `in T`    | `? extends T`      | —                    |
| 枚举         | `enum` / 字面量联合      | `enum class`        | `enum`             | `enum`               |
| 解构         | `const {a, b} = obj`     | `val (a, b) = pair` | —                  | `final (a, b) = ...` |
| 扩展运算符   | `...arr`                 | `*arr`              | —                  | `...arr`             |
