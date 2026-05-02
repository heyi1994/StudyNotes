# Tailwind CSS

Tailwind CSS 是一个原子化 CSS 框架，通过直接在 HTML/JSX 中组合预设的工具类来构建样式，无需编写自定义 CSS。当前版本为 **v4**，配置方式与 v3 有较大变化。

---

## 安装

### Vite 项目（推荐）

```shell
npm install tailwindcss @tailwindcss/vite
```

```typescript
// vite.config.ts
import { defineConfig } from "vite"
import tailwindcss from "@tailwindcss/vite"

export default defineConfig({
  plugins: [tailwindcss()],
})
```

```css
/* src/index.css */
@import "tailwindcss";
```

### Next.js

```shell
npm install tailwindcss @tailwindcss/postcss postcss
```

```javascript
// postcss.config.mjs
const config = { plugins: { "@tailwindcss/postcss": {} } }
export default config
```

```css
/* app/globals.css */
@import "tailwindcss";
```

---

## 核心概念

Tailwind 的每个类名对应一条或几条 CSS 规则：

```html
<!-- 传统 CSS 写法 -->
<div class="card">内容</div>

<!-- Tailwind 原子化写法 -->
<div class="rounded-lg bg-white p-6 shadow-md">内容</div>
```

### 命名规律

大多数工具类遵循 `属性-值` 的命名模式：

| 类名 | 对应 CSS |
|------|---------|
| `flex` | `display: flex` |
| `p-4` | `padding: 1rem` |
| `mt-2` | `margin-top: 0.5rem` |
| `text-lg` | `font-size: 1.125rem` |
| `bg-blue-500` | `background-color: #3b82f6` |
| `rounded-md` | `border-radius: 0.375rem` |
| `w-full` | `width: 100%` |
| `hidden` | `display: none` |

---

## 布局

### Display

```html
<div class="block">块级</div>
<div class="inline-block">行内块</div>
<div class="inline">行内</div>
<div class="hidden">隐藏（display: none）</div>
<div class="invisible">不可见（visibility: hidden，仍占位）</div>
```

### Flexbox

```html
<!-- 基础 flex 容器 -->
<div class="flex items-center justify-between gap-4">
  <div>左</div>
  <div>右</div>
</div>

<!-- 主轴方向 -->
<div class="flex flex-row">水平（默认）</div>
<div class="flex flex-col">垂直</div>
<div class="flex flex-row-reverse">反向水平</div>
<div class="flex flex-col-reverse">反向垂直</div>

<!-- 主轴对齐（justify-content） -->
<div class="flex justify-start">   <!-- flex-start -->
<div class="flex justify-center">  <!-- center -->
<div class="flex justify-end">     <!-- flex-end -->
<div class="flex justify-between"> <!-- space-between -->
<div class="flex justify-around">  <!-- space-around -->
<div class="flex justify-evenly">  <!-- space-evenly -->

<!-- 交叉轴对齐（align-items） -->
<div class="flex items-start">
<div class="flex items-center">
<div class="flex items-end">
<div class="flex items-stretch">
<div class="flex items-baseline">

<!-- 换行 -->
<div class="flex flex-wrap">
<div class="flex flex-nowrap">

<!-- flex 子项 -->
<div class="flex-1">      <!-- flex: 1 1 0%，占据剩余空间 -->
<div class="flex-auto">   <!-- flex: 1 1 auto -->
<div class="flex-none">   <!-- flex: none，不伸缩 -->
<div class="flex-shrink-0"> <!-- 禁止收缩 -->
<div class="grow">        <!-- flex-grow: 1 -->
<div class="shrink-0">    <!-- flex-shrink: 0 -->
<div class="order-first"> <!-- order: -9999 -->
<div class="order-last">  <!-- order: 9999 -->
<div class="order-2">     <!-- order: 2 -->
```

### Grid

```html
<!-- 定义列数 -->
<div class="grid grid-cols-3 gap-4">
  <div>1</div>
  <div>2</div>
  <div>3</div>
</div>

<!-- 自适应列（响应式卡片布局利器） -->
<div class="grid grid-cols-[repeat(auto-fill,minmax(200px,1fr))] gap-4">

<!-- 列跨越 -->
<div class="grid grid-cols-4">
  <div class="col-span-2">占 2 列</div>
  <div class="col-span-1">占 1 列</div>
  <div class="col-start-4">从第 4 列开始</div>
</div>

<!-- 行跨越 -->
<div class="row-span-2">占 2 行</div>

<!-- 定义行高 -->
<div class="grid grid-rows-3 grid-flow-col">

<!-- place-items：同时设置 align-items 和 justify-items -->
<div class="grid place-items-center h-screen">居中</div>
```

### Position

```html
<div class="relative">
  <div class="absolute top-0 right-0">右上角</div>
  <div class="absolute inset-0">完全覆盖父元素</div>
  <div class="absolute inset-x-0 bottom-0">底部横幅</div>
</div>

<div class="fixed bottom-4 right-4">固定在右下角</div>
<div class="sticky top-0 z-10">吸顶导航</div>

<!-- inset 简写 -->
<!-- inset-0 = top:0 right:0 bottom:0 left:0 -->
<!-- inset-x-0 = left:0 right:0 -->
<!-- inset-y-0 = top:0 bottom:0 -->
```

### 溢出与可见性

```html
<div class="overflow-hidden">
<div class="overflow-auto">
<div class="overflow-x-auto overflow-y-hidden">
<div class="truncate">文本过长时显示省略号</div>
<div class="line-clamp-3">最多显示 3 行，超出省略</div>
```

---

## 间距

### 默认尺寸比例（1 单位 = 0.25rem = 4px）

| 类名 | rem | px |
|------|-----|----|
| `p-0` | 0 | 0 |
| `p-1` | 0.25rem | 4px |
| `p-2` | 0.5rem | 8px |
| `p-4` | 1rem | 16px |
| `p-6` | 1.5rem | 24px |
| `p-8` | 2rem | 32px |
| `p-12` | 3rem | 48px |
| `p-16` | 4rem | 64px |

### Padding

```html
<div class="p-4">所有方向 1rem</div>
<div class="px-4">水平方向（padding-left + right）</div>
<div class="py-2">垂直方向（padding-top + bottom）</div>
<div class="pt-4 pb-2 pl-6 pr-8">分别设置四个方向</div>
<div class="ps-4 pe-4">逻辑属性（start/end，支持 RTL）</div>
```

### Margin

```html
<div class="m-4">所有方向</div>
<div class="mx-auto">水平居中</div>
<div class="mt-4 mb-8">上下</div>
<div class="-mt-2">负 margin（-0.5rem）</div>
```

### Gap（flex/grid 子项间距）

```html
<div class="flex gap-4">          <!-- 所有方向 -->
<div class="flex gap-x-4 gap-y-2"> <!-- 分别设置 -->
<div class="grid grid-cols-3 gap-6">
```

### Space Between（自动为子项添加间距）

```html
<div class="flex space-x-4">
  <div>A</div>   <!-- 除最后一个外，其他右侧加 1rem margin -->
  <div>B</div>
  <div>C</div>
</div>
```

---

## 尺寸

### Width / Height

```html
<!-- 固定尺寸 -->
<div class="w-4 h-4">   <!-- 1rem × 1rem -->
<div class="w-16 h-16"> <!-- 4rem × 4rem -->

<!-- 百分比 -->
<div class="w-1/2">   <!-- 50% -->
<div class="w-1/3">   <!-- 33.333% -->
<div class="w-2/3">   <!-- 66.666% -->
<div class="w-full">  <!-- 100% -->
<div class="h-full">  <!-- 100% -->
<div class="h-screen"> <!-- 100vh -->
<div class="h-dvh">    <!-- 100dvh（动态视口高度，适配移动端） -->

<!-- 自适应 -->
<div class="w-auto">
<div class="w-fit">    <!-- fit-content -->
<div class="w-max">    <!-- max-content -->
<div class="w-min">    <!-- min-content -->

<!-- 最大/最小尺寸 -->
<div class="max-w-sm">    <!-- max-width: 24rem -->
<div class="max-w-md">    <!-- max-width: 28rem -->
<div class="max-w-lg">    <!-- max-width: 32rem -->
<div class="max-w-xl">    <!-- max-width: 36rem -->
<div class="max-w-2xl">   <!-- max-width: 42rem -->
<div class="max-w-screen-lg"> <!-- max-width: 1024px -->
<div class="min-h-screen">
<div class="min-w-0">     <!-- 修复 flex 子项溢出问题 -->

<!-- 正方形/圆形 -->
<div class="size-10">     <!-- width: 2.5rem; height: 2.5rem -->
<div class="size-full">
```

---

## 排版

### 字体

```html
<!-- 字体族 -->
<p class="font-sans">   <!-- 无衬线（默认） -->
<p class="font-serif">  <!-- 衬线 -->
<p class="font-mono">   <!-- 等宽 -->

<!-- 字体大小 -->
<p class="text-xs">     <!-- 0.75rem / 12px -->
<p class="text-sm">     <!-- 0.875rem / 14px -->
<p class="text-base">   <!-- 1rem / 16px -->
<p class="text-lg">     <!-- 1.125rem / 18px -->
<p class="text-xl">     <!-- 1.25rem / 20px -->
<p class="text-2xl">    <!-- 1.5rem / 24px -->
<p class="text-3xl">    <!-- 1.875rem / 30px -->
<p class="text-4xl">    <!-- 2.25rem / 36px -->

<!-- 字体粗细 -->
<p class="font-thin">       <!-- 100 -->
<p class="font-light">      <!-- 300 -->
<p class="font-normal">     <!-- 400 -->
<p class="font-medium">     <!-- 500 -->
<p class="font-semibold">   <!-- 600 -->
<p class="font-bold">       <!-- 700 -->
<p class="font-extrabold">  <!-- 800 -->

<!-- 行高 -->
<p class="leading-none">    <!-- 1 -->
<p class="leading-tight">   <!-- 1.25 -->
<p class="leading-normal">  <!-- 1.5 -->
<p class="leading-loose">   <!-- 2 -->
<p class="leading-6">       <!-- 1.5rem -->

<!-- 字间距 -->
<p class="tracking-tight">   <!-- -0.025em -->
<p class="tracking-normal">  <!-- 0em -->
<p class="tracking-wide">    <!-- 0.025em -->
<p class="tracking-wider">   <!-- 0.05em -->
<p class="tracking-widest">  <!-- 0.1em -->
```

### 文本对齐与装饰

```html
<p class="text-left">
<p class="text-center">
<p class="text-right">
<p class="text-justify">

<p class="underline">下划线</p>
<p class="line-through">删除线</p>
<p class="no-underline">去掉下划线</p>
<p class="italic">斜体</p>
<p class="uppercase">全大写</p>
<p class="lowercase">全小写</p>
<p class="capitalize">首字母大写</p>

<p class="whitespace-nowrap">不换行</p>
<p class="whitespace-pre">保留空白</p>
<p class="break-words">长单词换行</p>
<p class="break-all">任意位置换行</p>
```

---

## 颜色

Tailwind 内置了丰富的调色板，格式为 `{属性}-{颜色}-{深浅}`：

```html
<!-- 文字颜色 -->
<p class="text-gray-900">深灰文字</p>
<p class="text-blue-600">蓝色文字</p>
<p class="text-red-500">红色文字</p>
<p class="text-white">白色</p>
<p class="text-transparent">透明</p>

<!-- 背景颜色 -->
<div class="bg-white">
<div class="bg-gray-50">
<div class="bg-blue-500">
<div class="bg-gradient-to-r from-blue-500 to-purple-600">渐变背景</div>

<!-- 边框颜色 -->
<div class="border border-gray-200">
<div class="border-2 border-blue-500">

<!-- 颜色透明度（/ 语法）-->
<div class="bg-blue-500/50">  <!-- 50% 透明度 -->
<p class="text-black/80">     <!-- 80% 不透明 -->
<div class="border border-gray-900/10">
```

### 内置颜色色阶

每种颜色有 50 ~ 950 共 11 个色阶：

```html
<!-- 以 blue 为例 -->
bg-blue-50    <!-- 极浅 -->
bg-blue-100
bg-blue-200
bg-blue-300
bg-blue-400
bg-blue-500   <!-- 标准色 -->
bg-blue-600
bg-blue-700
bg-blue-800
bg-blue-900
bg-blue-950   <!-- 极深 -->
```

内置颜色名：`slate` `gray` `zinc` `neutral` `stone` `red` `orange` `amber` `yellow` `lime` `green` `emerald` `teal` `cyan` `sky` `blue` `indigo` `violet` `purple` `fuchsia` `pink` `rose`

---

## 边框与圆角

```html
<!-- 边框宽度 -->
<div class="border">    <!-- 1px -->
<div class="border-2">  <!-- 2px -->
<div class="border-4">  <!-- 4px -->
<div class="border-t">  <!-- 仅上边框 -->
<div class="border-x">  <!-- 左右边框 -->

<!-- 圆角 -->
<div class="rounded">    <!-- 0.25rem -->
<div class="rounded-md"> <!-- 0.375rem -->
<div class="rounded-lg"> <!-- 0.5rem -->
<div class="rounded-xl"> <!-- 0.75rem -->
<div class="rounded-2xl"> <!-- 1rem -->
<div class="rounded-full"> <!-- 9999px，圆形/胶囊 -->
<div class="rounded-none"> <!-- 0 -->
<div class="rounded-t-lg"> <!-- 仅上方圆角 -->
<div class="rounded-tl-xl"><!-- 仅左上角 -->

<!-- 边框样式 -->
<div class="border border-solid">
<div class="border border-dashed">
<div class="border border-dotted">

<!-- outline -->
<button class="outline-none focus:outline-2 focus:outline-blue-500">
```

---

## 阴影

```html
<!-- box-shadow -->
<div class="shadow-sm">
<div class="shadow">
<div class="shadow-md">
<div class="shadow-lg">
<div class="shadow-xl">
<div class="shadow-2xl">
<div class="shadow-none">
<div class="shadow-inner">  <!-- 内阴影 -->

<!-- 彩色阴影 -->
<div class="shadow-lg shadow-blue-500/50">

<!-- 文字阴影 -->
<p class="text-shadow-sm">
<p class="text-shadow-md">
```

---

## 响应式设计

Tailwind 采用**移动优先**策略，断点前缀表示"此断点及以上生效"：

| 前缀 | 最小宽度 | 对应设备 |
|------|---------|---------|
| 无前缀 | 0px | 所有设备（移动端基础样式） |
| `sm:` | 640px | 大手机 |
| `md:` | 768px | 平板 |
| `lg:` | 1024px | 桌面 |
| `xl:` | 1280px | 大屏桌面 |
| `2xl:` | 1536px | 超宽屏 |

```html
<!-- 移动端 1 列，平板 2 列，桌面 4 列 -->
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">

<!-- 移动端隐藏，桌面显示 -->
<div class="hidden lg:block">桌面导航</div>

<!-- 移动端显示，桌面隐藏 -->
<div class="block lg:hidden">移动端菜单</div>

<!-- 字体随屏幕变化 -->
<h1 class="text-2xl md:text-4xl lg:text-6xl font-bold">

<!-- 内边距随屏幕变化 -->
<div class="p-4 md:p-8 lg:p-12">

<!-- 最大宽度容器 -->
<div class="max-w-sm md:max-w-2xl lg:max-w-5xl mx-auto px-4">
```

### 只在某个范围生效

```html
<!-- 只在 md 断点生效（md 到 lg 之间） -->
<div class="md:max-lg:bg-blue-500">
```

---

## 状态变体

### 伪类

```html
<!-- hover -->
<button class="bg-blue-500 hover:bg-blue-600 transition-colors">

<!-- focus -->
<input class="border focus:border-blue-500 focus:ring-2 focus:ring-blue-200 outline-none">

<!-- active（点击时） -->
<button class="active:scale-95 active:bg-blue-700">

<!-- disabled -->
<button class="disabled:opacity-50 disabled:cursor-not-allowed" disabled>

<!-- focus-within（子元素获焦时） -->
<div class="focus-within:ring-2 focus-within:ring-blue-500">
  <input type="text" />
</div>

<!-- first / last / odd / even -->
<ul>
  <li class="first:pt-0 last:pb-0 odd:bg-gray-50 even:bg-white py-2">
</ul>

<!-- 空状态 -->
<p class="hidden empty:block">暂无数据</p>

<!-- required / invalid / checked -->
<input class="border invalid:border-red-500 focus:invalid:ring-red-200" required>
<input type="checkbox" class="checked:bg-blue-500">
```

### 伪元素

```html
<!-- before / after -->
<div class="before:content-[''] before:block before:w-4 before:h-4 before:bg-red-500">

<!-- placeholder -->
<input class="placeholder:text-gray-400 placeholder:italic">

<!-- selection（文本选中） -->
<p class="selection:bg-blue-200 selection:text-blue-900">

<!-- first-line / first-letter -->
<p class="first-line:uppercase first-letter:text-4xl first-letter:font-bold">
```

### 暗色模式

```html
<!-- 跟随系统（class="dark" 也可以） -->
<div class="bg-white dark:bg-gray-900">
<p class="text-gray-900 dark:text-gray-100">
<img class="opacity-100 dark:opacity-80">
```

在 CSS 中开启 class 策略（手动切换）：

```css
/* src/index.css */
@import "tailwindcss";
@custom-variant dark (&:where(.dark, .dark *));
```

---

## 动画与过渡

### Transition

```html
<!-- 过渡（需搭配 hover 等状态变体） -->
<button class="bg-blue-500 hover:bg-blue-600 transition-colors duration-200">
<div class="opacity-0 hover:opacity-100 transition-opacity duration-300 ease-in-out">

<!-- transition-all：所有属性 -->
<div class="transition-all duration-200">

<!-- 常用 duration -->
<div class="duration-75">   <!-- 75ms -->
<div class="duration-150">  <!-- 150ms -->
<div class="duration-200">  <!-- 200ms -->
<div class="duration-300">  <!-- 300ms -->
<div class="duration-500">  <!-- 500ms -->

<!-- easing -->
<div class="ease-linear">
<div class="ease-in">
<div class="ease-out">
<div class="ease-in-out">
```

### Transform

```html
<!-- 缩放 -->
<div class="hover:scale-105 transition-transform">
<div class="hover:scale-x-110">
<div class="active:scale-95">

<!-- 位移 -->
<div class="hover:translate-x-1 hover:-translate-y-1">
<div class="-translate-x-1/2 -translate-y-1/2 absolute top-1/2 left-1/2"> <!-- 绝对居中 -->

<!-- 旋转 -->
<div class="hover:rotate-6">
<div class="-rotate-3">
<div class="rotate-180">  <!-- 旋转 180° -->

<!-- 倾斜 -->
<div class="skew-x-3">
```

### 内置动画

```html
<div class="animate-spin">    <!-- 旋转（loading 图标） -->
<div class="animate-ping">    <!-- 扩散（通知徽章） -->
<div class="animate-pulse">   <!-- 脉冲（骨架屏） -->
<div class="animate-bounce">  <!-- 弹跳 -->

<!-- 骨架屏示例 -->
<div class="animate-pulse flex space-x-4">
  <div class="rounded-full bg-gray-200 size-10"></div>
  <div class="flex-1 space-y-2 py-1">
    <div class="h-4 bg-gray-200 rounded w-3/4"></div>
    <div class="h-4 bg-gray-200 rounded w-1/2"></div>
  </div>
</div>
```

---

## 自定义配置（v4）

Tailwind v4 使用 CSS 文件配置，不再需要 `tailwind.config.js`：

```css
/* src/index.css */
@import "tailwindcss";

/* 自定义主题变量 */
@theme {
  /* 颜色 */
  --color-primary-50:  #eff6ff;
  --color-primary-100: #dbeafe;
  --color-primary-500: #3b82f6;
  --color-primary-600: #2563eb;
  --color-primary-900: #1e3a8a;

  --color-brand: #e11d48;

  /* 字体 */
  --font-sans: "Inter", ui-sans-serif, system-ui, sans-serif;
  --font-display: "Cal Sans", "Inter", sans-serif;

  /* 断点 */
  --breakpoint-xs: 480px;
  --breakpoint-3xl: 1920px;

  /* 间距 */
  --spacing-18: 4.5rem;
  --spacing-128: 32rem;

  /* 圆角 */
  --radius-4xl: 2rem;

  /* 动画 */
  --animate-fade-in: fade-in 0.3s ease-out;

  @keyframes fade-in {
    from { opacity: 0; transform: translateY(4px); }
    to   { opacity: 1; transform: translateY(0); }
  }
}

/* 自定义工具类 */
@utility container {
  width: 100%;
  max-width: 1200px;
  margin-inline: auto;
  padding-inline: 1rem;

  @media (width >= theme(--breakpoint-md)) {
    padding-inline: 2rem;
  }
}

/* 自定义变体 */
@custom-variant dark (&:where(.dark, .dark *));
@custom-variant hocus (&:hover, &:focus);
```

使用自定义主题变量：

```html
<div class="bg-primary-500 text-white">
<div class="font-display text-4xl">
<div class="p-18">
<div class="animate-fade-in">
<button class="hocus:bg-primary-600">  <!-- 自定义变体 -->
```

---

## 任意值（Arbitrary Values）

对于无法通过预设类名表达的值，使用 `[]` 语法直接写入 CSS 值：

```html
<!-- 任意尺寸 -->
<div class="w-[328px] h-[calc(100vh-64px)]">
<div class="max-w-[960px]">
<div class="top-[117px]">

<!-- 任意颜色 -->
<div class="bg-[#1da1f2]">
<div class="text-[rgb(99,102,241)]">
<div class="border-[color:var(--brand-color)]">

<!-- 任意字体大小 -->
<p class="text-[13px] leading-[1.6]">

<!-- 任意网格 -->
<div class="grid-cols-[1fr_2fr_1fr]">
<div class="grid-cols-[repeat(auto-fill,minmax(200px,1fr))]">

<!-- 任意变换 -->
<div class="translate-x-[calc(-50%-4px)]">

<!-- CSS 变量 -->
<div class="bg-[var(--card-bg)]">
<div class="text-[--text-color]">  <!-- v4 简写 -->

<!-- 任意属性（需要 Tailwind 不支持的 CSS 属性） -->
<div class="[mask-image:linear-gradient(to_bottom,white,transparent)]">
<div class="[scrollbar-width:none]">
<div class="[&>svg]:w-5 [&>svg]:fill-current"> <!-- 嵌套选择器 -->
```

---

## 提取组件（@apply）

当某组类名频繁重复时，可以用 `@apply` 提取到 CSS 类中：

```css
/* src/index.css */
@import "tailwindcss";

@layer components {
  .btn {
    @apply inline-flex items-center justify-center rounded-md px-4 py-2
           text-sm font-medium transition-colors focus-visible:outline-none
           focus-visible:ring-2 disabled:pointer-events-none disabled:opacity-50;
  }

  .btn-primary {
    @apply btn bg-primary-500 text-white hover:bg-primary-600
           focus-visible:ring-primary-500;
  }

  .btn-outline {
    @apply btn border border-gray-300 bg-transparent hover:bg-gray-100;
  }

  .card {
    @apply rounded-xl border border-gray-200 bg-white p-6 shadow-sm;
  }

  .input {
    @apply w-full rounded-md border border-gray-300 px-3 py-2 text-sm
           placeholder:text-gray-400 focus:border-primary-500
           focus:outline-none focus:ring-2 focus:ring-primary-200;
  }
}
```

> 大多数情况下，推荐在 React/Vue 中封装组件而不是 `@apply`，`@apply` 会增大 CSS 文件体积且降低可读性。

---

## 在 React 中的最佳实践

### 用 clsx / cn 合并条件类名

```shell
npm install clsx tailwind-merge
```

```typescript
// src/lib/cn.ts
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"

// twMerge 解决 Tailwind 类名冲突（如 p-4 和 px-2 同时存在时正确合并）
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

```tsx
import { cn } from "@/lib/cn"

interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "outline" | "ghost"
  size?: "sm" | "md" | "lg"
}

function Button({ variant = "primary", size = "md", className, ...props }: ButtonProps) {
  return (
    <button
      className={cn(
        // 基础样式
        "inline-flex items-center justify-center rounded-md font-medium transition-colors",
        "focus-visible:outline-none focus-visible:ring-2 disabled:opacity-50",
        // 变体
        {
          "bg-blue-500 text-white hover:bg-blue-600":  variant === "primary",
          "border border-gray-300 hover:bg-gray-100":  variant === "outline",
          "hover:bg-gray-100":                         variant === "ghost",
        },
        // 尺寸
        {
          "h-8 px-3 text-xs":   size === "sm",
          "h-10 px-4 text-sm":  size === "md",
          "h-12 px-6 text-base": size === "lg",
        },
        // 允许外部覆盖
        className,
      )}
      {...props}
    />
  )
}
```

### CVA（Class Variance Authority）

更系统的组件变体管理方案：

```shell
npm install class-variance-authority
```

```tsx
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/cn"

const buttonVariants = cva(
  // 基础类名
  "inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 disabled:opacity-50",
  {
    variants: {
      variant: {
        primary: "bg-blue-500 text-white hover:bg-blue-600 focus-visible:ring-blue-300",
        destructive: "bg-red-500 text-white hover:bg-red-600",
        outline: "border border-gray-300 bg-transparent hover:bg-gray-100",
        ghost: "hover:bg-gray-100 hover:text-gray-900",
        link: "text-blue-500 underline-offset-4 hover:underline",
      },
      size: {
        sm: "h-8 px-3 text-xs",
        md: "h-10 px-4",
        lg: "h-12 px-6 text-base",
        icon: "size-10",
      },
    },
    compoundVariants: [
      // 组合变体：primary + lg 时额外添加类
      { variant: "primary", size: "lg", class: "font-bold" },
    ],
    defaultVariants: {
      variant: "primary",
      size: "md",
    },
  }
)

interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {}

function Button({ variant, size, className, ...props }: ButtonProps) {
  return (
    <button className={cn(buttonVariants({ variant, size }), className)} {...props} />
  )
}

// 使用
<Button>默认</Button>
<Button variant="outline" size="sm">小号边框按钮</Button>
<Button variant="destructive" size="lg">大号危险按钮</Button>
```

---

## 常用组件片段

### 导航栏

```html
<nav class="sticky top-0 z-50 border-b border-gray-200 bg-white/80 backdrop-blur-sm">
  <div class="mx-auto flex h-16 max-w-7xl items-center justify-between px-4">
    <a href="/" class="text-xl font-bold text-gray-900">Logo</a>
    <div class="hidden md:flex items-center gap-6">
      <a class="text-sm font-medium text-gray-600 hover:text-gray-900 transition-colors">首页</a>
      <a class="text-sm font-medium text-gray-600 hover:text-gray-900 transition-colors">关于</a>
    </div>
    <button class="rounded-md bg-blue-500 px-4 py-2 text-sm font-medium text-white hover:bg-blue-600">
      登录
    </button>
  </div>
</nav>
```

### 卡片

```html
<div class="rounded-xl border border-gray-200 bg-white p-6 shadow-sm hover:shadow-md transition-shadow">
  <div class="flex items-center gap-3 mb-4">
    <img class="size-10 rounded-full object-cover" src="avatar.jpg" alt="">
    <div>
      <p class="text-sm font-semibold text-gray-900">张三</p>
      <p class="text-xs text-gray-500">2 小时前</p>
    </div>
  </div>
  <p class="text-sm text-gray-700 leading-relaxed line-clamp-3">内容文字...</p>
</div>
```

### 输入框

```html
<div class="space-y-1.5">
  <label class="text-sm font-medium text-gray-700">邮箱</label>
  <input
    type="email"
    placeholder="name@example.com"
    class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm
           placeholder:text-gray-400 transition-colors
           focus:border-blue-500 focus:outline-none focus:ring-2 focus:ring-blue-200
           invalid:border-red-500 invalid:focus:ring-red-200"
  >
  <p class="text-xs text-red-500 hidden peer-invalid:block">请输入有效邮箱</p>
</div>
```

### 徽章

```html
<span class="inline-flex items-center rounded-full bg-blue-50 px-2.5 py-0.5 text-xs font-medium text-blue-700 ring-1 ring-inset ring-blue-700/10">
  进行中
</span>
<span class="inline-flex items-center rounded-full bg-green-50 px-2.5 py-0.5 text-xs font-medium text-green-700 ring-1 ring-inset ring-green-600/20">
  已完成
</span>
```

### 绝对居中

```html
<!-- 方式一：flex -->
<div class="flex items-center justify-center min-h-screen">
  <div>居中内容</div>
</div>

<!-- 方式二：grid -->
<div class="grid place-items-center min-h-screen">
  <div>居中内容</div>
</div>

<!-- 方式三：absolute + translate -->
<div class="relative">
  <div class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2">
    居中内容
  </div>
</div>
```

---

## 常用速查

### 最常用的工具类组合

```html
<!-- 水平垂直居中 -->
flex items-center justify-center

<!-- 卡片 -->
rounded-lg border bg-white p-6 shadow-sm

<!-- 按钮基础 -->
inline-flex items-center rounded-md px-4 py-2 text-sm font-medium transition-colors

<!-- 输入框基础 -->
w-full rounded-md border px-3 py-2 text-sm focus:outline-none focus:ring-2

<!-- 截断文本 -->
truncate

<!-- 多行截断 -->
line-clamp-2

<!-- 响应式容器 -->
mx-auto max-w-7xl px-4 sm:px-6 lg:px-8

<!-- 分隔线 -->
border-t border-gray-200

<!-- 遮罩层 -->
fixed inset-0 bg-black/50 backdrop-blur-sm z-50

<!-- 隐藏滚动条但可滚动 -->
overflow-auto [scrollbar-width:none] [-ms-overflow-style:none] [&::-webkit-scrollbar]:hidden
```
