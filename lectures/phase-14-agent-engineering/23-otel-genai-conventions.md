---
chapter: OpenTelemetry GenAI Semantic Conventions
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/23-otel-genai-conventions
source_doc: phases/14-agent-engineering/23-otel-genai-conventions/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-23 · OpenTelemetry GenAI 语义约定

> OpenTelemetry's GenAI SIG (launched April 2024) defines the standard schema for agent telemetry. Span names, attributes, and content-capture rules converge across vendors so agent traces mean the same thing in Datadog, Grafana, Jaeger, and Honeycomb.

## 一、本章在整体中的位置

前几章讲了"如何造 agent"（14-13 ~ 14-22）。**造完怎么观测？** —— 这是本章的总问题。

> **Every vendor invents their own span names. Ops teams end up building per-framework dashboards. OpenTelemetry's GenAI SIG fixes this by defining one standard the whole ecosystem targets.**

**OTel GenAI SIG（2024-04 成立）的使命**：让 agent trace 在 **Datadog / Grafana / Jaeger / Honeycomb** 各种后端都长得一样。**一个标准，全生态对齐**。

---

## 二、Span 三大类（先记下来）

```
┌────────────────────────────────────────────────────────────┐
│  OTel GenAI Span 三类                                      │
│                                                            │
│  1. Model / client spans    — 原始 LLM 调用                 │
│                              （Anthropic / OpenAI / Bedrock SDK 发）│
│                                                            │
│  2. Agent spans             — `create_agent` + `invoke_agent`│
│                              （agent 构造 + agent 运行）      │
│                                                            │
│  3. Tool spans              — 每次工具调用一个 span           │
│                              （parent-child 挂 agent span）  │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**关系**：

```
invoke_agent (INTERNAL)
   ├── chat (LLM 调用)
   ├── tool: search_kb
   │     └── (嵌套 backend HTTP)
   ├── tool: send_email
   └── chat (LLM 续轮)
```

**三类 span 在树里有清晰归属**。**所有 vendor 都应遵循这套结构**。

---

## 三、Agent Span 命名：CLIENT vs INTERNAL

> **Span name**: `invoke_agent {gen_ai.agent.name}`（如果命名了）；fallback `invoke_agent`。
> **Span kind**:
> - **CLIENT** —— **远端 agent 服务**（OpenAI Assistants API、Bedrock Agents）
> - **INTERNAL** —— **进程内 agent 框架**（LangChain、CrewAI、ReAct）

| Span kind | 用在 |
|---|---|
| **CLIENT** | 你调的是远端服务（"我作为 client 调远端 agent"） |
| **INTERNAL** | agent 在你自己进程里跑（"我内部的 agent loop"） |

**这一条容易被忽略**。**把 ReAct loop 错标 CLIENT 会让 trace 含义错位**。**选 kind 的规则就一条：agent 跑在哪里**。

---

## 四、关键属性（Top-level GenAI Attributes）

| 属性 | 含义 | 例 |
|---|---|---|
| `gen_ai.provider.name` | provider 标识 | `anthropic` / `openai` / `aws.bedrock` / `google.vertex` |
| `gen_ai.request.model` | 请求的模型 ID | `claude-opus-4-5` |
| `gen_ai.response.model` | 实际响应模型（可能与请求不同，因为 routing） | `claude-sonnet-4-5`（fallback 后） |
| `gen_ai.agent.name` | agent 标识 | `triage_agent` |
| `gen_ai.operation.name` | 操作类型 | `chat` / `completion` / `invoke_agent` / `tool_call` |
| `gen_ai.data_source.id` | RAG 用：哪个 corpus / store 被查询 | `kb-customer-faq-2026q1` |

### request.model vs response.model 的微妙之处

> **may differ from request due to routing**

**多模型 routing 是常态**（你请求 sonnet 但后端 fallback 到 haiku）。**response.model 是真实模型**——**计费、SLA、latency 分析都看这个**。**别只看 request.model**。

### data_source.id 是 RAG 的金钥匙

> `gen_ai.data_source.id` — **for RAG: which corpus or store was consulted**

**RAG 答案不准确？** 第一反应是"我的 prompt 不行"。**但 OTel 属性告诉你：是哪个 corpus 命中的。** **这是 RAG debug 的起点**。

---

## 五、Content Capture 规则（**最重要的一条**）

> **Default rule**: instrumentations **SHOULD NOT** capture inputs/outputs by default.

**默认不 capture**。**Opt-in** 的方式是显式开：

- `gen_ai.system_instructions`
- `gen_ai.input.messages`
- `gen_ai.output.messages`

### 为什么默认不 capture

**PII、密钥、客户数据在 trace 里 = 安全事故**。**Ops / SRE 能看到 trace**——**把客户输入直接写 span 是泄漏**。

### 推荐生产模式

> **Store content externally (S3, your log store), record references on spans (pointer IDs, not prose).**

```
┌────────────────────────────────────────────────┐
│  Span (in trace)                              │
│  ├─ gen_ai.input.messages_ref: "s3://kb/p/7"  │  ← 只存引用
│  └─ gen_ai.output.messages_ref: "s3://kb/r/7" │
└────────────────────────────────────────────────┘
                    ↓
        ┌──────────────────────┐
        │  S3 / Log Store       │
        │  (受控访问)            │
        │  p/7: "用户问了..."   │  ← 原文
        │  r/7: "agent 回了..." │
        └──────────────────────┘
```

**原文在 S3，trace 上只有 ID**。**访问 S3 需要专门权限**——**trace 后端的人看不到内容**。**这正是 14-27 章 content-poisoning 防御在可观测性侧的落地**。

---

## 六、Stability：约定还在演进

> **Most conventions are experimental as of March 2026. Opt in to the stable preview with:**

```bash
OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental
```

**约定还在动**。**不设这个 env var，**后端升级时你的 attribute 名字可能被改**。**这是稳定性 vs 新功能的标准权衡**。

### 各后端支持情况

| 后端 | 支持 |
|---|---|
| **Datadog LLM Observability v1.37+** | **原生映射** GenAI 属性 |
| **Langfuse / Phoenix / Opik**（14-24 章） | auto-instrument 全生态 |
| **Jaeger / Honeycomb / Grafana Tempo** | 原始 OTel 属性，自建 dashboard |
| **Self-hosted OTel Collector + GenAI processor** | 完全自控 |

---

## 七、本章的失败模式（明文警告）

1. **Capturing full prompts in spans** —— PII、密钥、客户数据泄漏。**永远外部存**。
2. **No `gen_ai.provider.name`** —— 多 provider dashboard 失效。**没这个属性就别接 dashboard**。
3. **Spans without parent links** —— 孤立 tool span。**永远 propagate context**。
4. **Not setting stability opt-in** —— 升级后 attribute 名字改，dashboard 静默失效。

---

## 八、代码（`code/main.py`）速览

`code/main.py` 实现一个匹配 GenAI 约定的 stdlib span emitter：

```python
class Span: ...                  # GenAI 属性 schema
class Tracer:                    # start_span + 嵌套 context
   def start_span(name, kind): ...

# scripted agent run 发：
#   create_agent
#   invoke_agent (INTERNAL)
#   tool: search_kb
#   chat (LLM call)
#   tool: send_email
#   chat (续轮)
#   invoke_agent 结束
```

Content-capture 模式：prompts 进外部存（stub），span 上只挂 ID。

运行：
```bash
python3 code/main.py
```

输出：完整 span 树 + 外部存引用。

---

## 九、Use It：决策树

```
你的 trace 后端？
  ├─ Datadog v1.37+ → 原生 GenAI 支持
  ├─ Grafana / Honeycomb / Jaeger → 原始 OTel，自建 dashboard
  ├─ Langfuse / Phoenix / Opik → 14-24 章 auto-instrument
  └─ 自管 → OTel Collector + GenAI processor

content capture 模式？
  ├─ 调试 / 开发 → opt-in 全 capture
  └─ 生产 → 外部存 + ID ref（必）
```

---

## 十、Ship It：`outputs/skill-otel-genai.md`

把 OTel GenAI span 接到现有 agent：content-capture 默认关，生产模式外部存。

---

## 十一、课后练习 5 题核心思路

1. **把 14-01 ReAct loop 用 `invoke_agent` (INTERNAL) + per-tool span 装上**：发到 Jaeger。**这是"OTel GenAI 在我自己代码里跑起来"的实践**。
2. **加 content-capture 引用模式**：prompts 进 SQLite，span 只挂 row ID。**生产模式必须这样**。
3. **`gen_ai.data_source.id` 接到 14-09 Mem0 搜索**：corpus 命中的可追溯。**RAG debug 第一步**。
4. **设 `OTEL_SEMCONV_STABILITY_OPT_IN`**，验证 attribute 不被改。**这是"约定在动"的实战应对**。
5. **建 dashboard**："哪个 tool error 跟哪个 model 相关"——**只用 GenAI 属性**。**这是把"日志"变成"洞察"**。

---

## 十二、本章给我的工程启示

1. **OTel GenAI 是 agent 可观测性的"通用语"**。**学一次，所有后端都用得上**。**Datadog、Langfuse、Jaeger 都对齐它**。
2. **Span 三分类 + agent kind = 设计核心**。**造 agent 时就要想"我会发什么 span"**。**没有 trace = 没法 debug**。
3. **response.model 比 request.model 重要**。**真实模型是计费 + SLA + 性能分析的基础**。**别只看请求模型**。
4. **Content capture 默认关**。**生产模式必走"外部存 + ID ref"**。**这是"看得见"和"看得安全"的边界**。
5. **Stability opt-in 不是 nice-to-have**。**不设等于"约定升级时静默改你的 schema"**。**生产前必设**。

---

## 十三、横向对比

| 后端 | 原生 GenAI | auto-instrument | 推荐度 |
|---|---|---|---|
| **Datadog v1.37+** | **是** | 部分 | 商业首选 |
| **Langfuse / Phoenix / Opik** | 是 | **全** | 开源 / 自管 |
| **Jaeger / Honeycomb / Tempo** | 否（原始） | 需自配 | 灵活 / 工作量大 |
| **自管 OTel Collector** | 自定 | 自定 | 极度自控 |

**统一口诀**：

> **Span 三分类（model/agent/tool），kind 二选（CLIENT/INTERNAL），capture 默认关，外部存 + ID ref，stability opt-in 必设。**
