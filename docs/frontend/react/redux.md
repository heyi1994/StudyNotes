# Redux

Redux 是一个可预测的状态管理库，适合管理跨组件的复杂共享状态。现代 Redux 推荐使用 **Redux Toolkit（RTK）**，它封装了大量样板代码，是官方推荐的写法。

## 核心概念

| 概念 | 说明 |
|------|------|
| **Store** | 全局唯一的状态容器 |
| **State** | 存储在 Store 中的数据快照 |
| **Action** | 描述"发生了什么"的纯对象 `{ type, payload }` |
| **Reducer** | 纯函数，接收旧 State + Action，返回新 State |
| **Dispatch** | 派发 Action，触发 Reducer 更新 State |
| **Selector** | 从 State 中读取派生数据的函数 |
| **Slice** | RTK 概念，将 Reducer + Action 合并定义 |

数据流向：`UI → dispatch(action) → reducer → new state → UI 更新`

---

## 安装

```shell
npm install @reduxjs/toolkit react-redux
```

---

## 快速入门

### 1. 创建 Slice

Slice 是 RTK 的核心，一次性定义 state、reducer 和 action：

```typescript
// src/store/counterSlice.ts
import { createSlice, PayloadAction } from "@reduxjs/toolkit"

interface CounterState {
  value: number
  step: number
}

const initialState: CounterState = {
  value: 0,
  step: 1,
}

export const counterSlice = createSlice({
  name: "counter",
  initialState,
  reducers: {
    increment(state) {
      state.value += state.step   // RTK 内置 Immer，可直接"修改"state
    },
    decrement(state) {
      state.value -= state.step
    },
    incrementByAmount(state, action: PayloadAction<number>) {
      state.value += action.payload
    },
    setStep(state, action: PayloadAction<number>) {
      state.step = action.payload
    },
    reset(state) {
      state.value = 0
    },
  },
})

// 导出 actions
export const { increment, decrement, incrementByAmount, setStep, reset } =
  counterSlice.actions

// 导出 selectors
export const selectCount = (state: RootState) => state.counter.value
export const selectStep  = (state: RootState) => state.counter.step

export default counterSlice.reducer
```

### 2. 创建 Store

```typescript
// src/store/index.ts
import { configureStore } from "@reduxjs/toolkit"
import counterReducer from "./counterSlice"
import userReducer from "./userSlice"

export const store = configureStore({
  reducer: {
    counter: counterReducer,
    user: userReducer,
  },
})

// 从 store 本身推断类型，无需手动维护
export type RootState = ReturnType<typeof store.getState>
export type AppDispatch = typeof store.dispatch
```

### 3. 提供 Store

```tsx
// src/main.tsx
import { Provider } from "react-redux"
import { store } from "./store"

createRoot(document.getElementById("root")!).render(
  <Provider store={store}>
    <App />
  </Provider>
)
```

### 4. 在组件中使用

```tsx
// ✗ 不要直接用 useSelector/useDispatch，类型不安全
import { useSelector, useDispatch } from "react-redux"

// ✓ 封装类型化的 Hook（一次定义，到处复用）
// src/store/hooks.ts
import { useDispatch, useSelector } from "react-redux"
import type { RootState, AppDispatch } from "./index"

export const useAppDispatch = () => useDispatch<AppDispatch>()
export const useAppSelector = <T>(selector: (state: RootState) => T) =>
  useSelector(selector)
```

```tsx
// src/components/Counter.tsx
import { useAppDispatch, useAppSelector } from "@/store/hooks"
import {
  increment, decrement, incrementByAmount,
  setStep, reset, selectCount, selectStep,
} from "@/store/counterSlice"

export function Counter() {
  const dispatch = useAppDispatch()
  const count = useAppSelector(selectCount)
  const step  = useAppSelector(selectStep)

  return (
    <div>
      <p>计数：{count}，步长：{step}</p>
      <button onClick={() => dispatch(increment())}>+</button>
      <button onClick={() => dispatch(decrement())}>-</button>
      <button onClick={() => dispatch(incrementByAmount(10))}>+10</button>
      <button onClick={() => dispatch(setStep(5))}>步长设为 5</button>
      <button onClick={() => dispatch(reset())}>重置</button>
    </div>
  )
}
```

---

## Immer 与不可变更新

RTK 内置了 **Immer**，在 reducer 中可以直接"修改" state，Immer 会在背后生成新对象：

```typescript
reducers: {
  // ✓ RTK 写法：直接修改（Immer 处理不可变性）
  addTodo(state, action: PayloadAction<string>) {
    state.items.push({ id: Date.now(), text: action.payload, done: false })
  },
  removeTodo(state, action: PayloadAction<number>) {
    state.items = state.items.filter(item => item.id !== action.payload)
  },
  toggleTodo(state, action: PayloadAction<number>) {
    const todo = state.items.find(item => item.id === action.payload)
    if (todo) todo.done = !todo.done
  },

  // 也可以返回全新对象（两种方式不能混用）
  clearAll() {
    return { items: [], filter: "all" }
  },
}
```

---

## 异步操作：createAsyncThunk

`createAsyncThunk` 处理异步逻辑，自动生成 `pending / fulfilled / rejected` 三个 action：

```typescript
// src/store/userSlice.ts
import { createSlice, createAsyncThunk, PayloadAction } from "@reduxjs/toolkit"

interface User {
  id: number
  name: string
  email: string
}

interface UserState {
  list: User[]
  current: User | null
  status: "idle" | "loading" | "succeeded" | "failed"
  error: string | null
}

const initialState: UserState = {
  list: [],
  current: null,
  status: "idle",
  error: null,
}

// 定义异步 thunk
export const fetchUsers = createAsyncThunk(
  "users/fetchAll",              // action type 前缀
  async (_, { rejectWithValue }) => {
    try {
      const res = await fetch("/api/users")
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      return (await res.json()) as User[]
    } catch (err) {
      return rejectWithValue((err as Error).message)
    }
  }
)

export const fetchUserById = createAsyncThunk(
  "users/fetchById",
  async (userId: number, { rejectWithValue }) => {
    try {
      const res = await fetch(`/api/users/${userId}`)
      if (!res.ok) throw new Error("用户不存在")
      return (await res.json()) as User
    } catch (err) {
      return rejectWithValue((err as Error).message)
    }
  }
)

export const createUser = createAsyncThunk(
  "users/create",
  async (data: Omit<User, "id">, { rejectWithValue }) => {
    try {
      const res = await fetch("/api/users", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(data),
      })
      return (await res.json()) as User
    } catch (err) {
      return rejectWithValue((err as Error).message)
    }
  }
)

const userSlice = createSlice({
  name: "users",
  initialState,
  reducers: {
    clearCurrent(state) {
      state.current = null
    },
  },
  // 处理异步 action
  extraReducers(builder) {
    builder
      // fetchUsers
      .addCase(fetchUsers.pending, (state) => {
        state.status = "loading"
        state.error = null
      })
      .addCase(fetchUsers.fulfilled, (state, action) => {
        state.status = "succeeded"
        state.list = action.payload
      })
      .addCase(fetchUsers.rejected, (state, action) => {
        state.status = "failed"
        state.error = action.payload as string
      })

      // fetchUserById
      .addCase(fetchUserById.fulfilled, (state, action) => {
        state.current = action.payload
      })

      // createUser
      .addCase(createUser.fulfilled, (state, action) => {
        state.list.push(action.payload)
      })

      // 用 addMatcher 处理多个 action 的公共逻辑
      .addMatcher(
        (action) => action.type.endsWith("/pending"),
        (state) => { state.status = "loading" }
      )
  },
})

export const { clearCurrent } = userSlice.actions

// Selectors
export const selectAllUsers    = (state: RootState) => state.users.list
export const selectCurrentUser = (state: RootState) => state.users.current
export const selectUserStatus  = (state: RootState) => state.users.status
export const selectUserError   = (state: RootState) => state.users.error

export default userSlice.reducer
```

### 在组件中使用异步 thunk

```tsx
import { useEffect } from "react"
import { useAppDispatch, useAppSelector } from "@/store/hooks"
import { fetchUsers, selectAllUsers, selectUserStatus, selectUserError } from "@/store/userSlice"

export function UserList() {
  const dispatch = useAppDispatch()
  const users  = useAppSelector(selectAllUsers)
  const status = useAppSelector(selectUserStatus)
  const error  = useAppSelector(selectUserError)

  useEffect(() => {
    // 避免重复请求
    if (status === "idle") {
      dispatch(fetchUsers())
    }
  }, [status, dispatch])

  // 也可以在事件处理中 await dispatch
  async function handleRefresh() {
    const result = await dispatch(fetchUsers())
    if (fetchUsers.fulfilled.match(result)) {
      console.log("刷新成功", result.payload)
    }
  }

  if (status === "loading") return <p>加载中...</p>
  if (status === "failed")  return <p>错误：{error}</p>

  return (
    <ul>
      {users.map(u => <li key={u.id}>{u.name}</li>)}
    </ul>
  )
}
```

---

## RTK Query（数据请求）

RTK Query 是 RTK 内置的数据请求和缓存方案，类似 React Query，但深度集成 Redux。

### 创建 API Service

```typescript
// src/store/api.ts
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react"

export interface Post {
  id: number
  title: string
  body: string
  userId: number
}

export const postsApi = createApi({
  reducerPath: "postsApi",           // store 中的 key
  baseQuery: fetchBaseQuery({
    baseUrl: "/api",
    prepareHeaders(headers, { getState }) {
      // 自动附加 token
      const token = (getState() as RootState).auth.token
      if (token) headers.set("Authorization", `Bearer ${token}`)
      return headers
    },
  }),
  tagTypes: ["Post", "User"],        // 缓存标签，用于自动失效
  endpoints(builder) {
    return {
      // 查询（GET）
      getPosts: builder.query<Post[], void>({
        query: () => "/posts",
        providesTags: ["Post"],      // 提供 Post 标签
      }),

      getPostById: builder.query<Post, number>({
        query: (id) => `/posts/${id}`,
        providesTags: (result, error, id) => [{ type: "Post", id }],
      }),

      getPostsByUser: builder.query<Post[], number>({
        query: (userId) => `/posts?userId=${userId}`,
        providesTags: (result) =>
          result
            ? [
                ...result.map(({ id }) => ({ type: "Post" as const, id })),
                { type: "Post", id: "LIST" },
              ]
            : [{ type: "Post", id: "LIST" }],
      }),

      // 变更（POST / PUT / PATCH / DELETE）
      createPost: builder.mutation<Post, Omit<Post, "id">>({
        query: (body) => ({ url: "/posts", method: "POST", body }),
        invalidatesTags: ["Post"],   // 成功后使 Post 缓存失效，自动重新请求
      }),

      updatePost: builder.mutation<Post, Partial<Post> & Pick<Post, "id">>({
        query: ({ id, ...body }) => ({
          url: `/posts/${id}`,
          method: "PATCH",
          body,
        }),
        invalidatesTags: (result, error, { id }) => [{ type: "Post", id }],
      }),

      deletePost: builder.mutation<void, number>({
        query: (id) => ({ url: `/posts/${id}`, method: "DELETE" }),
        invalidatesTags: (result, error, id) => [{ type: "Post", id }],
      }),
    }
  },
})

// 导出自动生成的 hooks
export const {
  useGetPostsQuery,
  useGetPostByIdQuery,
  useGetPostsByUserQuery,
  useCreatePostMutation,
  useUpdatePostMutation,
  useDeletePostMutation,
} = postsApi
```

### 注册到 Store

```typescript
// src/store/index.ts
import { configureStore } from "@reduxjs/toolkit"
import { postsApi } from "./api"

export const store = configureStore({
  reducer: {
    [postsApi.reducerPath]: postsApi.reducer,
    // ...其他 reducer
  },
  // 添加中间件，启用缓存、失效、轮询等功能
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(postsApi.middleware),
})
```

### 使用查询 Hook

```tsx
import {
  useGetPostsQuery,
  useGetPostByIdQuery,
  useCreatePostMutation,
  useDeletePostMutation,
} from "@/store/api"

// 查询列表
function PostList() {
  const {
    data: posts,
    isLoading,
    isFetching,  // 缓存存在但正在后台刷新
    isError,
    error,
    refetch,     // 手动重新请求
  } = useGetPostsQuery()

  if (isLoading) return <p>首次加载...</p>
  if (isError)   return <p>加载失败：{JSON.stringify(error)}</p>

  return (
    <div>
      {isFetching && <span>刷新中...</span>}
      <button onClick={refetch}>手动刷新</button>
      <ul>
        {posts?.map(p => <li key={p.id}>{p.title}</li>)}
      </ul>
    </div>
  )
}

// 条件查询（skip 控制是否执行）
function UserPosts({ userId }: { userId: number | null }) {
  const { data } = useGetPostsByUserQuery(userId!, {
    skip: userId === null,          // userId 为 null 时跳过请求
    pollingInterval: 30_000,        // 每 30 秒轮询
    refetchOnMountOrArgChange: true, // 参数变化时重新请求
    refetchOnFocus: true,           // 窗口获焦时重新请求
  })
  return <ul>{data?.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

### 使用变更 Hook

```tsx
function NewPostForm() {
  const [createPost, { isLoading, isError, isSuccess }] = useCreatePostMutation()

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    const form = e.currentTarget
    const data = new FormData(form)

    try {
      await createPost({
        title: data.get("title") as string,
        body: data.get("body") as string,
        userId: 1,
      }).unwrap()  // unwrap 会在失败时抛出异常
      form.reset()
    } catch (err) {
      console.error("创建失败", err)
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="title" placeholder="标题" required />
      <textarea name="body" placeholder="内容" />
      <button type="submit" disabled={isLoading}>
        {isLoading ? "发布中..." : "发布"}
      </button>
      {isSuccess && <p>发布成功！</p>}
      {isError && <p>发布失败，请重试</p>}
    </form>
  )
}

// 删除示例（乐观更新）
function PostItem({ post }: { post: Post }) {
  const [deletePost, { isLoading }] = useDeletePostMutation()

  return (
    <li>
      {post.title}
      <button
        onClick={() => deletePost(post.id)}
        disabled={isLoading}
      >
        {isLoading ? "删除中..." : "删除"}
      </button>
    </li>
  )
}
```

### 乐观更新

```typescript
updatePost: builder.mutation<Post, Partial<Post> & Pick<Post, "id">>({
  query: ({ id, ...body }) => ({ url: `/posts/${id}`, method: "PATCH", body }),

  async onQueryStarted({ id, ...patch }, { dispatch, queryFulfilled }) {
    // 1. 立即更新缓存（乐观更新）
    const patchResult = dispatch(
      postsApi.util.updateQueryData("getPostById", id, (draft) => {
        Object.assign(draft, patch)
      })
    )
    try {
      await queryFulfilled   // 等待请求完成
    } catch {
      patchResult.undo()     // 请求失败则回滚
    }
  },
}),
```

---

## createSelector（Memoized Selectors）

使用 `createSelector` 创建有记忆功能的选择器，避免重复计算：

```typescript
import { createSelector } from "@reduxjs/toolkit"

// 基础 selector
const selectTodos    = (state: RootState) => state.todos.items
const selectFilter   = (state: RootState) => state.todos.filter
const selectSearchQ  = (state: RootState) => state.todos.searchQuery

// 组合 selector（只有依赖变化时才重新计算）
export const selectFilteredTodos = createSelector(
  [selectTodos, selectFilter, selectSearchQ],
  (todos, filter, query) => {
    const filtered = todos.filter(todo => {
      if (filter === "active") return !todo.done
      if (filter === "completed") return todo.done
      return true
    })
    if (!query) return filtered
    return filtered.filter(t =>
      t.text.toLowerCase().includes(query.toLowerCase())
    )
  }
)

export const selectTodoStats = createSelector(
  [selectTodos],
  (todos) => ({
    total:     todos.length,
    completed: todos.filter(t => t.done).length,
    active:    todos.filter(t => !t.done).length,
  })
)
```

---

## 中间件

### 内置中间件

`configureStore` 默认包含 `redux-thunk`（异步支持）和开发模式下的 `serializability-check`、`immutability-check`。

### 自定义中间件

```typescript
import { Middleware } from "@reduxjs/toolkit"

// 日志中间件
const loggerMiddleware: Middleware = (store) => (next) => (action) => {
  if (process.env.NODE_ENV !== "production") {
    console.group(String((action as any).type))
    console.log("Before:", store.getState())
    console.log("Action:", action)
    const result = next(action)
    console.log("After:", store.getState())
    console.groupEnd()
    return result
  }
  return next(action)
}

// 错误上报中间件
const errorReportMiddleware: Middleware = (store) => (next) => (action) => {
  try {
    return next(action)
  } catch (err) {
    reportError(err, { action, state: store.getState() })
    throw err
  }
}

export const store = configureStore({
  reducer: rootReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware()
      .concat(loggerMiddleware)
      .concat(errorReportMiddleware),
})
```

---

## 代码组织：Feature 目录结构

推荐按功能（Feature）组织，而不是按类型（reducers/actions/selectors 分目录）：

```
src/
├── store/
│   ├── index.ts           # configureStore + 类型导出
│   └── hooks.ts           # useAppDispatch / useAppSelector
├── features/
│   ├── counter/
│   │   ├── counterSlice.ts
│   │   └── Counter.tsx
│   ├── users/
│   │   ├── usersSlice.ts
│   │   ├── usersApi.ts    # RTK Query
│   │   ├── UserList.tsx
│   │   └── UserDetail.tsx
│   └── todos/
│       ├── todosSlice.ts
│       ├── TodoList.tsx
│       └── TodoItem.tsx
└── app/
    └── App.tsx
```

---

## 调试工具

安装 Redux DevTools 浏览器扩展后，`configureStore` 自动接入，无需额外配置。

可以查看：

- 每一个 action 的 type 和 payload
- action 触发前后的 state diff
- 时间旅行调试（撤销/重放 action）

---

## Redux vs 其他方案选型

| 场景 | 推荐方案 |
|------|---------|
| 服务端数据请求与缓存 | RTK Query / React Query（比 createAsyncThunk 更省心） |
| 跨组件共享的 UI 状态 | Redux Toolkit |
| 组件本地状态 | `useState` / `useReducer` |
| 简单全局状态 | React Context + `useReducer` |
| 高频更新的状态 | Zustand（比 Redux 更轻量） |

> 原则：**能用 `useState` 解决的就不要上 Redux**。Redux 适合真正需要跨多个组件、多个路由共享并且有复杂更新逻辑的状态。
