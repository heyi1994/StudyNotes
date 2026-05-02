# React

React 是由 Meta 开发的用于构建用户界面的 JavaScript 库。它采用组件化思想，通过声明式编程描述 UI 状态，并借助虚拟 DOM 高效更新界面。

---

## 安装

使用 Vite 创建项目（推荐）：

```shell
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install
npm run dev
```

---

## JSX

JSX 是 JavaScript 的语法扩展，允许在 JS 中书写类似 HTML 的标记。

```tsx
const element = <h1 className="title">Hello, React!</h1>;
```

### JSX 规则

- 只能返回**单个根元素**，可使用 `<>...</>` Fragment 包裹
- 所有标签必须**闭合**（包括 `<img />`, `<br />`）
- 属性名使用**驼峰命名**：`class` → `className`，`for` → `htmlFor`
- 用 `{}` 嵌入 JavaScript 表达式

```tsx
function Profile({ user }: { user: { name: string; avatar: string } }) {
  const greeting = `你好，${user.name}`;

  return (
    <>
      <h1>{greeting}</h1>
      <img src={user.avatar} alt={user.name} className="avatar" />
      <p>注册于：{new Date().getFullYear()} 年</p>
    </>
  );
}
```

---

## 组件

React 组件是返回 JSX 的函数，**组件名必须大写字母开头**。

```tsx
// 函数组件
function Greeting({ name }: { name: string }) {
  return <h1>你好，{name}！</h1>;
}

// 箭头函数写法
const Button = ({
  onClick,
  children,
}: React.ButtonHTMLAttributes<HTMLButtonElement>) => (
  <button onClick={onClick}>{children}</button>
);
```

### 导出组件

```tsx
// 命名导出
export function Card() { ... }

// 默认导出
export default function App() { ... }
```

---

## Props

Props 是父组件向子组件传递数据的方式，是只读的。

```tsx
interface CardProps {
  title: string;
  description?: string; // 可选 prop
  count: number;
  onClose: () => void;
}

function Card({ title, description = "暂无描述", count, onClose }: CardProps) {
  return (
    <div className="card">
      <h2>{title}</h2>
      <p>{description}</p>
      <span>数量：{count}</span>
      <button onClick={onClose}>关闭</button>
    </div>
  );
}

// 使用
<Card title="标题" count={5} onClose={() => console.log("closed")} />;
```

### children prop

```tsx
function Panel({
  title,
  children,
}: {
  title: string;
  children: React.ReactNode;
}) {
  return (
    <div className="panel">
      <h3>{title}</h3>
      <div>{children}</div>
    </div>
  );
}

// 使用
<Panel title="用户信息">
  <p>姓名：张三</p>
  <p>年龄：25</p>
</Panel>;
```

### 展开 Props

```tsx
function Input({
  label,
  ...inputProps
}: { label: string } & React.InputHTMLAttributes<HTMLInputElement>) {
  return (
    <div>
      <label>{label}</label>
      <input {...inputProps} />
    </div>
  );
}
```

---

## 条件渲染

```tsx
function UserStatus({
  isLoggedIn,
  isAdmin,
}: {
  isLoggedIn: boolean;
  isAdmin?: boolean;
}) {
  return (
    <div>
      {/* 三元运算符 */}
      {isLoggedIn ? <p>欢迎回来</p> : <p>请登录</p>}

      {/* && 短路（条件为 true 时渲染） */}
      {isAdmin && <button>管理后台</button>}

      {/* 注意：0 不会短路，要显式转为 boolean */}
      {!!count && <span>{count} 条消息</span>}
    </div>
  );
}
```

---

## 列表渲染

使用 `Array.map()` 渲染列表，每个元素必须有唯一的 `key`：

```tsx
interface Product {
  id: number;
  name: string;
  price: number;
}

function ProductList({ products }: { products: Product[] }) {
  if (products.length === 0) {
    return <p>暂无商品</p>;
  }

  return (
    <ul>
      {products.map((product) => (
        <li key={product.id}>
          {product.name} — ¥{product.price}
        </li>
      ))}
    </ul>
  );
}
```

> `key` 应使用数据的唯一标识（如 ID），避免使用数组下标，否则排序/删除时会出现 bug。

---

## 事件处理

```tsx
function EventDemo() {
  function handleClick(e: React.MouseEvent<HTMLButtonElement>) {
    e.preventDefault();
    console.log("点击了按钮");
  }

  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    console.log("输入值：", e.target.value);
  }

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    // 处理表单
  }

  return (
    <form onSubmit={handleSubmit}>
      <input onChange={handleChange} />
      <button onClick={handleClick}>提交</button>
    </form>
  );
}
```

---

## useState

`useState` 用于在组件中存储状态，状态更新会触发重新渲染。

```tsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState("");
  const [items, setItems] = useState<string[]>([]);

  function increment() {
    setCount((c) => c + 1); // 使用函数式更新确保基于最新值
  }

  function addItem(item: string) {
    setItems((prev) => [...prev, item]); // 不可变更新
  }

  return (
    <div>
      <p>计数：{count}</p>
      <button onClick={increment}>+1</button>
    </div>
  );
}
```

### 更新对象/数组状态

React 状态应视为**不可变的**，更新时需创建新引用：

```tsx
const [user, setUser] = useState({ name: "张三", age: 25 });
const [todos, setTodos] = useState([
  { id: 1, text: "学习 React", done: false },
]);

// 更新对象的某个字段
setUser((prev) => ({ ...prev, age: 26 }));

// 数组：添加
setTodos((prev) => [...prev, { id: 2, text: "写文档", done: false }]);

// 数组：删除
setTodos((prev) => prev.filter((todo) => todo.id !== 1));

// 数组：修改
setTodos((prev) =>
  prev.map((todo) => (todo.id === 1 ? { ...todo, done: true } : todo)),
);
```

### 惰性初始化

初始值需要复杂计算时，传入函数避免每次渲染都执行：

```tsx
const [data, setData] = useState(() => parseLocalStorage());
```

---

## useEffect

`useEffect` 用于处理副作用：数据请求、订阅、手动操作 DOM 等。

```tsx
import { useState, useEffect } from "react";

function UserProfile({ userId }: { userId: number }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);

    fetch(`/api/users/${userId}`)
      .then((r) => r.json())
      .then((data) => {
        if (!cancelled) {
          setUser(data);
          setLoading(false);
        }
      });

    return () => {
      cancelled = true; // 清理函数：组件卸载或 userId 变化时执行
    };
  }, [userId]); // 依赖数组：userId 变化时重新执行

  if (loading) return <p>加载中...</p>;
  return <div>{user?.name}</div>;
}
```

### 依赖数组规则

| 写法        | 行为                  |
| ----------- | --------------------- |
| 无依赖数组  | 每次渲染后都执行      |
| `[]` 空数组 | 仅在挂载时执行一次    |
| `[a, b]`    | `a` 或 `b` 变化时执行 |

### 常见副作用模式

```tsx
// 订阅事件
useEffect(() => {
  window.addEventListener("resize", handleResize);
  return () => window.removeEventListener("resize", handleResize);
}, []);

// 定时器
useEffect(() => {
  const timer = setInterval(() => setTime(new Date()), 1000);
  return () => clearInterval(timer);
}, []);

// 同步外部状态
useEffect(() => {
  document.title = `${count} 条未读消息`;
}, [count]);
```

---

## useContext

Context 用于跨组件层级共享数据，避免 prop drilling。

```tsx
import { createContext, useContext, useState } from "react";

// 1. 创建 Context
interface ThemeContextType {
  theme: "light" | "dark";
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextType | null>(null);

// 2. 提供 Context
function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<"light" | "dark">("light");

  function toggleTheme() {
    setTheme((t) => (t === "light" ? "dark" : "light"));
  }

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// 3. 消费 Context
function ThemeButton() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("ThemeButton must be inside ThemeProvider");

  return <button onClick={ctx.toggleTheme}>当前主题：{ctx.theme}</button>;
}

// 4. 包裹应用
function App() {
  return (
    <ThemeProvider>
      <ThemeButton />
    </ThemeProvider>
  );
}
```

---

## useRef

`useRef` 有两种用途：持有 DOM 引用和保存不触发渲染的可变值。

```tsx
import { useRef, useEffect } from "react";

function TextInput() {
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    inputRef.current?.focus(); // 挂载后自动聚焦
  }, []);

  return <input ref={inputRef} />;
}
```

保存不触发渲染的值（如定时器 ID、上一次的值）：

```tsx
function StopWatch() {
  const [time, setTime] = useState(0);
  const timerRef = useRef<number | null>(null);

  function start() {
    timerRef.current = setInterval(() => setTime((t) => t + 1), 1000);
  }

  function stop() {
    if (timerRef.current) clearInterval(timerRef.current);
  }

  return (
    <div>
      <p>{time}s</p>
      <button onClick={start}>开始</button>
      <button onClick={stop}>停止</button>
    </div>
  );
}
```

### 保存上一次的值

```tsx
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>();
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}
```

---

## useReducer

`useReducer` 适合管理复杂的状态逻辑，是 `useState` 的替代方案。

```tsx
import { useReducer } from "react";

interface State {
  count: number;
  step: number;
}

type Action =
  | { type: "increment" }
  | { type: "decrement" }
  | { type: "reset" }
  | { type: "setStep"; payload: number };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "increment":
      return { ...state, count: state.count + state.step };
    case "decrement":
      return { ...state, count: state.count - state.step };
    case "reset":
      return { count: 0, step: state.step };
    case "setStep":
      return { ...state, step: action.payload };
    default:
      return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0, step: 1 });

  return (
    <div>
      <p>
        计数：{state.count}，步长：{state.step}
      </p>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "decrement" })}>-</button>
      <button onClick={() => dispatch({ type: "reset" })}>重置</button>
      <input
        type="number"
        value={state.step}
        onChange={(e) =>
          dispatch({ type: "setStep", payload: Number(e.target.value) })
        }
      />
    </div>
  );
}
```

---

## useMemo

`useMemo` 缓存计算结果，仅在依赖变化时重新计算，避免每次渲染都执行昂贵运算。

```tsx
import { useMemo, useState } from "react";

function ProductList({ products }: { products: Product[] }) {
  const [filterText, setFilterText] = useState("");
  const [sortBy, setSortBy] = useState<"name" | "price">("name");

  const filtered = useMemo(() => {
    return products
      .filter((p) => p.name.includes(filterText))
      .sort((a, b) => (a[sortBy] > b[sortBy] ? 1 : -1));
  }, [products, filterText, sortBy]); // 依赖变化时才重新计算

  return (
    <ul>
      {filtered.map((p) => (
        <li key={p.id}>{p.name}</li>
      ))}
    </ul>
  );
}
```

> 不要过度使用 `useMemo`，只在真正有性能问题时才添加。

---

## useCallback

`useCallback` 缓存函数引用，避免子组件因函数引用变化而不必要地重渲染。

```tsx
import { useCallback, useState } from "react";

function Parent() {
  const [count, setCount] = useState(0);
  const [otherState, setOtherState] = useState(0);

  // 没有 useCallback：otherState 变化时 handleClick 会重新创建，导致 Child 重渲染
  // 有 useCallback：仅 count 变化时才重新创建
  const handleClick = useCallback(() => {
    setCount((c) => c + 1);
  }, []); // 空依赖，函数永不重建

  return (
    <>
      <Child onClick={handleClick} />
      <button onClick={() => setOtherState((n) => n + 1)}>其他操作</button>
    </>
  );
}

// 需配合 React.memo 才有实际意义
const Child = React.memo(({ onClick }: { onClick: () => void }) => {
  console.log("Child 渲染");
  return <button onClick={onClick}>+1</button>;
});
```

---

## useId

生成稳定唯一 ID，适合关联表单 label 和 input（支持服务端渲染）：

```tsx
import { useId } from "react";

function FormField({ label }: { label: string }) {
  const id = useId();
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} type="text" />
    </div>
  );
}
```

---

## useTransition

将状态更新标记为非紧急，保持 UI 响应性，避免大列表渲染时界面卡顿：

```tsx
import { useState, useTransition } from "react";

function SearchResults() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<string[]>([]);
  const [isPending, startTransition] = useTransition();

  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    setQuery(e.target.value); // 紧急更新：输入框立即响应

    startTransition(() => {
      setResults(searchItems(e.target.value)); // 非紧急：可以被中断
    });
  }

  return (
    <div>
      <input value={query} onChange={handleChange} />
      {isPending ? (
        <p>搜索中...</p>
      ) : (
        <ul>
          {results.map((r) => (
            <li key={r}>{r}</li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

## useDeferredValue

延迟某个值的更新，将渲染优先级降低，适合实时搜索等场景：

```tsx
import { useState, useDeferredValue } from "react";

function Search() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query); // 延迟跟随 query

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      {/* 使用 deferredQuery，不影响输入框的即时响应 */}
      <ResultList query={deferredQuery} />
    </div>
  );
}
```

---

## 自定义 Hook

将可复用的状态逻辑提取为自定义 Hook（以 `use` 开头）：

### useFetch

```tsx
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    setError(null);

    fetch(url)
      .then((r) => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);
        return r.json();
      })
      .then((data) => {
        if (!cancelled) {
          setData(data);
          setLoading(false);
        }
      })
      .catch((err) => {
        if (!cancelled) {
          setError(err);
          setLoading(false);
        }
      });

    return () => {
      cancelled = true;
    };
  }, [url]);

  return { data, loading, error };
}

// 使用
function UserProfile({ id }: { id: number }) {
  const { data: user, loading, error } = useFetch<User>(`/api/users/${id}`);

  if (loading) return <p>加载中...</p>;
  if (error) return <p>出错了：{error.message}</p>;
  return <div>{user?.name}</div>;
}
```

### useLocalStorage

```tsx
function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  function setStoredValue(newValue: T | ((prev: T) => T)) {
    const resolved = newValue instanceof Function ? newValue(value) : newValue;
    setValue(resolved);
    localStorage.setItem(key, JSON.stringify(resolved));
  }

  return [value, setStoredValue] as const;
}
```

### useDebounce

```tsx
function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debounced;
}
```

---

## React.memo

`React.memo` 包裹组件，仅在 props 变化时重新渲染：

```tsx
interface ItemProps {
  id: number;
  name: string;
  onDelete: (id: number) => void;
}

const Item = React.memo(function Item({ id, name, onDelete }: ItemProps) {
  console.log(`Item ${id} 渲染`);
  return (
    <li>
      {name}
      <button onClick={() => onDelete(id)}>删除</button>
    </li>
  );
});
```

配合 `useCallback` 避免因函数引用变化导致 memo 失效：

```tsx
function List({ items }: { items: { id: number; name: string }[] }) {
  const handleDelete = useCallback((id: number) => {
    // 删除逻辑
  }, []);

  return (
    <ul>
      {items.map((item) => (
        <Item
          key={item.id}
          id={item.id}
          name={item.name}
          onDelete={handleDelete}
        />
      ))}
    </ul>
  );
}
```

---

## 懒加载与 Suspense

`React.lazy` + `Suspense` 实现组件代码分割，减小初始包体积：

```tsx
import { lazy, Suspense } from "react";

const Dashboard = lazy(() => import("./pages/Dashboard"));
const Settings = lazy(() => import("./pages/Settings"));

function App() {
  return (
    <Suspense fallback={<div>页面加载中...</div>}>
      <Dashboard />
    </Suspense>
  );
}
```

嵌套 Suspense 边界可以更精细地控制加载状态：

```tsx
function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Header />
      <Suspense fallback={<ContentSkeleton />}>
        <MainContent />
      </Suspense>
    </Suspense>
  );
}
```

---

## forwardRef

将 ref 转发给子组件内部的 DOM 元素：

```tsx
import { forwardRef } from "react";

interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
}

const Input = forwardRef<HTMLInputElement, InputProps>(function Input(
  { label, ...props },
  ref,
) {
  return (
    <div>
      <label>{label}</label>
      <input ref={ref} {...props} />
    </div>
  );
});

// 使用
function Form() {
  const inputRef = useRef<HTMLInputElement>(null);

  return (
    <>
      <Input ref={inputRef} label="用户名" />
      <button onClick={() => inputRef.current?.focus()}>聚焦</button>
    </>
  );
}
```

---

## Portal

将子组件渲染到 DOM 树的其他位置（如 body），常用于模态框、提示框：

```tsx
import { createPortal } from "react-dom";

function Modal({
  isOpen,
  onClose,
  children,
}: {
  isOpen: boolean;
  onClose: () => void;
  children: React.ReactNode;
}) {
  if (!isOpen) return null;

  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        {children}
        <button onClick={onClose}>关闭</button>
      </div>
    </div>,
    document.body, // 渲染到 body，不受父级 overflow:hidden 影响
  );
}
```

---

## 错误边界

类组件实现的错误边界，捕获子树中的渲染错误，防止整个应用崩溃：

```tsx
import { Component, ErrorInfo, ReactNode } from "react";

interface Props {
  fallback: ReactNode;
  children: ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    console.error("组件错误：", error, info.componentStack);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}

// 使用
function App() {
  return (
    <ErrorBoundary fallback={<p>组件出错了，请刷新页面</p>}>
      <UserProfile />
    </ErrorBoundary>
  );
}
```

---

## 受控 vs 非受控组件

### 受控组件（推荐）

表单值由 React state 控制：

```tsx
function ControlledForm() {
  const [value, setValue] = useState("");

  return <input value={value} onChange={(e) => setValue(e.target.value)} />;
}
```

### 非受控组件

通过 ref 直接读取 DOM 值，适合文件上传等场景：

```tsx
function UncontrolledForm() {
  const inputRef = useRef<HTMLInputElement>(null);

  function handleSubmit() {
    console.log(inputRef.current?.value);
  }

  return (
    <>
      <input ref={inputRef} defaultValue="初始值" />
      <button onClick={handleSubmit}>提交</button>
    </>
  );
}
```

---

## 状态提升

当多个组件需要共享状态时，将状态提升到最近的公共父组件：

```tsx
function TemperatureConverter() {
  const [celsius, setCelsius] = useState("");

  const fahrenheit = celsius !== "" ? (Number(celsius) * 9) / 5 + 32 : "";

  return (
    <div>
      <TemperatureInput scale="摄氏度" value={celsius} onChange={setCelsius} />
      <TemperatureInput
        scale="华氏度"
        value={String(fahrenheit)}
        onChange={(f) => setCelsius(String(((Number(f) - 32) * 5) / 9))}
      />
    </div>
  );
}
```

---

## 常用 Hooks 速查

| Hook                  | 用途                            |
| --------------------- | ------------------------------- |
| `useState`            | 组件内部状态                    |
| `useEffect`           | 副作用（请求、订阅、DOM 操作）  |
| `useContext`          | 消费 Context                    |
| `useRef`              | DOM 引用 / 可变值（不触发渲染） |
| `useReducer`          | 复杂状态逻辑                    |
| `useMemo`             | 缓存计算结果                    |
| `useCallback`         | 缓存函数引用                    |
| `useId`               | 生成唯一 ID                     |
| `useTransition`       | 将更新标记为非紧急              |
| `useDeferredValue`    | 延迟值更新                      |
| `useLayoutEffect`     | 同步副作用（DOM 读取/测量）     |
| `useImperativeHandle` | 自定义 ref 暴露的方法           |
