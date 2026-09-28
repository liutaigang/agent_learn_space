# 阶段 2：掌握 LLM 应用基础

## 学习目标

能够独立开发一个基础 LLM 应用。

## 核心内容

### 一. LLM API 调用

下面从**工程接入**视角比较 Gemini API 与 OpenAI、Anthropic 的差异：

- 核心接口形态
- 上下文管理
- 结构化输出/JSON Schema 
- 工具调用
- 文件与多模态
- 流式输出
- 缓存和生态兼容

#### 1. 核心接口形态差异

| 维度   | OpenAI                            | Anthropic                   | Gemini                                    |
| ---- | --------------------------------- | --------------------------- | ----------------------------------------- |
| 主接口  | `responses.create()`              | `messages.create()`         | `models.generate_content()`               |
| 对话接口 | Responses + Conversations         | Messages                    | `chats.create()`                          |
| 消息单位 | input / message / content         | message + content block     | content + part                            |
| 角色命名 | `system/developer/user/assistant` | `system` + `user/assistant` | `user/model`，另有 `systemInstruction`       |
| 原生生态 | OpenAI SDK / Agents SDK           | Anthropic SDK / Claude Code | Google Gen AI SDK / AI Studio / Vertex AI |

Gemini 的最大区别在于：它不是以 OpenAI 的 `messages` 或 Anthropic 的 `messages` 为核心，而是以：

```text
contents = [
  {
    role: "user",
    parts: [
      { text: "..." },
      { file_data: ... },
      { inline_data: ... }
    ]
  }
]
```

为核心。

也就是说，Gemini 的底层抽象不是“聊天消息”，而是**多模态内容块**。

#### 2. 多轮对话上下文

##### OpenAI

OpenAI 可以手动传历史，也可以用 Responses API + Conversations API 持久化会话状态。Conversations API 会把 message、tool call、tool output 等作为 conversation items 保存。([OpenAI Developers][2])

```python
from openai import OpenAI

client = OpenAI()

r1 = client.responses.create(
    model="gpt-5.5",
    input="我有 2 只狗。"
)

r2 = client.responses.create(
    model="gpt-5.5",
    previous_response_id=r1.id,
    input="那一共有几只脚？"
)

print(r2.output_text)
```

##### Anthropic

Anthropic Messages API 默认无状态，每次都要传完整历史。([Claude API Docs][3])

```python
import anthropic

client = anthropic.Anthropic()

messages = [
    {"role": "user", "content": "我有 2 只狗。"},
    {"role": "assistant", "content": "好的，你有 2 只狗。"},
    {"role": "user", "content": "那一共有几只脚？"},
]

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=messages,
)

print(response.content[0].text)
```

##### Gemini

Gemini 有原生 `chat` 对象，可以创建 chat session，并传入初始 `history`；官方示例中使用 `client.chats.create(..., history=[Content(role="user"), Content(role="model")])`，再连续 `send_message()`。([Google AI for Developers][1])

```python
from google import genai
from google.genai import types

client = genai.Client()

chat = client.chats.create(
    model="gemini-3.5-flash",
    history=[
        types.Content(
            role="user",
            parts=[types.Part(text="我有 2 只狗。")]
        ),
        types.Content(
            role="model",
            parts=[types.Part(text="好的，你有 2 只狗。")]
        ),
    ],
)

response = chat.send_message(
    message="那一共有几只脚？"
)

print(response.text)
```

**关键差异：**

Gemini 的角色是 `user/model`，不是 `user/assistant`。这会影响你做统一 Adapter 时的消息转换。

```text
OpenAI assistant  -> Gemini model
Anthropic assistant -> Gemini model
Gemini model -> OpenAI/Anthropic assistant
```



#### 3. 结构化输出与 JSON Schema

##### OpenAI

OpenAI 的结构化输出主要通过 `text.format` + `json_schema`。OpenAI 文档说明 Structured Outputs 可用于生成符合 schema 的输出，并且 streaming 场景也可以处理结构化输出。([OpenAI Developers][4])

```python
from openai import OpenAI

client = OpenAI()

response = client.responses.create(
    model="gpt-5.5",
    input="我想去成都玩 3 天，不想太累。",
    text={
        "format": {
            "type": "json_schema",
            "name": "trip_intent",
            "strict": True,
            "schema": {
                "type": "object",
                "properties": {
                    "city": {"type": "string"},
                    "days": {"type": "integer"},
                    "relaxed": {"type": "boolean"}
                },
                "required": ["city", "days", "relaxed"],
                "additionalProperties": False
            }
        }
    }
)

print(response.output_text)
```

##### Anthropic

Anthropic 可以用 structured outputs，也可以通过 tool use 的 `input_schema` 来约束输出。Claude Messages API 更常见的工程做法是：把“我要的 JSON”建模成一个 tool input，让 Claude 调用这个 tool。

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "我想去成都玩 3 天，不想太累。"}
    ],
    tools=[
        {
            "name": "extract_trip_intent",
            "description": "抽取旅行意图",
            "input_schema": {
                "type": "object",
                "properties": {
                    "city": {"type": "string"},
                    "days": {"type": "integer"},
                    "relaxed": {"type": "boolean"}
                },
                "required": ["city", "days", "relaxed"]
            }
        }
    ]
)
```

##### Gemini

Gemini 的结构化输出是通过 `response_mime_type="application/json"` + `response_schema`。Google 文档明确说明 Gemini 可以配置模型生成符合 JSON Schema 的响应，适合数据抽取、分类、Agent workflows。([Google AI for Developers][5])

```python
from google import genai
from google.genai import types
from typing_extensions import TypedDict

class TripIntent(TypedDict):
    city: str
    days: int
    relaxed: bool

client = genai.Client()

response = client.models.generate_content(
    model="gemini-3.5-flash",
    contents="我想去成都玩 3 天，不想太累。",
    config=types.GenerateContentConfig(
        response_mime_type="application/json",
        response_schema=TripIntent,
    ),
)

print(response.text)
```

官方示例也展示了用 `response_mime_type="application/json"` 和 `response_schema=list[Recipe]` 控制输出。([Google AI for Developers][1])

**关键差异：**

| 维度        | OpenAI                 | Anthropic                      | Gemini                                 |
| --------- | ---------------------- | ------------------------------ | -------------------------------------- |
| JSON 输出入口 | `text.format`          | `output_config` 或 tool schema  | `response_mime_type + response_schema` |
| Schema 风格 | JSON Schema            | JSON Schema                    | Python 类型 / Schema / JSON Schema       |
| 工程体验      | 强 schema 约束清晰          | 常和 tool use 结合                 | 和 Python 类型集成较自然                       |
| 常见返回      | `response.output_text` | `content[0].text` 或 tool input | `response.text`                        |

---

#### 4. 工具调用 / Function Calling

三家都支持 function calling，但抽象方式不同。

| 维度     | OpenAI                                        | Anthropic                            | Gemini                                                                          |
| ------ | --------------------------------------------- | ------------------------------------ | ------------------------------------------------------------------------------- |
| 工具定义   | `tools=[{"type":"function", ...}]`            | `tools=[{"name", "input_schema"}]`   | `tools=[function]` 或 `function_declarations`                                    |
| 工具调用结果 | response item / tool call                     | `tool_use` block                     | function call part                                                              |
| 内置工具生态 | web search、file search、code interpreter、MCP 等 | web search、code execution、bash、MCP 等 | Google Search grounding、URL Context、Code Execution、Maps grounding、File Search 等 |
| 风格     | Agent 平台化                                     | Claude content block 化               | Google 多模态 + 工具配置化                                                              |

Gemini 官方文档说 function calling 让模型决定何时调用外部函数，并生成调用参数。([Google AI for Developers][6]) Gemini 的 Python SDK 还支持直接把 Python 函数放进 `tools`，官方示例是 `tools=[add, subtract, multiply, divide]`。([Google AI for Developers][1])

```python
from google import genai
from google.genai import types

client = genai.Client()

def get_weather(city: str, date: str) -> str:
    """查询指定城市和日期的天气"""
    return f"{city} 在 {date} 天气晴朗"

chat = client.chats.create(
    model="gemini-3.5-flash",
    config=types.GenerateContentConfig(
        tools=[get_weather]
    ),
)

response = chat.send_message(
    message="帮我查一下成都明天的天气"
)

print(response.text)
```

Gemini 的优势是 SDK 可以直接从函数签名和 docstring 推断工具定义；OpenAI 和 Anthropic 更常见的是显式写 JSON Schema。

#### 5. 多模态与文件输入

这是 Gemini 很明显的特色。

Google 文档中 Gemini `generateContent` 支持 images、audio、code、tools 等能力。([Google AI for Developers][1]) 官方示例中也展示了上传 PDF 后，将 `file_uri` 作为 `file_data` 传给模型进行总结。([Google AI for Developers][1])

Gemini 的输入天然是 `parts`，所以一个请求里可以混合：

```text
text part
image part
audio part
video part
PDF file part
function call part
code execution part
```

示意：

```python
from google import genai
from google.genai import types

client = genai.Client()

file = client.files.upload(file="report.pdf")

response = client.models.generate_content(
    model="gemini-3.5-flash",
    contents=[
        types.Content(
            role="user",
            parts=[
                types.Part(text="总结这个 PDF 的核心内容"),
                types.Part.from_uri(
                    file_uri=file.uri,
                    mime_type=file.mime_type
                ),
            ],
        )
    ],
)

print(response.text)
```

**对比理解：**

| 能力        | OpenAI        | Anthropic      | Gemini                           |
| --------- | ------------- | -------------- | -------------------------------- |
| 图片理解      | 强             | 强              | 强                                |
| PDF / 文档  | 通过文件/工具体系     | Claude 文档理解很强  | 原生 file part 很自然                 |
| 音频 / 视频   | 强，尤其 Realtime | 音频通常需外部转写或特定能力 | 原生多模态入口更统一                       |
| 图像/视频生成   | OpenAI 图像强    | 不是主打           | Imagen / Veo 生态更强                |
| Google 生态 | 一般            | 一般             | Search、Maps、Vertex、AI Studio 集成强 |


#### 6. 流式输出

##### OpenAI

OpenAI streaming 基于 SSE，Responses API 使用 typed semantic events。OpenAI 文档说明 streaming 可以边生成边处理输出。([OpenAI Developers][7])

```python
from openai import OpenAI

client = OpenAI()

stream = client.responses.create(
    model="gpt-5.5",
    input="写一个 Agent Runtime 的简短说明。",
    stream=True,
)

for event in stream:
    if event.type == "response.output_text.delta":
        print(event.delta, end="", flush=True)
```

##### Anthropic

Anthropic streaming 也是 SSE，但事件模型围绕 `message_start`、`content_block_delta`、`message_delta` 等 Claude content block。

```python
import anthropic

client = anthropic.Anthropic()

with client.messages.stream(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "写一个 Agent Runtime 的简短说明。"}
    ],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

##### Gemini

Gemini 用 `generate_content_stream()`。Google 官方示例中使用 `client.models.generate_content_stream(...)`，然后遍历 chunk，打印 `chunk.text`。([Google AI for Developers][1])

```python
from google import genai

client = genai.Client()

response = client.models.generate_content_stream(
    model="gemini-3.5-flash",
    contents="写一个 Agent Runtime 的简短说明。",
)

for chunk in response:
    print(chunk.text, end="", flush=True)
```

**关键差异：**

```text
OpenAI:
  事件类型丰富，适合复杂 Agent UI、tool call、code interpreter、file search 进度展示

Anthropic:
  content block 事件清晰，适合 Claude 原生 tool_use / thinking / message block

Gemini:
  chunk.text 使用简单，多模态流式入口统一；复杂事件抽象不如 OpenAI Responses 那么平台化
```

---

#### 7. 上下文缓存 / Prompt Caching

| 维度   | OpenAI            | Anthropic                    | Gemini            |
| ---- | ----------------- | ---------------------------- | ----------------- |
| 缓存风格 | 自动 prompt caching | 显式 cache control 更细          | 显式 cached content |
| 适合场景 | 重复长前缀             | 长 system prompt / 长文档 / 工具定义 | 长文档、视频、音频、知识上下文   |
| 工程控制 | 较少                | 较强                           | 较强                |

Gemini 的显式缓存叫 **Context Caching**：可以先把内容缓存起来，后续请求通过 `cachedContent` 引用，避免重复传同一大段上下文。Google 文档说明，可以先传一次内容给模型缓存，后续请求引用 cached tokens；在一定规模下，这比反复传相同语料更低成本。([Google AI for Developers][8]) Gemini API 请求里也有 `cachedContent` 字段，格式是 `cachedContents/{cachedContent}`。([Google AI for Developers][1])

这点和 Anthropic 很像：都偏**显式缓存控制**。OpenAI 则更偏自动缓存。

#### 8. OpenAI 兼容性

Gemini 支持 OpenAI 兼容接口，可以用 OpenAI SDK 调 Gemini，只需改 API key、base_url、model。Google 文档明确说 Gemini models 可以通过 OpenAI libraries 访问，但如果还没有使用 OpenAI libraries，建议直接调用 Gemini API。([Google AI for Developers][9])

```python
from openai import OpenAI

client = OpenAI(
    api_key="GEMINI_API_KEY",
    base_url="https://generativelanguage.googleapis.com/v1beta/openai/"
)

response = client.chat.completions.create(
    model="gemini-3.5-flash",
    messages=[
        {"role": "system", "content": "你是一个技术助手。"},
        {"role": "user", "content": "解释一下 Gemini API。"}
    ]
)

print(response.choices[0].message)
```

这对迁移很有用，但工程上要注意：

```text
OpenAI 兼容层 ≠ 完全等价于 OpenAI Responses API
```

如果你要充分使用 Gemini 的多模态 `parts`、Google Search grounding、URL Context、Maps grounding、cached content 等能力，最好用 Gemini 原生 SDK。

---

#### 10. 统一 Adapter 设计建议

如果你要做自己的 **Agent Runtime / Skill Scheduler**，建议不要直接绑定三家的原始结构，而是做统一抽象：

```text
LLMRequest
- system_instruction
- messages
- contents
- tools
- response_schema
- stream
- cache_policy
- reasoning_config
- files
- metadata
```

然后分别适配：

```text
OpenAIAdapter
- messages/input -> responses.create
- schema -> text.format
- tools -> tools
- state -> previous_response_id / conversation_id

AnthropicAdapter
- messages -> messages.create
- schema -> output_config 或 tool input_schema
- tools -> tool_use/tool_result
- state -> 业务侧 messages history

GeminiAdapter
- messages -> contents/parts
- assistant role -> model role
- schema -> response_mime_type + response_schema
- tools -> GenerateContentConfig(tools=...)
- cache -> cachedContent
```

最关键的是角色转换：

```text
OpenAI / Anthropic:
  assistant

Gemini:
  model
```

以及内容转换：

```text
OpenAI / Anthropic:
  message.content

Gemini:
  content.parts[]
```


### 二. Prompt 设计
[如何设计 Agent 场景下的单个 Prompt 模板](./references/如何设计%20Agent%20场景下的单个%20Prompt%20模板.md)

### 三. Prompt 测试

### 四. 错误重试与异常处理
[LLM API 调用中的错误重试与异常处理](./references/LLM%20API%20调用中的错误重试与异常处理.md)
