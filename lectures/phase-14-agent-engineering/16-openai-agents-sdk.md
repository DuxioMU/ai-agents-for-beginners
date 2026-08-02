---
chapter: OpenAI Agents SDK — Handoffs, Guardrails, Tracing
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/16-openai-agents-sdk
source_doc: phases/14-agent-engineering/16-openai-agents-sdk/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-16 · OpenAI Agents SDK：Handoffs、Guardrails、Tracing

> OpenAI Agents SDK is the lightweight multi-agent framework built on the Responses API. Five primitives: Agent, Handoff, Guardrail, Session, Tracing. Handoffs are tools named `transfer_to_<agent>`. Guardrails trip on input or output. Tracing is on by default.

## 一、本章在整体中的位置

第 14-13（LangGraph 状态图）、14-14（AutoGen actor 模型）、14-15（CrewAI 角色模板）都是"通用 multi-agent 框架"。本章切换到**具体厂商 SDK** 的代表 —— **OpenAI Agents SDK**，研究它把 multi-agent 抽象固化成什么形状。

对照记四章：

| 章 | 范式 | 谁拥有图 |
|---|---|---|
| 14-13 LangGraph | 节点 + 边 + 状态 | 工程师 |
| 14-14 AutoGen | Actor + 消息 | 运行时 |
| 14-15 CrewAI | 角色 + 任务 | LLM |
| **14-16 OpenAI Agents SDK** | **Agent + Handoff + Guardrail** | **LLM（通过 `transfer_to_X` 工具）** |

OpenAI Agents SDK 的形态最接近 14-15 的"角色模板"，但它把"换人"这件事**建模成调用一个工具** —— 这是个反直觉但极其关键的设计。

---

## 二、五个原语（先背下来）

```
┌─────────────────────────────────────────────────┐
│  Agent       LLM + instructions + tools + handoffs │
│  Handoff     Agent 之间的委托（建模成工具）        │
│  Guardrail   输入 / 输出 / 工具 三处校验           │
│  Session     跨轮会话历史（自动管理）              │
│  Tracing     OpenTelemetry 风格 span（默认开启）   │
└─────────────────────────────────────────────────┘
```

五个原语刚好对应"多 agent 是否能跑得起来、跑得安全、跑得可观测"这三件事。

---

## 三、核心洞察 #1：Handoffs 是工具

这是本章最重要的一招，必须理解：

> **对模型而言，"换人"是一个叫 `transfer_to_<agent_name>` 的工具。**

模型在它的工具列表里看到：

```python
[
    send_email(...),
    search_kb(...),
    transfer_to_billing_agent,      # ← 这是个"工具"，但它不走参数、走上下文
    transfer_to_support_agent,
]
```

当模型决定调用 `transfer_to_billing_agent` 时，运行时（不是模型）做的三件事：

1. **复制 / 折叠对话上下文**（参数 `input` 或 `nest_handoff_history` beta 决定是原样传还是先压缩）。
2. **用目标 agent 的 instructions 重新初始化**（它现在按"账务 agent 的人设"思考）。
3. **继续 run**。

### 为什么这个设计重要

- **对模型来说，普通工具调用和换人是一回事** —— 极大降低模型学习成本。
- **运行时决定一切副作用** —— 上下文复制、instructions 切换、是否启用 guardrail 都是运行时策略。
- **supervisor 模式被产品化**（14-13 / 14-28 提到的 supervisor 模式：调度 LLM 选 worker，原型在这里定型）。

### 一句话总结

> **Handoff = "把自己变形成另一个 agent" 的工具调用。**

---

## 四、核心洞察 #2：Guardrails 三种 × 两种模式

### 三个挂载点

| 挂载 | 触发时机 | 拦什么 |
|---|---|---|
| **Input guardrail** | 第一个 agent 收到输入时 | 不安全、越界、敏感请求 → 在 LLM 调用前挡掉 |
| **Output guardrail** | 最后一个 agent 给出输出时 | PII 泄漏、违反政策、格式错误 → 在出口挡掉 |
| **Tool guardrail** | 每个 function tool 被调用时 | 参数校验、权限检查、审计日志 |

挂载点选择的两条经验：

- **Input guardrail 优先拦截**比让 LLM 处理不请要求的请求便宜得多。
- **Output guardrail 是用户最后一道保护**，业务侧合规风控通常挂在这里。
- **Tool guardrail 是 RBAC + 副作用审计**的核心（删除 / 支付 / 写库 必挂）。

### 两种运行模式

```python
# 模式 1：Parallel（默认）
@input_guardrail(run_in_parallel=True)
async def check_input(ctx, input):
    return GuardrailFunctionOutput(...)

# 模式 2：Blocking
@input_guardrail(run_in_parallel=False)
async def check_input(ctx, input):
    return GuardrailFunctionOutput(...)
```

| 模式 | 行为 | 何时用 |
|---|---|---|
| **Parallel** | guardrail LLM 与主 LLM 并行跑 | 延迟敏感；trip 时浪费主 LLM 的 token |
| **Blocking** | guardrail LLM 先跑，trip 时主 LLM 根本不发 | 保护昂贵 token 预算；尾延迟高 |

### Tripwire 异常

```python
from openai_agents import InputGuardrailTripwireTriggered, OutputGuardrailTripwireTriggered
```

被触发的 guardrail 会抛出 tripwire 异常，**这是好事 —— 异常路径让兜底逻辑可写**：

```python
try:
    result = await Runner.run(triage, query)
except InputGuardrailTripwireTriggered:
    return "该请求超出处理范围"
```

---

## 五、Tracing：默认开启 + 可挂自定义

> OpenAI Agents SDK 默认开启 tracing。

每个 LLM 生成、工具调用、handoff、guardrail 都会发一个 span：

- **关掉**：`OPENAI_AGENTS_DISABLE_TRACING=1`
- **附加**：用 `add_trace_processor(processor)` 把 span 同步送到你自己的后端（Jaeger / Langfuse / 自家追踪）。

这是 OpenAI 团队最务实的设计选择 —— **让"看不到 agent 内部"这件事成立**。在生产里这是必选项。

### 与 OTel GenAI 的关系

Agents SDK 的 span 形状与 OTel GenAI 语义约定（第 14-23 章）一致。奥卡姆原理：**do not invent a new schema when the standard exists**。

### 隐私警告（本章明确警告）

> **敏感内容进 span = 泄漏。** 一律遵循 OTel GenAI 的 content-capture 规则（第 14-23 章细节）：
> - span 里只存 ref/ID；
> - 原文内容存外部安全存储；
> - 严格按角色 decide 何时 capture。

这是合规要求，不是 nice-to-have。

---

## 六、Session：跨轮对话的自动管理

```python
session = SQLiteSession("user-123")
result = await Runner.run(agent, input, session=session)
```

- SQLite / Redis / 自定义 backend 都行。
- Runner 自动加载历史、append 新消息 —— 业务代码不用管。

### 一个隐性陷阱

**Session is per-agent-call-binding.** 你的 handoff 链从 A 到 B 到 C，若三者绑同一个 session，记忆是连的；若绑不同 session，记忆断裂。这是你设计对话体验时必须想清楚的。

---

## 七、代码实现（`code/main.py`）

`code/main.py` 用 stdlib 实现 SDK 全部要素：

```python
class Agent:  # LLM + instructions + tools + handoffs
class FunctionTool:  # 普通工具
class Handoff:  # 特殊工具 — 触发"换人"语义

class Runner:
    run_in_parallel: bool      # Guardrail 模式
    tripwires: dict            # Guardrail 触发
    add_trace_processor(span)  # 自定义追踪

class Session:  # 历史存储（SQLite / Redis / 自定义）
```

Demo：一个 triage agent 根据 query 把会话交给 billing 或 support；故意让一个 input guardrail 触发出错 —— 看 trace 形状。

运行：
```bash
python3 code/main.py
```

trace 内容：
- 2 次成功 handoff（triage → billing → 回 triage → support）
- 1 次 input guardrail trip
- 1 棵 span 树

---

## 八、Use It：决策树

```
你要做 OpenAI 优先的产品？
  ├─ 是 → OpenAI Agents SDK（本 chapter）
  ├─ 否 → 
  │     ├─ Claude 优先 → 第 14-17 章 Claude Agent SDK
  │     ├─ 显式状态 + 持久 resume → 第 14-13 章 LangGraph
  │     ├─ 需要 actor 并发 + 故障隔离 → 第 14-14 章 AutoGen
  │     └─ 角色模板 + 拥抱 LLM 决策 → 第 14-15 章 CrewAI
  └─ 你需要某种细粒度控制（语音 / 多 provider / 联邦部署）→ 自建
```

---

## 九、常见踩坑（本章明文警告）

1. **Handoff drift（A↔B 无限循环）** —— A 交给 B，B 又交回 A。**加一个 hop counter**，超过 N 次强制拒绝。这是成熟 SDK 内置的，本章要你手写。
2. **Guardrail bypass** —— tool guardrail 只对 function tool 触发。**OpenAI 内置工具**（file_reader、web_fetch）走另一条路径，必须单独挂政策。
3. **Over-tracing** —— 把敏感内容直接写进 span。**永远先看 OTel GenAI content-capture 规则**（第 14-23 章），按角色决定能不能 capture。

---

## 十、课后练习 5 题核心思路

1. **Handoff hop counter**：跑 100 次对话，记 handoff 链长度分布。这一题的工程价值：你会发现"两个 agent 互相踢皮球"是真实失败模式。
2. **`nest_handoff_history`**：换人前用 LLM 压缩历史，对端看到一句摘要而不是 50 轮全文。**这是控制 token 预算的关键开关**。
3. **Blocking guardrail**：比较 trip 路径 vs pass 路径的延迟。**你会发现 parallel 在 P50 占优，但 blocking 在 P99 显著更低**（取决于 trip 概率）。
4. **`add_trace_processor` 接 JSON 日志**：拿到 SDK span 的完整形状 —— 比 OTel 简单但够用。**这一题让你看清"trace 信息够不够 debug"**。
5. **端口到真 `openai-agents-python`**：你会发现 toy 跳过的东西 —— **跨失败的 retry 策略、Session 跨进程一致性、guardrail 异常向上传播的语义**。这是从 demo 走到生产的台阶。

---

## 十一、本章给我的工程启示

1. **"换人是工具调用"是个反直觉但简洁的设计**。它让模型只用一种思考方式（"我有这些工具"），把复杂性全推到运行时。这对用户体验 + 调试体验都有好处。
2. **Guardrail 不是"风控后置"，是"早拒 + 出口双保险"**。早拒在 input 阻断乱请求，出口在 output 兜底 PII —— 任何一边漏一边。要 budget 的时候才上 Blocking。
3. **Tracing 默认开是对的**。"看得见"是 multi-agent 系统能不能 production 的硬指标。OpenAI 这个决定值得其他框架学。
4. **Handoff 需要 hop counter**。没有终止条件的递归是商用系统的最大风险。**这个 counter 应该写在最外层，避免任何一个 agent 内部破坏**。
5. **Session 跨 agent 一致性是用户体验的本源**。两个 agent 互相踢皮球但用户不知情，是糟糕体验的源头。**先把 session 边界画清楚，再设计 handoff 链**。

---

## 十二、横向对比（接前几章）

| 章 | 范式 | 模型看到什么 |
|---|---|---|
| 14-13 LangGraph | 状态图 | 下一节点（由调度器决定） |
| 14-14 AutoGen | Actor 模型 | 消息（投递给另一 actor） |
| 14-15 CrewAI | 角色 + 任务 | 自己的 prompt + 任务上下文 |
| **14-16 OpenAI Agents SDK** | **Agent + Handoff** | **工具列表（含 `transfer_to_X`）** |

把这四章放一起看：

- **LangGraph** 把图藏给模型 → 工程师有最终控制。
- **AutoGen** 让消息流转 → 运行时主导。
- **CrewAI** 把角色塞 prompt → LLM 在 prompt 内选角色。
- **Agents SDK** 把"换人"做成工具 → LLM 把它当普通工具顺手调。

**最 PHP-优雅的是 OpenAI 这个**：单一原语（工具调用）解决两类事件（做事 + 换人）。这是 SDK 表面小但能撑住实际产品的关键。
