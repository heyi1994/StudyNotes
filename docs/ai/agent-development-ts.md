# Agent 开发

> 语言：TypeScript
> 大模型：Claude

## 1. 核心心智模型

### 1.1 LLM vs Agent

|            | 普通 LLM 调用     | Agent                             |
| ---------- | ----------------- | --------------------------------- |
| 形态       | 问一句,答一句     | 给目标 + 工具,自主循环直到完成    |
| 编程类比   | 一次纯函数调用    | 围绕这个函数写的 `while` 循环     |
| 谁执行动作 | 没有动作,只有文本 | **你的代码**执行工具,把结果喂回去 |

一句话:**Agent = LLM + 工具 + 一个循环。**

### 1.2 Agent Loop(ReAct 循环)——一切的核心

```
┌─────────────────────────────────────────────┐
│  messages = [system, user_task]              │
│                                              │
│  loop:                                        │
│    response = llm(messages, tools)   ← 思考   │
│    if response 想调用工具:                     │
│       result = 你的代码执行工具       ← 行动   │
│       把 response 和 result 追加进 messages   │
│       continue                       ← 观察   │
│    else:                                      │
│       return response.text           ← 完成   │
└─────────────────────────────────────────────┘
```

关键认知(新手最容易误解的地方):

1. **模型从不自己执行工具**。它只输出「我想调用 `get_weather(city='北京')`」这个*意图*。真正执行的是你的代码。
2. **API 是无状态的**。Claude 不记得上一次对话,每次请求你都要把**完整历史**(`messages` 数组)发过去。
3. 循环的**终止条件**是:模型不再请求工具(`stop_reason === "end_turn"`)。

---

## 2. 环境准备 & 第一个 LLM 调用

### 2.1 安装

```bash
npm install @anthropic-ai/sdk
# 设置环境变量(不要把 key 硬编码进代码)
export ANTHROPIC_API_KEY="sk-ant-..."
```

### 2.2 最简单的一次调用

```typescript
import Anthropic from "@anthropic-ai/sdk";

// 默认从环境变量 ANTHROPIC_API_KEY 读取凭证
const client = new Anthropic();

const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  messages: [{ role: "user", content: "用一句话解释什么是 agent" }],
});

// response.content 是一个 ContentBlock[](区分联合类型)
// 必须先用 .type 收窄,才能访问 .text
for (const block of response.content) {
  if (block.type === "text") {
    console.log(block.text);
  }
}
```

要点:

- `max_tokens` 是**输出**上限。非流式建议默认 `16000`,太小会把输出截断。
- `content` 不是字符串,是**内容块数组**。文本、思考、工具调用都是不同类型的块,要用 `block.type` 判断。

### 2.3 System Prompt(定义 agent 的人设与规则)

```typescript
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  system: "你是一个严谨的代码助手,回答时总是给出 TypeScript 示例。",
  messages: [{ role: "user", content: "如何读取一个 JSON 文件?" }],
});
```

`system` 是「全局规则」,`messages` 是「对话」。agent 的「性格、能力边界、工作流程」都写在 system 里。

---

## 3. 工具调用(Tool Use)——agent 的「手」

LLM 本身不能联网、读文件、算数。**工具**让它能「动手」。

### 3.1 工具的定义结构

你给模型一份「工具说明书」:名字 + 描述 + 参数 schema(JSON Schema)。

```typescript
import Anthropic from "@anthropic-ai/sdk";

const tools: Anthropic.Tool[] = [
  {
    name: "get_weather",
    description: "查询某城市的当前天气。当用户询问天气、气温、是否下雨时调用。",
    input_schema: {
      type: "object",
      properties: {
        city: { type: "string", description: "城市名,例如:北京" },
        unit: {
          type: "string",
          enum: ["celsius", "fahrenheit"],
          description: "温度单位",
        },
      },
      required: ["city"],
    },
  },
];
```

> 💡 **描述决定一切**。`description` 不仅要说「这个工具做什么」,更要说「**什么时候**调用它」。Claude 靠它判断该不该用、用哪个。写得含糊,模型就会乱用或不用。

### 3.2 一次工具调用的完整往返

```typescript
const messages: Anthropic.MessageParam[] = [
  { role: "user", content: "北京今天冷吗?" },
];

// 第 1 步:模型决定调用工具
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  tools,
  messages,
});

console.log(response.stop_reason); // "tool_use" —— 模型想调工具

// 第 2 步:取出工具调用意图
for (const block of response.content) {
  if (block.type === "tool_use") {
    console.log(block.name); // "get_weather"
    console.log(block.input); // { city: "北京" } —— 已经是解析好的对象
  }
}
```

模型返回 `stop_reason: "tool_use"`,`content` 里有一个 `tool_use` 块。**接下来轮到你的代码执行,然后把结果塞回去** —— 这正是下一章的 agent loop。

---

## 4. 手写 Agent Loop(最重要的一章)

不要一上来就用框架。亲手实现一遍,你会彻底理解 agent 的本质。

### 4.1 完整可运行示例

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

// ---- 1. 定义工具的「说明书」(给模型看) ----
const tools: Anthropic.Tool[] = [
  {
    name: "get_weather",
    description: "查询某城市的当前天气。用户问天气/气温时调用。",
    input_schema: {
      type: "object",
      properties: { city: { type: "string", description: "城市名" } },
      required: ["city"],
    },
  },
  {
    name: "calculator",
    description: "计算一个数学表达式。用户需要算数时调用。",
    input_schema: {
      type: "object",
      properties: {
        expression: { type: "string", description: "如 '2 + 3 * 4'" },
      },
      required: ["expression"],
    },
  },
];

// ---- 2. 工具的「真正实现」(你的代码执行) ----
async function executeTool(name: string, input: any): Promise<string> {
  switch (name) {
    case "get_weather":
      // 真实场景这里会调天气 API,这里写死演示
      return `${input.city} 当前 -2°C,晴。`;
    case "calculator":
      // 演示用,生产环境别用 eval
      return String(eval(input.expression));
    default:
      return `未知工具: ${name}`;
  }
}

// ---- 3. Agent Loop ----
async function runAgent(userInput: string): Promise<string> {
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: userInput },
  ];

  while (true) {
    const response = await client.messages.create({
      model: "claude-opus-4-8",
      max_tokens: 16000,
      tools,
      messages,
    });

    // 情况 A:模型给出了最终答案,没有工具要调 → 结束
    if (response.stop_reason === "end_turn") {
      const textBlock = response.content.find((b) => b.type === "text");
      return textBlock && textBlock.type === "text" ? textBlock.text : "";
    }

    // 情况 B:模型想调用工具
    if (response.stop_reason === "tool_use") {
      // 把模型这一轮的回复(含 tool_use 块)原样追加进历史 —— 必须保留!
      messages.push({ role: "assistant", content: response.content });

      // 执行所有被请求的工具(模型可能一次请求多个)
      const toolResults: Anthropic.ToolResultBlockParam[] = [];
      for (const block of response.content) {
        if (block.type === "tool_use") {
          const result = await executeTool(block.name, block.input);
          toolResults.push({
            type: "tool_result",
            tool_use_id: block.id, // 必须对应上,否则 API 报错
            content: result,
          });
        }
      }

      // 把工具结果作为一条 user 消息追加,然后回到循环顶部让模型继续
      messages.push({ role: "user", content: toolResults });
      continue;
    }

    // 其他 stop_reason(refusal / max_tokens 等)
    throw new Error(`未处理的 stop_reason: ${response.stop_reason}`);
  }
}

// ---- 4. 跑起来 ----
console.log(await runAgent("北京今天冷吗?另外帮我算 23 * 17 等于多少"));
```

### 4.2 逐行讲解关键点

| 代码                                                              | 为什么                                                                                              |
| ----------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `messages.push({ role: "assistant", content: response.content })` | **必须把模型的完整回复(含 tool_use 块)追加回历史**。否则下一轮模型不知道自己请求过工具,会逻辑混乱。 |
| `tool_use_id: block.id`                                           | 工具结果必须用 `tool_use_id` 跟请求一一对应。模型一次可能调多个工具,靠 id 配对。                    |
| `for (const block of ...)` 遍历执行                               | 模型**一次可以请求多个工具**(并行)。要全部执行完,把结果合并成**一条** user 消息发回。               |
| `while(true)` + `stop_reason` 判断                                | 这就是 ReAct 循环。终止靠 `end_turn`。                                                              |

### 4.3 消息历史长什么样(跑完后)

```
[
  { role: "user",      content: "北京今天冷吗?另外帮我算 23*17" },
  { role: "assistant", content: [ {type:"tool_use", name:"get_weather"...},
                                  {type:"tool_use", name:"calculator"...} ] },
  { role: "user",      content: [ {type:"tool_result", ...},   // 天气结果
                                  {type:"tool_result", ...} ] },// 计算结果
  { role: "assistant", content: [ {type:"text", text:"北京今天 -2°C 比较冷..."} ] },
]
```

看懂这个数组的演变,你就真正理解 agent 了。

---

## 5. 用 SDK 的 Tool Runner 简化

第 4 章的循环很有教育意义,但生产里没必要每次手写。SDK 的 **Tool Runner**(beta)帮你自动跑这个循环。

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { betaZodTool } from "@anthropic-ai/sdk/helpers/beta/zod";
import { z } from "zod";

const client = new Anthropic();

// 用 Zod 定义工具:schema 自动生成,run 是真正的实现
const getWeather = betaZodTool({
  name: "get_weather",
  description: "查询某城市的当前天气。用户问天气/气温时调用。",
  inputSchema: z.object({
    city: z.string().describe("城市名,例如:北京"),
  }),
  run: async ({ city }) => `${city} 当前 -2°C,晴。`,
});

// toolRunner 自动处理整个 agent loop,直接返回最终消息
const finalMessage = await client.beta.messages.toolRunner({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  tools: [getWeather],
  messages: [{ role: "user", content: "北京今天冷吗?" }],
});

console.log(finalMessage.content);
```

Tool Runner 的好处:

- **不用手写循环**:自动调用工具、喂回结果、再循环,直到模型不再调工具。
- **类型安全**:Zod schema 既生成给模型的 JSON Schema,又给 `run` 的入参自动推导类型。

**什么时候还用手写循环?** 需要细粒度控制时:自定义日志、条件性执行某工具、**人工审批**(执行危险工具前要用户确认)、按工具做埋点等。

---

## 6. 记忆(Memory)与上下文管理

Agent 跑久了,`messages` 会越来越长,直到撑爆上下文窗口。Agent 的「记忆」其实分好几层,从单次会话内到跨会话的长期记忆:

### 6.1 短期记忆 = 对话历史本身

就是那个 `messages` 数组。它就是 agent 的「工作内存(RAM)」。每次请求都带上,模型才有连续性。

```typescript
const messages: Anthropic.MessageParam[] = [
  { role: "user", content: "我叫 Alice。" },
  { role: "assistant", content: "你好 Alice!" },
  { role: "user", content: "我叫什么?" }, // 模型能答出 Alice,因为历史在
];
```

规则:第一条必须是 `user`;API 无状态,**每次都要发完整历史**。

### 6.2 上下文太长怎么办?——Compaction(自动压缩)

当对话接近上下文上限,可以开启服务端**压缩**:API 自动把早期历史总结成一个紧凑的块。

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();
const messages: Anthropic.Beta.BetaMessageParam[] = [];

async function chat(userMessage: string): Promise<string> {
  messages.push({ role: "user", content: userMessage });

  const response = await client.beta.messages.create({
    betas: ["compact-2026-01-12"],
    model: "claude-opus-4-8",
    max_tokens: 16000,
    messages,
    context_management: {
      edits: [{ type: "compact_20260112" }],
    },
  });

  //  关键:把完整 content 追加回去(不是只追加文本!)
  // compaction 块必须保留,API 靠它在下次请求替换被压缩的历史
  messages.push({ role: "assistant", content: response.content });

  const textBlock = response.content.find(
    (b): b is Anthropic.Beta.BetaTextBlock => b.type === "text",
  );
  return textBlock?.text ?? "";
}
```

> ⚠️ **最常见的错误**:只把 `response` 的文本字符串追加回 `messages`。这会丢掉 compaction 状态。永远追加 `response.content`(整个数组)。

### 6.3 跨会话记忆

上面的都是「一次会话内」。如果要让 agent **跨会话**记住东西(比如用户偏好),需要把信息存到外部(数据库、文件),下次会话开始时再读出来拼进 system prompt 或第一条消息。Claude 也提供了 `memory` 工具(client-side,你实现存储后端)用于让模型自己读写一个记忆目录。

6.3 这种「全量读出来塞进 prompt」只适合**少量固定信息**(几条用户偏好)。一旦记忆越积越多——成百上千条事实、历史对话摘要——就塞不下了。这时需要**按相关性检索**:只取出和当前对话相关的那几条记忆。这就是**向量数据库**的用武之地。

### 6.4 长期记忆:用向量数据库做「语义记忆」

机制和 [第 7 章 RAG](#7-rag检索增强生成) **一模一样**,只是被检索的不再是文档,而是「记忆条目」:

```
【写入】产生一条值得记住的事实/对话摘要 ──embedding──▶ 存入向量数据库
【读取】新一轮对话进来 ──把当前语境 embedding──▶ 检索 top-k 相关记忆
        ──拼进 system prompt──▶ Claude 带着「回忆」回答
```

> 向量化的基础(`embed()`、余弦相似度 `cosineSim()`)在第 7 章讲,这里直接复用。

#### 一个语义记忆类

先用内存版理解原理,生产环境把存储换成真正的向量数据库即可(接口形状几乎不变):

```typescript
// 复用第 7 章的 embed() 与 cosineSim()
import { randomUUID } from "crypto";

type MemoryItem = {
  id: string;
  text: string;
  vector: number[];
  userId: string;
  createdAt: number;
};

class SemanticMemory {
  private items: MemoryItem[] = []; // 内存版;生产换成向量数据库

  // 写入一条记忆
  async remember(userId: string, text: string): Promise<void> {
    const [vector] = await embed([text]);
    this.items.push({
      id: randomUUID(),
      text,
      vector,
      userId,
      createdAt: Date.now(),
    });
    // 生产(Qdrant 示例):
    // await qdrant.upsert("memories", {
    //   points: [{ id, vector, payload: { text, userId, createdAt } }],
    // });
  }

  // 检索与当前语境最相关的 k 条记忆(按 userId 隔离不同用户)
  async recall(userId: string, query: string, k = 5): Promise<string[]> {
    const [q] = await embed([query]);
    return this.items
      .filter((m) => m.userId === userId) // 多用户隔离:只回忆这个用户的
      .map((m) => ({ text: m.text, score: cosineSim(q, m.vector) }))
      .sort((a, b) => b.score - a.score)
      .slice(0, k)
      .map((m) => m.text);
    // 生产(Qdrant 示例):
    // const hits = await qdrant.search("memories", {
    //   vector: q,
    //   limit: k,
    //   filter: { must: [{ key: "userId", match: { value: userId } }] },
    // });
    // return hits.map((h) => h.payload.text as string);
  }
}
```

#### 接进 agent

```typescript
const memory = new SemanticMemory();

async function chatWithMemory(
  userId: string,
  userInput: string,
): Promise<string> {
  // 1. 回忆:检索与当前输入相关的记忆
  const memories = await memory.recall(userId, userInput);
  const memoryBlock = memories.length
    ? `\n\n【关于该用户的已知记忆】\n- ${memories.join("\n- ")}`
    : "";

  // 2. 把记忆注入 system prompt(放固定内容之后,利于缓存)
  const response = await client.messages.create({
    model: "claude-opus-4-8",
    max_tokens: 16000,
    system: `你是一个贴心的助手,会利用已知记忆个性化回答。${memoryBlock}`,
    messages: [{ role: "user", content: userInput }],
  });

  const text = response.content.find((b) => b.type === "text");
  const answer = text && text.type === "text" ? text.text : "";

  // 3. 记住这轮里值得长期保留的信息(下面讲「该记什么」)
  await rememberIfWorthwhile(userId, userInput);

  return answer;
}
```

#### 该往记忆里写什么?(关键)

**别把每句话都存进去**,否则记忆库全是噪声,检索质量崩坏。两种常见策略:

1. **提炼后再存**:让一个 LLM 调用先判断「这轮有没有值得长期记住的事实」,只存提炼出来的结构化事实。

```typescript
async function rememberIfWorthwhile(
  userId: string,
  userInput: string,
): Promise<void> {
  const res = await client.messages.create({
    model: "claude-opus-4-8",
    max_tokens: 512,
    system:
      "从下面这句用户输入中,提取值得长期记住的【稳定事实】" +
      "(如偏好、身份、长期目标)。每条一行;如果没有,只输出 NONE。",
    messages: [{ role: "user", content: userInput }],
  });
  const text = res.content.find((b) => b.type === "text");
  const facts = text && text.type === "text" ? text.text.trim() : "";
  if (facts && facts !== "NONE") {
    for (const fact of facts.split("\n").filter(Boolean)) {
      await memory.remember(userId, fact);
    }
  }
}
```

2. **让模型自己存**:把 `save_memory` 做成一个**工具**(第 3 章),由模型在对话中自主决定何时记。这就是 Agentic 版的记忆。

#### 生产要点

- **向量库选型**(同第 7 章):`pgvector`(已有 PostgreSQL 就用它)、`Qdrant`/`Milvus`(自托管专用库)、`Pinecone`/`Weaviate`(托管)。接口都是 `upsert(向量)` + `search(向量, top_k, filter)`,把上面内存版的 `filter/map/sort` 换成数据库查询即可。
- **元数据**:存 `userId` 做多用户隔离,存 `createdAt` 可支持「时间衰减」(越新的记忆权重越高)。
- **去重/更新**:同一事实反复出现时,先检索是否已有相似记忆,有就更新而非新增。

#### 记忆分层小结

| 类型           | 存在哪                  | 解决什么                   | 小节 |
| -------------- | ----------------------- | -------------------------- | ---- |
| 短期记忆       | `messages` 数组         | 单次会话内的连续性         | 6.1  |
| 上下文压缩     | 服务端(compaction)      | 单次会话太长、超窗         | 6.2  |
| 跨会话(全量)   | 外部存储,全部读出       | **少量**固定信息(几条偏好) | 6.3  |
| 语义记忆(向量) | 向量数据库,**按需检索** | **海量**记忆,按相关性取用  | 6.4  |

> **语义记忆 vs RAG**:机制相同(都是 embedding + 向量检索),用途不同。RAG 检索的是**静态知识库**(文档、手册);语义记忆检索的是**动态积累**的、与具体用户/会话相关的经历。很多 agent 两者都用。

---

## 7. RAG(检索增强生成)

RAG = **R**etrieval **A**ugmented **G**eneration。让模型回答时,先去你的知识库里**检索**相关内容,再基于这些内容**生成**答案。

### 7.1 为什么需要 RAG

模型有两个天生限制:

1. **不知道你的私有数据**(公司内部文档、用户手册、最新代码库)。
2. **知识有截止日期**,也记不住海量细节。

最朴素的想法是「把所有文档都塞进 prompt」——但文档动辄几百万字,既超上下文窗口,又贵得离谱。RAG 的思路是:**只把和当前问题最相关的几段塞进去**。

> 类比:**开卷考试**。不靠死记硬背(模型参数),而是先翻到相关那几页(检索),再作答(生成)。

### 7.2 两个阶段

```
【索引阶段 · 离线,只做一次】
  文档 ──切块──▶ 文本块 ──embedding──▶ 向量 ──存入──▶ 向量库

【查询阶段 · 在线,每次提问】
  问题 ──embedding──▶ 查询向量 ──相似度检索 top-k──▶ 最相关的几块
       ──拼进 prompt──▶ Claude 生成答案
```

核心概念:**embedding(向量化)**。把一段文本变成一个数字向量(如 1024 维),语义相近的文本,向量在空间中也相近。检索就是「找和问题向量最接近的文本块」。

### 7.3 关于 Embedding(重要)

> **Claude / Anthropic 不提供 embedding 端点。** Claude 负责「生成」那一步;「向量化」需要第三方。Anthropic 官方推荐 **Voyage AI**,你也可以换成任意 embedding 服务或本地模型(如 `bge`、`e5` 系列)。

下面示例用 Voyage 的 REST API(用 `fetch`,不依赖额外 SDK,方便你替换):

```typescript
// 把一批文本转成向量。换 provider 时只改这一个函数即可。
async function embed(texts: string[]): Promise<number[][]> {
  const res = await fetch("https://api.voyageai.com/v1/embeddings", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.VOYAGE_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      input: texts,
      model: "voyage-3", // 通用文本 embedding 模型
      input_type: "document", // 索引文档用 "document",查询用 "query"
    }),
  });
  const json = await res.json();
  return json.data.map((d: { embedding: number[] }) => d.embedding);
}
```

### 7.4 最小可运行实现(内存版,无需向量数据库)

学习阶段不用上 Pinecone/pgvector,先用一个数组 + 余弦相似度理解原理。

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

// ---- 1. 切块:把长文档切成小段(这里按字数粗切,带重叠) ----
function chunkText(text: string, size = 500, overlap = 100): string[] {
  const chunks: string[] = [];
  for (let i = 0; i < text.length; i += size - overlap) {
    chunks.push(text.slice(i, i + size));
  }
  return chunks;
}

// ---- 2. 余弦相似度:衡量两个向量有多「像」 ----
function cosineSim(a: number[], b: number[]): number {
  let dot = 0,
    normA = 0,
    normB = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }
  return dot / (Math.sqrt(normA) * Math.sqrt(normB));
}

// ---- 3. 索引阶段:文档 → 块 → 向量,存进内存 ----
type Indexed = { text: string; vector: number[] };

async function buildIndex(documents: string[]): Promise<Indexed[]> {
  const chunks = documents.flatMap((doc) => chunkText(doc));
  const vectors = await embed(chunks);
  return chunks.map((text, i) => ({ text, vector: vectors[i] }));
}

// ---- 4. 检索阶段:找出与问题最相关的 top-k 块 ----
async function retrieve(
  index: Indexed[],
  query: string,
  k = 3,
): Promise<string[]> {
  const [queryVec] = await embed([query]);
  return index
    .map((item) => ({
      text: item.text,
      score: cosineSim(queryVec, item.vector),
    }))
    .sort((a, b) => b.score - a.score)
    .slice(0, k)
    .map((item) => item.text);
}

// ---- 5. 生成阶段:把检索到的内容拼进 prompt,交给 Claude ----
async function ragAnswer(index: Indexed[], question: string): Promise<string> {
  const context = await retrieve(index, question);

  const response = await client.messages.create({
    model: "claude-opus-4-8",
    max_tokens: 16000,
    system:
      "你是一个问答助手。只根据下面提供的【参考资料】回答问题;" +
      "如果资料里没有相关信息,就明确说『资料中没有提到』,不要编造。",
    messages: [
      {
        role: "user",
        content: `【参考资料】\n${context.join("\n---\n")}\n\n【问题】\n${question}`,
      },
    ],
  });

  const text = response.content.find((b) => b.type === "text");
  return text && text.type === "text" ? text.text : "";
}

// ---- 跑起来 ----
const docs = [
  "我们公司的退款政策:商品签收后 7 天内可无理由退款,生鲜类除外。",
  "客服工作时间为周一至周五 9:00-18:00。",
];
const index = await buildIndex(docs);
console.log(await ragAnswer(index, "买的水果可以退吗?"));
// 模型会基于「生鲜类除外」回答:不可以
```

注意 system prompt 里那句「只根据参考资料回答,没有就说没有」——这是 RAG 防止模型**幻觉**的关键约束。

### 7.5 进阶:Agentic RAG —— 把检索做成「工具」

上面是**固定流程**:每次都先检索再回答。但很多时候模型其实不需要检索(比如闲聊),或者需要**多次检索、改写问题**。

**Agentic RAG** 把「检索」包装成一个工具,让 agent **自己决定**:要不要搜、搜几次、用什么关键词。这就直接接回了[第 4 章的 loop](#4-手写-agent-loop最重要的一章) / [第 5 章的 Tool Runner](#5-用-sdk-的-tool-runner-简化)。

```typescript
import { betaZodTool } from "@anthropic-ai/sdk/helpers/beta/zod";
import { z } from "zod";

// 把检索包装成一个工具
const searchKnowledgeBase = betaZodTool({
  name: "search_knowledge_base",
  description:
    "在公司内部知识库中检索资料。当用户的问题涉及公司政策、产品细节、" +
    "或任何你不确定的内部信息时调用。可以多次调用、用不同关键词。",
  inputSchema: z.object({
    query: z.string().describe("检索关键词,应聚焦、具体"),
  }),
  run: async ({ query }) => {
    const results = await retrieve(index, query); // 复用 7.4 的 retrieve
    return results.join("\n---\n");
  },
});

const finalMessage = await client.beta.messages.toolRunner({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  tools: [searchKnowledgeBase],
  messages: [{ role: "user", content: "买的水果可以退吗?另外客服几点上班?" }],
});
// 模型会自主调用 search_knowledge_base(可能调两次:退款政策 + 客服时间)
console.log(finalMessage.content);
```

**固定 RAG vs Agentic RAG 怎么选?**

- 简单、单轮问答 → 固定流程(7.4),更可控、更省 token。
- 复杂、可能多步、需要模型自主判断要不要查 → Agentic RAG(7.5),更灵活。

### 7.6 提升检索质量的关键旋钮

| 旋钮                  | 作用                                     | 经验                               |
| --------------------- | ---------------------------------------- | ---------------------------------- |
| **chunk 大小 / 重叠** | 太大→检索不精准且费 token;太小→丢上下文  | 常见 300~800 字,重叠 10%~20%       |
| **top-k**             | 检索几块                                 | 一般 3~5;太多会引入噪声            |
| **元数据过滤**        | 先按 标签/时间/部门 缩小范围再做向量检索 | 大幅提升精度                       |
| **Rerank(重排)**      | 用专门的重排模型对召回结果二次排序       | 召回多→重排选优,质量明显提升       |
| **混合检索**          | 向量检索 + 关键词检索(BM25)结合          | 对专有名词、代码、ID 类查询更鲁棒  |
| **引用来源**          | 让答案标注出处,可追溯、可信              | Claude 有内置 citations 能力可考虑 |

### 7.7 从「内存版」到生产

7.4 的数组适合学习,几千块以上就要用**向量数据库**做持久化和高效检索:

- **pgvector**:在 PostgreSQL 里加向量列,适合已有 PG 的团队。
- **Qdrant / Milvus**:专用向量库,自托管。
- **Pinecone / Weaviate**:托管服务,省运维。

接口模式都一样:`upsert(向量)` + `query(向量, top_k, filter)`,你只需把 7.4 里的 `index.map(...).sort(...)` 换成数据库查询。

### 7.8 RAG 常见坑

| 坑                                     | 后果           | 正确做法                                       |
| -------------------------------------- | -------------- | ---------------------------------------------- |
| 以为 Claude 有 embedding 接口          | 找不到 API     | 用第三方(Voyage 等)做向量化,Claude 只做生成    |
| 索引用 `document`、查询也用 `document` | 检索不准       | 区分 `input_type`:文档用 document,问题用 query |
| 不约束模型「只根据资料回答」           | 幻觉、编造     | system prompt 明确约束 + 没有就说没有          |
| chunk 切太大/太小                      | 召回差         | 调 chunk size 与 overlap                       |
| 检索结果不加分隔直接拼接               | 模型分不清来源 | 用 `---` 等分隔,最好带来源标注                 |
| 把整个知识库塞进 prompt 而不检索       | 超窗、烧钱     | 这正是 RAG 要解决的——只塞 top-k                |

---

## 8. 流式输出(Streaming)

聊天 UI 需要「逐字蹦出来」的效果,而不是等全部生成完。任何可能产生**长输出**的请求都建议流式(还能避免 HTTP 超时)。

```typescript
const stream = client.messages.stream({
  model: "claude-opus-4-8",
  max_tokens: 64000, // 流式可以给大一点
  messages: [{ role: "user", content: "写一篇关于 agent 的短文" }],
});

for await (const event of stream) {
  if (
    event.type === "content_block_delta" &&
    event.delta.type === "text_delta"
  ) {
    process.stdout.write(event.delta.text); // 逐块打印
  }
}

// 需要完整消息时(拿 usage、stop_reason 等)
const finalMessage = await stream.finalMessage();
console.log("\n用了", finalMessage.usage.output_tokens, "个输出 token");
```

要点:

- 流式时 `max_tokens` 默认给 `64000`(不用担心超时,给模型留足空间)。
- 想要完整结果用 `stream.finalMessage()`,**别**自己用 `new Promise` 包 `.on()` 事件,SDK 已处理好所有完成/错误/中断状态。

---

## 9. 思考与 Effort

Claude Opus 4.8 支持**自适应思考(adaptive thinking)**:模型自己决定要不要思考、思考多深,并在工具调用之间自动穿插思考。

```typescript
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  thinking: { type: "adaptive" }, // 开启自适应思考
  output_config: { effort: "high" }, // 努力程度:low | medium | high | xhigh | max
  messages: [{ role: "user", content: "一步步解这道数学题..." }],
});

for (const block of response.content) {
  if (block.type === "thinking") console.log("思考:", block.thinking);
  else if (block.type === "text") console.log("回答:", block.text);
}
```

- **`thinking: { type: "adaptive" }`**:推荐设置。注意——**不写 `thinking` 字段就是不思考**,要显式开启。
- **`effort`**:控制思考深度和总体 token 花费。
  - `high`:大多数智能敏感任务的推荐起点。
  - `xhigh`:编码 / agentic 场景的最佳选择(Claude Code 的默认)。
  - `medium` / `low`:对成本/延迟敏感、任务简单时。
  - `max`:正确性远比成本重要时。
- Opus 4.8 **不支持** `temperature` / `top_p` / `top_k`(传了会 400 报错),也不支持旧的 `budget_tokens`。用 prompt 引导行为,用 `effort` 控制深度。

---

## 10. 多 Agent 协作

当任务太复杂,一个 agent 扛不住时,拆成多个专职 agent。最常见的是**协调者(orchestrator)+ 工人(workers)**模式:

```
            ┌──────────────┐
   用户 ───▶│ Orchestrator │  拆解任务、分发、汇总
            └──────┬───────┘
         ┌─────────┼─────────┐
         ▼         ▼         ▼
     搜索 agent  写码 agent  审查 agent
```

在「自己掌控循环」的 Claude API 里,多 agent 本质就是:**主循环里,把『调用某个子 agent』也实现成一个工具**。子 agent 是另一次独立的 `messages.create()`(可以用不同 system prompt、甚至更便宜的模型)。

```typescript
// 概念示例:把「子 agent」包装成主 agent 的一个工具
async function executeTool(name: string, input: any): Promise<string> {
  if (name === "research_subagent") {
    // 子 agent 就是另一个独立的对话/循环
    const sub = await client.messages.create({
      model: "claude-opus-4-8",
      max_tokens: 16000,
      system: "你是专职资料搜集 agent,只负责检索并总结。",
      messages: [{ role: "user", content: input.topic }],
    });
    const text = sub.content.find((b) => b.type === "text");
    return text && text.type === "text" ? text.text : "";
  }
  // ...其他工具
  return "";
}
```

设计原则:每个子 agent **单一职责**,就像把一个巨型函数重构成多个高内聚的模块。子 agent 之间不共享上下文,需要的信息要由协调者显式传递。

> 进阶:如果不想自己管编排和沙箱,Anthropic 还有 **Managed Agents**(服务端托管 agent 循环 + 容器),支持原生的 `multiagent` 协调者配置。新手阶段先掌握上面的「自己掌控循环」即可。

---

## 11. MCP 与智能体通信协议

前面我们的工具都是**手写**进每个 agent 里的。问题来了:如果你有 10 个 agent,都想用「查 GitHub」「读数据库」「发 Slack」这些能力,难道每个 agent 都把工具实现抄一遍?**协议**就是来解决这个的。

### 11.1 为什么需要协议:M×N → M+N

```
没有协议:M 个应用 × N 个数据源 = M×N 套定制集成(噩梦)

  Claude ─┬─ GitHub                每加一个应用 / 每加一个数据源,
  Cursor ─┼─ 数据库                 都要重写一堆胶水代码。
  你的App ─┴─ Slack

有了 MCP:M 个应用 + N 个 MCP Server = M+N(各做各的)

  Claude ─┐         ┌─ GitHub MCP Server
  Cursor ─┼─ MCP ───┼─ 数据库 MCP Server      写一次 Server,
  你的App ─┘         └─ Slack MCP Server       所有支持 MCP 的应用都能用。
```

**MCP(Model Context Protocol)** 是 Anthropic 开源的开放标准,用来标准化「LLM 应用 ↔ 外部工具/数据」的连接。常见比喻:**「AI 世界的 USB-C」**——统一接口,即插即用。

### 11.2 架构:Host / Client / Server

| 角色               | 是什么                                      | 例子                                      |
| ------------------ | ------------------------------------------- | ----------------------------------------- |
| **Host(宿主)**     | 用户实际使用的 AI 应用                      | Claude Desktop、Claude Code、你写的 agent |
| **Client(客户端)** | Host 内部,与一个 Server 建立 1:1 连接的组件 | SDK 里的 `Client`                         |
| **Server(服务端)** | 对外暴露能力(工具/数据)的独立进程或服务     | GitHub MCP Server、你自己写的 Server      |

**两种传输方式(transport):**

- **stdio**:Server 是本地子进程,通过标准输入输出通信。适合本地工具(读本地文件、运行本地命令)。
- **Streamable HTTP / SSE**:Server 是远程 HTTP 服务。适合云端服务、团队共享。

### 11.3 MCP Server 能暴露三种能力

| 能力                    | 含义                             | 类比                                |
| ----------------------- | -------------------------------- | ----------------------------------- |
| **Tools(工具)**         | 可执行的动作(模型决定调用)       | 函数调用 —— 和第 3 章的工具一个意思 |
| **Resources(资源)**     | 可读取的数据(类似只读文件/API)   | GET 接口 / 文件                     |
| **Prompts(提示词模板)** | 预设的提示词模板,供用户/应用选用 | 代码片段、模板                      |

> 目前 Claude API 的 **MCP connector 只支持 Tools**;Resources 和 Prompts 需要你自己用 MCP SDK 管理连接(见 11.6)。

### 11.4 三种「用上 MCP」的姿势

**姿势 A:通过支持 MCP 的客户端(零代码)**
Claude Desktop、Claude Code 等本身就是 MCP Host。你只要在配置文件里声明一个 Server,它就自动获得那些工具。这是最常见的日常用法,不用写代码。

**姿势 B:通过 Messages API 的 MCP connector(连远程 Server)**
你自己的程序里,直接在请求里挂上一个**远程 HTTP MCP Server**,Claude 会自动调用它的工具——你不用自己实现 MCP 客户端。

```typescript
import Anthropic from "@anthropic-ai/sdk";

const anthropic = new Anthropic();

const response = await anthropic.beta.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 1000,
  messages: [{ role: "user", content: "查一下 Jira 上还有哪些未关闭的 bug" }],
  // 声明要连接的远程 MCP Server(必须是 https://)
  mcp_servers: [
    {
      type: "url",
      url: "https://your-mcp-server.example.com/sse",
      name: "jira",
      authorization_token: "YOUR_OAUTH_TOKEN", // 需要鉴权时传(OAuth)
    },
  ],
  // 在 tools 里用 mcp_toolset 启用该 Server 的工具
  tools: [{ type: "mcp_toolset", mcp_server_name: "jira" }],
  betas: ["mcp-client-2025-11-20"], // 当前 beta header
});

console.log(response);
```

返回里会出现两种新内容块:`mcp_tool_use`(模型调用了 MCP 工具)和 `mcp_tool_result`(工具结果)。

> 限制:MCP connector 只支持 **URL(HTTPS)** 形式的 Server,**本地 stdio Server 连不了**;且当前只支持 **Tools**。

**姿势 C:自己管理连接(本地 stdio Server / 需要 Resources、Prompts)**
当你要连本地 Server,或要用 Resources/Prompts 时,用官方 MCP SDK 自己建连接,再用 Anthropic 提供的 helper 把 MCP 工具转成 Claude 工具,交给 Tool Runner(第 5 章)。

```bash
npm install @anthropic-ai/sdk @modelcontextprotocol/sdk
```

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { mcpTools } from "@anthropic-ai/sdk/helpers/beta/mcp";
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

const anthropic = new Anthropic();

// 1. 连接一个本地 MCP Server(作为子进程启动)
const transport = new StdioClientTransport({
  command: "mcp-server-weather",
  args: [],
});
const mcpClient = new Client({ name: "my-agent", version: "1.0.0" });
await mcpClient.connect(transport);

// 2. 列出 Server 的工具,用 mcpTools 转成 Claude 工具
const { tools } = await mcpClient.listTools();

// 3. 交给 Tool Runner,自动跑 agent loop(执行也由 mcpClient 代理)
const finalMessage = await anthropic.beta.messages.toolRunner({
  model: "claude-opus-4-8",
  max_tokens: 1024,
  messages: [{ role: "user", content: "北京天气怎么样?" }],
  tools: mcpTools(tools, mcpClient), // ★ 关键:MCP 工具 → Claude 工具
});

console.log(finalMessage);
```

> 这些 helper(`mcpTools` / `mcpMessages` / `mcpResourceToContent` / `mcpResourceToFile`)目前**只在 TypeScript SDK** 提供。

### 11.5 自己写一个 MCP Server(TS)

反过来,你也可以把自己的能力做成一个 MCP Server,让任何 MCP Host 都能用。下面是一个最小的天气 Server(stdio 传输):

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "weather", version: "1.0.0" });

// 注册一个工具
server.registerTool(
  "get_weather",
  {
    title: "查询天气",
    description: "查询某城市的当前天气。",
    inputSchema: { city: z.string().describe("城市名") },
  },
  async ({ city }) => ({
    content: [{ type: "text", text: `${city} 当前 -2°C,晴。` }],
  }),
);

// 用 stdio 传输启动,等待 Host 连接
const transport = new StdioServerTransport();
await server.connect(transport);
```

写好后,在 Claude Desktop / Claude Code 的配置里指向这个 Server(命令 + 参数),它就能用 `get_weather` 了——**这就是「写一次,处处可用」的威力**。

### 11.6 工具配置与安全

通过 connector 连一个 Server 时,可以精细控制启用哪些工具:

```typescript
// 白名单:默认全关,只开指定工具
tools: [
  {
    type: "mcp_toolset",
    mcp_server_name: "calendar",
    default_config: { enabled: false },
    configs: {
      search_events: { enabled: true },
      list_events: { enabled: true },
    },
  },
];

// 黑名单:默认全开,只关危险工具(做只读助手时强烈建议)
tools: [
  {
    type: "mcp_toolset",
    mcp_server_name: "calendar",
    configs: {
      delete_all_events: { enabled: false },
    },
  },
];
```

安全要点:

- **第三方 Server 是在「替模型执行动作」**,等于把权限交出去了。只连**可信**的 Server。
- 对会改数据/不可逆的工具(删除、发消息、转账),用**黑名单关掉**,或在你自己的 Host 里加**人工确认**。
- 鉴权 token 由**你**负责走 OAuth 流程获取和刷新;别硬编码进代码。
- 注意 MCP connector 的数据**不适用零数据保留(ZDR)**。

### 11.7 MCP vs 自己手写工具:怎么选

| 场景                                     | 用                               |
| ---------------------------------------- | -------------------------------- |
| 工具只在这一个 agent 里用、逻辑简单      | **手写工具**(第 3 章),最简单直接 |
| 能力要被多个应用/团队复用                | **MCP Server**,写一次到处用      |
| 想接现成的生态(GitHub、Slack、Sentry…)   | **连现成的 MCP Server**          |
| 需要在 Claude Desktop / Claude Code 里用 | **MCP**(它们原生支持)            |

新手建议:**先用手写工具把 agent 跑通(第 3~5 章),理解了工具的本质,再学 MCP 把工具「标准化、可复用」。** MCP 不是新概念,它就是「工具调用」的标准化封装。

### 11.8 另一类协议:Agent ↔ Agent(A2A)

MCP 解决的是 **agent ↔ 工具/数据**(纵向:让一个 agent 用上外部能力)。还有一类协议解决 **agent ↔ agent**(横向:让独立的 agent 互相发现、协作),代表是 **A2A(Agent2Agent)**:

- 每个 agent 发布一张「**Agent Card**」描述自己会什么、怎么调用。
- 其他 agent 通过 HTTP/JSON-RPC 发现它、给它派任务、收结果。
- 适合**跨组织、跨团队**的独立 agent 拼装成更大系统。

|      | MCP               | A2A                           |
| ---- | ----------------- | ----------------------------- |
| 连接 | agent ↔ 工具/数据 | agent ↔ agent                 |
| 方向 | 纵向(获取能力)    | 横向(分工协作)                |
| 类比 | 给 agent 装外设   | 让多个 agent 组队             |
| 归属 | Anthropic 开源    | 由 Linux 基金会托管的开放项目 |

两者**互补**:一个 A2A 系统里的每个 agent,内部往往各自用 MCP 去接它需要的工具。

> 新手阶段:**先吃透 MCP**(它和你已经会的工具调用一脉相承),A2A 了解概念即可,等你真要做「多个独立 agent 跨系统协作」时再深入。

### 11.9 MCP 常见坑

| 坑                                              | 后果            | 正确做法                                                 |
| ----------------------------------------------- | --------------- | -------------------------------------------------------- |
| 用 connector 想连本地 stdio Server              | 连不上          | connector 只支持 HTTPS URL;本地 Server 走姿势 C(MCP SDK) |
| 用了废弃的 beta header `mcp-client-2025-04-04`  | 行为不对/不支持 | 用当前的 `mcp-client-2025-11-20`                         |
| `mcp_servers` 里声明了 Server 但 `tools` 没引用 | API 校验失败    | 每个 Server 必须被**恰好一个** `mcp_toolset` 引用        |
| 盲目连接不可信第三方 Server                     | 安全/数据风险   | 只连可信 Server;危险工具加黑名单或人工确认               |
| 把所有能力都做成 MCP                            | 过度设计        | 一次性、简单的工具直接手写;要复用才上 MCP                |

---

## 12. 工程化:错误处理、成本、缓存

### 10.1 错误处理(用类型化异常,别字符串匹配)

```typescript
import Anthropic from "@anthropic-ai/sdk";

try {
  const response = await client.messages.create({
    /* ... */
  });
} catch (error) {
  if (error instanceof Anthropic.RateLimitError) {
    // 429 限流 —— SDK 默认会自动重试(指数退避)
  } else if (error instanceof Anthropic.BadRequestError) {
    // 400 请求格式问题
  } else if (error instanceof Anthropic.APIError) {
    console.error(`API 错误 ${error.status}:`, error.message);
  }
}
```

所有异常都继承 `Anthropic.APIError`(带 `status` 字段)。从**最具体**到最一般地判断。

### 10.2 Prompt 缓存(省钱省时间)

如果多次请求共享一大段固定前缀(比如长 system prompt、大段文档),用缓存可以让重复部分便宜约 90%。

```typescript
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  cache_control: { type: "ephemeral" }, // 自动缓存最后一个可缓存块
  system: largeDocumentText, // 例如 50KB 的固定上下文
  messages: [{ role: "user", content: "总结要点" }],
});

// 验证是否命中缓存
console.log(response.usage.cache_read_input_tokens); // >0 表示命中
```

**核心原理:缓存是前缀匹配**。前缀里任何一个字节变了(比如 system prompt 里塞了 `Date.now()`、未排序的 JSON),后面全部失效。所以:**保持 system prompt、工具列表稳定,把多变内容(时间戳、每次不同的问题)放最后。**

### 10.3 成本意识

- Opus 4.8:输入 $5 / 百万 token,输出 $25 / 百万 token。1M 上下文窗口,无长上下文溢价。
- 想压成本:先调低 `effort`(`high`→`medium`),用缓存,必要时把简单子任务交给更便宜的 Sonnet/Haiku。
- 别用 `tiktoken` 估算 token(那是 OpenAI 的,对 Claude 不准)。用 `client.messages.countTokens({...})`。

---

## 13. 常见坑速查

| 坑                                                   | 后果                     | 正确做法                                                         |
| ---------------------------------------------------- | ------------------------ | ---------------------------------------------------------------- |
| 访问 `response.content[0].text` 不判断类型           | TS 报错 / 拿到 undefined | 先 `if (block.type === "text")` 收窄                             |
| 工具循环里没把 `assistant` 回复追加回历史            | 模型逻辑混乱、死循环     | `messages.push({ role:"assistant", content: response.content })` |
| `tool_result` 没带对 `tool_use_id`                   | API 400                  | 用 `block.id` 一一对应                                           |
| compaction 时只追加文本                              | 丢失压缩状态             | 追加整个 `response.content`                                      |
| 在 Opus 4.8 传 `temperature`/`top_p`/`budget_tokens` | 400 报错                 | 删掉;用 `thinking: {type:"adaptive"}` + `effort`                 |
| `max_tokens` 给太小                                  | 输出被截断               | 非流式默认 16000,流式默认 64000                                  |
| `max_tokens > 16000` 还用非流式                      | HTTP 超时                | 改用 `.stream()` + `.finalMessage()`                             |
| system prompt 里放 `Date.now()`/UUID                 | 缓存永不命中             | 固定内容前置,多变内容放最后                                      |
| 自己定义 `interface ChatMessage`                     | 丢类型安全               | 用 `Anthropic.MessageParam` 等 SDK 类型                          |
| 用 `eval` 执行模型给的代码/表达式                    | 安全风险                 | 用安全的解析库;危险操作加人工确认                                |

---

## 参考

- [Anthropic 官方文档](https://platform.claude.com/docs/en/home)
