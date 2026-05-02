# shadcn/ui

shadcn/ui 不是传统的组件库——它不发布 npm 包，而是将组件源码直接复制到你的项目中。每个组件都是**你自己的代码**，可以随意修改，底层基于 **Radix UI**（无障碍原语）+ **Tailwind CSS**。

**核心理念：**

- 组件代码放在 `components/ui/` 目录，完全可控
- 基于 Radix UI 提供无障碍（a11y）支持
- 使用 Tailwind CSS + CSS 变量实现主题
- 按需添加，不安装用不到的组件

---

## 安装

### 前置要求

项目需要已配置 Tailwind CSS。以 Vite + React 为例：

```shell
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install tailwindcss @tailwindcss/vite
```

### 初始化 shadcn/ui

```shell
npx shadcn@latest init
```

交互式配置：

```
✔ Which style would you like to use? › Default
✔ Which color would you like to use as the base color? › Slate
✔ Would you like to use CSS variables for theming? › Yes
```

初始化后自动生成：

```
src/
├── components/
│   └── ui/            # 组件存放目录（后续 add 的组件放这里）
├── lib/
│   └── utils.ts       # cn() 工具函数
app/globals.css        # CSS 变量主题（或 src/index.css）
components.json        # shadcn 配置文件
```

### 添加组件

```shell
# 添加单个组件
npx shadcn@latest add button

# 添加多个组件
npx shadcn@latest add button input card dialog

# 添加所有组件
npx shadcn@latest add --all
```

---

## 主题系统

shadcn/ui 通过 CSS 变量实现主题，支持亮色/暗色模式切换：

```css
/* globals.css */
@import "tailwindcss";

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;
    --popover: 0 0% 100%;
    --popover-foreground: 222.2 84% 4.9%;
    --primary: 222.2 47.4% 11.2%;
    --primary-foreground: 210 40% 98%;
    --secondary: 210 40% 96%;
    --secondary-foreground: 222.2 47.4% 11.2%;
    --muted: 210 40% 96%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --accent: 210 40% 96%;
    --accent-foreground: 222.2 47.4% 11.2%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 210 40% 98%;
    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 222.2 84% 4.9%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --primary: 210 40% 98%;
    --primary-foreground: 222.2 47.4% 11.2%;
    /* ... 其他暗色变量 */
  }
}
```

### 切换暗色模式

```tsx
// 在 html 元素上切换 dark 类
document.documentElement.classList.toggle("dark")

// 搭配 next-themes
npm install next-themes

// providers.tsx
import { ThemeProvider } from "next-themes"

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <ThemeProvider attribute="class" defaultTheme="system" enableSystem>
      {children}
    </ThemeProvider>
  )
}

// 切换按钮
import { useTheme } from "next-themes"

function ThemeToggle() {
  const { theme, setTheme } = useTheme()
  return (
    <button onClick={() => setTheme(theme === "dark" ? "light" : "dark")}>
      切换主题
    </button>
  )
}
```

### 自定义主题色

直接修改 CSS 变量即可更换整套主题色，推荐使用 [shadcn/ui Themes](https://ui.shadcn.com/themes) 在线生成：

```css
:root {
  --primary: 262.1 83.3% 57.8%;        /* 紫色主色 */
  --primary-foreground: 210 40% 98%;
  --ring: 262.1 83.3% 57.8%;
  --radius: 0.75rem;                    /* 更大的圆角 */
}
```

---

## cn() 工具函数

shadcn 初始化时自动生成，用于合并 Tailwind 类名：

```typescript
// src/lib/utils.ts
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

---

## Button 按钮

```tsx
import { Button } from "@/components/ui/button"

// 变体
<Button>默认</Button>
<Button variant="destructive">危险操作</Button>
<Button variant="outline">边框</Button>
<Button variant="secondary">次要</Button>
<Button variant="ghost">幽灵</Button>
<Button variant="link">链接样式</Button>

// 尺寸
<Button size="sm">小</Button>
<Button size="default">默认</Button>
<Button size="lg">大</Button>
<Button size="icon"><SearchIcon /></Button>

// 加载状态
<Button disabled>
  <Loader2 className="mr-2 h-4 w-4 animate-spin" />
  加载中...
</Button>

// 与 Link 结合（asChild 模式）
import { Link } from "react-router"

<Button asChild>
  <Link to="/dashboard">进入控制台</Link>
</Button>
```

### 查看/修改按钮源码

```tsx
// components/ui/button.tsx（初始化后自动生成，可直接修改）
import { cva, type VariantProps } from "class-variance-authority"

const buttonVariants = cva(
  "inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground shadow hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground shadow-sm hover:bg-destructive/90",
        outline: "border border-input bg-background shadow-sm hover:bg-accent hover:text-accent-foreground",
        secondary: "bg-secondary text-secondary-foreground shadow-sm hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
      },
      size: {
        default: "h-9 px-4 py-2",
        sm: "h-8 rounded-md px-3 text-xs",
        lg: "h-10 rounded-md px-8",
        icon: "h-9 w-9",
      },
    },
    defaultVariants: { variant: "default", size: "default" },
  }
)

// 添加自定义变体
// 只需在 variants.variant 中添加即可：
// success: "bg-green-500 text-white hover:bg-green-600",
```

---

## Input 输入框

```tsx
import { Input } from "@/components/ui/input"
import { Label } from "@/components/ui/label"

// 基础
<Input placeholder="请输入邮箱" type="email" />

// 带 Label
<div className="grid w-full gap-1.5">
  <Label htmlFor="email">邮箱</Label>
  <Input id="email" type="email" placeholder="name@example.com" />
</div>

// 禁用
<Input disabled placeholder="禁用状态" />

// 带图标（Input 本身不含图标，需自己实现）
<div className="relative">
  <Search className="absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-muted-foreground" />
  <Input className="pl-9" placeholder="搜索..." />
</div>

// 带按钮
<div className="flex gap-2">
  <Input placeholder="输入邮箱订阅" />
  <Button type="submit">订阅</Button>
</div>
```

---

## Card 卡片

```tsx
import {
  Card,
  CardContent,
  CardDescription,
  CardFooter,
  CardHeader,
  CardTitle,
} from "@/components/ui/card"

<Card className="w-[380px]">
  <CardHeader>
    <CardTitle>账户设置</CardTitle>
    <CardDescription>修改你的账户信息和偏好设置</CardDescription>
  </CardHeader>
  <CardContent className="space-y-4">
    <div className="space-y-1.5">
      <Label htmlFor="name">用户名</Label>
      <Input id="name" defaultValue="张三" />
    </div>
    <div className="space-y-1.5">
      <Label htmlFor="email">邮箱</Label>
      <Input id="email" defaultValue="zhang@example.com" />
    </div>
  </CardContent>
  <CardFooter className="flex justify-between">
    <Button variant="outline">取消</Button>
    <Button>保存更改</Button>
  </CardFooter>
</Card>
```

---

## Dialog 对话框

```tsx
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
  DialogClose,
} from "@/components/ui/dialog"

<Dialog>
  <DialogTrigger asChild>
    <Button variant="outline">打开对话框</Button>
  </DialogTrigger>
  <DialogContent className="sm:max-w-[425px]">
    <DialogHeader>
      <DialogTitle>编辑个人资料</DialogTitle>
      <DialogDescription>修改后点击保存，更改将立即生效。</DialogDescription>
    </DialogHeader>
    <div className="grid gap-4 py-4">
      <div className="grid grid-cols-4 items-center gap-4">
        <Label htmlFor="name" className="text-right">姓名</Label>
        <Input id="name" defaultValue="张三" className="col-span-3" />
      </div>
    </div>
    <DialogFooter>
      <DialogClose asChild>
        <Button variant="outline">取消</Button>
      </DialogClose>
      <Button type="submit">保存</Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

### 受控 Dialog

```tsx
function ConfirmDialog({ onConfirm }: { onConfirm: () => void }) {
  const [open, setOpen] = useState(false)

  function handleConfirm() {
    onConfirm()
    setOpen(false)
  }

  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogTrigger asChild>
        <Button variant="destructive">删除账户</Button>
      </DialogTrigger>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>确认删除</DialogTitle>
          <DialogDescription>此操作不可撤销，账户数据将永久删除。</DialogDescription>
        </DialogHeader>
        <DialogFooter>
          <Button variant="outline" onClick={() => setOpen(false)}>取消</Button>
          <Button variant="destructive" onClick={handleConfirm}>确认删除</Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  )
}
```

---

## Sheet 侧边抽屉

```tsx
import {
  Sheet,
  SheetContent,
  SheetDescription,
  SheetHeader,
  SheetTitle,
  SheetTrigger,
  SheetFooter,
  SheetClose,
} from "@/components/ui/sheet"

// side 控制从哪侧滑出：top | right（默认）| bottom | left
<Sheet>
  <SheetTrigger asChild>
    <Button variant="outline">打开侧边栏</Button>
  </SheetTrigger>
  <SheetContent side="right">
    <SheetHeader>
      <SheetTitle>筛选</SheetTitle>
      <SheetDescription>选择筛选条件后点击应用</SheetDescription>
    </SheetHeader>
    <div className="py-6 space-y-4">
      {/* 筛选内容 */}
    </div>
    <SheetFooter>
      <SheetClose asChild>
        <Button className="w-full">应用筛选</Button>
      </SheetClose>
    </SheetFooter>
  </SheetContent>
</Sheet>
```

---

## DropdownMenu 下拉菜单

```tsx
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuLabel,
  DropdownMenuSeparator,
  DropdownMenuTrigger,
  DropdownMenuSub,
  DropdownMenuSubContent,
  DropdownMenuSubTrigger,
  DropdownMenuCheckboxItem,
  DropdownMenuRadioGroup,
  DropdownMenuRadioItem,
  DropdownMenuShortcut,
} from "@/components/ui/dropdown-menu"

<DropdownMenu>
  <DropdownMenuTrigger asChild>
    <Button variant="outline">
      我的账户 <ChevronDown className="ml-2 h-4 w-4" />
    </Button>
  </DropdownMenuTrigger>
  <DropdownMenuContent align="end" className="w-56">
    <DropdownMenuLabel>我的账户</DropdownMenuLabel>
    <DropdownMenuSeparator />

    <DropdownMenuItem onClick={() => router.push("/profile")}>
      <User className="mr-2 h-4 w-4" />
      个人资料
      <DropdownMenuShortcut>⇧⌘P</DropdownMenuShortcut>
    </DropdownMenuItem>

    <DropdownMenuItem onClick={() => router.push("/settings")}>
      <Settings className="mr-2 h-4 w-4" />
      设置
    </DropdownMenuItem>

    {/* 子菜单 */}
    <DropdownMenuSub>
      <DropdownMenuSubTrigger>
        <UserPlus className="mr-2 h-4 w-4" />
        邀请成员
      </DropdownMenuSubTrigger>
      <DropdownMenuSubContent>
        <DropdownMenuItem>通过邮件邀请</DropdownMenuItem>
        <DropdownMenuItem>复制邀请链接</DropdownMenuItem>
      </DropdownMenuSubContent>
    </DropdownMenuSub>

    <DropdownMenuSeparator />
    <DropdownMenuItem className="text-destructive" onClick={logout}>
      <LogOut className="mr-2 h-4 w-4" />
      退出登录
    </DropdownMenuItem>
  </DropdownMenuContent>
</DropdownMenu>
```

---

## Select 选择器

```tsx
import {
  Select,
  SelectContent,
  SelectGroup,
  SelectItem,
  SelectLabel,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select"

<Select onValueChange={(value) => console.log(value)}>
  <SelectTrigger className="w-[200px]">
    <SelectValue placeholder="选择框架" />
  </SelectTrigger>
  <SelectContent>
    <SelectGroup>
      <SelectLabel>前端框架</SelectLabel>
      <SelectItem value="react">React</SelectItem>
      <SelectItem value="vue">Vue</SelectItem>
      <SelectItem value="svelte">Svelte</SelectItem>
    </SelectGroup>
    <SelectGroup>
      <SelectLabel>后端框架</SelectLabel>
      <SelectItem value="nextjs">Next.js</SelectItem>
      <SelectItem value="remix">Remix</SelectItem>
    </SelectGroup>
  </SelectContent>
</Select>

// 受控
const [value, setValue] = useState("react")

<Select value={value} onValueChange={setValue}>
  ...
</Select>
```

---

## Tabs 标签页

```tsx
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs"

<Tabs defaultValue="overview" className="w-full">
  <TabsList className="grid w-full grid-cols-3">
    <TabsTrigger value="overview">概览</TabsTrigger>
    <TabsTrigger value="analytics">分析</TabsTrigger>
    <TabsTrigger value="settings">设置</TabsTrigger>
  </TabsList>

  <TabsContent value="overview">
    <Card>
      <CardHeader><CardTitle>概览</CardTitle></CardHeader>
      <CardContent>概览内容...</CardContent>
    </Card>
  </TabsContent>

  <TabsContent value="analytics">
    <Card>
      <CardHeader><CardTitle>分析</CardTitle></CardHeader>
      <CardContent>分析图表...</CardContent>
    </Card>
  </TabsContent>

  <TabsContent value="settings">
    <Card>
      <CardHeader><CardTitle>设置</CardTitle></CardHeader>
      <CardContent>设置选项...</CardContent>
    </Card>
  </TabsContent>
</Tabs>
```

---

## Form 表单（配合 react-hook-form）

shadcn 的 Form 组件深度封装了 `react-hook-form` + `zod`：

```shell
npm install react-hook-form zod @hookform/resolvers
npx shadcn@latest add form
```

```tsx
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"
import {
  Form,
  FormControl,
  FormDescription,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form"

// 1. 定义 Schema
const formSchema = z.object({
  username: z
    .string()
    .min(2, "用户名至少 2 个字符")
    .max(20, "用户名最多 20 个字符"),
  email: z.string().email("请输入有效邮箱"),
  age: z.coerce.number().int().min(1).max(120).optional(),
  role: z.enum(["admin", "user", "guest"], {
    required_error: "请选择角色",
  }),
  bio: z.string().max(200, "简介最多 200 字").optional(),
  notifications: z.boolean().default(false),
})

type FormValues = z.infer<typeof formSchema>

// 2. 构建表单
function ProfileForm() {
  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: {
      username: "",
      email: "",
      role: "user",
      notifications: false,
    },
  })

  function onSubmit(values: FormValues) {
    console.log(values)
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">

        {/* 文本输入 */}
        <FormField
          control={form.control}
          name="username"
          render={({ field }) => (
            <FormItem>
              <FormLabel>用户名</FormLabel>
              <FormControl>
                <Input placeholder="请输入用户名" {...field} />
              </FormControl>
              <FormDescription>这将是你的公开显示名称。</FormDescription>
              <FormMessage />  {/* 自动显示验证错误 */}
            </FormItem>
          )}
        />

        {/* 邮箱 */}
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>邮箱</FormLabel>
              <FormControl>
                <Input type="email" placeholder="name@example.com" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        {/* Select */}
        <FormField
          control={form.control}
          name="role"
          render={({ field }) => (
            <FormItem>
              <FormLabel>角色</FormLabel>
              <Select onValueChange={field.onChange} defaultValue={field.value}>
                <FormControl>
                  <SelectTrigger>
                    <SelectValue placeholder="选择角色" />
                  </SelectTrigger>
                </FormControl>
                <SelectContent>
                  <SelectItem value="admin">管理员</SelectItem>
                  <SelectItem value="user">普通用户</SelectItem>
                  <SelectItem value="guest">访客</SelectItem>
                </SelectContent>
              </Select>
              <FormMessage />
            </FormItem>
          )}
        />

        {/* Checkbox */}
        <FormField
          control={form.control}
          name="notifications"
          render={({ field }) => (
            <FormItem className="flex items-center space-x-3 space-y-0 rounded-md border p-4">
              <FormControl>
                <Checkbox
                  checked={field.value}
                  onCheckedChange={field.onChange}
                />
              </FormControl>
              <div>
                <FormLabel>接收通知</FormLabel>
                <FormDescription>接收产品更新和安全提醒。</FormDescription>
              </div>
            </FormItem>
          )}
        />

        <Button type="submit" disabled={form.formState.isSubmitting}>
          {form.formState.isSubmitting && (
            <Loader2 className="mr-2 h-4 w-4 animate-spin" />
          )}
          提交
        </Button>
      </form>
    </Form>
  )
}
```

---

## Table 表格

```tsx
import {
  Table,
  TableBody,
  TableCaption,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
  TableFooter,
} from "@/components/ui/table"

const invoices = [
  { id: "INV001", status: "已付款", method: "信用卡", amount: "¥250.00" },
  { id: "INV002", status: "待付款", method: "支付宝", amount: "¥150.00" },
  { id: "INV003", status: "已取消", method: "微信支付", amount: "¥350.00" },
]

<Table>
  <TableCaption>最近的账单列表</TableCaption>
  <TableHeader>
    <TableRow>
      <TableHead className="w-[100px]">账单号</TableHead>
      <TableHead>状态</TableHead>
      <TableHead>支付方式</TableHead>
      <TableHead className="text-right">金额</TableHead>
    </TableRow>
  </TableHeader>
  <TableBody>
    {invoices.map((inv) => (
      <TableRow key={inv.id}>
        <TableCell className="font-medium">{inv.id}</TableCell>
        <TableCell>{inv.status}</TableCell>
        <TableCell>{inv.method}</TableCell>
        <TableCell className="text-right">{inv.amount}</TableCell>
      </TableRow>
    ))}
  </TableBody>
  <TableFooter>
    <TableRow>
      <TableCell colSpan={3}>合计</TableCell>
      <TableCell className="text-right">¥750.00</TableCell>
    </TableRow>
  </TableFooter>
</Table>
```

---

## Toast 轻提示（Sonner）

shadcn 推荐使用 `sonner`：

```shell
npx shadcn@latest add sonner
```

```tsx
// 在根布局中添加
import { Toaster } from "@/components/ui/sonner"

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}
        <Toaster richColors position="top-right" />
      </body>
    </html>
  )
}

// 在任意组件中使用
import { toast } from "sonner"

toast("操作成功")
toast.success("保存成功！")
toast.error("保存失败，请重试")
toast.warning("注意：此操作不可逆")
toast.info("系统将在 5 分钟后维护")

// 带描述
toast.success("文件已上传", {
  description: "photo.jpg 已成功上传到云端",
})

// 带操作按钮
toast("已删除文章", {
  action: {
    label: "撤销",
    onClick: () => undoDelete(),
  },
})

// 加载态（Promise）
toast.promise(uploadFile(file), {
  loading: "上传中...",
  success: "上传成功！",
  error: "上传失败",
})
```

---

## Tooltip 工具提示

```tsx
import {
  Tooltip,
  TooltipContent,
  TooltipProvider,
  TooltipTrigger,
} from "@/components/ui/tooltip"

// TooltipProvider 通常放在根组件
<TooltipProvider>
  <Tooltip>
    <TooltipTrigger asChild>
      <Button variant="outline" size="icon">
        <Settings className="h-4 w-4" />
      </Button>
    </TooltipTrigger>
    <TooltipContent side="bottom">
      <p>打开设置</p>
    </TooltipContent>
  </Tooltip>
</TooltipProvider>
```

---

## Popover 弹出框

```tsx
import {
  Popover,
  PopoverContent,
  PopoverTrigger,
} from "@/components/ui/popover"

<Popover>
  <PopoverTrigger asChild>
    <Button variant="outline">打开</Button>
  </PopoverTrigger>
  <PopoverContent className="w-80" align="start">
    <div className="space-y-3">
      <h4 className="font-medium leading-none">尺寸</h4>
      <p className="text-sm text-muted-foreground">设置图层的尺寸。</p>
      <div className="grid grid-cols-3 items-center gap-4">
        <Label htmlFor="width">宽度</Label>
        <Input id="width" defaultValue="100%" className="col-span-2" />
      </div>
      <div className="grid grid-cols-3 items-center gap-4">
        <Label htmlFor="height">高度</Label>
        <Input id="height" defaultValue="25px" className="col-span-2" />
      </div>
    </div>
  </PopoverContent>
</Popover>
```

---

## Badge 徽章

```tsx
import { Badge } from "@/components/ui/badge"

<Badge>默认</Badge>
<Badge variant="secondary">次要</Badge>
<Badge variant="outline">边框</Badge>
<Badge variant="destructive">危险</Badge>

// 自定义颜色（直接加 className）
<Badge className="bg-green-500 hover:bg-green-600">已完成</Badge>
<Badge className="bg-yellow-500 hover:bg-yellow-600">进行中</Badge>
```

---

## Avatar 头像

```tsx
import { Avatar, AvatarFallback, AvatarImage } from "@/components/ui/avatar"

<Avatar>
  <AvatarImage src="https://github.com/shadcn.png" alt="@shadcn" />
  <AvatarFallback>CN</AvatarFallback>  {/* 图片加载失败时显示 */}
</Avatar>

// 头像组（叠加显示）
<div className="flex -space-x-3">
  {users.map((user) => (
    <Avatar key={user.id} className="border-2 border-background">
      <AvatarImage src={user.avatar} alt={user.name} />
      <AvatarFallback>{user.name[0]}</AvatarFallback>
    </Avatar>
  ))}
  <div className="flex h-10 w-10 items-center justify-center rounded-full border-2 border-background bg-muted text-sm">
    +5
  </div>
</div>
```

---

## Skeleton 骨架屏

```tsx
import { Skeleton } from "@/components/ui/skeleton"

// 直接使用
<Skeleton className="h-4 w-[250px]" />
<Skeleton className="h-4 w-[200px]" />

// 卡片骨架
function CardSkeleton() {
  return (
    <div className="flex items-center space-x-4 p-4">
      <Skeleton className="h-12 w-12 rounded-full" />
      <div className="space-y-2">
        <Skeleton className="h-4 w-[250px]" />
        <Skeleton className="h-4 w-[200px]" />
      </div>
    </div>
  )
}

// 搭配 Suspense 使用
function UserList() {
  return (
    <Suspense fallback={<>
      <CardSkeleton />
      <CardSkeleton />
      <CardSkeleton />
    </>}>
      <AsyncUserList />
    </Suspense>
  )
}
```

---

## Separator 分隔线

```tsx
import { Separator } from "@/components/ui/separator"

<div>
  <p>上方内容</p>
  <Separator className="my-4" />
  <p>下方内容</p>
</div>

// 垂直分隔线
<div className="flex h-5 items-center space-x-4 text-sm">
  <div>博客</div>
  <Separator orientation="vertical" />
  <div>文档</div>
  <Separator orientation="vertical" />
  <div>源码</div>
</div>
```

---

## Progress 进度条

```tsx
import { Progress } from "@/components/ui/progress"

<Progress value={60} className="w-full" />

// 动态进度
function UploadProgress() {
  const [progress, setProgress] = useState(0)

  useEffect(() => {
    const timer = setInterval(() => {
      setProgress((p) => {
        if (p >= 100) { clearInterval(timer); return 100 }
        return p + 10
      })
    }, 500)
    return () => clearInterval(timer)
  }, [])

  return (
    <div className="space-y-2">
      <div className="flex justify-between text-sm">
        <span>上传中...</span>
        <span>{progress}%</span>
      </div>
      <Progress value={progress} />
    </div>
  )
}
```

---

## 扩展与自定义组件

shadcn 组件源码完全开放，扩展非常直接：

```tsx
// 扩展 Button：添加 loading prop
import { Button, type ButtonProps } from "@/components/ui/button"
import { Loader2 } from "lucide-react"

interface LoadingButtonProps extends ButtonProps {
  loading?: boolean
}

export function LoadingButton({ loading, children, disabled, ...props }: LoadingButtonProps) {
  return (
    <Button disabled={loading || disabled} {...props}>
      {loading && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
      {children}
    </Button>
  )
}
```

```tsx
// 封装 FormField（减少重复代码）
import { useFormContext } from "react-hook-form"
import { FormControl, FormField, FormItem, FormLabel, FormMessage } from "@/components/ui/form"
import { Input } from "@/components/ui/input"

interface FormInputProps {
  name: string
  label: string
  placeholder?: string
  type?: string
}

export function FormInput({ name, label, placeholder, type = "text" }: FormInputProps) {
  const form = useFormContext()

  return (
    <FormField
      control={form.control}
      name={name}
      render={({ field }) => (
        <FormItem>
          <FormLabel>{label}</FormLabel>
          <FormControl>
            <Input type={type} placeholder={placeholder} {...field} />
          </FormControl>
          <FormMessage />
        </FormItem>
      )}
    />
  )
}

// 使用
<Form {...form}>
  <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
    <FormInput name="username" label="用户名" placeholder="请输入用户名" />
    <FormInput name="email" label="邮箱" type="email" placeholder="name@example.com" />
    <FormInput name="password" label="密码" type="password" />
    <Button type="submit">注册</Button>
  </form>
</Form>
```

---

## 常用组件速查

| 组件 | 安装命令 | 用途 |
|------|---------|------|
| Button | `add button` | 按钮 |
| Input | `add input` | 输入框 |
| Label | `add label` | 表单标签 |
| Card | `add card` | 卡片容器 |
| Dialog | `add dialog` | 模态对话框 |
| Sheet | `add sheet` | 侧边抽屉 |
| DropdownMenu | `add dropdown-menu` | 下拉菜单 |
| Select | `add select` | 选择器 |
| Tabs | `add tabs` | 标签页 |
| Form | `add form` | 表单（含 rhf 集成） |
| Table | `add table` | 表格 |
| Sonner | `add sonner` | Toast 提示 |
| Tooltip | `add tooltip` | 工具提示 |
| Popover | `add popover` | 弹出框 |
| Badge | `add badge` | 徽章 |
| Avatar | `add avatar` | 头像 |
| Skeleton | `add skeleton` | 骨架屏 |
| Separator | `add separator` | 分隔线 |
| Progress | `add progress` | 进度条 |
| Checkbox | `add checkbox` | 复选框 |
| Switch | `add switch` | 开关 |
| Slider | `add slider` | 滑块 |
| RadioGroup | `add radio-group` | 单选组 |
| Accordion | `add accordion` | 折叠面板 |
| Collapsible | `add collapsible` | 可折叠区域 |
| Calendar | `add calendar` | 日历 |
| DatePicker | `add date-picker` | 日期选择器 |
| Command | `add command` | 命令面板（⌘K） |
| Combobox | `add combobox` | 可搜索下拉 |
| DataTable | `add data-table` | 数据表格（含排序/筛选） |
| NavigationMenu | `add navigation-menu` | 导航菜单 |
| Breadcrumb | `add breadcrumb` | 面包屑 |
| Pagination | `add pagination` | 分页 |
| AlertDialog | `add alert-dialog` | 确认对话框 |
| Alert | `add alert` | 警告提示 |
| HoverCard | `add hover-card` | 悬停卡片 |
| ScrollArea | `add scroll-area` | 自定义滚动区域 |
| Resizable | `add resizable` | 可调整大小的面板 |
