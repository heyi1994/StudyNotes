# Next.js

Next.js 是基于 React 的全栈框架，由 Vercel 维护。它在 React 基础上提供了文件系统路由、服务端渲染、静态生成、API 路由、图片优化等开箱即用的能力。当前主流版本为 **App Router**（Next.js 13+），本文以此为准。

---

## 安装

```shell
npx create-next-app@latest my-app
cd my-app
npm run dev
```

创建时的选项推荐：

```
✔ TypeScript?              Yes
✔ ESLint?                  Yes
✔ Tailwind CSS?            Yes（按需）
✔ src/ directory?          Yes（推荐，结构更清晰）
✔ App Router?              Yes（新项目必选）
✔ import alias (@/*)?      Yes
```

---

## 项目结构

```
my-app/
├── src/
│   ├── app/                  # App Router 根目录
│   │   ├── layout.tsx        # 根布局（必须有）
│   │   ├── page.tsx          # 首页 /
│   │   ├── globals.css
│   │   ├── about/
│   │   │   └── page.tsx      # /about
│   │   ├── blog/
│   │   │   ├── page.tsx      # /blog
│   │   │   └── [slug]/
│   │   │       └── page.tsx  # /blog/:slug（动态路由）
│   │   └── api/
│   │       └── users/
│   │           └── route.ts  # API 路由 /api/users
│   ├── components/           # 可复用组件
│   ├── lib/                  # 工具函数、数据库等
│   └── types/                # 类型定义
├── public/                   # 静态资源
├── next.config.ts
└── tsconfig.json
```

---

## 文件系统路由

App Router 中，`app/` 目录下的文件夹名即为路由路径，每个路由段由以下特殊文件组成：

| 文件名 | 作用 |
|--------|------|
| `page.tsx` | 路由页面，公开可访问 |
| `layout.tsx` | 布局，包裹子路由，不随导航重新渲染 |
| `loading.tsx` | 加载 UI（自动包裹 Suspense） |
| `error.tsx` | 错误边界（必须是客户端组件） |
| `not-found.tsx` | 404 页面 |
| `template.tsx` | 类似 layout，但每次导航都重新挂载 |
| `route.ts` | API 端点，不能与 page.tsx 共存 |

### 动态路由

```
app/
├── blog/[slug]/page.tsx        # /blog/hello-world
├── shop/[...slug]/page.tsx     # /shop/a/b/c（捕获所有段）
├── shop/[[...slug]]/page.tsx   # /shop 和 /shop/a/b（可选捕获）
└── [lang]/[category]/page.tsx  # /en/tech
```

```tsx
// app/blog/[slug]/page.tsx
interface Props {
  params: Promise<{ slug: string }>
  searchParams: Promise<{ page?: string }>
}

export default async function BlogPost({ params, searchParams }: Props) {
  const { slug } = await params
  const { page = "1" } = await searchParams

  const post = await getPost(slug)

  return <article>{post.content}</article>
}
```

### 路由组（Route Groups）

用 `(folder)` 包裹文件夹，不影响 URL，用于组织代码或共享布局：

```
app/
├── (marketing)/
│   ├── layout.tsx        # marketing 专用布局
│   ├── page.tsx          # /
│   └── about/page.tsx    # /about
├── (dashboard)/
│   ├── layout.tsx        # dashboard 专用布局
│   └── dashboard/page.tsx # /dashboard
```

### 平行路由（Parallel Routes）

用 `@folder` 在同一布局中同时渲染多个页面：

```
app/
├── layout.tsx
├── @sidebar/
│   └── page.tsx
└── @content/
    └── page.tsx
```

```tsx
// app/layout.tsx
export default function Layout({
  children,
  sidebar,
  content,
}: {
  children: React.ReactNode
  sidebar: React.ReactNode
  content: React.ReactNode
}) {
  return (
    <div>
      <aside>{sidebar}</aside>
      <main>{content}</main>
    </div>
  )
}
```

### 拦截路由（Intercepting Routes）

在当前布局内打开其他路由（如在 Feed 页面弹出照片 Modal）：

```
app/
├── feed/page.tsx
├── photo/[id]/page.tsx          # 直接访问 /photo/1
└── feed/(..)photo/[id]/page.tsx # 从 feed 拦截 /photo/1，以 Modal 展示
```

---

## 服务端组件 vs 客户端组件

这是 App Router 最核心的概念。

| | 服务端组件（默认） | 客户端组件 |
|---|---|---|
| 标记方式 | 无需标记，默认即是 | 文件顶部加 `"use client"` |
| 运行环境 | 仅服务端 | 服务端（首次 SSR）+ 客户端（交互） |
| 可用特性 | 直接访问数据库、文件系统、密钥 | useState、useEffect、事件处理、浏览器 API |
| 不可用 | useState、useEffect、事件处理器、浏览器 API | 直接访问数据库、服务端密钥 |
| 包体积 | 不增加客户端 JS | 增加客户端 JS |

```tsx
// 服务端组件（默认）— 可直接 await，无需 useEffect
// app/users/page.tsx
async function UsersPage() {
  const users = await db.query("SELECT * FROM users")  // 直接访问数据库
  return (
    <ul>
      {users.map(u => <li key={u.id}>{u.name}</li>)}
    </ul>
  )
}
```

```tsx
// 客户端组件
"use client"

import { useState } from "react"

export function Counter() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```

### 组合模式

服务端组件可以将客户端组件作为 children 传入，实现"服务端壳 + 客户端岛"：

```tsx
// app/page.tsx（服务端组件）
import { ClientSearch } from "@/components/ClientSearch"

async function Page() {
  const initialData = await fetchData()  // 服务端获取数据

  return (
    <div>
      <h1>产品列表</h1>
      {/* 将服务端数据作为 props 传给客户端组件 */}
      <ClientSearch initialData={initialData} />
    </div>
  )
}
```

```tsx
// components/ClientSearch.tsx
"use client"

export function ClientSearch({ initialData }) {
  const [query, setQuery] = useState("")
  // 可以使用 hooks 和事件处理
}
```

---

## 数据获取

### 服务端组件直接获取

```tsx
// app/posts/page.tsx
async function PostsPage() {
  // 直接 await，无需 useEffect
  const posts = await fetch("https://api.example.com/posts", {
    next: { revalidate: 3600 },  // 每小时重新验证（ISR）
  }).then(r => r.json())

  return <PostList posts={posts} />
}
```

### fetch 缓存控制

```tsx
// 默认缓存（静态）
const data = await fetch(url)

// 不缓存（每次请求都重新获取，相当于 SSR）
const data = await fetch(url, { cache: "no-store" })

// 基于时间的重新验证（ISR）
const data = await fetch(url, { next: { revalidate: 60 } })  // 60 秒

// 基于标签的重新验证
const data = await fetch(url, { next: { tags: ["posts"] } })

// 在 Server Action 或 Route Handler 中手动触发
import { revalidateTag, revalidatePath } from "next/cache"
revalidateTag("posts")         // 使所有带 posts 标签的缓存失效
revalidatePath("/blog")        // 使 /blog 路径的缓存失效
```

### 并行数据获取

```tsx
async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params

  // 并行获取，而非串行
  const [user, posts, comments] = await Promise.all([
    fetchUser(id),
    fetchPosts(id),
    fetchComments(id),
  ])

  return <UserProfile user={user} posts={posts} comments={comments} />
}
```

### 使用 ORM 直接查询

```tsx
// lib/db.ts（使用 Prisma 示例）
import { PrismaClient } from "@prisma/client"
const prisma = new PrismaClient()

// app/users/page.tsx
import prisma from "@/lib/db"

async function UsersPage() {
  const users = await prisma.user.findMany({
    where: { active: true },
    orderBy: { createdAt: "desc" },
  })
  return <UserList users={users} />
}
```

---

## 布局与模板

### 根布局（必须）

```tsx
// app/layout.tsx
import type { Metadata } from "next"
import { Inter } from "next/font/google"
import "./globals.css"

const inter = Inter({ subsets: ["latin"] })

export const metadata: Metadata = {
  title: { template: "%s | My App", default: "My App" },
  description: "My Next.js Application",
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="zh">
      <body className={inter.className}>{children}</body>
    </html>
  )
}
```

### 嵌套布局

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <div className="dashboard">
      <DashboardNav />
      <main>{children}</main>
    </div>
  )
}
```

---

## 导航

### Link 组件

```tsx
import Link from "next/link"

// 基本用法
<Link href="/about">关于</Link>

// 动态路由
<Link href={`/blog/${post.slug}`}>{post.title}</Link>

// 对象形式
<Link href={{ pathname: "/shop", query: { category: "shoes" } }}>
  鞋类
</Link>

// 预加载控制（默认开启）
<Link href="/heavy-page" prefetch={false}>慢页面</Link>

// 替换历史记录
<Link href="/new" replace>替换</Link>
```

### 编程式导航

```tsx
"use client"
import { useRouter } from "next/navigation"

function LoginButton() {
  const router = useRouter()

  async function handleLogin() {
    await login()
    router.push("/dashboard")
    router.replace("/home")    // 不留历史记录
    router.back()              // 返回上一页
    router.refresh()           // 刷新当前路由（重新从服务端获取数据）
    router.prefetch("/shop")   // 手动预加载
  }
}
```

### usePathname / useSearchParams

```tsx
"use client"
import { usePathname, useSearchParams } from "next/navigation"

function NavItem({ href, label }: { href: string; label: string }) {
  const pathname = usePathname()
  const isActive = pathname === href || pathname.startsWith(href + "/")

  return (
    <Link href={href} className={isActive ? "active" : ""}>
      {label}
    </Link>
  )
}
```

---

## Server Actions

Server Actions 是在服务端执行的函数，可以直接在组件中调用，自动处理表单提交和数据变更。

```tsx
// app/actions.ts
"use server"

import { revalidatePath } from "next/cache"
import { redirect } from "next/navigation"

export async function createPost(formData: FormData) {
  const title = formData.get("title") as string
  const content = formData.get("content") as string

  // 表单验证
  if (!title || title.length < 3) {
    return { error: "标题至少 3 个字符" }
  }

  // 直接操作数据库
  await db.post.create({ data: { title, content } })

  // 刷新缓存并跳转
  revalidatePath("/blog")
  redirect("/blog")
}
```

### 在表单中使用

```tsx
// app/blog/new/page.tsx
import { createPost } from "@/app/actions"

export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="标题" required />
      <textarea name="content" placeholder="内容" />
      <button type="submit">发布</button>
    </form>
  )
}
```

### 与 useActionState 配合

```tsx
"use client"
import { useActionState } from "react"
import { createPost } from "@/app/actions"

export function PostForm() {
  const [state, formAction, isPending] = useActionState(createPost, null)

  return (
    <form action={formAction}>
      <input name="title" />
      <textarea name="content" />
      {state?.error && <p className="error">{state.error}</p>}
      <button type="submit" disabled={isPending}>
        {isPending ? "发布中..." : "发布"}
      </button>
    </form>
  )
}
```

### 在事件处理中调用

```tsx
"use client"
import { deletePost } from "@/app/actions"

export function DeleteButton({ id }: { id: number }) {
  return (
    <button
      onClick={async () => {
        await deletePost(id)
      }}
    >
      删除
    </button>
  )
}
```

---

## Metadata（SEO）

### 静态 Metadata

```tsx
// app/about/page.tsx
import type { Metadata } from "next"

export const metadata: Metadata = {
  title: "关于我们",
  description: "了解更多关于我们的信息",
  keywords: ["Next.js", "React", "Web"],
  authors: [{ name: "Alice" }],
  openGraph: {
    title: "关于我们",
    description: "了解更多关于我们的信息",
    images: [{ url: "/og-about.png", width: 1200, height: 630 }],
  },
  twitter: {
    card: "summary_large_image",
    title: "关于我们",
  },
}
```

### 动态 Metadata

```tsx
// app/blog/[slug]/page.tsx
import type { Metadata } from "next"

interface Props {
  params: Promise<{ slug: string }>
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params
  const post = await getPost(slug)

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      images: [{ url: post.coverImage }],
    },
  }
}
```

---

## 静态生成（SSG）

### generateStaticParams

预先生成动态路由的所有静态页面：

```tsx
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await getAllPosts()
  return posts.map(post => ({ slug: post.slug }))
}

// 生成 /blog/hello-world, /blog/second-post 等静态页面
export default async function BlogPost({ params }: Props) {
  const { slug } = await params
  const post = await getPost(slug)
  return <article>{post.content}</article>
}
```

### 动态 vs 静态渲染

```tsx
// 强制静态渲染
export const dynamic = "force-static"

// 强制动态渲染（每次请求都重新渲染）
export const dynamic = "force-dynamic"

// 重新验证间隔（秒）
export const revalidate = 3600

// 运行时
export const runtime = "edge"   // 或 "nodejs"（默认）
```

---

## API 路由（Route Handlers）

```tsx
// app/api/users/route.ts
import { NextRequest, NextResponse } from "next/server"

// GET /api/users
export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams
  const page = Number(searchParams.get("page") ?? "1")
  const limit = Number(searchParams.get("limit") ?? "10")

  const users = await db.user.findMany({
    skip: (page - 1) * limit,
    take: limit,
  })

  return NextResponse.json({ data: users, page, limit })
}

// POST /api/users
export async function POST(request: NextRequest) {
  const body = await request.json()

  if (!body.name || !body.email) {
    return NextResponse.json(
      { error: "name 和 email 是必填项" },
      { status: 400 }
    )
  }

  const user = await db.user.create({ data: body })
  return NextResponse.json(user, { status: 201 })
}
```

### 动态路由 API

```tsx
// app/api/users/[id]/route.ts
interface Context {
  params: Promise<{ id: string }>
}

export async function GET(request: NextRequest, { params }: Context) {
  const { id } = await params
  const user = await db.user.findUnique({ where: { id: Number(id) } })

  if (!user) {
    return NextResponse.json({ error: "用户不存在" }, { status: 404 })
  }

  return NextResponse.json(user)
}

export async function PATCH(request: NextRequest, { params }: Context) {
  const { id } = await params
  const body = await request.json()

  const user = await db.user.update({
    where: { id: Number(id) },
    data: body,
  })

  return NextResponse.json(user)
}

export async function DELETE(request: NextRequest, { params }: Context) {
  const { id } = await params
  await db.user.delete({ where: { id: Number(id) } })
  return new NextResponse(null, { status: 204 })
}
```

### 设置响应头 / Cookie

```tsx
export async function GET(request: NextRequest) {
  const response = NextResponse.json({ ok: true })

  response.headers.set("Cache-Control", "no-store")
  response.cookies.set("session", "abc123", {
    httpOnly: true,
    secure: true,
    maxAge: 60 * 60 * 24 * 7,  // 7 天
    sameSite: "lax",
  })

  return response
}
```

---

## 中间件

中间件在请求到达路由前执行，可用于鉴权、重定向、A/B 测试等：

```typescript
// middleware.ts（项目根目录）
import { NextResponse } from "next/server"
import type { NextRequest } from "next/server"

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl

  // 读取 Cookie
  const token = request.cookies.get("token")?.value

  // 保护 /dashboard 路由
  if (pathname.startsWith("/dashboard") && !token) {
    const loginUrl = new URL("/login", request.url)
    loginUrl.searchParams.set("from", pathname)
    return NextResponse.redirect(loginUrl)
  }

  // 国际化：根据 Accept-Language 重定向
  if (pathname === "/") {
    const lang = request.headers.get("accept-language")?.split(",")[0] ?? "zh"
    return NextResponse.redirect(new URL(`/${lang}`, request.url))
  }

  // 添加请求头（服务端组件可读取）
  const requestHeaders = new Headers(request.headers)
  requestHeaders.set("x-pathname", pathname)

  return NextResponse.next({ request: { headers: requestHeaders } })
}

// 配置中间件匹配的路径
export const config = {
  matcher: [
    // 排除静态资源和 API 路由
    "/((?!_next/static|_next/image|favicon.ico|api/).*)",
  ],
}
```

---

## 图片优化

```tsx
import Image from "next/image"

// 本地图片（自动获取尺寸）
import heroImage from "@/public/hero.png"

<Image src={heroImage} alt="Hero" priority />

// 远程图片（必须指定 width/height）
<Image
  src="https://example.com/photo.jpg"
  alt="Photo"
  width={800}
  height={600}
  quality={85}           // 默认 75
  placeholder="blur"     // 模糊占位
  blurDataURL="data:..."
/>

// 响应式图片（填充父容器）
<div style={{ position: "relative", height: "400px" }}>
  <Image
    src="/banner.jpg"
    alt="Banner"
    fill
    sizes="(max-width: 768px) 100vw, 50vw"
    style={{ objectFit: "cover" }}
    priority  // LCP 图片加 priority，跳过懒加载
  />
</div>
```

在 `next.config.ts` 中配置允许的远程图片域名：

```typescript
// next.config.ts
const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "example.com",
        pathname: "/images/**",
      },
    ],
  },
}
```

---

## 字体优化

```tsx
// app/layout.tsx
import { Inter, Noto_Sans_SC } from "next/font/google"
import localFont from "next/font/local"

const inter = Inter({
  subsets: ["latin"],
  variable: "--font-inter",  // CSS 变量方式
  display: "swap",
})

const notoSansSC = Noto_Sans_SC({
  subsets: ["chinese-simplified"],
  weight: ["400", "700"],
})

// 本地字体
const myFont = localFont({
  src: [
    { path: "../fonts/MyFont-Regular.woff2", weight: "400" },
    { path: "../fonts/MyFont-Bold.woff2", weight: "700" },
  ],
  variable: "--font-my",
})

export default function RootLayout({ children }) {
  return (
    <html className={`${inter.variable} ${myFont.variable}`}>
      <body className={notoSansSC.className}>{children}</body>
    </html>
  )
}
```

---

## 环境变量

```bash
# .env.local（本地开发，不提交 git）
DATABASE_URL=postgresql://localhost/mydb
JWT_SECRET=my-secret-key

# 客户端可访问（必须以 NEXT_PUBLIC_ 开头）
NEXT_PUBLIC_API_URL=https://api.example.com
NEXT_PUBLIC_GA_ID=G-XXXXXXXX
```

```tsx
// 服务端（任何服务端代码中可用）
const dbUrl = process.env.DATABASE_URL

// 客户端（只有 NEXT_PUBLIC_ 前缀的变量可用）
const apiUrl = process.env.NEXT_PUBLIC_API_URL
```

---

## Loading UI 与 Suspense

```tsx
// app/dashboard/loading.tsx
// 路由切换时自动展示，数据加载完毕后替换
export default function DashboardLoading() {
  return (
    <div className="skeleton">
      <div className="skeleton-header" />
      <div className="skeleton-body" />
    </div>
  )
}
```

手动使用 Suspense 实现流式渲染：

```tsx
// app/dashboard/page.tsx
import { Suspense } from "react"

async function Dashboard() {
  return (
    <div>
      <h1>仪表盘</h1>
      {/* 统计数据快速加载 */}
      <Suspense fallback={<StatsSkeleton />}>
        <Stats />
      </Suspense>
      {/* 图表慢速加载，不阻塞上面的内容 */}
      <Suspense fallback={<ChartSkeleton />}>
        <RevenueChart />
      </Suspense>
      {/* 列表最慢 */}
      <Suspense fallback={<TableSkeleton />}>
        <RecentActivity />
      </Suspense>
    </div>
  )
}

async function Stats() {
  const stats = await fetchStats()  // 假设需要 200ms
  return <StatsCards stats={stats} />
}

async function RevenueChart() {
  const data = await fetchChartData()  // 假设需要 1000ms
  return <Chart data={data} />
}
```

---

## 错误处理

```tsx
// app/dashboard/error.tsx
"use client"  // 错误边界必须是客户端组件

import { useEffect } from "react"

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    console.error(error)
  }, [error])

  return (
    <div>
      <h2>出了点问题</h2>
      <p>{error.message}</p>
      <button onClick={reset}>重试</button>
    </div>
  )
}
```

```tsx
// app/not-found.tsx
import Link from "next/link"

export default function NotFound() {
  return (
    <div>
      <h2>404 - 页面不存在</h2>
      <Link href="/">返回首页</Link>
    </div>
  )
}
```

在服务端组件中主动触发 404：

```tsx
import { notFound } from "next/navigation"

async function PostPage({ params }: Props) {
  const { slug } = await params
  const post = await getPost(slug)

  if (!post) notFound()  // 渲染 not-found.tsx

  return <article>{post.content}</article>
}
```

---

## 鉴权方案

推荐使用 [Auth.js（NextAuth）](https://authjs.dev)：

```shell
npm install next-auth@beta
```

```typescript
// auth.ts
import NextAuth from "next-auth"
import GitHub from "next-auth/providers/github"
import Credentials from "next-auth/providers/credentials"

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    GitHub,
    Credentials({
      async authorize(credentials) {
        const user = await verifyPassword(
          credentials.email as string,
          credentials.password as string,
        )
        return user ?? null
      },
    }),
  ],
  callbacks: {
    async session({ session, token }) {
      session.user.id = token.sub!
      return session
    },
  },
})
```

```tsx
// app/api/auth/[...nextauth]/route.ts
import { handlers } from "@/auth"
export const { GET, POST } = handlers
```

```tsx
// 在服务端组件中获取 session
import { auth } from "@/auth"

async function ProfilePage() {
  const session = await auth()
  if (!session) redirect("/login")

  return <div>欢迎，{session.user.name}</div>
}
```

---

## next.config.ts 常用配置

```typescript
import type { NextConfig } from "next"

const nextConfig: NextConfig = {
  // 重定向
  async redirects() {
    return [
      {
        source: "/old-blog/:slug",
        destination: "/blog/:slug",
        permanent: true,   // 308（SEO 友好）
      },
    ]
  },

  // URL 重写（代理，URL 不变）
  async rewrites() {
    return [
      {
        source: "/api/v1/:path*",
        destination: "https://backend.example.com/:path*",
      },
    ]
  },

  // 自定义响应头
  async headers() {
    return [
      {
        source: "/(.*)",
        headers: [
          { key: "X-Frame-Options", value: "DENY" },
          { key: "X-Content-Type-Options", value: "nosniff" },
        ],
      },
    ]
  },

  // 允许的远程图片
  images: {
    remotePatterns: [{ protocol: "https", hostname: "**.example.com" }],
  },

  // 开启实验性特性
  experimental: {
    ppr: true,         // Partial Pre-Rendering
    serverActions: { bodySizeLimit: "2mb" },
  },
}

export default nextConfig
```

---

## 渲染策略总结

| 策略 | 触发条件 | 适用场景 |
|------|---------|---------|
| **静态渲染（SSG）** | 默认，构建时生成 | 博客、文档、营销页 |
| **动态渲染（SSR）** | 使用 `cookies()`、`headers()` 或 `dynamic = "force-dynamic"` | 个人化页面、实时数据 |
| **增量静态再生（ISR）** | `revalidate` 设置秒数 或 按需 `revalidatePath` | 需要定期更新的静态页 |
| **流式渲染** | 使用 `Suspense` 包裹异步组件 | 仪表盘、数据较慢的页面 |
| **客户端渲染** | `"use client"` + `useEffect` 获取数据 | 高度交互、用户特定的 UI |
