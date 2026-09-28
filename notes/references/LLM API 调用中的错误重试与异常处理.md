# LLM API 调用中的错误重试与异常处理

LLM API 的错误处理，不能简单套用普通 HTTP 接口的“失败就重试”。因为 LLM 调用有几个特殊点：

1. **成本高**：每次重试都会消耗 token 和时间。
2. **结果不确定**：同一个请求重试，返回结果可能不同。
3. **可能有副作用**：如果模型触发了 tool calling，重试可能导致工具重复执行。
4. **链路长**：一次 Agent 调用可能包含多次 LLM 调用、工具调用、JSON 校验、状态流转。
5. **失败类型复杂**：网络失败、限流、模型超载、上下文超长、结构化输出失败、工具失败，都不能用同一种策略处理。

所以，LLM API 的异常处理核心不是“多重试几次”，而是：

> **先分类，再决策：哪些错误可以重试，哪些错误应该降级，哪些错误必须直接失败。**

---

## 一、LLM API 常见错误类型

### 1. 网络层错误

例如：

* DNS 失败
* 连接超时
* 读超时
* 连接被重置
* TLS 错误

这类错误通常是**临时性错误**，可以重试。

处理方式：

```text
网络错误 → 指数退避重试 → 超过次数后返回系统繁忙
```

---

### 2. 超时错误

LLM API 超时比较常见，原因可能是：

* prompt 太长
* 模型推理慢
* 服务端排队
* 流式响应中断
* 网络波动

超时可以重试，但要注意：

| 场景          |    是否重试 | 说明            |
| ----------- | ------: | ------------- |
| 请求还没发出去     |      可以 | 没有产生真实调用      |
| 请求已发送但无响应   |      谨慎 | 服务端可能已经在生成    |
| tool 已经执行过  |  不能简单重试 | 可能导致副作用重复     |
| 流式响应已经输出给用户 | 不建议直接重试 | 用户可能看到重复或冲突内容 |

---

### 3. 429 限流错误

常见原因：

* QPS 超限
* TPM / RPM 超限
* 并发请求过多
* 当前模型资源紧张

处理方式：

```text
429 → 读取 retry-after → 等待 → 重试
```

如果没有 `retry-after`，使用指数退避：

```text
1s → 2s → 4s → 8s
```

并增加随机抖动，避免所有请求同时重试，形成“重试风暴”。

---

### 4. 5xx 服务端错误

例如：

* 500 Internal Server Error
* 502 Bad Gateway
* 503 Service Unavailable
* 504 Gateway Timeout

这类通常可以重试。

但是不要无限重试，建议设置：

```text
max_retries = 2 ~ 3
total_timeout = 30s ~ 90s
```

LLM 场景下，超过 3 次重试通常收益不高，反而会增加延迟和成本。

---

### 5. 4xx 客户端错误

这类大部分**不应该重试**。

| 错误类型                     | 是否重试 | 处理方式          |
| ------------------------ | ---: | ------------- |
| 400 参数错误                 |    否 | 修复请求参数        |
| 401 认证失败                 |    否 | 检查 API Key    |
| 403 权限不足                 |    否 | 检查模型权限 / 账号权限 |
| 404 模型不存在                |    否 | 检查 model name |
| 413 请求过大                 |    否 | 压缩上下文         |
| context length exceeded  |    否 | 截断、摘要、RAG     |
| content policy violation |    否 | 走安全拒答或改写请求    |
| invalid JSON schema      |    否 | 修复 schema     |

这类错误重试没有意义，因为问题不在“临时失败”，而在“请求本身不合法”。

---

## 二、错误处理的核心分类

可以把 LLM API 错误分成三类。

### 1. 可重试错误

适合重试：

```text
网络波动
连接超时
读超时
429 限流
500 / 502 / 503 / 504
模型临时不可用
```

处理方式：

```text
retry with exponential backoff + jitter
```

---

### 2. 不可重试错误

不应该重试：

```text
API Key 错误
权限不足
模型不存在
参数错误
上下文超长
JSON Schema 不合法
余额不足
请求内容违反安全策略
```

处理方式：

```text
直接失败，并返回明确错误
```

---

### 3. 可修复后重试错误

这类很重要，尤其在 Agent 和结构化输出中经常遇到。

例如：

```text
模型输出不是合法 JSON
JSON 字段缺失
字段类型错误
tool_call 参数不符合 schema
回答不满足业务约束
```

这类不是 API 调用失败，而是**模型输出失败**。

处理方式不是简单重试原请求，而是：

```text
输出校验失败 → 构造修复提示词 → 要求模型修正 → 再校验
```

例如：

```text
模型返回：
{
  "name": "张三",
  "age": "二十岁"
}

Schema 要求：
age: integer

修复提示：
你的输出不符合 JSON Schema。
字段 age 应该是 integer，但你输出了 string。
请只返回修正后的 JSON。
```

这属于 **semantic retry**，也就是语义重试。

---

## 三、推荐的重试策略

### 1. 指数退避

不要固定间隔重试，例如每次都等 1 秒。

推荐：

```text
第 1 次失败：等待 1s
第 2 次失败：等待 2s
第 3 次失败：等待 4s
第 4 次失败：等待 8s
```

再加随机抖动：

```text
sleep = base * 2^attempt + random(0, 1)
```

这样可以避免大量请求在同一时间重新打到服务端。

---

### 2. 设置最大重试次数

建议：

| 场景        |  建议重试次数 |
| --------- | ------: |
| 普通聊天      | 1 ~ 2 次 |
| 后台任务      | 2 ~ 3 次 |
| 批处理任务     | 3 ~ 5 次 |
| 实时 Agent  | 1 ~ 2 次 |
| 用户正在等待的请求 |    不宜过多 |

不要为了“成功率”无限重试。
LLM 系统更应该关注：

```text
成功率 + 延迟 + 成本 + 可控性
```

---

### 3. 设置总超时时间

不要只设置单次请求超时，还要设置整个调用链的总超时。

例如：

```text
单次请求超时：30s
最大重试次数：2
总调用预算：60s
```

否则可能出现：

```text
第一次调用 30s 超时
第二次调用 30s 超时
第三次调用 30s 超时

用户等了 90s，体验极差
```

更合理的是：

```text
本次任务最多允许消耗 60s
超过后直接失败或降级
```

---

### 4. 区分 retry 和 fallback

重试是：

```text
同一个模型，再试一次
```

降级是：

```text
换一个模型 / 换一个供应商 / 换一个策略
```

例如：

```text
gpt-4.1 调用失败
→ 重试 1 次
→ 仍失败
→ fallback 到 gpt-4.1-mini
→ 仍失败
→ 返回系统繁忙
```

也可以按供应商降级：

```text
OpenAI 失败
→ Anthropic
→ Gemini
→ 本地模型
```

但是要注意：不同模型的输出格式、tool calling 能力、JSON 遵循能力可能不同，不能盲目切换。

---

## 四、异常处理分层设计

建议把 LLM 调用分成四层处理。

```text
业务代码
  ↓
LLM Service / LLM Gateway
  ↓
Provider Adapter
  ↓
具体 SDK / HTTP Client
```

---

### 1. Provider Adapter 层

这一层负责适配不同模型供应商。

例如：

```text
OpenAIAdapter
AnthropicAdapter
GeminiAdapter
LocalModelAdapter
```

职责：

```text
封装 SDK 调用
统一请求格式
统一响应格式
统一错误类型
```

不要让业务代码直接感知不同供应商的异常类型。

---

### 2. LLM Gateway 层

这一层是核心。

职责：

```text
错误分类
重试策略
超时控制
fallback 策略
限流控制
日志记录
token 统计
成本统计
trace_id 生成
```

业务层不应该到处写：

```python
try:
    response = client.chat.completions.create(...)
except Exception:
    ...
```

而应该统一走：

```python
llm_gateway.generate(request)
```

---

### 3. 业务层

业务层只关心：

```text
这次任务成功了吗？
输出是否满足业务要求？
失败后给用户什么反馈？
```

例如：

```text
行程规划失败 → 提示用户稍后重试
代码生成失败 → 返回明确错误
JSON 校验失败 → 要求模型修复
工具调用失败 → 转人工或中断
```

---

### 4. Workflow / Agent 层

如果是 Agent，不要只在单次 LLM API 层做异常处理，还要在流程节点层处理。

例如：

```text
intent_node 失败 → 返回无法识别意图
planner_node 失败 → 降级为简单计划
tool_node 失败 → 告诉用户工具暂时不可用
final_node 失败 → 尝试重新生成最终回答
```

Agent 中的异常处理应该是**节点级别的**，而不是整个 graph 一失败就崩掉。

---

## 五、推荐的错误决策表

| 错误              |      是否重试 | 是否降级 | 是否需要人工介入 | 处理方式           |
| --------------- | --------: | ---: | -------: | -------------- |
| 网络错误            |         是 |   可选 |        否 | 指数退避重试         |
| 请求超时            |         是 |   可选 |        否 | 重试或换快模型        |
| 429 限流          |         是 |    是 |        否 | 等待或切换模型        |
| 500/502/503/504 |         是 |    是 |        否 | 重试后 fallback   |
| API Key 无效      |         否 |    否 |        是 | 配置修复           |
| 模型不存在           |         否 |    是 |        是 | 检查模型路由         |
| 上下文超长           |         否 |    是 |        否 | 压缩上下文          |
| JSON 输出错误       | 是，但不是原样重试 |    否 |        否 | 语义修复重试         |
| Tool 参数错误       |   是，修复后重试 |    否 |       可选 | schema 校验 + 修复 |
| Tool 执行失败       |       视情况 |   可选 |       可选 | 看是否有副作用        |
| 内容安全拒绝          |         否 |    否 |        否 | 返回安全提示         |
| 余额不足            |         否 |    否 |        是 | 账户处理           |

---

## 六、结构化输出场景下的异常处理

对于 AI Coding、Agent、Workflow 来说，结构化输出失败非常常见。

推荐流程：

```text
LLM 输出
  ↓
JSON parse
  ↓
Pydantic 校验
  ↓
业务规则校验
  ↓
成功返回
```

如果失败：

```text
JSON 解析失败 → 要求模型修复 JSON
Pydantic 校验失败 → 告诉模型具体字段错误
业务规则失败 → 告诉模型违反了哪个业务约束
```

不要简单地：

```text
失败 → 重新问一遍
```

而应该把错误反馈给模型：

```text
你的输出不符合要求：
1. 字段 task_id 缺失
2. 字段 priority 应该是 enum: low | medium | high
3. 字段 deadline 应该是 YYYY-MM-DD

请只返回修正后的 JSON。
```

这比重新调用原始 prompt 更稳定。

---

## 七、Tool Calling 场景下的特殊处理

Tool Calling 是 LLM 重试中最容易出事故的地方。

例如模型要调用：

```text
create_order
send_email
delete_file
submit_payment
create_calendar_event
```

这些工具都有副作用。

如果 LLM API 调用超时，你不知道服务端是否已经生成了 tool call。
如果你直接重试，可能导致：

```text
重复创建订单
重复发送邮件
重复提交支付
重复删除文件
```

所以，Tool Calling 场景必须引入：

```text
idempotency_key
request_id
tool_call_id
checkpoint
state machine
```

推荐做法：

```text
LLM 生成 tool call
  ↓
校验 tool 参数
  ↓
生成 tool_call_id
  ↓
检查是否已执行
  ↓
未执行才执行工具
  ↓
保存工具结果
  ↓
继续后续流程
```

对于有副作用的工具，永远不要只依赖“重试次数”。

---

## 八、流式响应的异常处理

Streaming 比普通请求更复杂。

常见问题：

```text
流中断
部分 token 已输出
JSON 没输出完整
工具调用参数没输出完整
用户已经看到部分内容
```

处理建议：

### 普通文本流式输出

如果中途中断，可以提示：

```text
响应中断了，我可以继续补全。
```

不要无感重新生成，因为可能前后内容不一致。

---

### JSON 流式输出

不建议直接把 JSON 流式输出给业务消费。

更稳妥的方式：

```text
先内部完整接收
  ↓
JSON parse
  ↓
校验成功
  ↓
再返回业务系统
```

如果必须流式处理 JSON，要做 buffer 和完整性检测。

---

### Tool Calling 流式输出

工具调用参数没有完整生成前，不要执行工具。

```text
等待完整 tool_call
  ↓
解析参数
  ↓
schema 校验
  ↓
确认幂等
  ↓
执行工具
```

---

## 九、一个可落地的 Python 骨架

下面是一个简化版设计，重点是体现结构。

```python
import time
import random
from enum import Enum
from typing import Any, Optional


class ErrorType(str, Enum):
    RETRYABLE = "retryable"
    NON_RETRYABLE = "non_retryable"
    VALIDATION_RETRYABLE = "validation_retryable"
    FALLBACKABLE = "fallbackable"


class LLMCallError(Exception):
    def __init__(
        self,
        message: str,
        error_type: ErrorType,
        provider: Optional[str] = None,
        status_code: Optional[int] = None,
        raw_error: Optional[Exception] = None,
    ):
        super().__init__(message)
        self.error_type = error_type
        self.provider = provider
        self.status_code = status_code
        self.raw_error = raw_error


def classify_error(error: Exception) -> ErrorType:
    """
    实际项目中，这里应该根据：
    1. HTTP status code
    2. SDK exception type
    3. error code
    4. provider response body
    进行分类。
    """

    message = str(error).lower()

    if "timeout" in message:
        return ErrorType.RETRYABLE

    if "rate limit" in message or "429" in message:
        return ErrorType.RETRYABLE

    if "500" in message or "502" in message or "503" in message or "504" in message:
        return ErrorType.RETRYABLE

    if "context length" in message:
        return ErrorType.NON_RETRYABLE

    if "unauthorized" in message or "401" in message:
        return ErrorType.NON_RETRYABLE

    if "permission" in message or "403" in message:
        return ErrorType.NON_RETRYABLE

    return ErrorType.NON_RETRYABLE


def backoff_sleep(attempt: int, base: float = 1.0, max_sleep: float = 8.0):
    delay = min(base * (2 ** attempt), max_sleep)
    jitter = random.uniform(0, 0.5)
    time.sleep(delay + jitter)


class LLMGateway:
    def __init__(self, client: Any, max_retries: int = 2):
        self.client = client
        self.max_retries = max_retries

    def generate(self, request: dict) -> dict:
        last_error = None

        for attempt in range(self.max_retries + 1):
            try:
                return self._call_llm(request)

            except Exception as e:
                error_type = classify_error(e)
                last_error = e

                if error_type == ErrorType.NON_RETRYABLE:
                    raise LLMCallError(
                        message=f"LLM request failed with non-retryable error: {e}",
                        error_type=error_type,
                        raw_error=e,
                    )

                if attempt >= self.max_retries:
                    break

                backoff_sleep(attempt)

        raise LLMCallError(
            message=f"LLM request failed after retries: {last_error}",
            error_type=ErrorType.RETRYABLE,
            raw_error=last_error,
        )

    def _call_llm(self, request: dict) -> dict:
        """
        这里替换成具体 SDK 调用。
        例如 OpenAI / Anthropic / Gemini / LiteLLM。
        """
        response = self.client.call(request)
        return response
```

---

## 十、结构化输出修复重试示例

```python
from pydantic import BaseModel, ValidationError


class TaskOutput(BaseModel):
    task_id: str
    title: str
    priority: str
    done: bool


def parse_and_validate(raw_text: str) -> TaskOutput:
    return TaskOutput.model_validate_json(raw_text)


def generate_with_validation_retry(llm_gateway: LLMGateway, prompt: str, max_fix_retries: int = 2):
    raw_output = llm_gateway.generate({
        "prompt": prompt,
        "response_format": "json"
    })["content"]

    for attempt in range(max_fix_retries + 1):
        try:
            return parse_and_validate(raw_output)

        except ValidationError as e:
            if attempt >= max_fix_retries:
                raise

            fix_prompt = f"""
你上一次的输出不符合要求。

校验错误如下：
{e}

原始输出如下：
{raw_output}

请只返回修正后的 JSON，不要输出任何解释。
"""

            raw_output = llm_gateway.generate({
                "prompt": fix_prompt,
                "response_format": "json"
            })["content"]
```

这个模式非常适合：

```text
Agent 输出
AI Coding 任务规划
需求结构化
Tool 参数生成
JSON Schema 输出
```

---

## 十一、Agent / Workflow 中的异常处理

在 Agent 里，不建议只做全局 try-catch。

更好的方式是每个节点都有自己的异常处理策略。

例如：

```text
IntentNode
  - 失败：返回 intent_unknown

PlannerNode
  - 失败：重试一次
  - 仍失败：降级为 simple_plan

ToolNode
  - 参数错误：让模型修复参数
  - 工具失败：根据工具类型决定是否重试
  - 有副作用工具：必须检查幂等

FinalAnswerNode
  - 输出失败：重新生成最终回答
```

可以抽象成：

```text
node_error_policy:
  retryable_errors:
    - timeout
    - rate_limit
    - server_error
  max_retries: 2
  fallback_node: simple_planner
  on_failure: return_error
```

---

## 十二、日志与可观测性

LLM 调用的异常处理一定要配合日志，否则线上很难排查。

建议每次调用记录：

```text
trace_id
request_id
user_id
session_id
model
provider
prompt_tokens
completion_tokens
total_tokens
latency_ms
retry_count
error_type
error_code
status_code
fallback_used
temperature
response_format
tool_call_id
```

但要注意：

```text
不要默认完整记录用户 prompt
不要记录敏感信息
不要记录 API Key
生产环境需要脱敏
```

---

## 十三、推荐的最终处理流程

可以用这个流程作为标准：

```text
构造请求
  ↓
检查上下文长度
  ↓
调用 LLM API
  ↓
捕获异常
  ↓
错误分类
  ↓
判断是否可重试
  ↓
指数退避重试
  ↓
必要时 fallback
  ↓
解析输出
  ↓
结构化校验
  ↓
业务规则校验
  ↓
必要时语义修复重试
  ↓
返回结果
```

---

## 十四、核心原则总结

LLM API 的错误重试与异常处理，核心原则是：

1. **不是所有错误都能重试**
   参数错误、权限错误、上下文超长，重试没有意义。

2. **重试必须有上限**
   控制最大次数、最大耗时、最大成本。

3. **使用指数退避和随机抖动**
   避免重试风暴。

4. **区分 API 重试和语义重试**
   API 失败是技术问题，JSON 校验失败是输出质量问题。

5. **Tool Calling 必须考虑幂等性**
   有副作用的工具不能盲目重试。

6. **Agent 要做节点级异常处理**
   不同节点失败，恢复策略不同。

7. **异常要统一封装，不要散落在业务代码里**
   建议通过 LLM Gateway 统一处理。

8. **日志和 trace 必不可少**
   没有可观测性，就无法治理 LLM 系统。
