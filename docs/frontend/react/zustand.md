# Zustand

Zustand 是一个轻量、简洁的 React 状态管理库。相比 Redux，它没有 Action/Reducer/Provider 等概念，只需定义一个 Store 函数即可使用，非常适合中小型项目的全局状态管理。

**核心特点：**

- 无需 Provider 包裹，直接 import 使用
- 基于 Hook，语法极简
- 按需订阅，精细控制重渲染
- 内置 DevTools、持久化、Immer 等中间件支持
- 包体积极小（约 1KB）

---

## 安装

```shell
npm install zustand
```

---

## 基础用法

### 创建 Store

```typescript
// src/store/counterStore.ts
import { create } from "zustand"

interface CounterState {
  count: number
  step: number
  increment: () => void
  decrement: () => void
  incrementBy: (amount: number) => void
  setStep: (step: number) => void
  reset: () => void
}

export const useCounterStore = create<CounterState>((set, get) => ({
  count: 0,
  step: 1,

  increment: () =>
    set((state) => ({ count: state.count + state.step })),

  decrement: () =>
    set((state) => ({ count: state.count - state.step })),

  incrementBy: (amount) =>
    set((state) => ({ count: state.count + amount })),

  setStep: (step) => set({ step }),

  reset: () => set({ count: 0, step: 1 }),
}))
```

### 在组件中使用

```tsx
import { useCounterStore } from "@/store/counterStore"

function Counter() {
  // 按需订阅——只有 count 变化时才重渲染，step 变化不触发
  const count     = useCounterStore((state) => state.count)
  const step      = useCounterStore((state) => state.step)
  const increment = useCounterStore((state) => state.increment)
  const decrement = useCounterStore((state) => state.decrement)
  const reset     = useCounterStore((state) => state.reset)

  return (
    <div>
      <p>计数：{count}，步长：{step}</p>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
      <button onClick={reset}>重置</button>
    </div>
  )
}
```

---

## set 与 get

`set` 用于更新状态，`get` 用于在 action 内读取当前状态：

```typescript
export const useStore = create<State>((set, get) => ({
  count: 0,
  text: "",

  // set 合并更新（浅合并，不需要展开其他字段）
  setCount: (n) => set({ count: n }),

  // set 函数式更新（基于旧值）
  increment: () => set((state) => ({ count: state.count + 1 })),

  // 第二个参数 true：完全替换 state（慎用）
  replaceAll: () => set({ count: 0, text: "reset" }, true),

  // get 读取当前值（在异步或复杂逻辑中很有用）
  logCount: () => {
    const current = get().count
    console.log("当前计数：", current)
  },

  // 在 action 中调用其他 action
  doubleIncrement: () => {
    get().increment()
    get().increment()
  },
}))
```

---

## 订阅与重渲染控制

### 按需订阅（细粒度）

```tsx
// 只订阅 count，count 变化才重渲染
const count = useStore((state) => state.count)

// 只订阅 actions（actions 引用不变，永不触发重渲染）
const { increment, reset } = useStore((state) => ({
  increment: state.increment,
  reset: state.reset,
}))
```

### 使用 useShallow 订阅多个字段

当需要同时订阅多个字段时，用 `useShallow` 做浅比较，避免每次都触发重渲染：

```tsx
import { useShallow } from "zustand/react/shallow"

function UserCard() {
  // 没有 useShallow：每次 store 变化都返回新对象引用，导致重渲染
  // 有 useShallow：只有 name 或 email 真正变化时才重渲染
  const { name, email } = useStore(
    useShallow((state) => ({
      name: state.user.name,
      email: state.user.email,
    }))
  )

  return <div>{name} - {email}</div>
}

// 订阅数组同理
const [count, step] = useStore(
  useShallow((state) => [state.count, state.step])
)
```

### 在组件外部读取/订阅

```typescript
// 在非组件代码（如工具函数、路由守卫）中访问 store
const currentCount = useCounterStore.getState().count

// 在非组件代码中修改状态
useCounterStore.setState({ count: 100 })
useCounterStore.setState((state) => ({ count: state.count + 1 }))

// 订阅 state 变化（返回取消订阅函数）
const unsubscribe = useCounterStore.subscribe(
  (state) => console.log("count changed:", state.count)
)
unsubscribe() // 取消订阅
```

---

## 异步 Action

Zustand 的 action 天然支持异步，无需任何特殊处理：

```typescript
// src/store/userStore.ts
import { create } from "zustand"

interface User {
  id: number
  name: string
  email: string
}

interface UserState {
  users: User[]
  currentUser: User | null
  status: "idle" | "loading" | "error"
  error: string | null
  fetchUsers: () => Promise<void>
  fetchUserById: (id: number) => Promise<void>
  createUser: (data: Omit<User, "id">) => Promise<User>
  deleteUser: (id: number) => Promise<void>
  clearError: () => void
}

export const useUserStore = create<UserState>((set, get) => ({
  users: [],
  currentUser: null,
  status: "idle",
  error: null,

  fetchUsers: async () => {
    set({ status: "loading", error: null })
    try {
      const res = await fetch("/api/users")
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      const users: User[] = await res.json()
      set({ users, status: "idle" })
    } catch (err) {
      set({ error: (err as Error).message, status: "error" })
    }
  },

  fetchUserById: async (id) => {
    set({ status: "loading" })
    try {
      const res = await fetch(`/api/users/${id}`)
      if (!res.ok) throw new Error("用户不存在")
      const user: User = await res.json()
      set({ currentUser: user, status: "idle" })
    } catch (err) {
      set({ error: (err as Error).message, status: "error" })
    }
  },

  createUser: async (data) => {
    const res = await fetch("/api/users", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(data),
    })
    const newUser: User = await res.json()
    // 直接更新列表，无需重新请求
    set((state) => ({ users: [...state.users, newUser] }))
    return newUser
  },

  deleteUser: async (id) => {
    await fetch(`/api/users/${id}`, { method: "DELETE" })
    set((state) => ({
      users: state.users.filter((u) => u.id !== id),
    }))
  },

  clearError: () => set({ error: null, status: "idle" }),
}))
```

```tsx
function UserList() {
  const users      = useUserStore((state) => state.users)
  const status     = useUserStore((state) => state.status)
  const fetchUsers = useUserStore((state) => state.fetchUsers)
  const deleteUser = useUserStore((state) => state.deleteUser)

  useEffect(() => {
    fetchUsers()
  }, [])

  if (status === "loading") return <p>加载中...</p>

  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>
          {u.name}
          <button onClick={() => deleteUser(u.id)}>删除</button>
        </li>
      ))}
    </ul>
  )
}
```

---

## 中间件

### devtools：Redux DevTools 支持

```typescript
import { create } from "zustand"
import { devtools } from "zustand/middleware"

export const useStore = create<State>()(
  devtools(
    (set) => ({
      count: 0,
      increment: () => set((s) => ({ count: s.count + 1 }), false, "increment"),
      //                                                    ^^^^  ^^^^^^^^^^^
      //                                          false = 合并  action 名称（DevTools 显示）
    }),
    { name: "CounterStore", enabled: process.env.NODE_ENV === "development" }
  )
)
```

### persist：状态持久化

```typescript
import { create } from "zustand"
import { persist, createJSONStorage } from "zustand/middleware"

interface SettingsState {
  theme: "light" | "dark"
  language: string
  fontSize: number
  setTheme: (theme: "light" | "dark") => void
  setLanguage: (lang: string) => void
}

export const useSettingsStore = create<SettingsState>()(
  persist(
    (set) => ({
      theme: "light",
      language: "zh",
      fontSize: 16,
      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
    }),
    {
      name: "app-settings",                        // localStorage key
      storage: createJSONStorage(() => localStorage), // 默认即 localStorage
      // 只持久化部分字段
      partialize: (state) => ({
        theme: state.theme,
        language: state.language,
      }),
      // 版本迁移
      version: 2,
      migrate: (persistedState: any, version) => {
        if (version === 1) {
          // 从 v1 迁移到 v2：字段重命名
          persistedState.language = persistedState.lang ?? "zh"
          delete persistedState.lang
        }
        return persistedState as SettingsState
      },
    }
  )
)

// 使用 sessionStorage
export const useSessionStore = create<State>()(
  persist(
    (set) => ({ ... }),
    {
      name: "session-data",
      storage: createJSONStorage(() => sessionStorage),
    }
  )
)
```

### immer：直接修改嵌套状态

```typescript
import { create } from "zustand"
import { immer } from "zustand/middleware/immer"

interface TodoState {
  todos: { id: number; text: string; done: boolean }[]
  addTodo: (text: string) => void
  toggleTodo: (id: number) => void
  removeTodo: (id: number) => void
  editTodo: (id: number, text: string) => void
}

export const useTodoStore = create<TodoState>()(
  immer((set) => ({
    todos: [],

    addTodo: (text) =>
      set((state) => {
        // 直接 push，immer 处理不可变性
        state.todos.push({ id: Date.now(), text, done: false })
      }),

    toggleTodo: (id) =>
      set((state) => {
        const todo = state.todos.find((t) => t.id === id)
        if (todo) todo.done = !todo.done    // 直接修改
      }),

    removeTodo: (id) =>
      set((state) => {
        const index = state.todos.findIndex((t) => t.id === id)
        if (index !== -1) state.todos.splice(index, 1)
      }),

    editTodo: (id, text) =>
      set((state) => {
        const todo = state.todos.find((t) => t.id === id)
        if (todo) todo.text = text
      }),
  }))
)
```

### 组合多个中间件

```typescript
import { create } from "zustand"
import { devtools, persist, immer } from "zustand/middleware"

// 中间件从外到内包裹，顺序：devtools > persist > immer
export const useStore = create<State>()(
  devtools(
    persist(
      immer((set, get) => ({
        // store 定义
      })),
      { name: "my-store" }
    ),
    { name: "MyStore" }
  )
)
```

---

## 拆分与组合 Store

### 方式一：多个独立 Store（推荐）

```typescript
// 职责单一，按功能拆分
export const useAuthStore    = create<AuthState>(...)
export const useUIStore      = create<UIState>(...)
export const useProductStore = create<ProductState>(...)
```

### 方式二：Slice 模式（大型单一 Store）

```typescript
// src/store/slices/cartSlice.ts
import type { StateCreator } from "zustand"

interface CartSlice {
  items: CartItem[]
  total: number
  addItem: (item: CartItem) => void
  removeItem: (id: number) => void
  clearCart: () => void
}

export const createCartSlice: StateCreator<
  CartSlice & UserSlice,   // 整体 Store 类型（可访问其他 slice）
  [],
  [],
  CartSlice
> = (set, get) => ({
  items: [],
  total: 0,

  addItem: (item) =>
    set((state) => {
      const next = [...state.items, item]
      return {
        items: next,
        total: next.reduce((sum, i) => sum + i.price * i.qty, 0),
      }
    }),

  removeItem: (id) =>
    set((state) => {
      const next = state.items.filter((i) => i.id !== id)
      return {
        items: next,
        total: next.reduce((sum, i) => sum + i.price * i.qty, 0),
      }
    }),

  clearCart: () => set({ items: [], total: 0 }),
})
```

```typescript
// src/store/slices/userSlice.ts
import type { StateCreator } from "zustand"

interface UserSlice {
  user: User | null
  setUser: (user: User | null) => void
}

export const createUserSlice: StateCreator<
  CartSlice & UserSlice,
  [],
  [],
  UserSlice
> = (set) => ({
  user: null,
  setUser: (user) => set({ user }),
})
```

```typescript
// src/store/index.ts
import { create } from "zustand"
import { createCartSlice } from "./slices/cartSlice"
import { createUserSlice } from "./slices/userSlice"

export const useStore = create<CartSlice & UserSlice>()((...args) => ({
  ...createCartSlice(...args),
  ...createUserSlice(...args),
}))

// 可以为每个 slice 导出专用 hook，保持使用侧简洁
export const useCart = () =>
  useStore(
    useShallow((s) => ({
      items: s.items,
      total: s.total,
      addItem: s.addItem,
      removeItem: s.removeItem,
      clearCart: s.clearCart,
    }))
  )
```

---

## 派生状态

在 selector 中计算派生值，等同于 Redux 的 `createSelector`：

```typescript
interface TodoState {
  todos: { id: number; text: string; done: boolean }[]
  filter: "all" | "active" | "completed"
}

// 在组件中用 selector 计算派生值
function TodoStats() {
  const total     = useTodoStore((s) => s.todos.length)
  const completed = useTodoStore((s) => s.todos.filter((t) => t.done).length)
  const active    = useTodoStore((s) => s.todos.filter((t) => !t.done).length)

  return <p>全部 {total} / 已完成 {completed} / 进行中 {active}</p>
}

// 过滤列表
function TodoList() {
  const filtered = useTodoStore((s) => {
    if (s.filter === "active")    return s.todos.filter((t) => !t.done)
    if (s.filter === "completed") return s.todos.filter((t) => t.done)
    return s.todos
  })

  return <ul>{filtered.map((t) => <li key={t.id}>{t.text}</li>)}</ul>
}
```

对于开销较大的派生计算，配合 `useMemo` 或直接使用 `zustand-computed`：

```tsx
import { useMemo } from "react"

function ExpensiveList() {
  const todos  = useTodoStore((s) => s.todos)
  const filter = useTodoStore((s) => s.filter)

  // 只在 todos 或 filter 变化时重新计算
  const filtered = useMemo(() => {
    return todos.filter((t) => {
      if (filter === "active")    return !t.done
      if (filter === "completed") return t.done
      return true
    })
  }, [todos, filter])

  return <ul>{filtered.map((t) => <li key={t.id}>{t.text}</li>)}</ul>
}
```

---

## 在 React 外使用

Zustand store 是独立的，不依赖 React，可以在任何地方使用：

```typescript
// 路由守卫
function requireAuth() {
  const { user } = useAuthStore.getState()
  if (!user) {
    window.location.href = "/login"
    return false
  }
  return true
}

// axios 拦截器
axios.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

// 订阅特定字段变化（按需执行副作用）
const unsubscribe = useAuthStore.subscribe(
  (state) => state.user,          // 选取要监听的字段
  (user, prevUser) => {           // user 变化时执行
    if (!user && prevUser) {
      console.log("用户已登出")
      clearSensitiveData()
    }
  }
)
```

---

## 测试

```typescript
// 测试前重置 store
import { act } from "@testing-library/react"
import { useCounterStore } from "@/store/counterStore"

beforeEach(() => {
  useCounterStore.setState({ count: 0, step: 1 })
})

test("increment 增加 step", () => {
  const { increment, setStep } = useCounterStore.getState()
  act(() => setStep(5))
  act(() => increment())
  expect(useCounterStore.getState().count).toBe(5)
})

test("reset 清零", () => {
  useCounterStore.setState({ count: 100 })
  act(() => useCounterStore.getState().reset())
  expect(useCounterStore.getState().count).toBe(0)
})
```

---

## 完整示例：购物车

```typescript
// src/store/cartStore.ts
import { create } from "zustand"
import { persist, devtools } from "zustand/middleware"
import { immer } from "zustand/middleware/immer"
import { useShallow } from "zustand/react/shallow"

interface CartItem {
  id: number
  name: string
  price: number
  qty: number
}

interface CartState {
  items: CartItem[]
  // 派生值（通过 selector 计算，不存在 state 中）
  addItem: (item: Omit<CartItem, "qty">) => void
  removeItem: (id: number) => void
  updateQty: (id: number, qty: number) => void
  clearCart: () => void
}

export const useCartStore = create<CartState>()(
  devtools(
    persist(
      immer((set) => ({
        items: [],

        addItem: (product) =>
          set((state) => {
            const existing = state.items.find((i) => i.id === product.id)
            if (existing) {
              existing.qty += 1
            } else {
              state.items.push({ ...product, qty: 1 })
            }
          }),

        removeItem: (id) =>
          set((state) => {
            const index = state.items.findIndex((i) => i.id === id)
            if (index !== -1) state.items.splice(index, 1)
          }),

        updateQty: (id, qty) =>
          set((state) => {
            const item = state.items.find((i) => i.id === id)
            if (item) {
              if (qty <= 0) {
                state.items = state.items.filter((i) => i.id !== id)
              } else {
                item.qty = qty
              }
            }
          }),

        clearCart: () => set({ items: [] }),
      })),
      { name: "cart", partialize: (s) => ({ items: s.items }) }
    ),
    { name: "CartStore" }
  )
)

// 封装派生 selector
export const useCartItems  = () => useCartStore((s) => s.items)
export const useCartCount  = () => useCartStore((s) => s.items.reduce((n, i) => n + i.qty, 0))
export const useCartTotal  = () => useCartStore((s) => s.items.reduce((n, i) => n + i.price * i.qty, 0))
export const useCartActions = () =>
  useCartStore(
    useShallow((s) => ({
      addItem: s.addItem,
      removeItem: s.removeItem,
      updateQty: s.updateQty,
      clearCart: s.clearCart,
    }))
  )
```

```tsx
// 使用
function CartSummary() {
  const items   = useCartItems()
  const count   = useCartCount()
  const total   = useCartTotal()
  const { removeItem, updateQty, clearCart } = useCartActions()

  return (
    <div>
      <h2>购物车（{count} 件）</h2>
      {items.map((item) => (
        <div key={item.id}>
          <span>{item.name}</span>
          <input
            type="number"
            value={item.qty}
            onChange={(e) => updateQty(item.id, Number(e.target.value))}
          />
          <span>¥{(item.price * item.qty).toFixed(2)}</span>
          <button onClick={() => removeItem(item.id)}>移除</button>
        </div>
      ))}
      <p>合计：¥{total.toFixed(2)}</p>
      <button onClick={clearCart}>清空购物车</button>
    </div>
  )
}
```

---

## 与 Redux 对比

| | Zustand | Redux Toolkit |
|--|---------|---------------|
| 样板代码 | 极少 | 较多（Slice/Store/Provider） |
| 需要 Provider | 不需要 | 需要 `<Provider>` |
| 异步处理 | 原生支持，直接 async/await | 需要 `createAsyncThunk` |
| 中间件 | 内置 devtools/persist/immer | 需手动配置 |
| DevTools | 支持 | 支持（功能更完整） |
| 数据请求缓存 | 无内置（配合 React Query） | RTK Query |
| 适用规模 | 中小型项目 | 大型复杂项目 |
| 学习成本 | 低 | 中 |
