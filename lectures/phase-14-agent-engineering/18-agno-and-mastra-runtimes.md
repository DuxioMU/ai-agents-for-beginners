---
chapter: Production Agent Runtimes — Fast Instantiation and Typed Workflows
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/18-agno-and-mastra-runtimes
source_doc: phases/14-agent-engineering/18-agno-and-mastra-runtimes/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-18 · Production Agent Runtime：Agno 与 Mastra

> A production agent runtime optimizes what prototyping frameworks ignore: instantiation cost, typed workflow surfaces, and a serving-ready backend. The 2026 pairing: Agno (Python) aims at microsecond agent instantiation and stateless FastAPI backends. Mastra ships agents, tools, workflows, unified model routing, and composite storage on the Vercel AI SDK substrate.

## 一、本章在整体中的位置

第 14-13 到 14-17 章讲的都是**"多 agent 抽象"**（图、actor、role、handoff、subagent）。本章切换到**生产 runtime 的工程化层面**：

> **"框架太重，我只要 agent loop，跑得更快，跟我的 stack 紧贴"** —— 这就是 Agno / Mastra 的市场。

对照来看：

| 章 | 视角 | 关心什么 |
|---|---|---|
| 14-13 LangGraph | 状态机 | 显式持久化、可恢复 |
| 14-14 AutoGen | 并发 | actor 模型、分布 |
| 14-15 CrewAI | 角色 | LLM 主导的协作 |
| 14-16 OpenAI Agents SDK | 厂商 SDK | Handoff + Guardrail |
| 14-17 Claude Agent SDK | 厂商 SDK | Subagent + Hook + Session |
| **14-18 Agno / Mastra** | **Runtime** | **实例化开销 + stack 紧密度** |

Agno / Mastra **都不是新抽象**，而是对**"我已有的 stack 里跑得最快的 agent 是什么"**这个问题的两种回答。

---

## 二、Agno：Python 极简 runtime

### 定位

- **Python** runtime（前身 Phi-data）。
- 标语："**No graphs, chains, or convoluted patterns — just pure python.**"
- 与前面所有框架最大的区别：**没有图、没有 chain、没有复杂模式**，就是 Python 函数的薄包装。

### 性能目标（关键数字）

文档列的指标：
- **~2μs agent 实例化**
- **~3.75 KiB 每 agent 内存**
- **~23 个 model provider**

**这些数字什么时候重要？**
- **高扇入 chat 场景**：每秒上千个短命 agent（fan-in、evaluation pipelines）。
- **不重要**：单 agent 跑 10 分钟的长任务（那点实例化开销摊在 600 秒里 = 噪音）。

**一个反模式警告（本章明文警告）**：

> **Picking Agno because "2μs" sounds good when the workload is one slow agent call per request. Overhead is not the bottleneck.**

性能对比要看**端到端**，不是看 micro-benchmark。

### 推荐生产形态：**无状态 session-scoped FastAPI**

```
请求进来
  ↓
启动一个全新 agent
  ↓
session 状态全在 DB（不留在 agent 实例里）
  ↓
请求结束 → agent 销毁
```

**为什么是无状态**：

- **可水平扩展**：每个请求都是新 agent，无锁、无共享。
- **冷启动友好**：2μs 实例化让"按需起"在成本上划算。
- **故障隔离**：一个请求崩了不影响下一个。

**为什么是 session-scoped**：

- **状态外置**：DB 里靠 session_id 找历史。
- **重启不丢数据**：进程没了，DB 还在。
- **审计容易**：所有状态在 DB 里可查。

### 多模态与 RAG 一方内置

> Native multimodal (text, image, audio, video, file) and agentic RAG.

**Agno 把多模态和 RAG 作为一等公民**。这是它对很多只做文字 agent 的框架的差异化点。

---

## 三、Mastra：TypeScript on Vercel AI SDK

### 定位

- **TypeScript**，基于 Vercel AI SDK。
- 三大原语：**Agents + Tools（Zod 强类型） + Workflows**。

### 关键数字

- **Unified Model Router**：3,300+ 模型，跨 94 个 provider（2026-03 统计）。
- **22k+ GitHub stars，300k+ 周下载量**（1.0，2026-01）。

### 与 Agno 的核心差异：Mastra 是生态选择，Agno 是语言选择

| 维度 | Agno | Mastra |
|---|---|---|
| **语言** | Python | TypeScript |
| **底层** | 纯 Python | Vercel AI SDK |
| **原语** | 1 个：Agent | 3 个：Agent / Tool / Workflow |
| **Tool typing** | Python 类型提示 | **Zod schema**（运行时 + 编译时双校验） |
| **Model Router** | 23 个 provider | 3,300+ 模型 / 94 provider |
| **Server 适配** | FastAPI 为主 | Express / Hono / Fastify / Koa / **Next.js 一等公民** / Astro |
| **调试** | 第三方 | **Mastra Studio**（localhost:4111） |
| **License** | Apache 2.0 | Apache 2.0 + **`ee/` 目录 source-available** |

### 三个值得注意的设计点

**1. Zod 强类型 Tool**

Mastra 的 Tool 不是 Python-style 类型提示，而是 **Zod schema**：

```typescript
const tool = createTool({
  name: "search",
  description: "...",
  inputSchema: z.object({
    query: z.string(),
    limit: z.number().default(10),
  }),
  execute: async ({ query, limit }) => { ... }
})
```

**Zod 让 tool 在 TS 里享受完整类型流** —— 编译时检查、运行时校验、IDE 智能提示全有。**这是 TS-first 团队特别看重的**。

**2. Composite Storage**

> "Memory, workflows, observability to different backends; ClickHouse recommended for observability at scale."

不是一套 storage 解决所有问题，而是**按数据特性选最合适的 store**：memory 用 SQLite/Redis、workflow 用 Postgres、observability 用 ClickHouse。**这是认真生产的态度**。

**3. Mastra Studio**

`localhost:4111` 一个 UI，让你本地 introspect 正在跑的 agent。**对 TypeScript 生态来说，这个一等调试器是相对 Agno 的明显加分项**。

### License 注意（本章明文警告）

> **Mastra's `ee/` directories are source-available, not Apache 2.0. Read the licenses if you're planning to fork.**

`ee/` 目录用的不是 Apache 2.0，而是 source-available 企业 license —— **意味着能看源码但限制商用**。计划 fork 的团队必须看清。

---

## 四、它们共同关心的事：**生产部署形态**

> **Neither is trying to be LangGraph.** They compete on: language fit, runtime ergonomics, observability.

两个框架都默认你**用框架自带的方式部署**：

- **Agno** → FastAPI，每个请求一个 agent，session 进 DB。
- **Mastra** → Vercel/Express/Hono，studio 自带调试。

**这种"框架 + runtime 一体"的形态，对很多小到中型团队是好事**：你不用纠结 LangGraph 那套"图 vs 子图 vs 检查点"了。

---

## 五、什么时候选哪个

```
你的 stack 主语言？
  ├─ Python → 
  │     ├─ 需要极速实例化 / 高扇入 → Agno
  │     └─ 需要显式状态 / 持久化 / 子图 → LangGraph
  └─ TypeScript →
        ├─ 部署在 Vercel/Next.js → Mastra
        ├─ 需要多 provider 路由 → Mastra（3,300+ 模型）
        └─ 想要 Zod 强类型 Tool → Mastra
```

**本章给的最关键判断**：

> **Agno for "many short-lived agents, FastAPI shop".**
> **Mastra for "TypeScript, Vercel/Next.js, multi-provider".**

---

## 六、本章的"不写什么"

本章**Type: Learn**，没有大段代码。`code/main.py` 只做一件事：用 Agno 风格和 Mastra 风格各实现一遍 "run agent → stream output → persist session"。**两条 trace 结构不同但功能等价**。

**这一章的精髓是判断力**。代码量少，但每个决策（"用哪个 stack"）都决定了你未来 12 个月的运维形态。

---

## 七、Use It：决策树

```
Python 团队：
  ├─ 高扇入 / 短命 agent → Agno
  ├─ 长任务 / 显式状态图 → LangGraph
  └─ OpenAI/Claude 优先 → 第 14-16 / 14-17 章

TypeScript 团队：
  ├─ 部署在 Vercel/Next.js → Mastra
  ├─ 多 provider 路由需求 → Mastra
  └─ 想要 Zod 强类型 → Mastra
```

---

## 八、常见踩坑（本章明文警告）

1. **Perf-for-perf's-sake** —— "2μs" 听起来好但你 workload 是单 agent 长任务，那 2μs 在 600 秒里是噪音。**先看端到端瓶颈在哪**。
2. **Ecosystem lock-in** —— Mastra 的 Vercel 集成在 Vercel 上是加分；离开 Vercel 就是减分。**你的部署环境决定取舍**。
3. **Enterprise license confusion** —— Mastra `ee/` 不是 Apache 2.0。**要 fork 之前必看 license**。

---

## 九、课后练习 5 题核心思路

1. **把第 14-01 章 ReAct 端口到 Agno**：你会发现"框架消失了"——Agno 几乎是裸 Python。**这正是它的卖点**。
2. **同一个 ReAct 端口到 Mastra**：你会发现 tool 定义变成 **Zod schema**——类型更安全，但写起来稍重。**用 Zod 的人爱得不行，不用的觉得啰嗦**。
3. **Benchmark**：测你的 stack 上 agent 实例化延迟。**真实数字会告诉你 Agno 的 2μs 对你 workload 是否重要**。
4. **CrewAI → Agno 迁移**：你会丢掉 `role + goal + backstory` 这一整套 CrewAI 抽象。**aggressive 简化是好是坏取决于你的产品**。
5. **读 Mastra `ee/` license**：fork 之前必看。**这是法律问题不是工程问题**。

---

## 十、本章给我的工程启示

1. **不是所有 production 都需要显式状态机**。如果你的 workload 是高扇入短命 agent，LangGraph 那套"图"就是 overkill。**Agno / Mastra 都在做"反 LangGraph"的简化**。
2. **Zod vs Python 类型提示**是 TS vs Python 的本质差异。TS-first 团队一旦用过 Zod，很难回头。**这是选 Mastra 还是 Agno 的隐藏技术债考虑**。
3. **"无状态 session-scoped"是 Agno 最重要的设计选择**。它把"实例化开销"问题变成"实例化快"问题，从根本上改变了扩展模型。**其他多 agent 框架都该认真抄这个形态**。
4. **License 透明度比性能数字重要**。一个 2μs 的运行时如果 license 不清，写不写得进生产都是问题。**Mastra `ee/` 是个真问题**。
5. **Mastra Studio 这种"框架自带调试 UI"值得其他框架学**。一等公民的可观测性是 TS-first 团队的硬需求。

---

## 十一、横向对比（接前几章）

| 维度 | LangGraph | Agno | Mastra |
|---|---|---|---|
| **抽象** | 状态图 | 函数薄包装 | Agent + Tool + Workflow |
| **语言** | Python | Python | TypeScript |
| **实例化开销** | 较高 | **~2μs** | 较低（基于 Vercel AI SDK） |
| **Tool 类型** | Python / Zod | Python type | **Zod schema** |
| **状态** | 显式持久 | DB（无状态） | 多种 backend（composite） |
| **调试** | LangGraph Studio | 第三方 | **Mastra Studio 自带** |
| **何时用** | 显式状态 / 子图 | 高扇入 / FastAPI | Vercel / 多 provider |

记一个口诀：

> **LangGraph 给控制，Agno 给速度，Mastra 给生态。**

你的产品首要约束是什么？**这是这三选一的核心问题**。
