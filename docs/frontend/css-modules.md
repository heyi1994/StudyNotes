# CSS Modules

## 什么是 CSS Modules

CSS Modules 是一种 CSS 文件的模块化方案。每个类名在编译时会被自动转换为唯一的哈希值，从根本上解决全局样式冲突问题。

```
.card  →  .card_3xH7k2  （编译后）
```

Vite 和 Create React App 原生支持，无需额外配置。

---

## 文件命名规则

文件名必须以 `.module.css` 结尾，否则不会被当作 CSS Modules 处理。

```
Team.module.css    CSS Module
Team.css           普通 CSS（全局生效）
```

---

## 基础用法

### 1. 引入

```tsx
import styles from "./Team.module.css";
```

### 2. 使用类名

```tsx
// 单个类名
<div className={styles.card}>

// 带连字符的类名（不能用点语法）
<div className={styles['card-team']}>
```

### 3. 多个类名

```tsx
// 方法一：模板字符串
<div className={`${styles.card} ${styles.active}`}>

// 方法二：数组 join
<div className={[styles.card, styles.active].join(' ')}>

// 方法三：第三方库 classnames（推荐）
import cx from 'classnames'
<div className={cx(styles.card, { [styles.active]: isActive })}>
```

---

## CSS 文件写法

```css
/* Team.module.css */

.card {
  border: 1px solid #000;
  border-radius: 10px;
}

/* 命名建议用 camelCase，这样可以用点语法访问 */
.cardTitle {
  font-size: 18px;
  font-weight: bold;
}

/* 连字符命名需要用中括号访问 styles['card-title'] */
.card-title {
  font-size: 18px;
}
```

---

## 全局样式（:global）

有时需要在 CSS Module 文件里写不被哈希的全局样式：

```css
/* 局部（默认）：会被哈希 */
.title {
  color: red;
}

/* 全局：不会被哈希，直接生效 */
:global(.title) {
  color: red;
}

/* 混合使用 */
.container :global(.third-party-class) {
  margin: 0;
}
```

---

## 样式复用（composes）

```css
/* base.module.css */
.button {
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
}

/* Team.module.css */
.primaryButton {
  composes: button from "./base.module.css"; /* 复用其他模块的类 */
  background-color: blue;
  color: white;
}

.secondaryButton {
  composes: button from "./base.module.css";
  background-color: gray;
}
```

---

## 普通 CSS vs CSS Modules 对比

| 对比项   | 普通 CSS             | CSS Modules                             |
| -------- | -------------------- | --------------------------------------- |
| 引入方式 | `import './App.css'` | `import styles from './App.module.css'` |
| 类名使用 | `className="card"`   | `className={styles.card}`               |
| 作用域   | 全局                 | 组件局部                                |
| 类名冲突 | 容易冲突             | 自动避免                                |
| 适用场景 | 全局重置、主题变量   | 组件样式                                |

---

## 常见错误

### 错误：普通方式引入 CSS Module

```tsx
import './Team.module.css'
<div className="card-team">  {/* 类名不会生效 */}
```

### 正确

```tsx
import styles from './Team.module.css'
<div className={styles['card-team']}>
```

---

### 错误：用点语法访问连字符类名

```tsx
<div className={styles.card-team}>  {/* 语法错误 */}
```

### 正确

```tsx
<div className={styles['card-team']}>
```

---

## TypeScript 类型提示

默认情况下 `styles` 的类型是 `any`，可以用 `vite-plugin-svgr` 或手动声明来获得类型提示。

简单方案：在项目 `src` 目录下创建 `declarations.d.ts`：

```ts
declare module "*.module.css" {
  const classes: { [key: string]: string };
  export default classes;
}
```
