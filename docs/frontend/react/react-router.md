# React Router

React Router 是 React 生态中最流行的客户端路由库，支持声明式路由、嵌套路由、数据加载等特性。当前版本为 v7，与 v6 API 基本兼容。

## 安装

使用官方脚手架创建项目：

```shell
npx create-react-router@latest my-react-router-app
cd my-react-router-app
npm i
npm run dev
```

在已有 React 项目中安装：

```shell
npm install react-router
```

---

## 路由模式

React Router 提供三种路由模式：

| 模式       | API                   | 说明                                 |
| ---------- | --------------------- | ------------------------------------ |
| 浏览器路由 | `createBrowserRouter` | 基于 HTML5 History API，最常用       |
| Hash 路由  | `createHashRouter`    | URL 包含 `#`，适合静态文件服务       |
| 内存路由   | `createMemoryRouter`  | 不依赖浏览器，适合测试和非浏览器环境 |

---

## 基本配置

### 创建路由

推荐使用 `createBrowserRouter` + 对象配置方式：

```tsx
// src/router.tsx
import { createBrowserRouter } from "react-router";
import Home from "./pages/Home";
import About from "./pages/About";
import NotFound from "./pages/NotFound";

const router = createBrowserRouter([
  {
    path: "/",
    element: <Home />,
  },
  {
    path: "/about",
    element: <About />,
  },
  {
    path: "*",
    element: <NotFound />,
  },
]);

export default router;
```

### 挂载路由

```tsx
// src/main.tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { RouterProvider } from "react-router";
import router from "./router";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <RouterProvider router={router} />
  </StrictMode>,
);
```

---

## 路由导航

### Link 组件

`<Link>` 用于声明式导航，不会触发整页刷新：

```tsx
import { Link } from "react-router";

function Nav() {
  return (
    <nav>
      <Link to="/">首页</Link>
      <Link to="/about">关于</Link>
      <Link to="/users/42">用户详情</Link>
    </nav>
  );
}
```

### NavLink 组件

`<NavLink>` 是 `<Link>` 的增强版，会在链接激活时自动添加 `active` 类名，适合导航菜单：

```tsx
import { NavLink } from "react-router";

function Nav() {
  return (
    <nav>
      <NavLink to="/" className={({ isActive }) => (isActive ? "active" : "")}>
        首页
      </NavLink>
      <NavLink
        to="/about"
        style={({ isActive }) => ({ fontWeight: isActive ? "bold" : "normal" })}
      >
        关于
      </NavLink>
    </nav>
  );
}
```

### useNavigate 编程式导航

```tsx
import { useNavigate } from "react-router";

function LoginForm() {
  const navigate = useNavigate();

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    await login();
    navigate("/dashboard"); // 跳转到新页面
    navigate(-1); // 返回上一页
    navigate("/home", { replace: true }); // 替换当前历史记录
    navigate("/checkout", { state: { from: "cart" } }); // 携带状态
  }

  return <form onSubmit={handleSubmit}>...</form>;
}
```

### redirect 函数

在 loader/action 中使用 `redirect` 进行服务端风格的重定向：

```tsx
import { redirect } from "react-router";

export async function loader() {
  const user = await getUser();
  if (!user) {
    return redirect("/login");
  }
  return user;
}
```

---

## 动态路由参数

在路径中使用 `:paramName` 定义动态段：

```tsx
const router = createBrowserRouter([
  {
    path: "/users/:userId",
    element: <UserDetail />,
  },
  {
    path: "/posts/:postId/comments/:commentId",
    element: <Comment />,
  },
]);
```

### useParams 获取参数

```tsx
import { useParams } from "react-router";

function UserDetail() {
  const { userId } = useParams<{ userId: string }>();

  return <div>用户 ID：{userId}</div>;
}
```

---

## 嵌套路由

嵌套路由允许子路由在父路由的布局中渲染，通过 `<Outlet>` 指定子路由渲染位置。

```tsx
const router = createBrowserRouter([
  {
    path: "/dashboard",
    element: <DashboardLayout />, // 包含 <Outlet />
    children: [
      {
        index: true, // 默认子路由（访问 /dashboard 时渲染）
        element: <DashboardHome />,
      },
      {
        path: "profile", // 访问 /dashboard/profile
        element: <Profile />,
      },
      {
        path: "settings",
        element: <Settings />,
      },
    ],
  },
]);
```

父级布局组件中使用 `<Outlet>`：

```tsx
import { Outlet, NavLink } from "react-router";

function DashboardLayout() {
  return (
    <div className="dashboard">
      <aside>
        <NavLink to="/dashboard">概览</NavLink>
        <NavLink to="/dashboard/profile">个人资料</NavLink>
        <NavLink to="/dashboard/settings">设置</NavLink>
      </aside>
      <main>
        <Outlet /> {/* 子路由在这里渲染 */}
      </main>
    </div>
  );
}
```

### 布局路由（无 path）

不设置 `path` 的路由仅提供布局，不影响 URL：

```tsx
const router = createBrowserRouter([
  {
    element: <MainLayout />, // 无 path，仅作为布局容器
    children: [
      { path: "/", element: <Home /> },
      { path: "/about", element: <About /> },
    ],
  },
]);
```

### 向 Outlet 传递数据

```tsx
import { Outlet, useOutletContext } from "react-router";

// 父组件
function Parent() {
  const [count, setCount] = useState(0);
  return <Outlet context={{ count, setCount }} />;
}

// 子组件
function Child() {
  const { count, setCount } = useOutletContext<{
    count: number;
    setCount: (n: number) => void;
  }>();
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

---

## 索引路由

`index: true` 表示索引路由，当父路由路径完全匹配但没有子路径时渲染：

```tsx
{
  path: '/users',
  element: <UsersLayout />,
  children: [
    { index: true, element: <UserList /> },     // 访问 /users
    { path: ':id', element: <UserDetail /> },   // 访问 /users/42
  ],
}
```

---

## Search Params（查询参数）

```tsx
import { useSearchParams } from "react-router";

function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();

  const page = searchParams.get("page") ?? "1";
  const category = searchParams.get("category") ?? "all";

  function handlePageChange(newPage: number) {
    setSearchParams({ page: String(newPage), category });
  }

  return (
    <div>
      <p>
        第 {page} 页，分类：{category}
      </p>
      <button onClick={() => handlePageChange(Number(page) + 1)}>下一页</button>
    </div>
  );
}
```

---

## 获取当前路由信息

### useLocation

```tsx
import { useLocation } from "react-router";

function CurrentPath() {
  const location = useLocation();

  // location.pathname  → '/users/42'
  // location.search    → '?tab=profile'
  // location.hash      → '#section1'
  // location.state     → navigate 时传入的 state

  return <div>当前路径：{location.pathname}</div>;
}
```

### useMatch

检查当前路径是否匹配指定模式：

```tsx
import { useMatch } from "react-router";

function Nav() {
  const isHome = useMatch("/");
  const isUserPage = useMatch("/users/:id");

  return (
    <nav>
      <span className={isHome ? "active" : ""}>首页</span>
    </nav>
  );
}
```

---

## 数据加载（Loader）

`loader` 在路由渲染前预先获取数据，避免 useEffect 带来的加载闪烁问题。

```tsx
// router.tsx
import { createBrowserRouter } from "react-router";

const router = createBrowserRouter([
  {
    path: "/users/:userId",
    element: <UserDetail />,
    loader: async ({ params }) => {
      const user = await fetch(`/api/users/${params.userId}`).then((r) =>
        r.json(),
      );
      return user;
    },
  },
]);
```

在组件中使用 `useLoaderData` 获取数据：

```tsx
import { useLoaderData } from "react-router";

interface User {
  id: number;
  name: string;
  email: string;
}

function UserDetail() {
  const user = useLoaderData() as User;

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}
```

### 加载状态

使用 `useNavigation` 监听加载状态：

```tsx
import { useNavigation } from "react-router";

function Layout() {
  const navigation = useNavigation();
  const isLoading = navigation.state === "loading";

  return (
    <div>
      {isLoading && <div className="loading-bar" />}
      <Outlet />
    </div>
  );
}
```

---

## 数据提交（Action）

`action` 处理表单提交和数据变更，对应 `<Form>` 或 `useFetcher`。

```tsx
import { Form, useActionData, redirect } from "react-router";

// 定义 action
async function createUserAction({ request }: { request: Request }) {
  const formData = await request.formData();
  const name = formData.get("name") as string;
  const email = formData.get("email") as string;

  const errors: Record<string, string> = {};
  if (!name) errors.name = "姓名不能为空";
  if (!email) errors.email = "邮箱不能为空";
  if (Object.keys(errors).length > 0) return errors;

  await createUser({ name, email });
  return redirect("/users");
}

// 路由配置
const router = createBrowserRouter([
  {
    path: "/users/new",
    element: <NewUser />,
    action: createUserAction,
  },
]);

// 组件
function NewUser() {
  const errors = useActionData() as Record<string, string> | undefined;

  return (
    <Form method="post">
      <div>
        <input name="name" placeholder="姓名" />
        {errors?.name && <span className="error">{errors.name}</span>}
      </div>
      <div>
        <input name="email" placeholder="邮箱" />
        {errors?.email && <span className="error">{errors.email}</span>}
      </div>
      <button type="submit">创建</button>
    </Form>
  );
}
```

---

## 错误处理

使用 `errorElement` 捕获路由加载或渲染中的错误：

```tsx
import { useRouteError, isRouteErrorResponse } from "react-router";

function ErrorBoundary() {
  const error = useRouteError();

  if (isRouteErrorResponse(error)) {
    return (
      <div>
        <h1>
          {error.status} {error.statusText}
        </h1>
        <p>{error.data}</p>
      </div>
    );
  }

  return <div>发生了未知错误</div>;
}

const router = createBrowserRouter([
  {
    path: "/",
    element: <Root />,
    errorElement: <ErrorBoundary />, // 捕获此路由及其子路由的错误
    children: [
      {
        path: "users/:id",
        element: <UserDetail />,
        errorElement: <UserError />, // 局部错误边界
        loader: userLoader,
      },
    ],
  },
]);
```

在 loader 中抛出 HTTP 错误响应：

```tsx
import { data } from "react-router";

async function userLoader({ params }: { params: { id: string } }) {
  const user = await fetchUser(params.id);
  if (!user) {
    throw data("用户不存在", { status: 404 });
  }
  return user;
}
```

---

## 路由懒加载

使用 `lazy` 按需加载路由组件，减小初始包体积：

```tsx
const router = createBrowserRouter([
  {
    path: "/dashboard",
    lazy: async () => {
      const { default: Dashboard } = await import("./pages/Dashboard");
      return { Component: Dashboard };
    },
  },
  {
    path: "/settings",
    lazy: async () => {
      const mod = await import("./pages/Settings");
      return {
        Component: mod.default,
        loader: mod.loader,
        action: mod.action,
      };
    },
  },
]);
```

---

## 保护路由（鉴权）

通过在 loader 中检查登录状态来实现路由保护：

```tsx
// 通用鉴权 loader
async function requireAuth({ request }: { request: Request }) {
  const user = await getUser();
  if (!user) {
    const url = new URL(request.url);
    return redirect(`/login?from=${url.pathname}`);
  }
  return user;
}

const router = createBrowserRouter([
  {
    path: "/dashboard",
    loader: requireAuth,
    element: <Dashboard />,
  },
  {
    path: "/login",
    element: <Login />,
    action: loginAction,
  },
]);
```

登录后跳回原页面：

```tsx
function Login() {
  const [searchParams] = useSearchParams();
  const navigate = useNavigate();

  async function handleLogin() {
    await login();
    const from = searchParams.get("from") ?? "/dashboard";
    navigate(from, { replace: true });
  }
}
```

---

## useFetcher

`useFetcher` 允许在不导航的情况下调用 loader 或 action，适合局部数据更新、即时搜索等场景：

```tsx
import { useFetcher } from "react-router";

function LikeButton({ postId }: { postId: number }) {
  const fetcher = useFetcher();
  const isLiking = fetcher.state !== "idle";

  return (
    <fetcher.Form method="post" action={`/posts/${postId}/like`}>
      <button type="submit" disabled={isLiking}>
        {isLiking ? "处理中..." : "点赞"}
      </button>
    </fetcher.Form>
  );
}
```

即时搜索示例：

```tsx
function SearchBar() {
  const fetcher = useFetcher<{ results: string[] }>();

  return (
    <div>
      <fetcher.Form method="get" action="/search">
        <input
          name="q"
          onChange={(e) => {
            fetcher.submit(e.currentTarget.form);
          }}
        />
      </fetcher.Form>
      {fetcher.data?.results.map((item) => (
        <div key={item}>{item}</div>
      ))}
    </div>
  );
}
```

---

## 滚动恢复

React Router 默认在导航时滚动到页面顶部，使用 `<ScrollRestoration>` 恢复滚动位置：

```tsx
import { ScrollRestoration } from "react-router";

function Root() {
  return (
    <>
      <Nav />
      <Outlet />
      <ScrollRestoration />
    </>
  );
}
```

---

## 常用 Hooks 速查

| Hook                 | 说明                                        |
| -------------------- | ------------------------------------------- |
| `useNavigate()`      | 返回导航函数，用于编程式跳转                |
| `useParams()`        | 获取动态路由参数                            |
| `useSearchParams()`  | 读写 URL 查询参数                           |
| `useLocation()`      | 获取当前 location 对象                      |
| `useMatch(pattern)`  | 检查当前路径是否匹配                        |
| `useLoaderData()`    | 获取当前路由 loader 返回的数据              |
| `useActionData()`    | 获取最近一次 action 的返回值                |
| `useNavigation()`    | 获取导航状态（idle / loading / submitting） |
| `useRouteError()`    | 在 errorElement 中获取错误对象              |
| `useFetcher()`       | 在不导航的情况下加载/提交数据               |
| `useOutletContext()` | 获取父路由通过 Outlet 传递的上下文          |
| `useRevalidator()`   | 手动触发当前路由数据重新验证                |
