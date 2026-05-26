# Webhook

## 1. 什么是 Webhook

### 1.1 概念定义

Webhook 是一种基于 HTTP 的**回调机制**，也常被称为"反向 API"（Reverse API）或"HTTP Push"。

传统 API 的调用方向是：**你 → 服务**（主动查询）
Webhook 的调用方向是：**服务 → 你**（事件推送）

当第三方平台（如 GitLab、GitHub、支付宝）发生某个事件时，它会**主动向你预先配置的 URL 发送一个 HTTP POST 请求**，请求体中包含该事件的详细数据。

### 1.2 生活类比

想象你在网购等快递：

| 方式          | 描述                               | 对应技术  |
| ------------- | ---------------------------------- | --------- |
| **主动轮询**  | 每隔 10 分钟打电话问快递员"到了吗" | Polling   |
| **Webhook**   | 快递员送到门口时主动给你打电话     | Webhook   |
| **WebSocket** | 和快递员建立一条专线，随时通话     | WebSocket |

### 1.3 核心特征

- **事件驱动**：只在事件发生时触发，不浪费资源
- **单向推送**：发送方推送数据，接收方被动接收
- **无状态**：每次 HTTP 请求相互独立
- **近实时**：事件发生后通常在毫秒到秒级内触达

---

## 2. Webhook 与轮询的对比

### 2.1 轮询（Polling）模式

```
客户端                    服务端
  |                         |
  |──── GET /status ───────>|
  |<─── { status: "..." } ──|   ← 无更新，资源浪费
  |                         |
  | [等待 60 秒]             |
  |                         |
  |──── GET /status ───────>|
  |<─── { status: "..." } ──|   ← 还是无更新
  |                         |
  | [等待 60 秒]             |
  |                         |
  |──── GET /status ───────>|
  |<─── { status: "ok"  } ──|   ← 终于有更新！但已延迟 60 秒
```

**缺点：**

- 浪费带宽和服务器资源
- 存在固有延迟（轮询间隔）
- 高并发下服务端压力大

### 2.2 Webhook 模式

```
客户端                    服务端（GitLab）
  |                         |
  |  [预先注册 Webhook URL]  |
  |                         |
  |         [事件发生！]      |
  |                         |
  |<─── POST /webhook ──────|   ← 立即推送，无延迟
  |─── HTTP 200 ───────────>|   ← 立即确认收到
```

**优点：**

- 近实时，延迟极低
- 服务端只在有事件时发送请求
- 接收端不需要持续轮询

### 2.3 综合对比

| 维度       | Polling          | Webhook        | WebSocket      |
| ---------- | ---------------- | -------------- | -------------- |
| 实时性     | 低（取决于间隔） | 高             | 最高           |
| 服务端压力 | 高               | 低             | 中（维持连接） |
| 客户端实现 | 简单             | 需要公网服务器 | 复杂           |
| 适用场景   | 简单查询         | 事件通知       | 双向实时通信   |
| 防火墙穿透 | 容易             | 需要入站端口   | 需要入站端口   |

---

## 3. Webhook 工作原理

### 3.1 完整生命周期

```
┌─────────────────────────────────────────────────────────────┐
│                     Webhook 完整流程                         │
└─────────────────────────────────────────────────────────────┘

Step 1: 注册
  开发者 ──[配置 Webhook URL + Secret]──> GitLab 服务器
                                           ↓
                                    GitLab 保存配置

Step 2: 事件触发
  开发者 A ──[提交代码 / 创建 MR]──> GitLab
                                      ↓
                                  GitLab 检测到事件
                                      ↓
                              构建 Webhook 请求体（JSON）
                                      ↓
                              附加 Secret 到请求头

Step 3: HTTP 推送
  GitLab ──[POST /webhook/gitlab]──> 你的服务器
              Headers:
                X-Gitlab-Event: Push Hook
                X-Gitlab-Token: your-secret
              Body:
                { "object_kind": "push", ... }

Step 4: 处理与响应
  你的服务器:
    1. 验证 X-Gitlab-Token
    2. 解析请求体
    3. 立即返回 HTTP 200（≤10 秒内）
    4. 异步执行业务逻辑

Step 5: 重试机制（若失败）
  GitLab 检测到非 2xx 响应 or 超时
    ↓
  按间隔重试（通常 1min/5min/10min...）
```

### 3.2 请求体结构（以 GitLab Push 为例）

```json
{
  "object_kind": "push",
  "event_name": "push",
  "before": "95790bf891e76fee5726fa501f52d1a0d468a4b3",
  "after": "da1560886d4f094c3e6c9ef40349f7d38b5d27d7",
  "ref": "refs/heads/main",
  "checkout_sha": "da1560886d4f094c3e6c9ef40349f7d38b5d27d7",
  "user_id": 4,
  "user_name": "张三",
  "user_username": "zhangsan",
  "project_id": 15,
  "project": {
    "id": 15,
    "name": "my-project",
    "web_url": "https://gitlab.example.com/my-project",
    "path_with_namespace": "my-group/my-project"
  },
  "commits": [
    {
      "id": "da1560886d4f094c3e6c9ef40349f7d38b5d27d7",
      "message": "fix: 修复登录 bug\n\nCloses #42",
      "title": "fix: 修复登录 bug",
      "timestamp": "2024-01-15T10:30:00+08:00",
      "url": "https://gitlab.example.com/...",
      "author": { "name": "张三", "email": "zhangsan@example.com" },
      "added": ["src/auth/login.ts"],
      "modified": ["src/auth/token.ts"],
      "removed": []
    }
  ],
  "total_commits_count": 1
}
```

---

## 4. HTTP 请求详解

### 4.1 请求头（Headers）

GitLab 发送的 Webhook 请求包含以下关键请求头：

```http
POST /webhook/gitlab HTTP/1.1
Host: your-server.com
Content-Type: application/json
X-Gitlab-Event: Push Hook          ← 事件类型
X-Gitlab-Token: your-secret-token  ← 安全令牌
X-Gitlab-Instance: https://gitlab.example.com
X-Gitlab-Webhook-UUID: 12345678-...
User-Agent: GitLab/16.0.0
Content-Length: 2048
```

| Header                  | 含义                                           |
| ----------------------- | ---------------------------------------------- |
| `X-Gitlab-Event`        | 事件类型，如 `Push Hook`、`Merge Request Hook` |
| `X-Gitlab-Token`        | 你配置的 Secret Token，用于验证来源            |
| `X-Gitlab-Instance`     | 发送请求的 GitLab 实例地址                     |
| `X-Gitlab-Webhook-UUID` | 本次请求的唯一 ID，用于幂等处理                |

### 4.2 所有 GitLab 事件类型

| `X-Gitlab-Event` 值  | 触发场景                        |
| -------------------- | ------------------------------- |
| `Push Hook`          | 代码 Push                       |
| `Tag Push Hook`      | Tag 创建/删除                   |
| `Issue Hook`         | Issue 创建/更新/关闭            |
| `Merge Request Hook` | MR 创建/更新/合并/关闭          |
| `Note Hook`          | 评论（Issue/MR/Commit/Snippet） |
| `Pipeline Hook`      | Pipeline 状态变化               |
| `Job Hook`           | 单个 Job 状态变化               |
| `Wiki Page Hook`     | Wiki 页面创建/更新              |
| `Release Hook`       | Release 创建/更新/删除          |
| `Member Hook`        | 成员加入/离开项目               |
| `Deployment Hook`    | 部署事件                        |

### 4.3 响应要求

```
接收方必须在 10 秒内返回 2xx 响应，否则 GitLab 视为失败并重试。

  正确做法：立即响应，异步处理
  收到请求 → 验证 Secret → 返回 200 → 放入队列/异步处理业务

  错误做法：同步处理所有逻辑再响应
  收到请求 → 调用 API → 发送通知 → 写数据库 → 返回 200
                ↑ 可能超过 10 秒，导致 GitLab 重试
```

**推荐响应格式：**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"received": true}
```

---

## 5. Webhook 安全机制

### 5.1 为什么需要安全验证

不验证来源的 Webhook 存在严重风险：

```
攻击者
  |
  |──[伪造 POST /webhook/gitlab]──> 你的服务器
  |   Body: { "action": "merge", ... 假数据 }
  |                                    |
  |                             触发自动部署 ← 危险！
```

### 5.2 Secret Token 验证

最基础也是最常用的验证方式。

**GitLab 配置侧：**

```
项目 Settings → Webhooks → Secret token: my-super-secret-token
```

**服务端验证代码：**

```typescript
function verifyGitLabSecret(req: Request, res: Response, next: NextFunction) {
  const token = req.headers["x-gitlab-token"];
  const expected = process.env.WEBHOOK_SECRET;

  if (!expected) {
    // 未配置 Secret 则跳过验证（仅开发环境）
    return next();
  }

  if (!token || token !== expected) {
    console.warn("[security] Webhook token mismatch, rejecting request");
    return res.status(401).json({ error: "Unauthorized" });
  }

  next();
}
```

> **注意**：GitLab 使用的是**明文比较**（`token === expected`），而非 HMAC 签名。
> GitHub 使用 HMAC-SHA256 签名，安全性更高（详见下方对比章节）。

### 5.3 HMAC 签名验证（GitHub 风格，更安全）

若你自己开发的系统需要给第三方提供 Webhook，推荐使用 HMAC 签名：

```typescript
import crypto from "crypto";

// 发送方（你的系统）生成签名
function generateSignature(payload: string, secret: string): string {
  return (
    "sha256=" +
    crypto.createHmac("sha256", secret).update(payload, "utf8").digest("hex")
  );
}

// 接收方验证签名
function verifySignature(
  payload: string,
  signature: string,
  secret: string,
): boolean {
  const expected = generateSignature(payload, secret);
  // 使用 timingSafeEqual 防止时序攻击
  return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
}

// Express 中间件
function verifyHmac(req: Request, res: Response, next: NextFunction) {
  const signature = req.headers["x-hub-signature-256"] as string;
  const rawBody = JSON.stringify(req.body); // 注意：需要原始 body

  if (!verifySignature(rawBody, signature, process.env.WEBHOOK_SECRET!)) {
    return res.status(403).json({ error: "Invalid signature" });
  }

  next();
}
```

**签名验证流程：**

```
发送方:
  payload = JSON.stringify(event)
  signature = HMAC-SHA256(payload, secret)
  Headers["X-Hub-Signature-256"] = "sha256=" + signature

接收方:
  expected = HMAC-SHA256(rawBody, secret)
  actual   = Headers["X-Hub-Signature-256"]
  安全 = timingSafeEqual(expected, actual)
```

> **为什么用 `timingSafeEqual` 而不是 `===`？**
> 普通字符串比较在字符不匹配时提前返回，攻击者可通过测量响应时间逐字节猜测签名（时序攻击）。`timingSafeEqual` 始终比较完整长度，耗时固定，消除此风险。

### 5.4 IP 白名单

GitLab.com 的 Webhook 出站 IP 范围是固定的，可以在防火墙层面过滤：

```nginx
# Nginx 示例：只允许 GitLab.com IP 段
location /webhook/gitlab {
    allow 35.231.145.151;    # GitLab.com IP（示例，需查官方文档获取最新）
    allow 34.74.90.64;
    deny all;
    proxy_pass http://localhost:3000;
}
```

自托管 GitLab 则配置你自己 GitLab 服务器的 IP。

### 5.5 HTTPS 强制要求

```
 http://your-server.com/webhook    （明文传输，Secret 可被截获）
 https://your-server.com/webhook   （TLS 加密，安全）
```

生产环境**必须**使用 HTTPS。

---

## 6. GitLab Webhook 详解

### 6.1 配置步骤

```
GitLab 项目主页
  └── Settings（左侧菜单）
        └── Webhooks
              ├── URL: https://your-server.com/webhook/gitlab
              ├── Secret token: xxxxxxxx
              ├── Trigger（勾选需要监听的事件）:
              │     ☑ Push events
              │     ☑ Tag push events
              │     ☑ Issues events
              │     ☑ Merge requests events
              │     ☑ Pipeline events
              │     ☑ Job events
              │     □ Comments
              │     □ Confidential issues events
              ├── SSL verification: ☑ Enable
              └── [Add webhook]
```

### 6.2 各事件 Payload 详解

#### Push Hook

```json
{
  "object_kind": "push",
  "ref": "refs/heads/feature/login", // 分支
  "before": "abc123", // push 前的 commit SHA
  "after": "def456", // push 后的 commit SHA
  "commits": [
    // 本次 push 的所有提交
    {
      "id": "def456...",
      "message": "feat: 新增登录功能",
      "added": ["src/login.ts"], // 新增文件
      "modified": ["package.json"], // 修改文件
      "removed": [] // 删除文件
    }
  ],
  "total_commits_count": 1
}
```

**常见用途：**

- 触发 CI/CD 流程
- 通知团队有新代码提交
- 检查提交信息格式是否符合规范

#### Merge Request Hook

```json
{
  "object_kind": "merge_request",
  "object_attributes": {
    "iid": 42, // MR 编号（项目内唯一）
    "title": "feat: 新增用户权限模块",
    "state": "opened", // opened/closed/merged/locked
    "action": "open", // open/close/reopen/update/merge/approved
    "source_branch": "feature/auth",
    "target_branch": "main",
    "draft": false, // 是否为草稿 MR
    "url": "https://gitlab.example.com/..."
  },
  "user": { "username": "zhangsan" }, // 触发操作的用户
  "assignees": [], // 指派人
  "reviewers": [] // 审者
}
```

**`action` 字段完整值：**

| action       | 触发时机               |
| ------------ | ---------------------- |
| `open`       | MR 首次创建            |
| `close`      | MR 被关闭              |
| `reopen`     | MR 被重新打开          |
| `update`     | MR 标题/描述等信息更新 |
| `approved`   | MR 被审者批准          |
| `unapproved` | 批准被撤销             |
| `approval`   | 审批状态变更           |
| `merge`      | MR 被合并              |

#### Issue Hook

```json
{
  "object_kind": "issue",
  "object_attributes": {
    "iid": 15,
    "title": "登录页面样式错乱",
    "description": "在 Safari 浏览器下...",
    "state": "opened",
    "action": "open", // open/close/reopen/update
    "labels": [{ "title": "bug", "color": "#d9534f" }]
  },
  "user": { "username": "lisi" }
}
```

#### Pipeline Hook

```json
{
  "object_kind": "pipeline",
  "object_attributes": {
    "id": 1234,
    "ref": "main",
    "status": "failed", // pending/running/success/failed/canceled
    "duration": 120, // 秒
    "url": "https://gitlab.example.com/..."
  },
  "builds": [
    {
      "name": "unit-test",
      "stage": "test",
      "status": "failed",
      "duration": 45
    }
  ],
  "merge_request": {
    // 若 Pipeline 关联了 MR
    "iid": 42,
    "title": "feat: ..."
  }
}
```

### 6.3 在 GitLab UI 中测试 Webhook

GitLab 提供了测试功能，无需真实触发事件：

```
Settings → Webhooks → [找到已创建的 Webhook] → Test → 选择事件类型
  ↓
GitLab 发送一个模拟请求到你配置的 URL
  ↓
页面显示响应状态码和响应体
```

同时可以查看历史请求记录：

```
Settings → Webhooks → Edit → Recent Deliveries
  ├── 请求时间
  ├── 响应状态码
  ├── 请求 Headers
  ├── 请求 Body
  └── 响应 Body
```

---

## 7. 服务端实现

### 7.1 基础结构（Node.js / Express）

```typescript
import express from "express";
import "dotenv/config";

const app = express();
app.use(express.json());

app.post("/webhook/gitlab", (req, res) => {
  const event = req.headers["x-gitlab-event"];
  const token = req.headers["x-gitlab-token"];

  // Step 1: 验证 Secret
  if (token !== process.env.WEBHOOK_SECRET) {
    return res.status(401).json({ error: "Unauthorized" });
  }

  // Step 2: 立即响应（必须在 10 秒内）
  res.status(200).json({ received: true });

  // Step 3: 异步处理业务逻辑
  setImmediate(() => {
    handleEvent(event as string, req.body).catch(console.error);
  });
});

async function handleEvent(eventType: string, payload: unknown) {
  switch (eventType) {
    case "Push Hook":
      // 处理 Push 事件...
      break;
    case "Merge Request Hook":
      // 处理 MR 事件...
      break;
    default:
      console.log(`未处理的事件类型: ${eventType}`);
  }
}

app.listen(3000);
```

### 7.2 幂等处理（防重复消费）

GitLab 在响应超时或失败时会重试，导致同一事件被多次推送。

```typescript
// 使用 X-Gitlab-Webhook-UUID 做幂等控制
const processedEvents = new Set<string>();

app.post("/webhook/gitlab", (req, res) => {
  const webhookId = req.headers["x-gitlab-webhook-uuid"] as string;

  if (webhookId && processedEvents.has(webhookId)) {
    console.log(`[idempotent] 重复请求，跳过: ${webhookId}`);
    return res.status(200).json({ received: true, duplicate: true });
  }

  res.status(200).json({ received: true });

  setImmediate(async () => {
    try {
      await handleEvent(req.headers["x-gitlab-event"] as string, req.body);
      if (webhookId) {
        processedEvents.add(webhookId);
        // 10 分钟后清理，防止内存无限增长
        setTimeout(() => processedEvents.delete(webhookId), 10 * 60 * 1000);
      }
    } catch (err) {
      console.error("处理失败:", err);
    }
  });
});
```

> **生产环境**建议使用 Redis 替代内存 Set，跨进程共享状态。

### 7.3 消息队列解耦（推荐生产方案）

```
GitLab
  |
  |──[POST /webhook]──> Webhook 接收服务
                              |
                        立即返回 200
                              |
                       推入消息队列（Redis/RabbitMQ）
                              |
                    ┌─────────┴─────────┐
                    ↓                   ↓
              Worker 1              Worker 2
           (处理 MR 事件)         (发送通知)
```

**使用 Bull 队列（Redis）示例：**

```typescript
import Queue from "bull";

const webhookQueue = new Queue("gitlab-webhooks", {
  redis: { host: "localhost", port: 6379 },
});

// 接收端：快速入队
app.post("/webhook/gitlab", (req, res) => {
  res.status(200).json({ received: true });
  webhookQueue.add({
    event: req.headers["x-gitlab-event"],
    payload: req.body,
    receivedAt: Date.now(),
  });
});

// 消费端：异步处理
webhookQueue.process(async (job) => {
  const { event, payload } = job.data;
  await handleEvent(event, payload);
});
```

### 7.4 本地开发调试（内网穿透）

本地开发时没有公网地址，使用内网穿透工具让 GitLab 能访问你的本地服务：

#### 方案一：ngrok（推荐）

```bash
# 安装
brew install ngrok

# 启动本地服务
npm run dev  # 监听 3000 端口

# 开启内网穿透
ngrok http 3000

# 输出示例：
# Forwarding  https://abc123.ngrok.io -> http://localhost:3000
```

将 `https://abc123.ngrok.io/webhook/gitlab` 填入 GitLab Webhook 配置。

#### 方案二：Cloudflare Tunnel（免费，稳定）

```bash
# 安装 cloudflared
brew install cloudflare/cloudflare/cloudflared

# 临时隧道（无需账号）
cloudflared tunnel --url http://localhost:3000

# 输出：
# https://random-name.trycloudflare.com
```

#### 方案三：localtunnel

```bash
npx localtunnel --port 3000
# 输出：your url is: https://xxxxx.loca.lt
```

---

## 8. 常见问题与调试

### 8.1 GitLab 报告 Webhook 失败

**排查步骤：**

```
1. 查看 GitLab Webhook 历史
   Settings → Webhooks → Edit → Recent Deliveries
   检查：响应码、响应体、耗时

2. 检查服务器日志
   是否收到请求？
   是否有报错？

3. 手动测试 Webhook 端点
   curl -X POST https://your-server.com/webhook/gitlab \
     -H "Content-Type: application/json" \
     -H "X-Gitlab-Token: your-secret" \
     -H "X-Gitlab-Event: Push Hook" \
     -d '{"object_kind":"push","ref":"refs/heads/main",...}'
```

### 8.2 常见错误及解决方案

| 错误        | 原因                | 解决方案                                |
| ----------- | ------------------- | --------------------------------------- |
| `HTTP 401`  | Secret Token 不匹配 | 检查 `.env` 与 GitLab 配置是否一致      |
| `HTTP 404`  | 路由不存在          | 检查 URL 路径是否正确                   |
| `HTTP 500`  | 服务端报错          | 查看服务器日志，检查代码异常            |
| `Timeout`   | 处理超时 > 10s      | 改为立即返回 200，异步处理              |
| `SSL Error` | 证书问题            | 检查 HTTPS 证书是否有效                 |
| 事件未触发  | 未勾选对应事件类型  | GitLab → Settings → Webhooks → 勾选事件 |

### 8.3 调试技巧

#### 记录所有入站请求

```typescript
app.use("/webhook", (req, _res, next) => {
  console.log("[webhook] 收到请求");
  console.log("  Event:", req.headers["x-gitlab-event"]);
  console.log("  UUID:", req.headers["x-gitlab-webhook-uuid"]);
  console.log("  Body:", JSON.stringify(req.body, null, 2));
  next();
});
```

#### 使用 RequestBin 捕获请求

[Webhook.site](https://webhook.site) 提供免费的临时 URL，可以实时查看所有入站 Webhook 请求的完整内容，非常适合开发阶段分析 payload 结构。

#### 本地重放历史请求

```bash
# 从 GitLab Recent Deliveries 复制请求体，保存为 payload.json
curl -X POST http://localhost:3000/webhook/gitlab \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Token: your-secret" \
  -H "X-Gitlab-Event: Merge Request Hook" \
  -d @payload.json
```

### 8.4 处理 GitLab 重试

当你的服务临时不可用时，GitLab 会重试。确保业务逻辑是**幂等的**：

```typescript
//  非幂等：每次收到事件都新增评论
await client.addMRComment(mrIid, "自动审查中...");

//  幂等：先检查是否已有相同评论
async function addCommentIfNotExists(
  mrIid: number,
  marker: string,
  body: string,
) {
  const notes = await client.getMRNotes(mrIid);
  if (notes.some((n: { body: string }) => n.body.includes(marker))) {
    return; // 已存在，跳过
  }
  await client.addMRComment(mrIid, body);
}
```

---

## 9. 生产环境最佳实践

### 9.1 架构设计

```
                         ┌──────────────────────┐
                         │    负载均衡 / Nginx    │
                         │  (TLS 终止 + 限流)     │
                         └──────────┬───────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ↓               ↓               ↓
             ┌─────────┐    ┌─────────┐    ┌─────────┐
             │ Bot实例1 │    │ Bot实例2 │    │ Bot实例3 │
             └────┬────┘    └────┬────┘    └────┬────┘
                  │              │              │
                  └──────────────┼──────────────┘
                                 │
                         ┌───────┴──────┐
                         │  Redis 队列   │
                         │  + 幂等去重   │
                         └───────┬──────┘
                                 │
                    ┌────────────┼────────────┐
                    ↓            ↓            ↓
             ┌──────────┐ ┌──────────┐ ┌──────────┐
             │ Worker:  │ │ Worker:  │ │ Worker:  │
             │ 标签/审者 │ │  通知    │ │  评论    │
             └──────────┘ └──────────┘ └──────────┘
```

### 9.2 配置清单

```bash
# .env 生产配置
PORT=3000
WEBHOOK_SECRET=<32位以上随机字符串>    # openssl rand -hex 32

GITLAB_URL=https://gitlab.your-company.com
GITLAB_TOKEN=<最小权限的 Project Access Token>

# 通知渠道（按需配置）
DINGTALK_WEBHOOK_URL=https://oapi.dingtalk.com/robot/send?access_token=xxx
DINGTALK_SECRET=<钉钉机器人加签密钥>
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/xxx

# Redis（生产环境幂等控制）
REDIS_URL=redis://localhost:6379
```

### 9.3 限流保护

防止 Webhook 风暴（短时间大量事件）：

```typescript
import rateLimit from "express-rate-limit";

const webhookLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 分钟窗口
  max: 200, // 最多 200 个请求
  message: { error: "Too many requests" },
  standardHeaders: true,
  legacyHeaders: false,
});

app.use("/webhook", webhookLimiter);
```

### 9.4 健康检查与监控

```typescript
// 健康检查端点（供 Kubernetes / 负载均衡探活）
app.get("/health", (_req, res) => {
  res.json({
    status: "ok",
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
    queue: {
      pending: webhookQueue.count(),
      failed: webhookQueue.failedCount(),
    },
  });
});

// 关键指标（Prometheus 格式）
app.get("/metrics", (_req, res) => {
  res.set("Content-Type", "text/plain");
  res.send(
    [
      `webhook_received_total ${totalReceived}`,
      `webhook_processed_total ${totalProcessed}`,
      `webhook_failed_total ${totalFailed}`,
    ].join("\n"),
  );
});
```

### 9.5 GitLab Access Token 权限最小化

为机器人创建专用 Token，只授予必需权限：

```
GitLab → 项目 → Settings → Access Tokens → Add new token

名称: gitlab-bot
角色: Developer（或按需选择）
权限范围（Scopes）:
  ☑ api          （调用 REST API）
  □ read_user    （读取用户信息，按需）
  □ write_repository（按需）
过期时间: 设置合理的过期时间，定期轮换
```

### 9.6 日志规范

```typescript
// 结构化日志，便于日志平台（ELK/Loki）查询
function log(level: "info" | "warn" | "error", event: string, data?: unknown) {
  console.log(
    JSON.stringify({
      timestamp: new Date().toISOString(),
      level,
      event,
      ...(data ? { data } : {}),
    }),
  );
}

// 使用示例
log("info", "webhook.received", { eventType: "Merge Request Hook", mrIid: 42 });
log("error", "gitlab.api.failed", { error: err.message, mrIid: 42 });
```

---

## 10. 与其他平台的对比

### 10.1 GitLab vs GitHub Webhook 差异

| 特性         | GitLab                   | GitHub                                                    |
| ------------ | ------------------------ | --------------------------------------------------------- |
| 签名 Header  | `X-Gitlab-Token`（明文） | `X-Hub-Signature-256`（HMAC-SHA256）                      |
| 签名方式     | 明文对比                 | HMAC 签名，更安全                                         |
| 事件 Header  | `X-Gitlab-Event`         | `X-GitHub-Event`                                          |
| 超时时间     | 10 秒                    | 10 秒                                                     |
| 重试次数     | 3 次                     | ~3 次                                                     |
| 测试功能     | UI 内置测试              | UI 内置测试 + Redeliver                                   |
| Payload 格式 | JSON                     | JSON                                                      |
| 内容类型     | `application/json`       | `application/json` 或 `application/x-www-form-urlencoded` |

### 10.2 GitHub 签名验证实现（对比参考）

```typescript
// GitHub 使用 HMAC-SHA256 签名
function verifyGitHubSignature(req: Request): boolean {
  const signature = req.headers["x-hub-signature-256"] as string;
  const secret = process.env.WEBHOOK_SECRET!;

  // 注意：必须使用原始 Buffer，不能用 JSON.stringify
  const hmac = crypto.createHmac("sha256", secret);
  hmac.update(req.body); // req.body 需是原始 Buffer
  const expected = "sha256=" + hmac.digest("hex");

  return crypto.timingSafeEqual(
    Buffer.from(signature ?? ""),
    Buffer.from(expected),
  );
}
```

### 10.3 钉钉 / 飞书 Webhook 作为「发送端」

机器人向外部发送通知时（钉钉/飞书机器人），方向相反——你是发送方：

```typescript
// 发送钉钉 Markdown 消息
async function sendDingTalk(title: string, content: string) {
  await axios.post(process.env.DINGTALK_WEBHOOK_URL!, {
    msgtype: "markdown",
    markdown: { title, text: content },
  });
}

// 发送飞书卡片消息
async function sendFeishu(title: string, content: string) {
  await axios.post(process.env.FEISHU_WEBHOOK_URL!, {
    msg_type: "interactive",
    card: {
      header: { title: { tag: "plain_text", content: title } },
      elements: [{ tag: "div", text: { tag: "lark_md", content } }],
    },
  });
}
```

---

## 附录：快速参考卡

### Webhook 接收端检查清单

```
✅ 基础功能
□ HTTPS 端点（非 HTTP）
□ 验证 X-Gitlab-Token
□ 10 秒内返回 200
□ 异步处理业务逻辑

✅ 可靠性
□ 幂等处理（防重复消费）
□ 记录 Webhook UUID
□ 业务逻辑有 try/catch

✅ 安全性
□ Secret 存放在环境变量，不写死在代码里
□ 接口有限流保护
□ 最小权限的 GitLab Token

✅ 可观测性
□ 结构化日志
□ /health 健康检查端点
□ 错误告警
```

### 常用调试命令

```bash
# 本地测试 Webhook 端点
curl -X POST http://localhost:3000/webhook/gitlab \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Token: your-secret" \
  -H "X-Gitlab-Event: Push Hook" \
  -d '{"object_kind":"push","ref":"refs/heads/main","user_name":"test","project":{"id":1,"name":"test","web_url":"http://example.com"},"commits":[],"total_commits_count":0}'

# 查看服务器日志
tail -f /var/log/gitlab-bot/app.log | jq .

# 启动内网穿透（开发调试）
ngrok http 3000
```

### GitLab Webhook 事件 → action 速查

```
Merge Request Hook
  open      MR 创建
  update    MR 信息更新（标题/描述等）
  close     MR 关闭
  reopen    MR 重新打开
  merge     MR 合并
  approved  MR 被批准
  unapproved 批准被撤销

Issue Hook
  open      Issue 创建
  update    Issue 更新
  close     Issue 关闭
  reopen    Issue 重新打开

Pipeline Hook（通过 status 字段区分）
  pending   等待执行
  running   执行中
  success   执行成功
  failed    执行失败
  canceled  已取消
  skipped   已跳过
```
