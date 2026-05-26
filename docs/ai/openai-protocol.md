# OpenAI API 协议

## 一、基础概念

### 1.1 什么是 OpenAI API 协议

OpenAI API 是一套基于 **HTTP/HTTPS + JSON** 的 RESTful 接口规范。由于 OpenAI 的先发优势，该协议已成为大语言模型领域的**事实标准（de facto standard）**，众多厂商和开源项目均提供兼容接口。

```
客户端                           OpenAI 服务器
  │                                    │
  │── POST /v1/chat/completions ───────▶│
  │   Headers: Authorization: Bearer   │
  │   Body: JSON                       │
  │                                    │
  │◀────────────── JSON Response ──────│
  │                  or SSE Stream     │
```

### 1.2 Token 概念

所有计费和限制都基于 **Token**（词元），而非字符数：

- 英文：约 1 token ≈ 4 个字符（1 个单词 ≈ 1.3 token）
- 中文：约 1 token ≈ 1-2 个汉字
- 代码：比普通文本消耗更多 token

```python
# 估算 token 数（本地）
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o")
tokens = enc.encode("你好，世界！Hello World!")
print(len(tokens))  # token 数量
```

### 1.3 上下文窗口（Context Window）

模型一次能处理的最大 token 数，包含输入+输出：

| 模型        | 上下文窗口  |
| ----------- | ----------- |
| gpt-4o      | 128K tokens |
| gpt-4o-mini | 128K tokens |
| gpt-4-turbo | 128K tokens |
| o1          | 200K tokens |

---

## 二、认证与鉴权

### 2.1 API Key

所有请求必须在 HTTP Header 中携带 API Key：

```http
Authorization: Bearer sk-proj-xxxxxxxxxxxxxxxxxxxxxxxx
```

```python
from openai import OpenAI

# 方式1：直接传入
client = OpenAI(api_key="sk-proj-xxx")

# 方式2：环境变量（推荐）
# export OPENAI_API_KEY="sk-proj-xxx"
client = OpenAI()  # 自动读取 OPENAI_API_KEY
```

```javascript
// Node.js
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});
```

### 2.2 Organization ID（可选）

归属到某个组织时需要传递：

```http
OpenAI-Organization: org-xxxxxxxxxxxxxxxx
```

```python
client = OpenAI(
    api_key="sk-xxx",
    organization="org-xxx"
)
```

### 2.3 Project ID（可选）

用于精细化计费和权限管理：

```http
OpenAI-Project: proj-xxxxxxxxxxxxxxxx
```

---

## 三、核心端点总览

```
基础 URL: https://api.openai.com/v1
```

| 端点                    | 方法            | 说明                     |
| ----------------------- | --------------- | ------------------------ |
| `/chat/completions`     | POST            | 对话补全（核心接口）     |
| `/completions`          | POST            | 文本补全（旧版，已弃用） |
| `/embeddings`           | POST            | 文本向量化               |
| `/images/generations`   | POST            | 文生图                   |
| `/images/edits`         | POST            | 图像编辑                 |
| `/images/variations`    | POST            | 图像变体                 |
| `/audio/transcriptions` | POST            | 语音转文字（Whisper）    |
| `/audio/translations`   | POST            | 语音翻译为英文           |
| `/audio/speech`         | POST            | 文字转语音（TTS）        |
| `/files`                | POST/GET/DELETE | 文件管理                 |
| `/batches`              | POST/GET        | 批量请求                 |
| `/models`               | GET             | 列出可用模型             |
| `/models/{model}`       | GET             | 模型详情                 |
| `/moderations`          | POST            | 内容审核                 |

---

## 四、Chat Completions 详解

### 4.1 基础请求结构

```json
POST /v1/chat/completions
Content-Type: application/json
Authorization: Bearer sk-xxx

{
  "model": "gpt-4o",
  "messages": [
    {
      "role": "system",
      "content": "你是一个专业的代码助手，用中文回答问题。"
    },
    {
      "role": "user",
      "content": "解释什么是递归"
    }
  ]
}
```

### 4.2 完整请求参数

```typescript
interface ChatCompletionRequest {
  // 必填
  model: string; // 模型名称
  messages: Message[]; // 对话历史

  // 采样参数
  temperature?: number; // 0.0-2.0，默认 1.0
  top_p?: number; // 0.0-1.0，核采样
  n?: number; // 生成几个候选，默认 1
  seed?: number; // 随机种子（可重复性）

  // 输出控制
  max_tokens?: number; // 最大输出 token 数
  max_completion_tokens?: number; // 新版，替代 max_tokens
  stop?: string | string[]; // 停止词
  stream?: boolean; // 是否流式输出
  stream_options?: {
    include_usage?: boolean; // 流式时是否返回 usage
  };

  // 惩罚参数
  presence_penalty?: number; // -2.0 到 2.0，鼓励新话题
  frequency_penalty?: number; // -2.0 到 2.0，减少重复
  logit_bias?: Record<string, number>; // 调整特定 token 概率

  // 工具
  tools?: Tool[]; // 工具定义列表
  tool_choice?: ToolChoice; // 工具选择策略
  parallel_tool_calls?: boolean; // 是否允许并行工具调用

  // 输出格式
  response_format?: ResponseFormat; // 输出格式约束

  // 其他
  user?: string; // 终端用户标识（用于审计）
  logprobs?: boolean; // 是否返回 log 概率
  top_logprobs?: number; // 返回 top N 的 logprob
}
```

### 4.3 消息角色（Message Roles）

#### system —— 系统提示

```json
{
  "role": "system",
  "content": "你是一个专业的数据分析师。始终用中文回答，并以结构化的方式呈现数据。"
}
```

system 消息的最佳实践：

- 放在 messages 数组的**第一条**
- 设定人格、能力边界、输出格式
- 避免过长（影响性能和成本）

#### user —— 用户输入

```json
// 纯文本
{
  "role": "user",
  "content": "帮我分析这份数据"
}

// 多模态（文本+图片）
{
  "role": "user",
  "content": [
    {
      "type": "text",
      "text": "这张图里有什么？"
    },
    {
      "type": "image_url",
      "image_url": {
        "url": "https://example.com/image.jpg",
        "detail": "high"
      }
    }
  ]
}
```

#### assistant —— 模型回复

```json
// 普通回复
{
  "role": "assistant",
  "content": "好的，我来分析这份数据..."
}

// 包含工具调用的回复
{
  "role": "assistant",
  "content": null,
  "tool_calls": [
    {
      "id": "call_abc123",
      "type": "function",
      "function": {
        "name": "get_data",
        "arguments": "{\"query\": \"销售数据\"}"
      }
    }
  ]
}
```

#### tool —— 工具结果

```json
{
  "role": "tool",
  "tool_call_id": "call_abc123",
  "content": "{\"sales\": 1000, \"unit\": \"万元\"}"
}
```

### 4.4 完整响应结构

```json
{
  "id": "chatcmpl-abc123xyz",
  "object": "chat.completion",
  "created": 1716000000,
  "model": "gpt-4o-2024-08-06",
  "system_fingerprint": "fp_abc123",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "递归是指函数调用自身的编程技术..."
      },
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 45,
    "completion_tokens": 187,
    "total_tokens": 232,
    "prompt_tokens_details": {
      "cached_tokens": 0,
      "audio_tokens": 0
    },
    "completion_tokens_details": {
      "reasoning_tokens": 0,
      "audio_tokens": 0,
      "accepted_prediction_tokens": 0,
      "rejected_prediction_tokens": 0
    }
  }
}
```

### 4.5 finish_reason 枚举值

| 值               | 含义                             |
| ---------------- | -------------------------------- |
| `stop`           | 正常结束（遇到停止词或自然结束） |
| `length`         | 达到 max_tokens 限制被截断       |
| `tool_calls`     | 模型调用了工具，需要处理工具结果 |
| `content_filter` | 被内容过滤器拦截                 |
| `function_call`  | 旧版函数调用（已弃用）           |

### 4.6 采样参数详解

**temperature**（温度）

```
temperature = 0.0 → 几乎确定性输出，适合代码/数学
temperature = 0.7 → 平衡创意与准确性，适合对话
temperature = 1.0 → 默认值
temperature = 1.5+ → 高创意，可能不连贯，适合创意写作
```

**top_p**（核采样）

```
top_p = 0.1 → 只从概率最高的 10% token 中采样（保守）
top_p = 0.9 → 从概率最高的 90% token 中采样（平衡）
top_p = 1.0 → 不限制（默认）
```

> 建议：temperature 和 top_p 只调整其中一个，不要同时修改。

**presence_penalty / frequency_penalty**

```
presence_penalty  = 正值 → 惩罚已出现过的 token（鼓励引入新话题）
frequency_penalty = 正值 → 惩罚高频出现的 token（减少重复）
两者均为负值时     → 反向效果
```

---

## 五、流式输出（SSE）

### 5.1 协议原理

流式输出基于 **Server-Sent Events（SSE）** 协议，HTTP 连接保持打开，服务器持续推送数据块：

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"id":"chatcmpl-x","object":"chat.completion.chunk","choices":[{"delta":{"role":"assistant"},"index":0}]}

data: {"id":"chatcmpl-x","object":"chat.completion.chunk","choices":[{"delta":{"content":"递"},"index":0}]}

data: {"id":"chatcmpl-x","object":"chat.completion.chunk","choices":[{"delta":{"content":"归"},"index":0}]}

data: {"id":"chatcmpl-x","object":"chat.completion.chunk","choices":[{"delta":{},"finish_reason":"stop","index":0}]}

data: [DONE]
```

### 5.2 流式响应数据块（Chunk）结构

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion.chunk",
  "created": 1716000000,
  "model": "gpt-4o",
  "choices": [
    {
      "index": 0,
      "delta": {
        "role": "assistant", // 只在第一个 chunk 出现
        "content": "递归" // 增量内容
      },
      "finish_reason": null // 结束时为 "stop" 等值
    }
  ]
}
```

### 5.3 Python 流式实现

```python
from openai import OpenAI

client = OpenAI()

# 基础流式
def stream_chat(prompt: str):
    stream = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        stream=True
    )

    full_content = ""
    for chunk in stream:
        delta = chunk.choices[0].delta
        if delta.content:
            print(delta.content, end="", flush=True)
            full_content += delta.content

    print()  # 换行
    return full_content


# 包含 usage 统计的流式
def stream_with_usage(prompt: str):
    stream = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        stream=True,
        stream_options={"include_usage": True}
    )

    for chunk in stream:
        if chunk.choices and chunk.choices[0].delta.content:
            print(chunk.choices[0].delta.content, end="", flush=True)

        # usage 在最后一个 chunk 返回
        if chunk.usage:
            print(f"\n[Token 用量: {chunk.usage.total_tokens}]")
```

### 5.4 JavaScript/Node.js 流式实现

```javascript
import OpenAI from "openai";

const client = new OpenAI();

async function streamChat(prompt) {
  const stream = await client.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: prompt }],
    stream: true,
  });

  let fullContent = "";

  for await (const chunk of stream) {
    const content = chunk.choices[0]?.delta?.content || "";
    process.stdout.write(content);
    fullContent += content;
  }

  console.log();
  return fullContent;
}

// 前端 Fetch 实现（浏览器）
async function streamChatBrowser(prompt) {
  const response = await fetch("/api/chat", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ message: prompt }),
  });

  const reader = response.body.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    const lines = decoder.decode(value).split("\n");
    for (const line of lines) {
      if (line.startsWith("data: ") && line !== "data: [DONE]") {
        const data = JSON.parse(line.slice(6));
        const content = data.choices[0]?.delta?.content;
        if (content) {
          // 更新 UI
          document.getElementById("output").textContent += content;
        }
      }
    }
  }
}
```

---

## 六、工具调用（Function Calling）

### 6.1 工具定义结构

```json
{
  "type": "function",
  "function": {
    "name": "get_weather",
    "description": "获取指定城市和日期的天气信息。当用户询问天气时调用此函数。",
    "parameters": {
      "type": "object",
      "properties": {
        "city": {
          "type": "string",
          "description": "城市名称，例如：北京、上海"
        },
        "date": {
          "type": "string",
          "description": "日期，格式：YYYY-MM-DD，不填则为今天"
        },
        "unit": {
          "type": "string",
          "enum": ["celsius", "fahrenheit"],
          "description": "温度单位"
        }
      },
      "required": ["city"],
      "additionalProperties": false
    },
    "strict": true
  }
}
```

> `"strict": true` 配合 `"additionalProperties": false` 可强制模型严格遵循 schema，减少幻觉字段。

### 6.2 tool_choice 参数

```json
// 自动决定（默认）
"tool_choice": "auto"

// 禁止调用工具，强制文本回复
"tool_choice": "none"

// 必须调用至少一个工具
"tool_choice": "required"

// 强制调用指定工具
"tool_choice": {
  "type": "function",
  "function": { "name": "get_weather" }
}
```

### 6.3 完整工具调用流程

```python
from openai import OpenAI
import json

client = OpenAI()

# 1. 定义工具
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取城市天气",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "城市名"},
                    "date": {"type": "string", "description": "日期 YYYY-MM-DD"}
                },
                "required": ["city"],
                "additionalProperties": False
            },
            "strict": True
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search_web",
            "description": "搜索互联网信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "搜索关键词"}
                },
                "required": ["query"],
                "additionalProperties": False
            },
            "strict": True
        }
    }
]

# 2. 工具执行函数
def execute_tool(name: str, arguments: dict) -> str:
    if name == "get_weather":
        city = arguments["city"]
        date = arguments.get("date", "今天")
        # 调用真实天气 API...
        return json.dumps({"city": city, "date": date, "temp": 25, "condition": "晴"}, ensure_ascii=False)

    elif name == "search_web":
        query = arguments["query"]
        # 调用搜索 API...
        return f"搜索结果：关于'{query}'的相关信息..."

    return "工具不存在"

# 3. Agent 循环
def run_agent(user_input: str):
    messages = [{"role": "user", "content": user_input}]

    while True:
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=tools,
            tool_choice="auto",
            parallel_tool_calls=True   # 允许并行调用多个工具
        )

        choice = response.choices[0]
        messages.append(choice.message.model_dump())

        # 无工具调用 → 完成
        if choice.finish_reason == "stop":
            return choice.message.content

        # 有工具调用 → 执行并返回结果
        if choice.finish_reason == "tool_calls":
            tool_results = []

            for tool_call in choice.message.tool_calls:
                print(f"[调用工具] {tool_call.function.name}: {tool_call.function.arguments}")

                arguments = json.loads(tool_call.function.arguments)
                result = execute_tool(tool_call.function.name, arguments)

                print(f"[工具结果] {result}\n")

                tool_results.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result
                })

            messages.extend(tool_results)

# 运行
answer = run_agent("北京今天天气怎么样？顺便搜索一下明天有什么活动")
print(f"\n最终回复：{answer}")
```

### 6.4 并行工具调用

当 `parallel_tool_calls=True`（默认），模型可以在一次响应中调用多个工具：

```json
{
  "role": "assistant",
  "content": null,
  "tool_calls": [
    {
      "id": "call_001",
      "type": "function",
      "function": { "name": "get_weather", "arguments": "{\"city\": \"北京\"}" }
    },
    {
      "id": "call_002",
      "type": "function",
      "function": { "name": "get_weather", "arguments": "{\"city\": \"上海\"}" }
    },
    {
      "id": "call_003",
      "type": "function",
      "function": {
        "name": "search_web",
        "arguments": "{\"query\": \"旅游攻略\"}"
      }
    }
  ]
}
```

处理时可并行执行这三个工具调用，提高效率。

---

## 七、结构化输出（Structured Output）

### 7.1 JSON Mode（基础）

只保证输出合法 JSON，但不约束具体结构：

```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "始终以 JSON 格式输出"},
        {"role": "user", "content": "提取以下文本中的人名和地点：张三在北京工作，李四在上海"}
    ],
    response_format={"type": "json_object"}
)

data = json.loads(response.choices[0].message.content)
```

### 7.2 JSON Schema（严格约束）

指定精确的输出结构，模型 100% 遵循：

```python
from pydantic import BaseModel
from typing import List

# 使用 Pydantic 定义结构
class Person(BaseModel):
    name: str
    location: str
    role: str | None = None

class ExtractResult(BaseModel):
    persons: List[Person]
    total_count: int

# 方式1：直接用 Pydantic 模型（推荐）
response = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[
        {"role": "system", "content": "从文本中提取人物信息"},
        {"role": "user", "content": "张三是北京的工程师，李四在上海做设计师"}
    ],
    response_format=ExtractResult
)

result = response.choices[0].message.parsed
print(result.persons)       # [Person(name='张三', ...), Person(name='李四', ...)]
print(result.total_count)   # 2


# 方式2：手动写 JSON Schema
response = client.chat.completions.create(
    model="gpt-4o-2024-08-06",
    messages=[...],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "extract_result",
            "strict": True,
            "schema": {
                "type": "object",
                "properties": {
                    "persons": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "name": {"type": "string"},
                                "location": {"type": "string"},
                                "role": {"type": ["string", "null"]}
                            },
                            "required": ["name", "location", "role"],
                            "additionalProperties": False
                        }
                    },
                    "total_count": {"type": "integer"}
                },
                "required": ["persons", "total_count"],
                "additionalProperties": False
            }
        }
    }
)
```

### 7.3 支持的 JSON Schema 类型

```
string, number, integer, boolean, null, array, object

// 支持的关键字
enum, anyOf, $ref, $defs

// 不支持的关键字（strict 模式）
oneOf, allOf, not, if/then/else, pattern, minLength, ...
```

---

## 八、多模态（Vision）

### 8.1 图片输入

```python
import base64

# 方式1：URL（公开图片）
messages = [
    {
        "role": "user",
        "content": [
            {"type": "text", "text": "这张图片展示了什么？"},
            {
                "type": "image_url",
                "image_url": {
                    "url": "https://example.com/image.jpg",
                    "detail": "auto"  # auto | low | high
                }
            }
        ]
    }
]

# 方式2：Base64 编码（本地图片）
with open("screenshot.png", "rb") as f:
    image_data = base64.b64encode(f.read()).decode("utf-8")

messages = [
    {
        "role": "user",
        "content": [
            {"type": "text", "text": "分析这个界面截图"},
            {
                "type": "image_url",
                "image_url": {
                    "url": f"data:image/png;base64,{image_data}",
                    "detail": "high"
                }
            }
        ]
    }
]

response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages,
    max_tokens=1000
)
print(response.choices[0].message.content)
```

### 8.2 detail 参数说明

| 值     | Token 消耗     | 适用场景                 |
| ------ | -------------- | ------------------------ |
| `low`  | 固定 85 tokens | 图片整体理解，不需要细节 |
| `high` | 按分辨率计算   | 需要识别细节文字、图表   |
| `auto` | 自动决定       | 默认，根据图片大小选择   |

### 8.3 多图片输入

```python
messages = [
    {
        "role": "user",
        "content": [
            {"type": "text", "text": "比较这两张图片的差异"},
            {"type": "image_url", "image_url": {"url": "https://example.com/before.jpg"}},
            {"type": "image_url", "image_url": {"url": "https://example.com/after.jpg"}}
        ]
    }
]
```

---

## 九、Embeddings

### 9.1 基础用法

```python
response = client.embeddings.create(
    model="text-embedding-3-large",
    input="人工智能是未来的核心技术",
    encoding_format="float"   # float | base64
)

vector = response.data[0].embedding  # 3072 维向量
print(len(vector))  # 3072
```

### 9.2 批量 Embedding

```python
texts = ["文本一", "文本二", "文本三"]

response = client.embeddings.create(
    model="text-embedding-3-small",
    input=texts
)

vectors = [item.embedding for item in response.data]
```

### 9.3 模型对比

| 模型                     | 维度 | 特点               |
| ------------------------ | ---- | ------------------ |
| `text-embedding-3-large` | 3072 | 最高精度，适合生产 |
| `text-embedding-3-small` | 1536 | 性价比高           |
| `text-embedding-ada-002` | 1536 | 旧版，已被替代     |

### 9.4 相似度计算

```python
import numpy as np

def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# 示例：语义搜索
def semantic_search(query: str, documents: list[str], top_k: int = 3):
    # 获取 query 向量
    query_resp = client.embeddings.create(model="text-embedding-3-small", input=query)
    query_vec = query_resp.data[0].embedding

    # 获取文档向量
    docs_resp = client.embeddings.create(model="text-embedding-3-small", input=documents)
    doc_vecs = [item.embedding for item in docs_resp.data]

    # 计算相似度排序
    scores = [(i, cosine_similarity(query_vec, doc_vec))
              for i, doc_vec in enumerate(doc_vecs)]
    scores.sort(key=lambda x: x[1], reverse=True)

    return [(documents[i], score) for i, score in scores[:top_k]]
```

---

## 十、Batch API

批量处理大量请求，成本降低 **50%**，响应时间 24 小时内。

### 10.1 准备批量请求文件

```python
import json

# 构建 JSONL 格式的请求文件
requests = [
    {
        "custom_id": "request-001",
        "method": "POST",
        "url": "/v1/chat/completions",
        "body": {
            "model": "gpt-4o-mini",
            "messages": [{"role": "user", "content": "翻译成英文：你好世界"}],
            "max_tokens": 100
        }
    },
    {
        "custom_id": "request-002",
        "method": "POST",
        "url": "/v1/chat/completions",
        "body": {
            "model": "gpt-4o-mini",
            "messages": [{"role": "user", "content": "翻译成英文：人工智能"}],
            "max_tokens": 100
        }
    }
]

# 写入 JSONL 文件
with open("batch_requests.jsonl", "w") as f:
    for req in requests:
        f.write(json.dumps(req, ensure_ascii=False) + "\n")
```

### 10.2 提交批量任务

```python
# 上传文件
with open("batch_requests.jsonl", "rb") as f:
    file = client.files.create(file=f, purpose="batch")

# 创建批量任务
batch = client.batches.create(
    input_file_id=file.id,
    endpoint="/v1/chat/completions",
    completion_window="24h"
)

print(f"Batch ID: {batch.id}")
print(f"Status: {batch.status}")
```

### 10.3 查询结果

```python
import time

# 轮询状态
while True:
    batch = client.batches.retrieve(batch.id)
    print(f"Status: {batch.status} ({batch.request_counts.completed}/{batch.request_counts.total})")

    if batch.status == "completed":
        break
    elif batch.status in ["failed", "cancelled", "expired"]:
        raise Exception(f"Batch failed: {batch.status}")

    time.sleep(60)

# 下载结果
result_file = client.files.content(batch.output_file_id)
results = [json.loads(line) for line in result_file.text.split("\n") if line]

for result in results:
    custom_id = result["custom_id"]
    content = result["response"]["body"]["choices"][0]["message"]["content"]
    print(f"{custom_id}: {content}")
```

---

## 十一、Files API

### 11.1 上传文件

```python
# 上传（支持：jsonl, pdf, txt, png, jpg 等）
with open("data.jsonl", "rb") as f:
    file = client.files.create(
        file=f,
        purpose="batch"       # batch | fine-tune | assistants
    )

print(f"File ID: {file.id}")
print(f"Size: {file.bytes} bytes")
```

### 11.2 管理文件

```python
# 列出所有文件
files = client.files.list()
for f in files.data:
    print(f.id, f.filename, f.purpose)

# 获取文件信息
file = client.files.retrieve("file-xxx")

# 下载文件内容
content = client.files.content("file-xxx")
print(content.text)

# 删除文件
client.files.delete("file-xxx")
```

---

## 十二、错误处理

### 12.1 HTTP 错误码

| 状态码 | 类型          | 常见原因               | 处理建议        |
| ------ | ------------- | ---------------------- | --------------- |
| 400    | Bad Request   | 请求格式错误、参数非法 | 检查请求体      |
| 401    | Unauthorized  | API Key 无效或过期     | 检查密钥        |
| 403    | Forbidden     | 无权访问（IP 封禁等）  | 联系支持        |
| 404    | Not Found     | 模型不存在、端点错误   | 检查 model 名称 |
| 422    | Unprocessable | 参数类型正确但值非法   | 检查参数范围    |
| 429    | Rate Limit    | 超出请求频率限制       | 退避重试        |
| 500    | Server Error  | 服务端内部错误         | 重试            |
| 503    | Unavailable   | 服务过载               | 重试            |

### 12.2 错误响应格式

```json
{
  "error": {
    "message": "Invalid API key provided.",
    "type": "invalid_request_error",
    "param": null,
    "code": "invalid_api_key"
  }
}
```

### 12.3 Python 异常处理

```python
from openai import (
    OpenAI,
    APIError,
    APIConnectionError,
    APITimeoutError,
    AuthenticationError,
    PermissionDeniedError,
    NotFoundError,
    UnprocessableEntityError,
    RateLimitError,
    InternalServerError,
)
import time

client = OpenAI()

def robust_completion(messages, max_retries=3, base_delay=1.0):
    for attempt in range(max_retries):
        try:
            return client.chat.completions.create(
                model="gpt-4o",
                messages=messages
            )

        except AuthenticationError as e:
            # API Key 问题，不重试
            raise RuntimeError(f"认证失败，请检查 API Key: {e}") from e

        except PermissionDeniedError as e:
            raise RuntimeError(f"无权限: {e}") from e

        except NotFoundError as e:
            raise RuntimeError(f"资源不存在: {e}") from e

        except UnprocessableEntityError as e:
            raise ValueError(f"请求参数错误: {e}") from e

        except RateLimitError as e:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt)  # 指数退避
            print(f"触发限流，{delay}s 后重试...")
            time.sleep(delay)

        except (APIConnectionError, APITimeoutError) as e:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt)
            print(f"网络错误，{delay}s 后重试: {e}")
            time.sleep(delay)

        except InternalServerError as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(base_delay)
            print(f"服务器错误，重试中...")

        except APIError as e:
            # 其他 API 错误
            raise
```

### 12.4 retry-after Header

限流时，服务器会返回 `retry-after` Header 指示等待时间：

```python
except RateLimitError as e:
    retry_after = e.response.headers.get("retry-after", 60)
    time.sleep(float(retry_after))
```

---

## 十三、限流与配额

### 13.1 限流维度

| 维度                      | 说明            |
| ------------------------- | --------------- |
| RPM (Requests Per Minute) | 每分钟请求数    |
| TPM (Tokens Per Minute)   | 每分钟 token 数 |
| RPD (Requests Per Day)    | 每天请求数      |
| TPD (Tokens Per Day)      | 每天 token 数   |

### 13.2 响应 Header 中的限流信息

```http
x-ratelimit-limit-requests: 5000
x-ratelimit-limit-tokens: 4000000
x-ratelimit-remaining-requests: 4999
x-ratelimit-remaining-tokens: 3998500
x-ratelimit-reset-requests: 12ms
x-ratelimit-reset-tokens: 90ms
```

### 13.3 令牌桶算法（客户端限流）

```python
import threading
import time

class RateLimiter:
    def __init__(self, rpm: int = 500, tpm: int = 100000):
        self.rpm = rpm
        self.tpm = tpm
        self.request_times = []
        self.token_counts = []
        self.lock = threading.Lock()

    def wait_if_needed(self, estimated_tokens: int = 1000):
        with self.lock:
            now = time.time()
            minute_ago = now - 60

            # 清理 1 分钟前的记录
            self.request_times = [t for t in self.request_times if t > minute_ago]
            self.token_counts = [(t, c) for t, c in self.token_counts if t > minute_ago]

            current_rpm = len(self.request_times)
            current_tpm = sum(c for _, c in self.token_counts)

            if current_rpm >= self.rpm or current_tpm + estimated_tokens > self.tpm:
                # 等待到最早的请求过期
                oldest = min(self.request_times + [t for t, _ in self.token_counts])
                wait_time = oldest + 60 - now + 0.1
                if wait_time > 0:
                    time.sleep(wait_time)

            self.request_times.append(time.time())
            self.token_counts.append((time.time(), estimated_tokens))

limiter = RateLimiter(rpm=500)

def limited_call(messages):
    limiter.wait_if_needed()
    return client.chat.completions.create(model="gpt-4o", messages=messages)
```

---

## 十四、兼容厂商接入

### 14.1 切换方式

OpenAI SDK 支持通过 `base_url` 切换到任何兼容厂商：

```python
from openai import OpenAI

# ===== 各厂商配置 =====

# Anthropic Claude（OpenAI 兼容接口）
claude = OpenAI(
    base_url="https://api.anthropic.com/v1",
    api_key="sk-ant-xxx",
    default_headers={"anthropic-version": "2023-06-01"}
)

# Azure OpenAI
azure = OpenAI(
    base_url="https://{resource}.openai.azure.com/openai/deployments/{deployment}",
    api_key="your-azure-key",
    default_query={"api-version": "2024-02-01"}
)

# 本地 Ollama
ollama = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama"
)

# 硅基流动
siliconflow = OpenAI(
    base_url="https://api.siliconflow.cn/v1",
    api_key="sk-xxx"
)

# 月之暗面（Kimi）
kimi = OpenAI(
    base_url="https://api.moonshot.cn/v1",
    api_key="sk-xxx"
)

# 智谱 AI（GLM）
zhipu = OpenAI(
    base_url="https://open.bigmodel.cn/api/paas/v4",
    api_key="xxx"
)

# 阿里云百炼（通义千问）
qwen = OpenAI(
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
    api_key="sk-xxx"
)

# DeepSeek
deepseek = OpenAI(
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx"
)
```

### 14.2 多厂商路由

```python
from typing import Literal
from dataclasses import dataclass

@dataclass
class LLMConfig:
    provider: str
    model: str
    base_url: str
    api_key: str

CONFIGS = {
    "gpt-4o": LLMConfig("openai", "gpt-4o", "https://api.openai.com/v1", "sk-openai-xxx"),
    "claude-3-5-sonnet": LLMConfig("anthropic", "claude-3-5-sonnet-20241022", "https://api.anthropic.com/v1", "sk-ant-xxx"),
    "deepseek-chat": LLMConfig("deepseek", "deepseek-chat", "https://api.deepseek.com/v1", "sk-deepseek-xxx"),
}

def get_client(model_alias: str) -> tuple[OpenAI, str]:
    config = CONFIGS[model_alias]
    client = OpenAI(base_url=config.base_url, api_key=config.api_key)
    return client, config.model

def chat(model_alias: str, messages: list) -> str:
    client, model_name = get_client(model_alias)
    resp = client.chat.completions.create(model=model_name, messages=messages)
    return resp.choices[0].message.content
```

### 14.3 兼容性注意事项

| 特性             | 支持情况       |
| ---------------- | -------------- |
| 基础对话         | 所有厂商支持   |
| 流式输出         | 大多数厂商支持 |
| 工具调用         | 主流厂商支持   |
| JSON Schema 输出 | 部分厂商支持   |
| 并行工具调用     | 少数厂商支持   |
| logprobs         | 部分厂商支持   |
| 图片输入         | 多模态模型支持 |

---

## 十五、最佳实践

### 15.1 成本优化

```python
# 1. 选择合适的模型
# gpt-4o-mini 成本约为 gpt-4o 的 1/15，适合简单任务

# 2. 复用 system prompt（Prompt Caching）
# 相同 system prompt 的多次调用，许多厂商会自动缓存

# 3. 控制输出长度
client.chat.completions.create(
    model="gpt-4o-mini",        # 简单任务用小模型
    messages=messages,
    max_tokens=500,              # 明确限制输出长度
    temperature=0               # 确定性任务设为 0
)

# 4. 批量请求（节省 50% 成本）
# 非实时任务使用 Batch API

# 5. 避免重复发送长文档
# 使用 RAG 只传递相关片段，而非整个文档
```

### 15.2 性能优化

```python
# 并发请求
import asyncio
from openai import AsyncOpenAI

async_client = AsyncOpenAI()

async def batch_chat(prompts: list[str]) -> list[str]:
    tasks = [
        async_client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": p}]
        )
        for p in prompts
    ]
    responses = await asyncio.gather(*tasks)
    return [r.choices[0].message.content for r in responses]

# 运行
results = asyncio.run(batch_chat(["问题1", "问题2", "问题3"]))
```

### 15.3 System Prompt 设计原则

```python
SYSTEM_PROMPT = """
# 角色
你是一个专业的 Python 代码审查助手。

# 能力
- 发现代码中的 Bug 和安全漏洞
- 提供性能优化建议
- 检查代码风格规范（PEP8）

# 输出格式
必须以 JSON 格式输出，包含以下字段：
{
  "issues": [{"type": "bug|security|performance|style", "line": <行号>, "description": "...", "fix": "..."}],
  "score": <0-100>,
  "summary": "..."
}

# 约束
- 只审查 Python 代码
- 如果输入不是代码，返回 {"error": "非 Python 代码"}
- 不要输出 JSON 以外的内容
"""
```

### 15.4 对话历史管理

```python
class ConversationManager:
    def __init__(self, system_prompt: str, max_tokens: int = 100000):
        self.system = system_prompt
        self.history = []
        self.max_tokens = max_tokens

    def add_user(self, content: str):
        self.history.append({"role": "user", "content": content})

    def add_assistant(self, content: str):
        self.history.append({"role": "assistant", "content": content})

    def _estimate_tokens(self) -> int:
        total = len(self.system) // 4
        for msg in self.history:
            total += len(str(msg["content"])) // 4
        return total

    def _trim(self):
        # 超出限制时，移除最早的对话（保留 system）
        while self._estimate_tokens() > self.max_tokens and len(self.history) > 2:
            self.history.pop(0)
            self.history.pop(0)

    def get_messages(self) -> list:
        self._trim()
        return [{"role": "system", "content": self.system}] + self.history

    def chat(self, user_input: str) -> str:
        self.add_user(user_input)

        response = client.chat.completions.create(
            model="gpt-4o",
            messages=self.get_messages()
        )

        reply = response.choices[0].message.content
        self.add_assistant(reply)
        return reply
```

### 15.5 安全注意事项

```python
# Prompt Injection 防护
def safe_user_input(user_input: str) -> str:
    # 不要将用户输入直接拼接到 system prompt
    # 错误做法：
    # system = f"你是助手。用户信息：{user_input}"

    # 正确做法：用户输入单独在 user 消息中
    return user_input

# 内容审核
def check_content(text: str) -> bool:
    response = client.moderations.create(input=text)
    result = response.results[0]
    return not result.flagged

# 输出长度限制（防止成本爆炸）
MAX_OUTPUT_TOKENS = 2000
```

---

## 十六、完整示例：Agent 实现

```python
#!/usr/bin/env python3
"""
完整的 Tool-Use Agent 示例
支持：网络搜索、Python 代码执行、文件读取
"""

import json
import subprocess
import tempfile
import os
from typing import Any
from openai import OpenAI

client = OpenAI()

# ========== 工具定义 ==========

TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "search",
            "description": "搜索互联网获取信息。当需要最新信息或不确定某个知识点时调用。",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "搜索关键词"},
                    "max_results": {"type": "integer", "description": "返回结果数量，默认3"}
                },
                "required": ["query"],
                "additionalProperties": False
            },
            "strict": True
        }
    },
    {
        "type": "function",
        "function": {
            "name": "run_python",
            "description": "执行 Python 代码并返回标准输出。适合计算、数据处理等任务。",
            "parameters": {
                "type": "object",
                "properties": {
                    "code": {"type": "string", "description": "要执行的 Python 代码"},
                    "timeout": {"type": "integer", "description": "超时秒数，默认10"}
                },
                "required": ["code"],
                "additionalProperties": False
            },
            "strict": True
        }
    },
    {
        "type": "function",
        "function": {
            "name": "finish",
            "description": "任务完成时调用，返回最终答案给用户。",
            "parameters": {
                "type": "object",
                "properties": {
                    "answer": {"type": "string", "description": "最终答案"},
                    "confidence": {
                        "type": "string",
                        "enum": ["high", "medium", "low"],
                        "description": "答案置信度"
                    }
                },
                "required": ["answer", "confidence"],
                "additionalProperties": False
            },
            "strict": True
        }
    }
]

# ========== 工具实现 ==========

def search(query: str, max_results: int = 3) -> str:
    """模拟搜索（实际应接入 Bing/Google/Serper API）"""
    return f"搜索「{query}」的结果：[此处返回真实搜索结果]"

def run_python(code: str, timeout: int = 10) -> str:
    with tempfile.NamedTemporaryFile(mode="w", suffix=".py", delete=False) as f:
        f.write(code)
        tmp_path = f.name

    try:
        result = subprocess.run(
            ["python3", tmp_path],
            capture_output=True,
            text=True,
            timeout=timeout
        )
        output = result.stdout
        if result.returncode != 0:
            output += f"\n[Error]\n{result.stderr}"
        return output or "(无输出)"
    except subprocess.TimeoutExpired:
        return f"[超时] 代码执行超过 {timeout} 秒"
    finally:
        os.unlink(tmp_path)

def execute_tool(name: str, arguments: dict) -> Any:
    print(f"\n  [Tool] {name}({json.dumps(arguments, ensure_ascii=False)})")

    if name == "search":
        result = search(**arguments)
    elif name == "run_python":
        result = run_python(**arguments)
    elif name == "finish":
        return arguments  # 特殊处理
    else:
        result = f"未知工具: {name}"

    print(f"  [Result] {str(result)[:200]}")
    return result

# ========== Agent 主循环 ==========

def run_agent(task: str, max_steps: int = 10) -> str:
    print(f"\n{'='*50}")
    print(f"任务: {task}")
    print('='*50)

    messages = [
        {
            "role": "system",
            "content": (
                "你是一个能力强大的 AI Agent，可以使用工具完成各种任务。\n"
                "策略：先思考任务需要哪些信息，按需调用工具，最后用 finish 返回答案。\n"
                "如果需要计算，优先用 run_python；如果需要最新信息，先用 search。"
            )
        },
        {"role": "user", "content": task}
    ]

    for step in range(max_steps):
        print(f"\n[Step {step + 1}]")

        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=TOOLS,
            tool_choice="auto",
            parallel_tool_calls=True
        )

        choice = response.choices[0]
        messages.append(choice.message.model_dump(exclude_none=True))

        if choice.finish_reason == "stop":
            return choice.message.content

        if choice.finish_reason == "tool_calls":
            tool_results = []

            for tool_call in choice.message.tool_calls:
                name = tool_call.function.name
                args = json.loads(tool_call.function.arguments)
                result = execute_tool(name, args)

                # finish 工具意味着任务完成
                if name == "finish":
                    print(f"\n{'='*50}")
                    print(f"[完成] 置信度: {result['confidence']}")
                    print(f"答案: {result['answer']}")
                    return result["answer"]

                tool_results.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": str(result)
                })

            messages.extend(tool_results)

    return "达到最大步数限制，任务未完成"

# ========== 运行示例 ==========

if __name__ == "__main__":
    # 示例1：数学计算
    result = run_agent("计算 1到100 所有奇数的平方和")

    # 示例2：信息查询（需要真实搜索 API）
    # result = run_agent("Python 3.13 有哪些新特性？")

    # 示例3：复合任务
    # result = run_agent("生成一个包含10个随机数的列表，并计算其均值和标准差")
```

---

## 附录：快速参考

### 常用模型速查

| 模型          | 上下文 | 特点         | 适用场景         |
| ------------- | ------ | ------------ | ---------------- |
| `gpt-4o`      | 128K   | 最强，多模态 | 复杂推理、代码   |
| `gpt-4o-mini` | 128K   | 快速，低成本 | 简单任务、高并发 |
| `o1`          | 200K   | 深度推理     | 数学、科学       |
| `o1-mini`     | 128K   | 推理+低成本  | STEM 任务        |
| `o3-mini`     | 200K   | 最新推理模型 | 复杂推理         |

### 请求模板（Python）

```python
from openai import OpenAI

client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "系统提示"},
        {"role": "user", "content": "用户消息"}
    ],
    temperature=0.7,
    max_tokens=2048,
    stream=False
)

print(response.choices[0].message.content)
print(f"Token 用量: {response.usage.total_tokens}")
```

### 错误处理模板

```python
from openai import OpenAI, RateLimitError, APIError
import time

def safe_call(messages, retries=3):
    for i in range(retries):
        try:
            return client.chat.completions.create(
                model="gpt-4o", messages=messages
            )
        except RateLimitError:
            if i < retries - 1:
                time.sleep(2 ** i)
        except APIError as e:
            if e.status_code >= 500 and i < retries - 1:
                time.sleep(1)
            else:
                raise
```
