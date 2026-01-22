# Langfuse SDK v3 集成指南

本指南详细介绍如何使用 Langfuse SDK v3 来追踪和观测 LLM 应用。Langfuse SDK v3 是基于 OpenTelemetry 原生构建的，提供了开箱即用的 LLM 可观测性功能。

> 📌 本指南同时涵盖 **Python** 和 **JavaScript/TypeScript** 两种语言的使用方式。

## 目录

- [1. 安装与配置](#1-安装与配置)
- [2. 核心概念](#2-核心概念)
- [3. 埋点方式](#3-埋点方式)
  - [3.1 装饰器/包装器模式](#31-装饰器包装器模式)
  - [3.2 上下文管理器模式](#32-上下文管理器模式)
  - [3.3 手动创建观测](#33-手动创建观测)
- [4. 嵌套追踪](#4-嵌套追踪)
- [5. 属性与元数据](#5-属性与元数据)
- [6. 与 OpenAI 集成](#6-与-openai-集成)
- [7. 与 LangChain 集成](#7-与-langchain-集成)
- [8. 评分与反馈](#8-评分与反馈)
- [9. 完整示例](#9-完整示例)
- [10. 常见问题](#10-常见问题)

---

## 1. 安装与配置

### Python 安装

```bash
pip install langfuse
```

### JavaScript/TypeScript 安装

```bash
npm install @langfuse/tracing @langfuse/otel @opentelemetry/sdk-node
# 可选包
npm install @langfuse/client   # API 客户端
npm install @langfuse/openai   # OpenAI 自动埋点
npm install @langfuse/langchain # LangChain 集成
```

### 环境变量配置

```bash
# 必需配置
export LANGFUSE_PUBLIC_KEY="pk-lf-xxx"
export LANGFUSE_SECRET_KEY="sk-lf-xxx"

# 可选配置（默认为 Langfuse Cloud EU）
export LANGFUSE_BASE_URL="https://cloud.langfuse.com"  # EU 区域（默认）
# export LANGFUSE_BASE_URL="https://us.cloud.langfuse.com"  # US 区域
# export LANGFUSE_BASE_URL="http://localhost:3000"  # 本地部署
```

### 代码中配置

<details>
<summary>🐍 Python</summary>

```python
from langfuse import Langfuse

# 方式1：使用环境变量（推荐）
langfuse = Langfuse()

# 方式2：代码中直接配置
langfuse = Langfuse(
    public_key="pk-lf-xxx",
    secret_key="sk-lf-xxx",
    host="https://cloud.langfuse.com"
)
```
</details>

<details>
<summary>📘 JavaScript/TypeScript</summary>

**Step 1: 创建 instrumentation.ts（初始化 OpenTelemetry）**

```typescript
// instrumentation.ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { LangfuseSpanProcessor } from "@langfuse/otel";

export const sdk = new NodeSDK({
  spanProcessors: [new LangfuseSpanProcessor()],
});

sdk.start();
```

**Step 2: 初始化客户端**

```typescript
import { LangfuseClient } from "@langfuse/client";

// 方式1：使用环境变量（推荐）
const langfuse = new LangfuseClient();

// 方式2：代码中直接配置
const langfuse = new LangfuseClient({
  publicKey: "pk-lf-xxx",
  secretKey: "sk-lf-xxx",
  baseUrl: "https://cloud.langfuse.com",
});
```
</details>

---

## 2. 核心概念

### Trace（追踪）
代表一次完整的请求或交互，包含多个 Observation。

### Observation（观测）
Trace 中的单个步骤，有以下类型：

| 类型 | 说明 | 典型用途 |
|------|------|----------|
| **span** | 通用步骤 | 函数调用、处理流程 |
| **generation** | LLM 生成 | GPT/Claude 等模型调用 |
| **event** | 事件 | 用户点击、系统事件 |

### 层级关系

```
Trace
├── Span: "user-request-pipeline"
│   ├── Generation: "gpt-4-call"
│   ├── Span: "data-processing"
│   └── Generation: "summarization"
└── Event: "user-feedback"
```

---

## 3. 埋点方式

Langfuse SDK v3 提供三种埋点方式，适用于不同场景。

### 3.1 装饰器/包装器模式

最简单的方式，自动捕获函数的输入、输出、执行时间和错误。

<details>
<summary>🐍 Python - @observe 装饰器</summary>

#### 基础用法

```python
from langfuse import observe

@observe()
def process_user_query(query: str) -> dict:
    """处理用户查询"""
    result = {"answer": f"处理结果: {query}"}
    return result

# 调用函数时自动创建 trace 和 span
result = process_user_query("什么是 Langfuse?")
```

#### 指定观测类型

```python
from langfuse import observe

# 标记为 generation 类型（用于 LLM 调用）
@observe(name="llm-generation", as_type="generation")
def call_llm(prompt: str) -> str:
    return f"LLM 响应: {prompt}"

# 标记为普通 span
@observe(name="data-processing", as_type="span")
def process_data(data: dict) -> dict:
    return {"processed": True, **data}
```

#### 禁用输入/输出捕获

```python
@observe(capture_input=False, capture_output=False)
def handle_large_data(large_data: bytes) -> bytes:
    return large_data
```
</details>

<details>
<summary>📘 JavaScript/TypeScript - observe 包装器</summary>

#### 基础用法

```typescript
import { observe, updateActiveObservation } from "@langfuse/tracing";

async function fetchData(source: string) {
  updateActiveObservation({ metadata: { source: "API" } });
  return { data: `some data from ${source}` };
}

// 使用 observe 包装函数
const tracedFetchData = observe(fetchData, {
  name: "fetch-data",
  asType: "span",
});

const result = await tracedFetchData("API");
```

#### 指定观测类型

```typescript
import { observe } from "@langfuse/tracing";

// 标记为 generation 类型
const tracedLLMCall = observe(
  async (prompt: string) => {
    return `LLM 响应: ${prompt}`;
  },
  { name: "llm-call", asType: "generation" }
);

// 标记为 span 类型
const tracedProcess = observe(
  (data: any) => ({ processed: true, ...data }),
  { name: "data-processing", asType: "span" }
);
```

#### 禁用输入/输出捕获

```typescript
const tracedFn = observe(myFunction, {
  name: "my-function",
  captureInput: false,
  captureOutput: false,
});
```
</details>

---

### 3.2 上下文管理器模式

提供更灵活的控制，适合需要手动更新观测数据的场景。

<details>
<summary>🐍 Python</summary>

#### 基础用法

```python
from langfuse import get_client

langfuse = get_client()

with langfuse.start_as_current_observation(
    as_type="span",
    name="user-request-pipeline",
    input={"user_query": "告诉我一个笑话"}
) as span:
    # 在这个上下文中执行的所有操作都会被追踪
    span.update(output={"joke": "为什么程序员总是冷？因为他们喜欢 Windows..."})
```

#### 嵌套创建 generation

```python
from langfuse import get_client, propagate_attributes

langfuse = get_client()

with langfuse.start_as_current_observation(
    as_type="span",
    name="chat-workflow",
    input={"user_message": "你好"}
) as root_span:
    
    with propagate_attributes(user_id="user_123", session_id="session_abc"):
        
        with langfuse.start_as_current_observation(
            as_type="generation",
            name="gpt-4-call",
            model="gpt-4",
            input=[{"role": "user", "content": "你好"}]
        ) as generation:
            
            llm_response = "你好！有什么我可以帮你的吗？"
            generation.update(
                output=llm_response,
                usage={"input_tokens": 10, "output_tokens": 15}
            )
    
    root_span.update(output={"response": llm_response})
```
</details>

<details>
<summary>📘 JavaScript/TypeScript</summary>

#### 基础用法

```typescript
import { startActiveObservation } from "@langfuse/tracing";

await startActiveObservation("user-request-pipeline", async (span) => {
  span.update({
    input: { user_query: "告诉我一个笑话" },
  });

  // 执行业务逻辑
  const joke = "为什么程序员总是冷？因为他们喜欢 Windows...";

  span.update({
    output: { joke },
  });
});
```

#### 嵌套创建 generation

```typescript
import {
  startActiveObservation,
  startObservation,
  propagateAttributes,
} from "@langfuse/tracing";

await startActiveObservation("chat-workflow", async (rootSpan) => {
  rootSpan.update({ input: { user_message: "你好" } });

  await propagateAttributes(
    {
      userId: "user_123",
      sessionId: "session_abc",
    },
    async () => {
      const generation = startObservation(
        "gpt-4-call",
        {
          model: "gpt-4",
          input: [{ role: "user", content: "你好" }],
        },
        { asType: "generation" }
      );

      const llmResponse = "你好！有什么我可以帮你的吗？";

      generation.update({
        output: { content: llmResponse },
        usage: { inputTokens: 10, outputTokens: 15 },
      });
      generation.end();

      rootSpan.update({ output: { response: llmResponse } });
    }
  );
});
```
</details>

---

### 3.3 手动创建观测

适合需要完全控制 span 生命周期的场景。

<details>
<summary>🐍 Python</summary>

```python
from langfuse import get_client

langfuse = get_client()

# 创建父 span
parent = langfuse.start_observation(name="manual-parent")

# 创建子 span
child_span = parent.start_observation(name="processing-step")
child_span.update(output={"status": "completed"})
child_span.end()

# 创建子 generation
child_gen = parent.start_observation(
    name="llm-call",
    as_type="generation",
    model="gpt-4"
)
child_gen.update(
    output="LLM 响应内容",
    usage={"input_tokens": 50, "output_tokens": 100}
)
child_gen.end()

# 结束父 span
parent.update(output={"final_result": "success"})
parent.end()

# 确保数据发送完成
langfuse.flush()
```
</details>

<details>
<summary>📘 JavaScript/TypeScript</summary>

```typescript
import { startObservation } from "@langfuse/tracing";
import { sdk } from "./instrumentation";

// 创建父 span
const parent = startObservation("manual-parent");

// 创建子 span（手动方式需要通过参数关联）
const childSpan = startObservation("processing-step", {}, { asType: "span" });
childSpan.update({ output: { status: "completed" } });
childSpan.end();

// 创建子 generation
const childGen = startObservation(
  "llm-call",
  { model: "gpt-4" },
  { asType: "generation" }
);
childGen.update({
  output: "LLM 响应内容",
  usage: { inputTokens: 50, outputTokens: 100 },
});
childGen.end();

// 结束父 span
parent.update({ output: { finalResult: "success" } });
parent.end();

// 确保数据发送完成（短生命周期应用需要）
await sdk.shutdown();
```
</details>

---

## 4. 嵌套追踪

Langfuse 自动处理观测的层级嵌套关系。

<details>
<summary>🐍 Python - 使用装饰器自动嵌套</summary>

```python
from langfuse import observe

@observe()
def main_pipeline(user_input: str):
    """主流程 - 自动成为 root span"""
    preprocessed = preprocess(user_input)
    response = generate_response(preprocessed)
    return postprocess(response)

@observe()
def preprocess(text: str):
    """预处理 - 自动成为 main_pipeline 的子 span"""
    return text.strip().lower()

@observe(as_type="generation")
def generate_response(text: str):
    """生成响应 - 自动成为 main_pipeline 的子 generation"""
    return f"AI 响应: {text}"

@observe()
def postprocess(response: str):
    """后处理 - 自动成为 main_pipeline 的子 span"""
    return {"response": response, "timestamp": "2024-01-01"}

# 调用时自动生成完整的 trace 树
result = main_pipeline("Hello World")
```

**生成的 Trace 结构：**
```
Trace: main_pipeline
├── Span: preprocess
├── Generation: generate_response
└── Span: postprocess
```
</details>

<details>
<summary>📘 JavaScript/TypeScript - 使用 startActiveObservation 嵌套</summary>

```typescript
import { startActiveObservation, startObservation } from "@langfuse/tracing";

async function mainPipeline(userInput: string) {
  return await startActiveObservation("main-pipeline", async (rootSpan) => {
    rootSpan.update({ input: { userInput } });

    // 子 span - preprocess
    const preprocessed = await startActiveObservation(
      "preprocess",
      async (span) => {
        const result = userInput.trim().toLowerCase();
        span.update({ output: { result } });
        return result;
      }
    );

    // 子 generation
    const generation = startObservation(
      "generate-response",
      { model: "gpt-4" },
      { asType: "generation" }
    );
    const response = `AI 响应: ${preprocessed}`;
    generation.update({ output: { response } });
    generation.end();

    // 子 span - postprocess
    const finalResult = await startActiveObservation(
      "postprocess",
      async (span) => {
        const result = { response, timestamp: new Date().toISOString() };
        span.update({ output: result });
        return result;
      }
    );

    rootSpan.update({ output: finalResult });
    return finalResult;
  });
}

// 使用
const result = await mainPipeline("Hello World");
```
</details>

---

## 5. 属性与元数据

### 可用属性

| 属性 | 说明 | 用途 |
|------|------|------|
| `user_id` / `userId` | 用户 ID | 用户级别分析 |
| `session_id` / `sessionId` | 会话 ID | 会话级别分析 |
| `metadata` | 元数据字典 | 自定义键值对 |
| `version` | 应用版本 | 版本追踪 |
| `tags` | 标签列表 | 分类和过滤 |
| `trace_name` / `traceName` | Trace 名称 | 覆盖默认名称 |

<details>
<summary>🐍 Python - 使用 propagate_attributes</summary>

```python
from langfuse import observe, propagate_attributes

@observe()
def handle_user_request(user_id: str, query: str):
    with propagate_attributes(
        user_id=user_id,
        session_id="session_" + user_id,
        metadata={
            "env": "production",
            "feature_flag": "new_model_v2",
            "source": "web_app"
        },
        version="1.2.0",
        tags=["production", "priority-high"]
    ):
        result = process_query(query)
        return result

@observe()
def process_query(query: str):
    # 自动继承 user_id, session_id 等属性
    return {"result": query.upper()}

handle_user_request("user_123", "你好")
```
</details>

<details>
<summary>📘 JavaScript/TypeScript - 使用 propagateAttributes</summary>

```typescript
import {
  startActiveObservation,
  propagateAttributes,
  startObservation,
} from "@langfuse/tracing";

await startActiveObservation("user-workflow", async () => {
  await propagateAttributes(
    {
      userId: "user_123",
      sessionId: "session_abc",
      metadata: { experiment: "variant_a", env: "prod" },
      version: "1.0",
      traceName: "user-workflow",
    },
    async () => {
      const generation = startObservation(
        "llm-call",
        { model: "gpt-4" },
        { asType: "generation" }
      );
      generation.end();
    }
  );
});
```
</details>

---

## 6. 与 OpenAI 集成

<details>
<summary>🐍 Python</summary>

### 方式一：使用 Langfuse 封装的 OpenAI 客户端

```python
from langfuse.openai import openai

# 直接使用，所有调用自动追踪
response = openai.chat.completions.create(
    model="gpt-4",
    messages=[
        {"role": "system", "content": "你是一个有帮助的助手。"},
        {"role": "user", "content": "什么是机器学习？"}
    ]
)

print(response.choices[0].message.content)
```

### 方式二：包装现有 OpenAI 客户端

```python
from openai import OpenAI
from langfuse.openai import wrap_openai

client = OpenAI(api_key="your-api-key")
wrapped_client = wrap_openai(client)

response = wrapped_client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "你好"}]
)
```

### 在 @observe 中使用

```python
from langfuse import observe
from langfuse.openai import openai

@observe()
def chat_with_user(user_message: str):
    response = openai.chat.completions.create(
        model="gpt-4",
        messages=[
            {"role": "system", "content": "你是一个友好的助手。"},
            {"role": "user", "content": user_message}
        ],
        temperature=0.7
    )
    return response.choices[0].message.content

result = chat_with_user("介绍一下 Python")
```
</details>

<details>
<summary>📘 JavaScript/TypeScript</summary>

### 安装

```bash
npm install @langfuse/openai openai
```

### 使用 Langfuse 封装的 OpenAI 客户端

```typescript
import { withTracing } from "@langfuse/openai";
import OpenAI from "openai";

const openai = withTracing(new OpenAI({ apiKey: "your-api-key" }));

const response = await openai.chat.completions.create({
  model: "gpt-4",
  messages: [
    { role: "system", content: "你是一个有帮助的助手。" },
    { role: "user", content: "什么是机器学习？" },
  ],
});

console.log(response.choices[0].message.content);
```

### 在 startActiveObservation 中使用

```typescript
import { startActiveObservation } from "@langfuse/tracing";
import { withTracing } from "@langfuse/openai";
import OpenAI from "openai";

const openai = withTracing(new OpenAI());

await startActiveObservation("chat-with-user", async (span) => {
  span.update({ input: { userMessage: "介绍一下 TypeScript" } });

  const response = await openai.chat.completions.create({
    model: "gpt-4",
    messages: [
      { role: "system", content: "你是一个友好的助手。" },
      { role: "user", content: "介绍一下 TypeScript" },
    ],
    temperature: 0.7,
  });

  const content = response.choices[0].message.content;
  span.update({ output: { response: content } });

  return content;
});
```
</details>

---

## 7. 与 LangChain 集成

<details>
<summary>🐍 Python</summary>

```python
from langfuse.callback import CallbackHandler
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

# 创建 Langfuse 回调处理器
langfuse_handler = CallbackHandler(
    public_key="pk-lf-xxx",
    secret_key="sk-lf-xxx",
    host="https://cloud.langfuse.com"
)

# 创建 LangChain 组件
llm = ChatOpenAI(model="gpt-4", temperature=0.7)
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有帮助的助手。"),
    ("user", "{input}")
])

chain = prompt | llm

# 使用回调追踪
response = chain.invoke(
    {"input": "什么是 LangChain?"},
    config={"callbacks": [langfuse_handler]}
)

print(response.content)
```

### 设置用户和会话信息

```python
langfuse_handler = CallbackHandler(
    user_id="user_123",
    session_id="session_abc",
    metadata={"source": "chatbot"},
    tags=["langchain", "production"]
)
```
</details>

<details>
<summary>📘 JavaScript/TypeScript</summary>

### 安装

```bash
npm install @langfuse/langchain @langchain/core @langchain/openai
```

### 使用

```typescript
import { CallbackHandler } from "@langfuse/langchain";
import { ChatOpenAI } from "@langchain/openai";
import { ChatPromptTemplate } from "@langchain/core/prompts";

// 创建 Langfuse 回调处理器
const langfuseHandler = new CallbackHandler({
  publicKey: "pk-lf-xxx",
  secretKey: "sk-lf-xxx",
  baseUrl: "https://cloud.langfuse.com",
});

// 创建 LangChain 组件
const llm = new ChatOpenAI({ modelName: "gpt-4", temperature: 0.7 });
const prompt = ChatPromptTemplate.fromMessages([
  ["system", "你是一个有帮助的助手。"],
  ["user", "{input}"],
]);

const chain = prompt.pipe(llm);

// 使用回调追踪
const response = await chain.invoke(
  { input: "什么是 LangChain?" },
  { callbacks: [langfuseHandler] }
);

console.log(response.content);
```

### 设置用户和会话信息

```typescript
const langfuseHandler = new CallbackHandler({
  userId: "user_123",
  sessionId: "session_abc",
  metadata: { source: "chatbot" },
  tags: ["langchain", "production"],
});
```
</details>

---

## 8. 评分与反馈

<details>
<summary>🐍 Python</summary>

```python
from langfuse import get_client, get_current_trace_id, observe

langfuse = get_client()

# 直接通过 trace_id 添加评分
langfuse.score(
    trace_id="your-trace-id",
    name="user-feedback",
    value=1,
    comment="回答很有帮助"
)

# 数值评分
langfuse.score(
    trace_id="your-trace-id",
    name="quality-score",
    value=0.85,
    data_type="NUMERIC"
)

# 在追踪中获取 trace_id
@observe()
def my_function():
    trace_id = get_current_trace_id()
    print(f"Trace ID: {trace_id}")
    return {"trace_id": trace_id, "result": "success"}
```
</details>

<details>
<summary>📘 JavaScript/TypeScript</summary>

```typescript
import { LangfuseClient } from "@langfuse/client";
import {
  startActiveObservation,
  getActiveTraceId,
} from "@langfuse/tracing";

const langfuse = new LangfuseClient();

// 直接通过 trace_id 添加评分
await langfuse.score({
  traceId: "your-trace-id",
  name: "user-feedback",
  value: 1,
  comment: "回答很有帮助",
});

// 数值评分
await langfuse.score({
  traceId: "your-trace-id",
  name: "quality-score",
  value: 0.85,
  dataType: "NUMERIC",
});

// 在追踪中获取 trace_id
await startActiveObservation("my-function", async (span) => {
  const traceId = getActiveTraceId();
  console.log(`Trace ID: ${traceId}`);

  span.update({ output: { traceId, result: "success" } });
});
```
</details>

---

## 9. 完整示例

### RAG（检索增强生成）应用示例

<details>
<summary>🐍 Python 完整示例</summary>

```python
from langfuse import observe, get_client, propagate_attributes
from langfuse.openai import openai

langfuse = get_client()

# 模拟知识库
KNOWLEDGE_BASE = {
    "langfuse": "Langfuse 是一个开源的 LLM 可观测性平台。",
    "opentelemetry": "OpenTelemetry 是一个开源的可观测性框架。",
}

@observe(name="search-knowledge", as_type="span")
def search_knowledge_base(query: str) -> list:
    """模拟向量搜索"""
    results = []
    for key, value in KNOWLEDGE_BASE.items():
        if key in query.lower():
            results.append({"topic": key, "content": value, "score": 0.9})
    return results

@observe(name="generate-response", as_type="generation")
def generate_response(query: str, context: list) -> str:
    """使用 LLM 生成响应"""
    context_text = "\n".join([f"- {item['content']}" for item in context])

    response = openai.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": f"基于上下文回答：\n{context_text}"},
            {"role": "user", "content": query}
        ]
    )
    return response.choices[0].message.content

@observe(name="rag-pipeline")
def rag_pipeline(user_id: str, query: str) -> dict:
    """完整的 RAG 流程"""
    with propagate_attributes(
        user_id=user_id,
        metadata={"pipeline": "rag"},
        tags=["rag", "production"]
    ):
        retrieved_docs = search_knowledge_base(query)
        response = generate_response(query, retrieved_docs) if retrieved_docs else "未找到相关信息"

        return {
            "query": query,
            "response": response,
            "sources": [doc["topic"] for doc in retrieved_docs]
        }

if __name__ == "__main__":
    result = rag_pipeline("user_001", "什么是 Langfuse？")
    print(f"问题: {result['query']}")
    print(f"回答: {result['response']}")
    langfuse.flush()
```
</details>

<details>
<summary>📘 JavaScript/TypeScript 完整示例</summary>

```typescript
// instrumentation.ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { LangfuseSpanProcessor } from "@langfuse/otel";

export const sdk = new NodeSDK({
  spanProcessors: [new LangfuseSpanProcessor()],
});
sdk.start();

// main.ts
import { sdk } from "./instrumentation";
import {
  startActiveObservation,
  startObservation,
  propagateAttributes,
} from "@langfuse/tracing";
import { withTracing } from "@langfuse/openai";
import OpenAI from "openai";

const openai = withTracing(new OpenAI());

// 模拟知识库
const KNOWLEDGE_BASE: Record<string, string> = {
  langfuse: "Langfuse 是一个开源的 LLM 可观测性平台。",
  opentelemetry: "OpenTelemetry 是一个开源的可观测性框架。",
};

async function searchKnowledgeBase(query: string) {
  return await startActiveObservation("search-knowledge", async (span) => {
    const results: Array<{ topic: string; content: string; score: number }> = [];

    for (const [key, value] of Object.entries(KNOWLEDGE_BASE)) {
      if (query.toLowerCase().includes(key)) {
        results.push({ topic: key, content: value, score: 0.9 });
      }
    }

    span.update({ output: { results } });
    return results;
  });
}

async function generateResponse(query: string, context: typeof results) {
  const generation = startObservation(
    "generate-response",
    { model: "gpt-3.5-turbo" },
    { asType: "generation" }
  );

  const contextText = context.map((item) => `- ${item.content}`).join("\n");

  const response = await openai.chat.completions.create({
    model: "gpt-3.5-turbo",
    messages: [
      { role: "system", content: `基于上下文回答：\n${contextText}` },
      { role: "user", content: query },
    ],
  });

  const content = response.choices[0].message.content;
  generation.update({ output: { content } });
  generation.end();

  return content;
}

async function ragPipeline(userId: string, query: string) {
  return await startActiveObservation("rag-pipeline", async (span) => {
    span.update({ input: { userId, query } });

    return await propagateAttributes(
      { userId, metadata: { pipeline: "rag" }, tags: ["rag", "production"] },
      async () => {
        const retrievedDocs = await searchKnowledgeBase(query);
        const response = retrievedDocs.length
          ? await generateResponse(query, retrievedDocs)
          : "未找到相关信息";

        const result = {
          query,
          response,
          sources: retrievedDocs.map((doc) => doc.topic),
        };

        span.update({ output: result });
        return result;
      }
    );
  });
}

// 运行
async function main() {
  const result = await ragPipeline("user_001", "什么是 Langfuse？");
  console.log(`问题: ${result.query}`);
  console.log(`回答: ${result.response}`);
}

main().finally(() => sdk.shutdown());
```
</details>

---

## 10. 常见问题

### Q1: 数据没有出现在 Langfuse 控制台？

<details>
<summary>🐍 Python</summary>

```python
from langfuse import get_client
langfuse = get_client()
# ... 你的代码 ...
langfuse.flush()  # 强制发送所有待处理数据
```
</details>

<details>
<summary>📘 JavaScript/TypeScript</summary>

```typescript
import { sdk } from "./instrumentation";
// ... 你的代码 ...
await sdk.shutdown(); // 对于短生命周期应用必须调用
```
</details>

### Q2: 如何在异步代码中使用？

两种语言的 SDK 都原生支持异步操作，装饰器和包装器可以直接用于异步函数。

### Q3: 如何调试连接问题？

<details>
<summary>🐍 Python</summary>

```python
import logging
logging.basicConfig(level=logging.DEBUG)
# 或 export LANGFUSE_DEBUG=true
```
</details>

<details>
<summary>📘 JavaScript/TypeScript</summary>

```typescript
// 设置环境变量
// export LANGFUSE_DEBUG=true
```
</details>

### Q4: Python 和 JS SDK 的主要区别？

| 特性 | Python | JavaScript/TypeScript |
|------|--------|----------------------|
| OpenTelemetry 初始化 | 自动 | 需手动配置 NodeSDK |
| 装饰器语法 | `@observe()` | `observe(fn, options)` |
| 上下文管理器 | `with ... as span:` | `await startActiveObservation(name, async (span) => {})` |
| 属性传播 | `with propagate_attributes():` | `await propagateAttributes({}, async () => {})` |
| 生命周期管理 | `langfuse.flush()` | `sdk.shutdown()` |

---

## 参考链接

- [Langfuse 官方文档](https://langfuse.com/docs)
- [Langfuse Python SDK 参考](https://python.reference.langfuse.com)
- [Langfuse JS SDK 参考](https://langfuse-js.vercel.app)
- [OpenTelemetry 官方文档](https://opentelemetry.io/docs)
- [Langfuse GitHub](https://github.com/langfuse/langfuse)

---

## 许可证

MIT License