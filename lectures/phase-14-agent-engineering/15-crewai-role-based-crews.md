---
chapter: Role-Based Agent Teams — Roles, Tasks, Processes
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/15-crewai-role-based-crews
source_doc: phases/14-agent-engineering/15-crewai-role-based-crews/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-15 · 基于角色的 Agent 团队：Role、Task、Process

> Four primitives: Agent, Task, Crew, Process. Two top-level shapes: Crews (autonomous, role-based collaboration) and Flows (event-driven, deterministic). CrewAI is the 2026 reference implementation, and its docs are blunt: "for any production-ready application, start with a Flow."

## 一、本章在整体中的位置

第 14-13（LangGraph）、14-14（AutoGen actor 模型）都是"工程范式" —— 把 agent 当图或 actor。本章是另一种范式：**把 agent 当"角色"** —— 用自然语言描述人设和工作流。

**CrewAI 是 2026 年这套思路的参考实现**，它的文档特别诚实：

> *"for any production-ready application, start with a Flow."*

这一句话是本章的总纲。你可能觉得奇怪 —— CrewAI 教 Crew，结果推荐 Flow？—— 这正是本章要建立的认知：**角色编排是原型利器，生产需要 Flow 包一层审计。**

---

## 二、四个原语：先背下来

```
┌─────────────────────────────────────┐
│  Agent   ── 角色 + 目标 + 背景故事 + 工具 │
│  Task    ── 描述 + 期望输出 + 归属  │
│  Crew    ── 容器：agents + tasks + process │
│  Process ── 执行策略：顺序 / 层级 / （未来：共识）│
└─────────────────────────────────────┘
```

它们之间有清晰的归属：

| 原语 | 持有 | 不持有 |
|---|---|---|
| Agent | 自身背景、工具 | 任务 |
| Task | 自身描述、期望输出 | agent 之间关系 |
| Crew | agents + tasks + process + memory + verbose | 单 agent 内部细节 |
| Process | 派发策略 | 任何 agent 自身的 prompt |

**重要原则**：Agents 之间**不直接看到对方**。它们通过 Tasks 关联，Tasks 描述输入/输出，Process 决定谁先谁后。这是排错时"边界感"的来源。

### 一句重要警告：本章的"角色 vs 工具"决策权

> "Backstory is load-bearing. It shapes tone, judgment, when the agent stops."

**Backstory 不是调味的，是承重的**。它直接塑造：口吻、判断标准、何时停下。一个 200 字的 backstory 真的会影响 Agent 表现 —— 这一章后面会专门警告"backstory 不要超过 200 字"。

---

## 三、三种 Process：Sequential / Hierarchical / Consensus

| Process | 行为 | 成本 | 何时用 |
|---|---|---|---|
| **Sequential** | 任务按声明顺序串行执行；输出当 context 传给下一个 | 最低、最可预测 | 顺序固定的工作流 |
| **Hierarchical** | manager Agent（额外 LLM 调用）每轮决定谁做下一个 | **每轮多一个 LLM 调用** | 4+ specialists，顺序依赖前一步输出 |
| **Consensus** | 计划中（投票），public API 暂未实现 | — | **现在别用** |

### 关键工程教训：manager LLM 的 token 税

> **Hierarchical 在 N 个 specialist 任务上实际产生 N+1 个 LLM 调用**（多出来的那个是 manager）。
>
> 5 个任务的 crew：Sequential 是 5 次 LLM 调用，Hierarchical 是 6 次 —— 而且 manager 的 prompt 要塞下"任务清单 + 历史输出"。
>
> **开关原则**：只在"路由真依赖输出"时打开 Hierarchical。

### 注意事项：版本与现实

本章核对版本是 **CrewAI 0.86（2026-05）**，新版本可能改名/合并 process 类型。所以生产代码要 **[查 CrewAI Processes 文档](https://docs.crewai.com/concepts/processes)**，别依赖本文此刻的命名。

---

## 四、本章核心二分：**Crews vs Flows**

这是 14-15 章的**真正主菜**：

| | **Crew** | **Flow** |
|---|---|---|
| **控制** | LLM 实时决定 | 工程师在代码里写死 |
| **路径** | 自由 | 图（`@start` / `@listen(topic)`） |
| **可重放性** | 难 | 高 |
| **可测试性** | 难 | 高 |
| **观察性** | 弱 | 强 |
| **何时用** | 研究 / 头脑风暴 / 初稿 | 生产 |

```python
# Flow 写法
@start()
def begin(self):
    self.state.researched = research_crew.kickoff(...)

@listen("researched")
def draft(self, payload):
    self.state.drafted = write_crew.kickoff(...)

@listen("drafted")
def finalize(self, payload):
    publish(payload)
```

事件驱动、显式 topic、各步骤是普通 Python（可以内部调用一个 Crew）。

### CrewAI 2026 文档的推荐

> **生产应用先从 Flow 起步**。把 Crew 内嵌到 Flow 的步骤里 —— `Crew.kickoff()` 作为一步。
>
> **Crew 提供"探索型"的形状**（适合 brainstorming、drafts）；
> **Flow 提供"生产型"的形状**（适合审计、测试、确定性）。
>
> **组合，而不是二选一**。

这是 multi-agent 框架里**最重要的一条工程规则**，值得背下来：

> Crew 是流体的（重跑结果可能不同），Flow 是固定的（必须给一样的输出）。

---

## 五、工具集成：三种递进复杂度

工具是 Agent"能做什么"的能力，三种接法按复杂度递增：

### 1. `@tool` 装饰器（函数工具）—— 90% 场景

```python
from crewai.tools import tool

@tool("Search the web")
def search(query: str) -> str:
    """Return top results for the query."""   # ← docstring 是 LLM 看到的描述
    return run_search(query)
```

**最适合**：一次性辅助函数。

### 2. `BaseTool` 子类（类工具）—— 多状态时

```python
class SearchTool(BaseTool):
    name = "web_search"
    description = "Search the web and return top results."
    args_schema = SearchArgs   # Pydantic，结构化入参

    def _run(self, query: str, limit: int = 10) -> str:
        return self.client.search(query, limit=limit)
```

**最适合**：工具本身有状态（client、cache）或需要结构化参数。

### 3. 一方 adapter 工具箱 —— 5 行接入

CrewAI 自带：`SerperDevTool`、`FileReadTool`、`DirectoryReadTool`、`CodeInterpreterTool`、`RagTool`、`WebsiteSearchTool`。

### 结构化输出：**核心工程价值**

```python
Task(
    description="Write a brief.",
    agent=editor_agent,
    expected_output="JSON with title, summary, sections",
    output_pydantic=Brief       # ← 关键
)
```

- LLM 输出 → Pydantic 校验 → 不合规重试。
- 配合 `expected_output` 字符串（即"合同"），下游 step 拿到的是**类型化对象**，不是自由文本。

**为什么这事至关重要**：避免 Brittle Handoff —— 任务 N 写"提纲"，任务 N+1 读出 N 段或 4 段时被迫即兴发挥。**结构化输出在 CrewAI 里是被鼓励的，而在 free-form 工作流里这要自己造轮子。**

---

## 六、四种 Memory（按需组合）

| 类型 | 存活期 | 用途 |
|---|---|---|
| **Short-term** | 单次 run | 同一 kickoff 内多轮传递 |
| **Long-term** | 跨 run | 向量库按相似度检索，跨 kickoff 保留 |
| **Entity** | 跨 run（按实体键） | "客户 X 是 enterprise 套餐" 这种事实档 |
| **Contextual** | 即时 retrieval | Agent 需要时按需取，不预载 |

CrewAI 默认走 OpenAI Embeddings，可换本地模型。

### 重要警告

- **Memory 不是默认全开**，开了 `memory=True` 后长时记忆每次都写库 → 向量库爆炸 → 检索变噪。
- **范围应限定到"事实确实持久"的任务**。

### 版本注脚

新版 CrewAI 把这四类收纳到一个统一的 `Memory` 系统下 —— 概念还在，但类 API 入口可能在变。生产前同样要查文档。

---

## 七、什么时候选（不选）CrewAI

### ✅ 适合

- **3–6 个角色**清晰的协作（drafting / reviewing / planning / brainstorming）。
- 用 LLM 判断"下一步该谁"是有价值的事（Hierarchical）。
- 团队对 `role + goal + backstory` 比对"图"更熟。

### ❌ 不适合

- **强 DAG / 严顺序**：用 LangGraph（14-13）—— 图才是对的抽象。
- **亚秒级延迟预算**：Hierarchical 加 round trips，Sequential 也因 backstory + 历史输出把 prompt 撑大。
- **单 agent 循环**：直接用 chapter 14-01 的 agent loop 加上 tool registry，没必要上框架。

### 一个比较点

> 14-17 章（Agent Framework Tradeoffs）会摆矩阵比较这些框架。本章最关键的"一句话总结"：**CrewAI 坐镇"协作型 + 角色化"四角**。

---

## 八、四种 CrewAI 失败模式（本章明文警告）

1. **Prompt-bloat from backstories** —— 五个 agent 各 2000 字 backstory，第一轮工具还没调用就把 context 烧光。**保持每个 backstory ≤ 200 字**，跨 agent 复用风格描述，别重复。

2. **Manager-LLM token tax** —— 5 任务 Sequential 是 5 次 LLM；5 任务 Hierarchical 是 6 次，再加上 manager 携带的"全部任务清单 + 历史输出"。**Sequential 不够用才切 Hierarchical**。

3. **Brittle handoffs** —— 任务 N 写"提纲"，任务 N+1 自由读取发现段数对不上，被迫即兴发挥。**用 `output_pydantic` 让 N+1 读到的是类型化对象，不是 free text**。

4. **Crew-as-prod** —— 把 free-form Crew 直接 ship 到生产，没 Flow 包一层。结果变异大、不可重放、on-call 无法比较。**永远用 Flow 包 Crew**。

这四条是本章最实战的部分，抄到备忘录不亏。

---

## 九、代码实现（`code/main.py`）速览

```python
@dataclass
class Agent:  role, goal, backstory, tools

@dataclass
class Task:   description, expected_output, agent, context, output_pydantic

class SequentialCrew:
    def kickoff(self, inputs):  # 按声明顺序，输出当 context

class HierarchicalCrew:
    def kickoff(self, topic):   # 加 manager，每轮选下一位，选"done"停下

@start()
@listen("topic")
class Flow: ...                  # 显式 topic 图

def tool(name):  ...             # @tool 装饰器

class Memory:                    # short_term, long_term, entity
    ...
```

Demo 演示：**Researcher → Writer → Editor** 三角色 crew 出简报，然后用 Flow 包同样的三步定形对照。

两条 trace 对比：

- **Crew trace 流体**（manager 在原理上可重排）；
- **Flow trace 固定**。

这一对比是本章的视觉重点。

---

## 十、Use It：决策树

```
生产应用：
  └─ CrewAI Flow（兜底）——
        内部可调：Crew.kickoff()

Crews:
  ├─ Sequential —— 顺序明确、初稿、审阅
  ├─ Hierarchical —— 4+ specialists，路由依赖输出
  └─ Consensus —— 暂不可用

替代技术（按范式）：
  ├─ LangGraph —— 显式状态机
  ├─ AutoGen v0.4 —— actor 并发 / 故障隔离
  ├─ OpenAI Agents SDK —— OpenAI 优先，handoffs + guardrails
  └─ Claude Agent SDK —— Claude 优先，subagents + session store
```

注意最后两行 —— **每个框架都有自己的主场**。这一章最后把它们摆在一起看，是为了不让读者把 CrewAI 当万能锤。

---

## 十一、Ship It：`outputs/skill-crew-or-flow.md`

这个 ship 输出会自动判断一个任务该用 **Crew 还是 Flow** 并生成最小可用脚手架。

**硬拒绝规则**：
- Crew 没有 backstory（不要写"会说话的工具人"，回去补 backstory）。
- Flow 没有显式 topic（流程图要清晰）。
- Hierarchical + 不到 3 specialists（浪费 manager 税）。

---

## 十二、课后练习 7 题核心思路

1. **Sequential → Flow 转换**：把同样的 3 步塞进 Flow，画两个 trace 对比"灵活性下降点 vs 可读性上升点"。
2. **Entity Memory 实战**：每个 kickoff 写客户事实；下次 retrieve 必须命中 —— Entity 是按 key 不是按相似度的，这是和 Long-term 的关键差别。
3. **Hierarchical manager 拒绝路由**：manager 必须看到"writer 输出≥3 段"才允许接 editor。练习会要求 trace 显示 retry 路径。
4. **`BaseTool` vs `@tool`**：同一 mock web search，两种接法的 trace 差异（schema 与重试暴露程度）。
5. **`output_pydantic=Brief` 容错**：故意喂一次坏 JSON，看 CrewAI retry 行为。练习目的：熟悉真实生产中怎么**观察重试**。
6. **真把 toy 端口到 `crewai`**：会发现 toy 跳过的东西 —— **消息序列化、跨进程 capability、重试 policy、topic 路由**。这是把 demo 推到生产的台阶。
7. **接 AgentOps / Langfuse（24 章）**：stdlib 版本丢失的 trace 之一 — **manager 决策的中间 prompt**，这往往是调试 CrewAI 失败模式的关键证据。

---

## 十三、本章给我的工程启示

1. **Crew 是 expression 不是 plan**。当你看到 `role + goal + backstory` 这种自然语言声明，要意识到 —— 那是诗而不是图。如果你需要重放与确定，请尽快落进 Flow。
2. **Backstory ≠ 调味的，是承重的**。它就是 system prompt 的一部分，会直接决定 agent 在哪儿停、是否拒绝、对工具的信心曲线。所有"agent 性格化"的产品都要严肃 A/B。
3. **免费框架的代价在版本和 API 漂移**。CrewAI 0.86 → future 的 Memory 整合就是证明；任何 CrewAI 代码上线前，都要查对应版本文档。
4. **结构化输出是 handoff 的护城河**。Crew 间用 free text 交接是 brittle 的源头；Pydantic 在 task N 出、task N+1 读，能消灭一类失效。
5. **Production 先 Flow，内嵌 Crew**。这是 Anhtropic 「building effective agents」思想在 CrewAI 上的具体落地 —— 把 Crew 当 exploration step，把 Flow 当 audit boundary。

---

## 十四、横向对比（接前几章口诀）

| 章 | 范式 | 何时用 |
|---|---|---|
| 14-12 Workflow | 写死路径 | 步骤可枚举 |
| 14-13 LangGraph | 状态图 | 持久状态、长流程 |
| 14-14 AutoGen | Actor 模型 | 并发 + 故障隔离 |
| **14-15 CrewAI** | **角色 + 任务模板** | **协作 + 多角色** |

把这四章摆在一起看，**抽象层级在升**：路径 → 图 → actor → 角色。最抽象的是 CrewAI，也最容易"看起来在工作，但生产很难调试"。

工程建议（同时是这一章想传达的）：

> 把 Crew 当研究工具，把 Flow 当生产工具。如果团队坚持要用 Crew 上生产，请在前面套 Flow 留审计边界。
