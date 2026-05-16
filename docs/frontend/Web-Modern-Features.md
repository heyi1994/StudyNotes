# 现代 Web 平台新特性全览

## HTML 新特性 {#html}

### dialog 元素

```html
<dialog id="modal">
  <p>这是一个原生对话框</p>
  <button onclick="this.closest('dialog').close()">关闭</button>
</dialog>

<button onclick="document.getElementById('modal').showModal()">打开</button>
```

```js
const dialog = document.querySelector("dialog");
dialog.showModal(); // 模态（带 backdrop）
dialog.show(); // 非模态
dialog.close();
dialog.returnValue; // 关闭时的返回值

dialog.addEventListener("close", () => console.log(dialog.returnValue));
```

---

### popover API（2024）

```html
<!-- 无需 JS，纯 HTML 实现弹出层 -->
<button popovertarget="tip">显示提示</button>
<div id="tip" popover>
  <p>这是一个 Popover</p>
</div>

<!-- 手动控制 -->
<div id="menu" popover="manual">...</div>
```

```js
const popover = document.querySelector("[popover]");
popover.showPopover();
popover.hidePopover();
popover.togglePopover();
```

---

### details / summary

```html
<details>
  <summary>点击展开</summary>
  <p>折叠内容...</p>
</details>

<!-- 手风琴效果：name 相同的 details 同时只能开一个 -->
<details name="faq">
  <summary>问题 1</summary>
  ...
</details>
<details name="faq">
  <summary>问题 2</summary>
  ...
</details>
```

---

### loading="lazy" 懒加载

```html
<img src="photo.jpg" loading="lazy" alt="..." />
<iframe src="embed.html" loading="lazy"></iframe>
```

---

### fetchpriority

```html
<!-- 提升关键资源优先级 -->
<img src="hero.jpg" fetchpriority="high" />
<link rel="preload" href="font.woff2" as="font" fetchpriority="high" />
<script src="non-critical.js" fetchpriority="low"></script>
```

---

### inert 属性

```html
<!-- 让整个区域不可交互（不可聚焦、不可点击、对 AT 不可见） -->
<div inert>
  <button>此按钮不可用</button>
  <input type="text" />
</div>
```

---

### input 新类型和属性

```html
<input type="color" />
<input type="range" min="0" max="100" step="5" />
<input type="date" />
<input type="datetime-local" />
<input type="search" />

<!-- datalist：输入建议 -->
<input list="fruits" placeholder="选择水果" />
<datalist id="fruits">
  <option value="Apple"></option>
  <option value="Banana"></option>
  <option value="Cherry"></option>
</datalist>
```

---

### form 验证属性

```html
<form>
  <input required minlength="3" maxlength="20" pattern="[a-z]+" />
  <input type="email" />
  <input type="url" />
  <button formnovalidate>跳过验证提交</button>
</form>
```

```js
input.setCustomValidity("用户名已存在");
input.reportValidity();
form.checkValidity();
```

---

### View Transitions API（2024）

```js
// 页面切换时添加过渡动画
document.startViewTransition(() => {
  updateDOM(); // 更新 DOM
});

// CSS 控制动画
// ::view-transition-old(root) — 旧页面
// ::view-transition-new(root) — 新页面
```

```css
::view-transition-old(root) {
  animation: slide-out 300ms ease;
}
::view-transition-new(root) {
  animation: slide-in 300ms ease;
}
```

---

## CSS 新特性 {#css}

### CSS 自定义属性（变量）

```css
:root {
  --color-primary: #007bff;
  --spacing-base: 8px;
  --font-size: 16px;
}

.button {
  background: var(--color-primary);
  padding: calc(var(--spacing-base) * 2);
  /* 回退值 */
  color: var(--text-color, #333);
}
```

```js
// JS 读写 CSS 变量
getComputedStyle(el).getPropertyValue("--color-primary");
el.style.setProperty("--color-primary", "#ff0000");
```

---

### CSS Grid

```css
.container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: auto;
  gap: 16px;

  /* 命名区域 */
  grid-template-areas:
    "header header header"
    "sidebar main main"
    "footer footer footer";
}

.header {
  grid-area: header;
}
.sidebar {
  grid-area: sidebar;
}

/* 自动填充 */
grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));

/* 子项定位 */
.item {
  grid-column: 1 / 3; /* 跨 2 列 */
  grid-row: span 2; /* 跨 2 行 */
}
```

---

### CSS Flexbox 新属性

```css
.container {
  display: flex;
  flex-wrap: wrap;
  gap: 16px; /* 现代写法，替代 margin hack */
}

.item {
  flex: 1 1 200px; /* grow shrink basis */
}
```

---

### Container Queries（2023）

```css
/* 根据容器大小（而非视口）响应式布局 */
.card-wrapper {
  container-type: inline-size;
  container-name: card;
}

@container card (min-width: 400px) {
  .card {
    display: flex;
    flex-direction: row;
  }
}

/* 容器查询单位 */
.card h2 {
  font-size: clamp(1rem, 5cqi, 2rem); /* cqi = 容器行内尺寸的 1% */
}
```

---

### CSS 嵌套（2024）

```css
/* 原生支持类似 Sass 的嵌套 */
.card {
  padding: 1rem;
  background: white;

  .title {
    font-size: 1.5rem;
    color: navy;
  }

  &:hover {
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  }

  @media (max-width: 600px) {
    padding: 0.5rem;
  }
}
```

---

### :is() / :where() / :has()

```css
/* :is() — 选择器列表，权重取最高的 */
:is(h1, h2, h3) > a {
  color: blue;
}

/* :where() — 同 :is()，但权重为 0 */
:where(header, footer) a {
  color: inherit;
}

/* :has() — 父选择器，根据子元素匹配父 */
.card:has(img) {
  padding: 0;
}
label:has(+ input:required)::after {
  content: " *";
  color: red;
}
li:has(> details[open]) {
  background: #f0f8ff;
}
```

---

### CSS 逻辑属性

```css
/* 替代方向性属性，支持国际化/RTL */
.box {
  margin-inline: auto; /* margin-left + margin-right */
  padding-block: 1rem; /* padding-top + padding-bottom */
  border-inline-start: 2px solid blue; /* RTL 自动翻转 */
  inset-inline-start: 0; /* left in LTR, right in RTL */
}
```

---

### clamp() / min() / max()

```css
/* 响应式字体大小，无需媒体查询 */
h1 {
  font-size: clamp(1.5rem, 4vw, 3rem); /* 最小 最佳 最大 */
}

.container {
  width: min(100%, 1200px); /* 取较小值 */
  padding: max(1rem, 5%); /* 取较大值 */
}
```

---

### CSS 颜色函数（2023）

```css
/* oklch — 感知均匀的颜色空间 */
color: oklch(70% 0.15 220);
color: oklch(from var(--base-color) l c h); /* 相对颜色语法 */

/* color-mix */
color: color-mix(in oklch, blue 30%, red);
background: color-mix(in srgb, var(--primary) 80%, transparent);

/* display-p3 — 广色域 */
color: color(display-p3 0.5 0.9 0.3);
```

---

### 滚动相关 CSS

```css
/* 滚动捕捉 */
.scroll-container {
  scroll-snap-type: x mandatory;
  overflow-x: scroll;
}
.scroll-item {
  scroll-snap-align: start;
}

/* 滚动条样式 */
.custom-scroll {
  scrollbar-width: thin;
  scrollbar-color: #888 #f0f0f0;
}

/* 滚动驱动动画（2024） */
@keyframes fade-in {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: none;
  }
}
.hero {
  animation: fade-in linear;
  animation-timeline: scroll();
  animation-range: entry 0% entry 50%;
}
```

---

### @layer（级联层）

```css
/* 控制样式优先级，解决特异性冲突 */
@layer reset, base, components, utilities;

@layer reset {
  * {
    margin: 0;
    box-sizing: border-box;
  }
}

@layer components {
  .button {
    padding: 8px 16px;
  }
}

@layer utilities {
  .mt-4 {
    margin-top: 1rem !important;
  }
}
/* 后声明的 layer 优先级更高 */
```

---

### @property（自定义属性类型声明）

```css
/* 让 CSS 变量支持动画和类型检查 */
@property --rotation {
  syntax: "<angle>";
  inherits: false;
  initial-value: 0deg;
}

.spinner {
  --rotation: 0deg;
  transform: rotate(var(--rotation));
  transition: --rotation 1s ease;
}
.spinner:hover {
  --rotation: 360deg;
}
```

---

### text-wrap: balance / pretty

```css
/* 均衡换行，防止最后一行只剩一个词 */
h1,
h2 {
  text-wrap: balance; /* 适合标题 */
}

p {
  text-wrap: pretty; /* 适合正文，优化孤行 */
}
```

---

### CSS 子网格（subgrid）

```css
.parent {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
}

.child {
  grid-column: span 3;
  display: grid;
  grid-template-columns: subgrid; /* 继承父网格轨道 */
}
```

---

## Storage & 离线 {#storage}

### localStorage / sessionStorage

```js
localStorage.setItem("token", "abc123");
localStorage.getItem("token");
localStorage.removeItem("token");
localStorage.clear();

// sessionStorage 在标签关闭后清除
sessionStorage.setItem("draft", JSON.stringify(data));
```

---

### IndexedDB

```js
const request = indexedDB.open("MyDB", 1);

request.onupgradeneeded = (e) => {
  const db = e.target.result;
  const store = db.createObjectStore("users", { keyPath: "id" });
  store.createIndex("email", "email", { unique: true });
};

request.onsuccess = (e) => {
  const db = e.target.result;
  const tx = db.transaction("users", "readwrite");
  const store = tx.objectStore("users");
  store.add({ id: 1, name: "Alice", email: "alice@example.com" });
};

// 推荐使用封装库：idb（Jake Archibald）
import { openDB } from "idb";
const db = await openDB("MyDB", 1, {
  upgrade(db) {
    db.createObjectStore("users", { keyPath: "id" });
  },
});
await db.put("users", { id: 1, name: "Alice" });
const user = await db.get("users", 1);
```

---

### Cache API

```js
// 通常在 Service Worker 中使用
const cache = await caches.open("v1");
await cache.addAll(["/index.html", "/app.js", "/style.css"]);

// 缓存优先策略
const response = (await caches.match(request)) ?? (await fetch(request));
```

---

### File System Access API

```js
// 读取本地文件
const [fileHandle] = await window.showOpenFilePicker({
  types: [{ accept: { "text/*": [".txt", ".md"] } }],
});
const file = await fileHandle.getFile();
const text = await file.text();

// 写入本地文件
const writable = await fileHandle.createWritable();
await writable.write("Hello World");
await writable.close();

// 选择目录
const dirHandle = await window.showDirectoryPicker();
for await (const [name, handle] of dirHandle) {
  console.log(name, handle.kind);
}
```

---

### Storage Manager

```js
// 估算存储用量
const { quota, usage } = await navigator.storage.estimate();
console.log(`已用 ${((usage / quota) * 100).toFixed(1)}%`);

// 申请持久化存储（不被浏览器自动清除）
const granted = await navigator.storage.persist();
```

---

## 网络与通信 {#network}

### Fetch API

```js
const res = await fetch("/api/users", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "Alice" }),
  signal: AbortController.signal,
});

if (!res.ok) throw new Error(`HTTP ${res.status}`);

const data = await res.json();
// 其他格式：res.text(), res.blob(), res.arrayBuffer(), res.formData()

// 流式响应
const reader = res.body.getReader();
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  process(value);
}
```

---

### AbortController

```js
const controller = new AbortController();
const { signal } = controller;

setTimeout(() => controller.abort(), 5000); // 5 秒超时

try {
  const res = await fetch("/api/data", { signal });
} catch (err) {
  if (err.name === "AbortError") console.log("请求已取消");
}

// AbortSignal 工具方法
const signal = AbortSignal.timeout(5000); // 超时信号
const combined = AbortSignal.any([s1, s2]); // 任一中止即触发
```

---

### WebSocket

```js
const ws = new WebSocket("wss://example.com/socket");

ws.onopen = () => ws.send(JSON.stringify({ type: "hello" }));
ws.onmessage = (e) => console.log(JSON.parse(e.data));
ws.onerror = (e) => console.error(e);
ws.onclose = (e) => console.log("closed", e.code);

// 关闭
ws.close(1000, "Normal closure");
```

---

### Server-Sent Events (SSE)

```js
// 服务器单向推送，自动重连
const es = new EventSource("/api/stream");

es.onmessage = (e) => console.log(e.data);
es.addEventListener("update", (e) => console.log(e.data));
es.onerror = () => console.error("连接断开，将自动重连");
es.close();

// 服务端（Node.js）
res.setHeader("Content-Type", "text/event-stream");
res.write("data: Hello\n\n");
res.write('event: update\ndata: {"count":1}\n\n');
```

---

### WebRTC

```js
// 点对点音视频通信
const pc = new RTCPeerConnection({
  iceServers: [{ urls: "stun:stun.l.google.com:19302" }],
});

// 添加媒体流
const stream = await navigator.mediaDevices.getUserMedia({
  video: true,
  audio: true,
});
stream.getTracks().forEach((track) => pc.addTrack(track, stream));

// 数据通道
const channel = pc.createDataChannel("chat");
channel.send("Hello peer!");
```

---

### Beacon API

```js
// 页面卸载时可靠发送数据（不阻塞页面关闭）
window.addEventListener("visibilitychange", () => {
  if (document.visibilityState === "hidden") {
    navigator.sendBeacon("/analytics", JSON.stringify({ event: "page_exit" }));
  }
});
```

---

## 多线程与并发 {#workers}

### Web Worker

```js
// main.js
const worker = new Worker("worker.js");
worker.postMessage({ data: [1, 2, 3] });
worker.onmessage = (e) => console.log(e.data);
worker.terminate();

// Transferable — 零拷贝转移（转移后原线程不可用）
const buffer = new ArrayBuffer(1024 * 1024);
worker.postMessage(buffer, [buffer]);

// worker.js
self.onmessage = ({ data }) => {
  const result = heavyCompute(data.data);
  self.postMessage(result);
};
```

---

### Shared Worker

```js
// 多个标签页/窗口共享同一个 worker
const sw = new SharedWorker("shared.js");
sw.port.start();
sw.port.postMessage("hello");
sw.port.onmessage = (e) => console.log(e.data);

// shared.js
self.onconnect = (e) => {
  const port = e.ports[0];
  port.onmessage = ({ data }) => port.postMessage(data);
};
```

---

### Service Worker

```js
// 注册
navigator.serviceWorker.register("/sw.js", { scope: "/" });

// sw.js
self.addEventListener("install", (e) => {
  e.waitUntil(
    caches.open("v1").then((c) => c.addAll(["/index.html", "/app.js"])),
  );
});

self.addEventListener("fetch", (e) => {
  e.respondWith(
    caches.match(e.request).then((cached) => cached ?? fetch(e.request)),
  );
});

self.addEventListener("push", (e) => {
  e.waitUntil(
    self.registration.showNotification("新消息", { body: e.data.text() }),
  );
});
```

---

## 图形与媒体 {#graphics}

### Canvas 2D

```js
const canvas = document.querySelector("canvas");
const ctx = canvas.getContext("2d");

ctx.fillStyle = "#007bff";
ctx.fillRect(10, 10, 100, 50);

ctx.beginPath();
ctx.arc(100, 100, 40, 0, Math.PI * 2);
ctx.stroke();

// 图片
const img = new Image();
img.onload = () => ctx.drawImage(img, 0, 0);
img.src = "/photo.jpg";

// 导出
canvas.toBlob((blob) => saveFile(blob), "image/webp", 0.9);
```

---

### WebGL / WebGL2

```js
const gl = canvas.getContext("webgl2");
// 着色器 + 缓冲区 + 绘制调用
// 实际项目推荐使用 Three.js / Babylon.js
```

---

### WebGPU（2024）

```js
// 下一代 GPU API，支持计算着色器
const adapter = await navigator.gpu.requestAdapter();
const device = await adapter.requestDevice();

// 比 WebGL 更低层级、性能更好
// 适合 ML 推理、复杂渲染、通用计算
```

---

### OffscreenCanvas

```js
// 在 Worker 中进行 Canvas 渲染，不阻塞主线程
const offscreen = canvas.transferControlToOffscreen();
worker.postMessage({ canvas: offscreen }, [offscreen]);

// worker.js
self.onmessage = ({ data }) => {
  const ctx = data.canvas.getContext("2d");
  ctx.fillRect(0, 0, 100, 100);
};
```

---

### Media Session API

```js
// 控制系统媒体通知/锁屏控件
navigator.mediaSession.metadata = new MediaMetadata({
  title: "歌曲名称",
  artist: "艺术家",
  artwork: [{ src: "/cover.jpg", sizes: "512x512" }],
});

navigator.mediaSession.setActionHandler("play", () => audio.play());
navigator.mediaSession.setActionHandler("pause", () => audio.pause());
navigator.mediaSession.setActionHandler("nexttrack", () => playNext());
```

---

### Screen Capture API

```js
// 录制屏幕
const stream = await navigator.mediaDevices.getDisplayMedia({
  video: { displaySurface: "window" },
  audio: true,
});
const recorder = new MediaRecorder(stream);
recorder.ondataavailable = (e) => chunks.push(e.data);
recorder.start(1000);
```

---

## 性能 API {#performance}

### Performance Observer

```js
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(entry.name, entry.duration);
  }
});

observer.observe({ type: "longtask", buffered: true });
observer.observe({ type: "layout-shift", buffered: true });
observer.observe({ type: "largest-contentful-paint", buffered: true });
observer.observe({ type: "navigation", buffered: true });
observer.observe({ type: "resource", buffered: true });
```

---

### User Timing API

```js
performance.mark("start-task");
// ... 执行任务
performance.mark("end-task");
performance.measure("task-duration", "start-task", "end-task");

const [measure] = performance.getEntriesByName("task-duration");
console.log(`耗时：${measure.duration.toFixed(2)}ms`);
```

---

### Navigation Timing

```js
const [nav] = performance.getEntriesByType("navigation");
console.log("DNS:", nav.domainLookupEnd - nav.domainLookupStart);
console.log("TCP:", nav.connectEnd - nav.connectStart);
console.log("TTFB:", nav.responseStart - nav.requestStart);
console.log("DOM解析:", nav.domContentLoadedEventEnd - nav.responseEnd);
```

---

### Scheduler API（2024）

```js
// 调度任务，让出主线程
await scheduler.yield(); // 让出控制权，处理用户输入

scheduler.postTask(
  () => {
    // 低优先级后台任务
  },
  { priority: "background" },
); // user-blocking | user-visible | background
```

---

### requestIdleCallback

```js
// 浏览器空闲时执行非关键任务
requestIdleCallback((deadline) => {
  while (deadline.timeRemaining() > 0 && tasks.length > 0) {
    tasks.shift()(); // 执行一个任务
  }
  if (tasks.length > 0) requestIdleCallback(processTasks);
});
```

---

## 观察者 API {#observers}

### IntersectionObserver

```js
// 元素进入/离开视口
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        entry.target.classList.add("visible");
        observer.unobserve(entry.target);
      }
    });
  },
  {
    threshold: 0.1, // 10% 可见时触发
    rootMargin: "0px 0px -100px 0px",
  },
);

document.querySelectorAll(".lazy").forEach((el) => observer.observe(el));
```

---

### MutationObserver

```js
// 监听 DOM 变化
const observer = new MutationObserver((mutations) => {
  for (const mutation of mutations) {
    if (mutation.type === "childList") {
      mutation.addedNodes.forEach((node) => init(node));
    }
    if (mutation.type === "attributes") {
      console.log(`属性 ${mutation.attributeName} 变化`);
    }
  }
});

observer.observe(document.body, {
  childList: true,
  subtree: true,
  attributes: true,
  attributeFilter: ["class", "data-state"],
});
```

---

### ResizeObserver

```js
// 监听元素尺寸变化（非 window resize）
const observer = new ResizeObserver((entries) => {
  for (const entry of entries) {
    const { width, height } = entry.contentRect;
    updateLayout(width, height);
  }
});

observer.observe(document.querySelector(".chart"));
```

---

## 设备与硬件 {#device}

### Geolocation

```js
navigator.geolocation.getCurrentPosition(
  ({ coords }) => console.log(coords.latitude, coords.longitude),
  (err) => console.error(err),
  { enableHighAccuracy: true, timeout: 5000 },
);

const watchId = navigator.geolocation.watchPosition(handler);
navigator.geolocation.clearWatch(watchId);
```

---

### Clipboard API

```js
// 写入剪贴板
await navigator.clipboard.writeText("复制内容");
await navigator.clipboard.write([new ClipboardItem({ "image/png": blob })]);

// 读取剪贴板
const text = await navigator.clipboard.readText();
const items = await navigator.clipboard.read();
```

---

### Web Bluetooth

```js
const device = await navigator.bluetooth.requestDevice({
  filters: [{ services: ["heart_rate"] }],
});
const server = await device.gatt.connect();
const service = await server.getPrimaryService("heart_rate");
const char = await service.getCharacteristic("heart_rate_measurement");
char.startNotifications();
char.addEventListener("characteristicvaluechanged", handler);
```

---

### Vibration API

```js
navigator.vibrate(200); // 振动 200ms
navigator.vibrate([100, 50, 100]); // 振动-停-振动
navigator.vibrate(0); // 停止
```

---

### Screen Orientation

```js
screen.orientation.type; // 'portrait-primary'
await screen.orientation.lock("landscape");
screen.orientation.unlock();
screen.orientation.addEventListener("change", handler);
```

---

### Battery Status API

```js
const battery = await navigator.getBattery();
console.log(battery.level, battery.charging);
battery.addEventListener("levelchange", () => {
  if (battery.level < 0.2) showLowBatteryWarning();
});
```

---

### Device Orientation / Motion

```js
window.addEventListener("deviceorientation", ({ alpha, beta, gamma }) => {
  // alpha: 绕 Z 轴旋转 (0-360)
  // beta:  绕 X 轴旋转 (-180~180，前后倾斜)
  // gamma: 绕 Y 轴旋转 (-90~90，左右倾斜)
});

window.addEventListener("devicemotion", ({ acceleration, rotationRate }) => {
  console.log(acceleration.x, acceleration.y, acceleration.z);
});
```

---

## Web Components {#web-components}

### Custom Elements

```js
class MyCard extends HTMLElement {
  static observedAttributes = ["title", "theme"];

  constructor() {
    super();
    this.attachShadow({ mode: "open" });
  }

  connectedCallback() {
    this.render();
  }

  attributeChangedCallback(name, oldVal, newVal) {
    this.render();
  }

  render() {
    this.shadowRoot.innerHTML = `
      <style>
        :host { display: block; padding: 1rem; }
        :host([theme="dark"]) { background: #333; color: white; }
      </style>
      <h2>${this.getAttribute("title")}</h2>
      <slot></slot>
    `;
  }
}

customElements.define("my-card", MyCard);
```

```html
<my-card title="Hello" theme="dark">
  <p>内容在这里</p>
</my-card>
```

---

### Shadow DOM

```js
const shadow = element.attachShadow({ mode: "open" });

// 样式隔离
shadow.innerHTML = `
  <style>p { color: red; }</style>  <!-- 不影响外部 -->
  <p>Shadow DOM 内容</p>
`;

// CSS 穿透
// ::slotted(p) — 选择插槽内的元素
// :host — 选择宿主元素
// :host-context(.dark) — 宿主在 .dark 祖先下
```

---

### HTML Templates

```html
<template id="card-tpl">
  <div class="card">
    <h2 class="title"></h2>
    <slot name="content"></slot>
  </div>
</template>
```

```js
const tpl = document.getElementById("card-tpl");
const clone = tpl.content.cloneNode(true);
clone.querySelector(".title").textContent = "Hello";
element.appendChild(clone);
```

---

## 安全与权限 {#security}

### Permissions API

```js
const result = await navigator.permissions.query({ name: "camera" });
// result.state: 'granted' | 'denied' | 'prompt'

result.addEventListener("change", () => {
  console.log("权限状态变化:", result.state);
});
```

---

### Content Security Policy (CSP)

```html
<meta
  http-equiv="Content-Security-Policy"
  content="default-src 'self'; script-src 'self' 'nonce-abc123'; img-src *"
/>
```

```js
// 随机 nonce（服务端生成）
<script nonce="abc123">/* 允许执行 */</script>
```

---

### SubtleCrypto API

```js
// 生成密钥
const key = await crypto.subtle.generateKey(
  { name: "AES-GCM", length: 256 },
  true,
  ["encrypt", "decrypt"],
);

// 加密
const iv = crypto.getRandomValues(new Uint8Array(12));
const encrypted = await crypto.subtle.encrypt(
  { name: "AES-GCM", iv },
  key,
  new TextEncoder().encode("secret message"),
);

// 哈希
const hash = await crypto.subtle.digest("SHA-256", data);

// 随机数
crypto.randomUUID(); // UUID v4
crypto.getRandomValues(new Uint8Array(16)); // 随机字节
```

---

### Sanitizer API（实验性）

```js
// 安全地插入 HTML，防止 XSS
const sanitizer = new Sanitizer();
element.setHTML("<b>粗体</b><script>alert(1)</script>", { sanitizer });
// script 标签被移除
```

---

## PWA（Progressive Web App）{#pwa}

### Web App Manifest

```json
{
  "name": "My App",
  "short_name": "App",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#007bff",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ],
  "screenshots": [...],
  "shortcuts": [
    { "name": "新建", "url": "/new", "icons": [...] }
  ]
}
```

---

### 安装提示

```js
let deferredPrompt;
window.addEventListener("beforeinstallprompt", (e) => {
  e.preventDefault();
  deferredPrompt = e;
  showInstallButton();
});

installButton.addEventListener("click", async () => {
  deferredPrompt.prompt();
  const { outcome } = await deferredPrompt.userChoice;
  console.log(outcome); // 'accepted' | 'dismissed'
});
```

---

### Push Notifications

```js
// 订阅推送
const sub = await registration.pushManager.subscribe({
  userVisibleOnly: true,
  applicationServerKey: urlBase64ToUint8Array(VAPID_PUBLIC_KEY),
});

// 将 sub 发送到服务器
await fetch("/api/push/subscribe", {
  method: "POST",
  body: JSON.stringify(sub),
});

// Service Worker 接收推送
self.addEventListener("push", (e) => {
  const { title, body } = e.data.json();
  e.waitUntil(
    self.registration.showNotification(title, {
      body,
      icon: "/icon.png",
      badge: "/badge.png",
      actions: [{ action: "open", title: "查看" }],
    }),
  );
});
```

---

### Background Sync

```js
// 网络恢复后自动重试
navigator.serviceWorker.ready.then((reg) => {
  reg.sync.register("send-message");
});

// sw.js
self.addEventListener("sync", (e) => {
  if (e.tag === "send-message") {
    e.waitUntil(sendPendingMessages());
  }
});
```

---

## 速查表

| 特性                 | 类别     | 要点                  |
| -------------------- | -------- | --------------------- |
| `dialog`             | HTML     | 原生模态/非模态对话框 |
| `popover`            | HTML     | 纯 HTML 弹出层        |
| `loading="lazy"`     | HTML     | 图片/iframe 懒加载    |
| `inert`              | HTML     | 禁用整个区域交互      |
| View Transitions     | HTML API | 页面切换动画          |
| CSS 变量             | CSS      | `--var` 全局主题      |
| CSS Grid             | CSS      | 二维布局              |
| Container Queries    | CSS      | 基于容器响应式        |
| CSS 嵌套             | CSS      | 原生 Sass 式嵌套      |
| `:has()`             | CSS      | 父选择器              |
| `clamp()`            | CSS      | 流体尺寸              |
| `@layer`             | CSS      | 级联层控制            |
| `oklch`              | CSS      | 现代颜色空间          |
| `text-wrap: balance` | CSS      | 均衡换行              |
| 滚动驱动动画         | CSS      | 无 JS 滚动联动        |
| Fetch API            | 网络     | 现代请求              |
| AbortController      | 网络     | 取消请求              |
| WebSocket            | 网络     | 双向实时通信          |
| SSE                  | 网络     | 服务器推送            |
| Beacon               | 网络     | 页面卸载发数据        |
| Web Worker           | 并发     | 后台线程              |
| Service Worker       | 并发     | 离线/缓存/推送        |
| IndexedDB            | 存储     | 客户端数据库          |
| Cache API            | 存储     | 请求级缓存            |
| File System Access   | 存储     | 读写本地文件          |
| IntersectionObserver | 观察者   | 视口进入检测          |
| MutationObserver     | 观察者   | DOM 变化监听          |
| ResizeObserver       | 观察者   | 元素尺寸变化          |
| Clipboard API        | 设备     | 读写剪贴板            |
| SubtleCrypto         | 安全     | 加密/哈希             |
| Custom Elements      | 组件     | 自定义 HTML 元素      |
| Shadow DOM           | 组件     | 样式/结构隔离         |
| Push Notifications   | PWA      | 推送通知              |
| Background Sync      | PWA      | 离线数据同步          |
