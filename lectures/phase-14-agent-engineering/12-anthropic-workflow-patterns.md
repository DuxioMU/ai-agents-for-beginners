---
chapter: Anthropic's Workflow Patterns — Simple Over Complex
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/12-anthropic-workflow-patterns
source_doc: phases/14-agent-engineering/12-anthropic-workflow-patterns/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-12 · Anthropic 工作流模式：简单优于复杂

> Schluntz and Zhang (Anthropic, Dec 2024) distinguish workflows (predefined paths) from agents (dynamic tool-use). Five workflow patterns cover most cases. Start with direct API calls. Add agents only when steps cannot be predicted.

## 一、本章在整体中的位置

第 14-02 到 14-11 章都在讲"如何让 agent 更强大"。本章是 **Anthropic 团队 2024 年 12 月发布**的那篇著名博文 —— *Building Effective Agents* —— 的精读，是整个 phase 14 的**方法论校正**章节：

> 大多数团队在实际问题还没列清楚之前就跳上多 agent 框架了。该用 workflow 的场景被强塞了 agent。

---

## 二、最重要的一个区分：Workflow vs Agent

这是本章的核心对立概念，先把它钉死：

| | **Workflow** | **Agent** |
|---|---|---|
| **谁拥有图（graph）** | 工程师在代码里写死 | 模型在每一步自己决定 |
| **控制流** | 预定义路径 | 动态工具选择 |
| **调试性** | 高 —— 静态读图 | 低 —— 读轨迹反推 |
| **成本** | 有界（步数固定） | 可无界（循环失控） |
| **适用** | 步骤可枚举的任务 | 开放性、变长任务 |
| **审计友好性** | 高（合规想看图） | 低 |

**论文的核心论点**：
> *能写死路径，就写死路径。Agent 是把"用模型解决开放问题"的能力买回来，同时把"可解释性 + 可控性"卖出去 —— 工程师要先确认买回的是真东西才付得起这个价。*

---

## 三、起点单元：增强型 LLM（The Augmented LLM）

所有五种模式都依赖同一个原子单位 —— **单个 LLM 加三种外接能力**：

```
   ┌────────────────────────────────┐
   │  Augmented LLM（原子单元）       │
   │                                │
   │  ┌─────┐ ┌──────┐ ┌─────────┐ │
   │  │LLM  │ │Tools │ │ Memory  │ │
   │  │call │ │(动作)│ │(持久化) │ │
   │  └──┬──┘ └──────┘ └─────────┘ │
   │     │                          │
   │     └─→ Search（检索）          │
   │                                │
   └────────────────────────────────┘
```

任何 API 调用都可以同时挂这三件套。一切工作流都是这个原子单位的某种组合。

---

## 四、五种 Workflow 模式（重点）

### 模式 1：Prompt Chaining（提示链）

```
输入 → [LLM₁] → 中间产物 → [LLM₂] → ... → [LLMₙ] → 输出
                ↑                       ↑
        （可选程序化闸门）        （可选程序化闸门）
```

- **适用**：任务有清晰的线性分解（生成大纲 → 写每节 → 翻译 → 校对）
- **关键点**：LLM 调用之间可以插**程序化判断**（不是 LLM，是 `if score < 0.6: raise`）。这很重要 —— 大多数失败模式不需要再调一次 LLM。
- **常见错误**：把链做得太碎，让每次 LLM 调用只见局部，丢失全局。

### 模式 2：Routing（路由）

```
输入 → [Classifier LLM] → 类别 A → [Handler A]
                       → 类别 B → [Handler B]
                       → 类别 C → [Handler C]
```

- **适用**：输入**性质不同**需要不同处理（一线 support vs 退款 vs bug vs 销售）。
- **变体**：可以多级路由（先业务路由，再语言路由）。
- **风险**：分类错误会级联到下游 —— 分类器必须有置信度，必要时降级到人工。
- **本题对应第 1 课后练习**：routing + 置信度阈值。

### 模式 3：Parallelization（并行）

两种形态：

| 形态 | 描述 | 聚合方式 |
|---|---|---|
| **Sectioning（分块）** | 不同块给不同 LLM | 程序化拼接 |
| **Voting（投票）** | 同一 prompt 跑 N 次 | 多数票 / LLM 综合 |

- **sectioning**：把长文档切片，每片摘要；或每个研究员查一个子问题；或代码评审每文件并发。
- **voting**：用 N 个推理样本来"投票"，对**有明确对错**的任务（数学、分类）效果好；对**主观**任务退化为"花更多 token 取平均意见"。
- **工程盲点**：并行调用不是免费的 —— 调度与归约都是工程量。本章练习 2 专门讨论 hang 的聚合策略（这是真实生产痛点）。

### 模式 4：Orchestrator-Workers（编排-工人）

```
┌───────────────────────────────┐
│ Orchestrator LLM（一次性规划） │
│                               │
│ 任务："起草项目计划"            │
│       ↓                       │
│ 选出 workers：                 │
│  - 工时估算 worker             │
│  - 风险评估 worker             │
│  - 依赖关系 worker             │
└───────────────────────────────┘
         ↓        ↓        ↓
       [W₁]     [W₂]     [W₃]
         ↓        ↓        ↓
         └────→ Synthesizer ←┘
                  ↓
            最终输出
```

- 与 agent 循环**形态相似但本质不同**：
  - orchestrator **不会无限迭代**，它一次性决定调用哪些 worker。
  - worker 也不自循环。
- 是 OpenAI Agents SDK 的产品化形态（课件"Use It"那一节明说了）。
- **设计陷阱**：orchestrator LLM 在做的事"看起来像 agent 在做"。但实际上它是**一次性的规划函数**，不是真正的自治循环。

### 模式 5：Evaluator-Optimizer（评估-优化）

```
[Proposer LLM] → 草稿
     ↓
[Evaluator LLM] → 反馈
     ↓
[Proposer LLM] → 草稿 2
     ↓
[Evaluator LLM] → ...
     ↓
（直到 Evaluator 通过或达到 max_iter）
```

- 这是第 14-05 章 Self-Refine 的**形式化与推广**（论文原文就这么说的）。
- 关键设计：
  - **评估准则要精确**（"足够好"会让循环停不下来）。
  - **最大迭代次数**是硬上限，避免失控。
  - **多态**：proposer/evaluator 可以是同一模型不同 prompt，也可以是不同模型（评估用强模型、proposer 用快模型）。
- 本章练习 3 把这一模式变成多臂老虎机 —— 保留 top-2 防止"晚出好答案被晚出差答案覆盖"，这是很实用的工程改造。

---

## 五、对照表：什么时候用哪个？

| 场景 | 选 | 理由 |
|---|---|---|
| 步骤可枚举 | **Workflow** | 图写死就好 |
| 步骤数未知 / 取决于前一步结果 | **Agent** | 用 LLM 的动态决策买回灵活性 |
| 成本敏感、有上限要求 | **Workflow** | 有界步数 = 有界账单 |
| 合规、可审计要求高 | **Workflow** | 审计员读图不读轨迹 |
| 开放性研究、长任务 | **Agent** | 真需要模型自治 |
| 新域、无现成 path | **先 Agent 探索 → 后 Workflow 固化** | 跑出来后常能固化成更便宜的 workflow |

最后一行是**最重要的工程经验**之一 —— 这就是"agent 探索 → workflow 落地"的两阶段法。

---

## 六、代码实现（`code/main.py`）结构

`code/main.py` 用 stdlib + 一个 `ScriptedLLM` 把五种模式都实现了一遍（每种 ~10–15 行）：

```python
prompt_chain(input, steps)          # 顺序串接
route(input, classifier, handlers)  # 分类 + 派发
parallel_vote(prompt, n, aggregator)# N 路并发 + 聚合
orchestrator_workers(task, workers) # 一次性规划
evaluator_optimizer(task, proposer,
                    evaluator, max_iter)  # 循环直到通过
```

`ScriptedLLM` 让这一切离线可跑（不调真实 API），每种模式打印 trace 便于教学。

---

## 七、Use It：什么时候才用框架

> Direct API calls for most tasks.
>
> Framework only when the pattern genuinely needs **durable state (LangGraph)**, **actor-model concurrency (AutoGen v0.4)**, or **role templating (CrewAI)**.
>
> Reach for the Claude Agent SDK when you want the Claude Code harness shape without rebuilding it.

这句话翻译成决策树：

```
任务能直接 API 调吗？
   ├─ 是 → 直接调
   └─ 否 → 需要持久状态吗？
              ├─ 是 → LangGraph
              └─ 否 → 需要 actor-model 并发吗？
                        ├─ 是 → AutoGen
                        └─ 否 → 需要角色模板吗？
                                  ├─ 是 → CrewAI
                                  └─ 否 → Claude Agent SDK
```

注意：**这些框架都是"为了实现某一种特定 workflow 模式而存在"**，不是"为了更强大而存在"。

---

## 八、上下文工程配套（论文提名）

> *Effective context engineering for AI agents* (Anthropic 2025)

本章提到了一个配套论文 —— 紧贴本主题但已超出范围：

- **200k 窗口是预算，不是容器**。
- 包含什么、什么时机压缩、什么时候放上下文自然增长 —— 这是相邻学科。
- 详细讲解在 phase 14 较早的某节课（原文标注：本章编号前）。这里不展开。

---

## 九、课后练习 5 题核心思路

1. **Routing + 置信度阈值**：低置信度降级为人工；tier-1 支持通常 0.7–0.85 是合理阈值。
2. **`parallel_vote` 加超时**：一条挂掉怎么处理？常见策略 —— 超时视为弃票（多数派容忍），或让 aggregator LLM 综合剩余结果，或 fallback 到无投票版本。
3. **Evaluator-optimizer 改老虎机**：保留 top-2 而不是单一 best，模拟了"小集合求最优"防御"过拟合某次评估"。
4. **Routing + Chaining 组合 + token 成本对比**：路由省钱，但 router 本身是一次额外 LLM 调用；小任务未必优于大 prompt，要量化对比。
5. **给一个生产功能画 workflow 图、数步骤**：很多被宣称为 agent 的功能实际上画出来就是个 workflow（可能只 4–6 步）。这一题的工程价值最大 —— 让你看清"是不是该退回 workflow"。

---

## 十、本章给我的工程启示

1. **"复杂度是成本"** 是这一章的总纲。每加一层框架，你都要问一句：**它买回了什么？** 如果答案是"未来灵活"，那它不值得当前成本 —— 灵活性是推断出来的，不是免费的。
2. **Augmented LLM 才是真正的原子**。所有五种模式只是组装方式不同。在 agent 框架里找"最小可控单元"的工作，常常会落到这一层。
3. **Orchestrator-workers 是"伪 agent 的最大公约数"** —— 它长得像 agent，但成本和可控性都偏向 workflow。这是你面对"看起来该用 agent"的需求时**先试一步**的折中方案。
4. **先 Agent 探索，后 Workflow 落地** —— 这条路径在 2025 年很多生产案例中被验证。
5. **避免 framework lock-in**。直接 API 调 + 10–15 行的胶水代码绝大多数情况就够。等到框架真的回本了再引入。

---

## 十一、横向对比（与前后章节）

| 章 | 模型 | 何时用 |
|---|---|---|
| 14-12 Workflow | 工程师写死图 | 步骤可枚举 |
| 14-13 LangGraph | 状态图 + 节点 + 边 | 持久状态、长流程 |
| **14-14 AutoGen** | **Actor / 消息 / 异步** | **天然并发 + 故障隔离 + 分布** |
| 14-15 CrewAI | 角色 + 任务模板 | 多个角色"按剧本" |

记一个口诀：**图 → 状态图 → actor 消息 → 角色剧本**，从最静态到最动态。如果你正在设计一个新系统，从静态往动态方向探索 —— 这种顺序决定了你能多快固化不必要的复杂度。
