# Sass

Sass（Syntactically Awesome Style Sheets）是 CSS 的预处理器，提供变量、嵌套、混入、函数等特性，编译后生成标准 CSS。

Sass 有两种语法格式：

| 格式 | 扩展名 | 特点 |
|------|--------|------|
| SCSS | `.scss` | 是 CSS 的超集，兼容所有 CSS 语法，更常用 |
| Sass | `.sass` | 缩进语法，无大括号和分号，更简洁 |

---

## 安装

### 独立安装

```shell
npm install -g sass
```

### 在项目中使用

```shell
npm install -D sass
```

Vite、webpack 等构建工具检测到 `sass` 依赖后会自动处理 `.scss` 文件，无需额外配置。

### 编译命令

```shell
# 编译单个文件
sass input.scss output.css

# 监听文件变化
sass --watch input.scss output.css

# 监听整个目录
sass --watch src/styles:dist/styles

# 压缩输出
sass --style=compressed input.scss output.css
```

---

## 变量

使用 `$` 定义变量，支持任意 CSS 值：

```scss
$primary-color: #3498db;
$font-size-base: 16px;
$font-stack: "Helvetica Neue", Arial, sans-serif;
$border-radius: 4px;
$spacing-unit: 8px;

.button {
  background-color: $primary-color;
  font-size: $font-size-base;
  font-family: $font-stack;
  border-radius: $border-radius;
  padding: $spacing-unit $spacing-unit * 2;
}
```

### 变量作用域

变量默认是局部作用域，使用 `!global` 提升为全局：

```scss
$color: red;

.component {
  $color: blue;           // 局部变量，不影响外部
  $size: 14px !global;   // 提升为全局变量
  color: $color;          // blue
}

.other {
  color: $color;   // red（使用全局 $color）
  font-size: $size; // 14px（使用上面提升的全局变量）
}
```

### 默认值 `!default`

常用于库/主题开发，允许外部覆盖：

```scss
// _variables.scss（库内默认值）
$primary-color: #3498db !default;
$border-radius: 4px !default;

// main.scss（用户覆盖，必须在 @use 之前定义）
@use 'variables' with (
  $primary-color: #e74c3c,
  $border-radius: 8px
);
```

---

## 嵌套

### 选择器嵌套

```scss
nav {
  background: #333;

  ul {
    list-style: none;
    margin: 0;
  }

  li {
    display: inline-block;
  }

  a {
    color: white;
    text-decoration: none;

    &:hover {
      color: #aaa;
    }
  }
}
```

编译后：

```css
nav { background: #333; }
nav ul { list-style: none; margin: 0; }
nav li { display: inline-block; }
nav a { color: white; text-decoration: none; }
nav a:hover { color: #aaa; }
```

### `&` 父选择器引用

`&` 代表当前选择器，用于伪类、BEM 命名等：

```scss
.button {
  background: blue;

  &:hover { background: darkblue; }    // .button:hover
  &:disabled { opacity: 0.5; }          // .button:disabled
  &.active { background: green; }       // .button.active
  &--primary { background: #3498db; }   // .button--primary（BEM 修饰符）
  &__icon { margin-right: 8px; }        // .button__icon（BEM 元素）

  .dark-theme & { background: white; }  // .dark-theme .button（反向嵌套）
}
```

### 属性嵌套

相同命名空间的属性可以嵌套书写：

```scss
.element {
  font: {
    family: Arial;
    size: 16px;
    weight: bold;
  }

  border: {
    top: 1px solid #ccc;
    bottom: 2px solid #999;
  }

  background: {
    color: #fff;
    image: url('bg.png');
    repeat: no-repeat;
  }
}
```

---

## @mixin 混入

混入定义可复用的样式块，可以接受参数。

### 基本用法

```scss
@mixin flex-center {
  display: flex;
  justify-content: center;
  align-items: center;
}

.hero {
  @include flex-center;
  height: 100vh;
}
```

### 带参数的混入

```scss
@mixin button($bg-color, $text-color: white, $padding: 8px 16px) {
  background-color: $bg-color;
  color: $text-color;
  padding: $padding;
  border: none;
  border-radius: 4px;
  cursor: pointer;

  &:hover {
    background-color: darken($bg-color, 10%);
  }
}

.btn-primary { @include button(#3498db); }
.btn-danger  { @include button(#e74c3c, white, 10px 20px); }
.btn-outline { @include button(transparent, #333); }
```

### 可变参数

```scss
@mixin box-shadow($shadows...) {
  box-shadow: $shadows;
}

.card {
  @include box-shadow(0 2px 4px rgba(0,0,0,0.1), 0 4px 8px rgba(0,0,0,0.05));
}
```

### `@content` 内容块

混入可以接受一段 CSS 内容块：

```scss
@mixin respond-to($breakpoint) {
  @if $breakpoint == 'sm' {
    @media (max-width: 576px) { @content; }
  } @else if $breakpoint == 'md' {
    @media (max-width: 768px) { @content; }
  } @else if $breakpoint == 'lg' {
    @media (max-width: 1024px) { @content; }
  }
}

.container {
  width: 1200px;

  @include respond-to('md') {
    width: 100%;
    padding: 0 16px;
  }

  @include respond-to('sm') {
    padding: 0 8px;
  }
}
```

---

## @extend 继承

`@extend` 让一个选择器继承另一个选择器的所有样式：

```scss
%message-base {
  padding: 12px 16px;
  border-radius: 4px;
  margin-bottom: 16px;
  font-size: 14px;
}

.message-success {
  @extend %message-base;
  background: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

.message-error {
  @extend %message-base;
  background: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

.message-warning {
  @extend %message-base;
  background: #fff3cd;
  color: #856404;
}
```

`%` 开头是**占位符选择器**，不会被编译到 CSS 中，只用于被继承（推荐用这种方式代替直接继承类名）。

### @extend vs @mixin

| | `@extend` | `@mixin` |
|---|---|---|
| 编译结果 | 合并选择器 | 复制样式 |
| 传参 | 不支持 | 支持 |
| 跨媒体查询 | 不可用 | 可用 |
| 适用场景 | 共享相同样式 | 复用带参数的样式模式 |

---

## 函数

### 内置函数

**颜色函数：**

```scss
$base: #3498db;

.box {
  color: darken($base, 10%);          // 加深 10%
  background: lighten($base, 20%);    // 变亮 20%
  border-color: saturate($base, 15%); // 增加饱和度
  outline-color: desaturate($base, 20%); // 降低饱和度
  box-shadow: rgba($base, 0.3);       // 设置 alpha

  // 混合颜色（50% 混合）
  color: mix($base, #e74c3c, 50%);

  // 根据背景色自动选择黑/白文字
  color: if(lightness($base) > 50%, #000, #fff);
}
```

**数学函数：**

```scss
.element {
  width: math.div(100%, 3);    // 33.333%（推荐用 math.div 替代 /）
  height: math.round(16.7px);  // 17px
  padding: math.abs(-8px);     // 8px
  font-size: math.max(12px, 1rem); // 取最大值
}
```

**字符串函数：**

```scss
$name: "hello";

.element {
  content: to-upper-case($name);   // "HELLO"
  content: str-length($name);      // 5
  content: str-slice($name, 1, 3); // "hel"（下标从 1 开始）
}
```

**列表函数：**

```scss
$sizes: 8px 16px 24px 32px;

.element {
  padding: nth($sizes, 2);         // 16px（下标从 1 开始）
  font-size: nth($sizes, length($sizes)); // 32px（最后一个）
}
```

### 自定义函数

```scss
@use 'sass:math';

@function rem($px, $base: 16) {
  @return math.div($px, $base) * 1rem;
}

@function spacing($multiplier) {
  @return $multiplier * 8px;
}

@function z-index($layer) {
  $layers: (
    'base': 0,
    'dropdown': 100,
    'modal': 200,
    'toast': 300,
    'tooltip': 400
  );
  @return map.get($layers, $layer);
}

.card {
  font-size: rem(14px);       // 0.875rem
  padding: spacing(2);        // 16px
  margin: spacing(3);         // 24px
  z-index: z-index('modal');  // 200
}
```

---

## 控制流

### @if / @else

```scss
@mixin theme-color($theme) {
  @if $theme == 'dark' {
    background: #1a1a1a;
    color: #fff;
  } @else if $theme == 'light' {
    background: #fff;
    color: #333;
  } @else {
    @warn "未知主题：#{$theme}，使用默认值";
    background: #f5f5f5;
    color: #333;
  }
}

.dark-panel  { @include theme-color('dark'); }
.light-panel { @include theme-color('light'); }
```

### @each

遍历列表或映射：

```scss
// 遍历列表
$colors: red, green, blue, yellow;

@each $color in $colors {
  .text-#{$color} {
    color: $color;
  }
}

// 遍历映射
$icons: (
  'home': '\e001',
  'user': '\e002',
  'settings': '\e003',
);

@each $name, $code in $icons {
  .icon-#{$name}::before {
    content: $code;
  }
}
```

### @for

```scss
// @for $i from 1 through 5（包含 5）
@for $i from 1 through 5 {
  .col-#{$i} {
    width: math.div(100%, 12) * $i;
  }
}

// @for $i from 1 to 5（不包含 5）
@for $i from 1 to 5 {
  .mt-#{$i} {
    margin-top: $i * 4px;
  }
}
```

### @while

```scss
$i: 1;

@while $i <= 4 {
  .opacity-#{$i * 25} {
    opacity: math.div($i, 4);
  }
  $i: $i + 1;
}
```

---

## 映射（Map）

映射是键值对集合，相当于对象/字典：

```scss
@use 'sass:map';

$breakpoints: (
  'sm': 576px,
  'md': 768px,
  'lg': 1024px,
  'xl': 1280px,
  'xxl': 1536px,
);

$theme-colors: (
  'primary':   #3498db,
  'secondary': #95a5a6,
  'success':   #2ecc71,
  'danger':    #e74c3c,
  'warning':   #f39c12,
);

// 读取
$md: map.get($breakpoints, 'md');  // 768px

// 检查是否存在
@if map.has-key($breakpoints, 'xs') {
  // ...
}

// 合并
$extended: map.merge($breakpoints, ('xxxl': 1920px));

// 遍历生成工具类
@each $name, $color in $theme-colors {
  .bg-#{$name}   { background-color: $color; }
  .text-#{$name} { color: $color; }
  .border-#{$name} { border-color: $color; }
}

// 响应式 mixin
@mixin breakpoint($name) {
  $value: map.get($breakpoints, $name);
  @media (min-width: $value) {
    @content;
  }
}

.container {
  width: 100%;
  @include breakpoint('md') { max-width: 768px; }
  @include breakpoint('lg') { max-width: 1024px; }
}
```

---

## 模块系统

### @use（推荐）

`@use` 是现代 Sass 的模块导入方式，有命名空间，不会污染全局：

```scss
// _colors.scss
$primary: #3498db;
$danger: #e74c3c;

@mixin highlight($color) {
  background: lighten($color, 30%);
  color: darken($color, 20%);
}
```

```scss
// main.scss
@use 'colors';          // 默认命名空间为文件名
@use 'colors' as c;     // 自定义命名空间
@use 'colors' as *;     // 不使用命名空间（谨慎使用）

.button {
  color: colors.$primary;    // 通过命名空间访问
  color: c.$primary;         // 自定义命名空间
}
```

以 `_` 开头的文件（partials）是局部文件，`@use` 时可省略下划线和扩展名：

```scss
@use 'variables';    // 对应 _variables.scss
@use 'mixins';       // 对应 _mixins.scss
@use 'sass:math';    // 内置模块
@use 'sass:color';
@use 'sass:map';
@use 'sass:list';
@use 'sass:string';
```

### @forward

`@forward` 将一个模块的内容转发给上层模块，用于构建模块入口文件：

```scss
// styles/_index.scss
@forward 'variables';
@forward 'mixins';
@forward 'functions';

// 控制转发的内容
@forward 'variables' show $primary-color, $font-size-base;
@forward 'mixins' hide flex-debug;

// 添加前缀避免命名冲突
@forward 'buttons' as btn-*;
```

```scss
// 外部使用，只需引入入口文件
@use 'styles';

.element {
  color: styles.$primary-color;
  @include styles.flex-center;
}
```

### @import（旧版，不推荐）

`@import` 会将所有内容合并到全局作用域，容易造成变量名冲突，Sass 官方已标记为废弃：

```scss
// 旧写法（不推荐）
@import 'variables';
@import 'mixins';
```

---

## 插值 `#{}`

在选择器、属性名、字符串中嵌入变量或表达式：

```scss
$property: 'margin';
$side: 'top';
$theme: 'dark';

.element {
  #{$property}-#{$side}: 16px;   // margin-top: 16px
}

.theme-#{$theme} {               // .theme-dark
  background: #1a1a1a;
}

@mixin icon($name) {
  &::before {
    content: url('icons/#{$name}.svg');
    background-image: url('/assets/#{$name}.png');
  }
}
```

---

## @at-root

将样式提升到根层级，打破嵌套：

```scss
.parent {
  color: red;

  @at-root .child {
    color: blue;   // 编译为 .child，不是 .parent .child
  }

  @at-root (without: media) {
    // 跳出 @media 包裹
    color: green;
  }
}
```

常与 BEM 结合使用：

```scss
.block {
  $self: &;

  &__element {
    color: blue;
  }

  &--modifier {
    color: red;

    @at-root #{$self}__element {
      color: green;  // .block__element（在 modifier 上下文中修改 element）
    }
  }
}
```

---

## 注释

```scss
// 单行注释 —— 不会编译到 CSS

/* 多行注释 —— 会编译到 CSS */

/*!
 * 强制注释 —— 压缩模式下也会保留（常用于版权声明）
 */
```

---

## @warn / @error / @debug

```scss
@mixin deprecated-mixin() {
  @warn "此 mixin 已废弃，请改用 new-mixin()";
}

@function validate-color($color) {
  @if type-of($color) != 'color' {
    @error "参数必须是颜色值，收到：#{$color}";
  }
  @return $color;
}

$value: 42px;
@debug "调试值：#{$value}";  // 输出到控制台，不影响编译
```

---

## 项目文件组织

推荐的 7-1 架构（小项目可按需简化）：

```
styles/
├── abstracts/          # 不生成 CSS 的工具
│   ├── _variables.scss
│   ├── _mixins.scss
│   ├── _functions.scss
│   └── _index.scss
├── base/               # 全局基础样式
│   ├── _reset.scss
│   ├── _typography.scss
│   └── _index.scss
├── components/         # 独立组件
│   ├── _button.scss
│   ├── _card.scss
│   ├── _modal.scss
│   └── _index.scss
├── layout/             # 页面布局
│   ├── _header.scss
│   ├── _footer.scss
│   ├── _grid.scss
│   └── _index.scss
├── pages/              # 页面特有样式
│   ├── _home.scss
│   └── _about.scss
├── themes/             # 主题（暗色模式等）
│   └── _dark.scss
└── main.scss           # 入口文件
```

`main.scss` 入口文件：

```scss
@use 'abstracts';
@use 'base';
@use 'layout';
@use 'components';
@use 'pages/home';
```

---

## 在 Vue / React 中使用

### Vite 项目

安装 `sass` 依赖后直接使用，无需额外配置：

```shell
npm install -D sass
```

```vue
<!-- Vue SFC -->
<style lang="scss">
.container {
  $padding: 16px;
  padding: $padding;
}
</style>

<!-- scoped 样式（推荐） -->
<style lang="scss" scoped>
.button {
  &:hover { opacity: 0.8; }
}
</style>
```

### 全局注入变量/混入

在 `vite.config.ts` 中配置，每个 `.scss` 文件都自动导入：

```ts
// vite.config.ts
export default defineConfig({
  css: {
    preprocessorOptions: {
      scss: {
        additionalData: `@use "@/styles/abstracts" as *;`,
      },
    },
  },
})
```

### CSS Modules + Sass

```tsx
// React
import styles from './Button.module.scss'

function Button({ children }) {
  return <button className={styles.button}>{children}</button>
}
```

```scss
// Button.module.scss
.button {
  padding: 8px 16px;
  &:hover { opacity: 0.9; }
}
```

---

## 常用内置模块速查

| 模块 | 常用函数 |
|------|---------|
| `sass:math` | `math.div()`, `math.round()`, `math.floor()`, `math.ceil()`, `math.abs()`, `math.max()`, `math.min()`, `math.pow()`, `math.sqrt()` |
| `sass:color` | `color.adjust()`, `color.scale()`, `color.mix()`, `color.invert()`, `color.grayscale()` |
| `sass:map` | `map.get()`, `map.set()`, `map.merge()`, `map.remove()`, `map.has-key()`, `map.keys()`, `map.values()` |
| `sass:list` | `list.nth()`, `list.length()`, `list.append()`, `list.join()`, `list.index()` |
| `sass:string` | `string.length()`, `string.slice()`, `string.to-upper-case()`, `string.quote()` |
| `sass:meta` | `meta.type-of()`, `meta.inspect()`, `meta.call()`, `meta.load-css()` |
