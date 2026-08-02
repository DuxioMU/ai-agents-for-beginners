---
chapter: The Actor Model for Agents — Async Messages and Typed Runtimes
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/14-autogen-actor-model
source_doc: phases/14-agent-engineering/14-autogen-actor-model/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-14 · Actor 模型与 Agent：异步消息与类型化运行时

> Agents as actors: async message exchange, event-driven handlers, fault isolation, natural concurrency. AutoGen v0.4 (Microsoft Research, Jan 2025) redesigned agent orchestration around this model; the framework is now in maintenance mode, with Microsoft Agent Framework (public preview Oct 2025) as its production successor.

## 一、本章在整体中的位置

前面的章节都在讲"agent 内部如何思考"（ReAct、ReWOO、HTN、Self-Refine）或"agent 之间如何合作"（workflow 模式）。本章是第一次把视角切到**运行时形态本身** —— 智能体如何作为软件实体被调度、通信、隔离。

主题：**AutoGen v0.4**（Microsoft Research，2025 年 1 月）提出的 actor-model 重构，以及它对"分布式、可观测、可失败"的工程影响。

补充信息：AutoGen 目前已经进入 **maintenance mode**，微软把生产开发转向了 Microsoft Agent Framework（2025-10 公开预览，2026 Q1 末 GA）。但 **actor 模型作为思想是耐用的** —— 模式可以平滑迁移。

---

## 二、要解决的问题：同步调用的工程痛点

传统 agent 框架（包括 AutoGen v0.2 之前）是**同步调用式**：

```python
result = agent_b.chat(agent_a)   # 同步阻塞直到对方返回
```

这会带来三个连锁问题：

| 问题 | 后果 |
|---|---|
| **崩溃传染** | Agent B 抛异常 → 调用栈断 → Agent A 也死了 |
| **并发别扭** | "多 agent 同时进行"靠线程/async 包一层 |
| **分布困难** | 本地调用 ≠ 网络调用 → 要重写 |

AutoGen v0.4 的回答：**actor model**。

---

## 三、Actor 模型（本章的理论基础）

Actor 模型是 1973 年由 Hewitt 等人提出的并发计算模型，近年被 Erlang/Akka 推到工业级。AutoGen v0.4 是它首次系统化进入 LLM agent 编排领域。

### 一个 actor 的三件套

```
┌──────────────────────────────┐
│  Actor                       │
│                              │
│  ┌────────────────────────┐  │
│  │ Private state          │  │  ← 外部任何代码都不能直接读写
│  └────────────────────────┘  │
│                              │
│  ┌──────┐                    │
│  │Inbox │ ← messages waiting│
│  └──┬───┘                    │
│     │                        │
│     ▼                        │
│  handler(msg, runtime)       │  ← 决定 effects
│                              │
│  effects 可选：              │
│   • reply(msg)               │
│   • send(other, msg)         │
│   • spawn(new_actor)         │
│   • update_state(...)        │
│   • stop_self()              │
└──────────────────────────────┘
```

### 核心约束（也是最大威力）

> **两个 actor 之间没有任何共享内存。**
> **它们之间唯一的交互方式：发消息。**

这条约束是所有好处的源头：

| 特性 | 怎么来的 |
|---|---|
| **故障隔离** | handler 抛异常 → 运行时捕获 → 其他 actor 不受影响 |
| **天然并发** | 每个 actor 处理自己的 inbox，可以并发 |
| **分布透明** | "投递" 是一个抽象 —— 本地队列 / HTTP / NATS / gRPC 都是它的实现 |
| **可推理性** | 没有共享状态意味着每个 actor 是隔离单元，推理边界清晰 |

---

## 四、AutoGen v0.4 的三层 API

AutoGen v0.4 把整个表面拆成三层，**自下而上越来越高级**，这是它最被低估的设计点：

```
┌────────────────────────────────────┐
│  Extensions（集成）                │
│   - OpenAI / Anthropic / Azure    │
│   - 工具、Memory                   │
├────────────────────────────────────┤
│  AgentChat（任务驱动高层 API）      │
│   - AssistantAgent                │
│   - UserProxyAgent                │
│   - RoundRobinGroupChat           │
│   - SelectorGroupChat             │
├────────────────────────────────────┤
│  Core（actor framework 底层）       │
│   - AgentRuntime                 │
│   - Agent                         │
│   - Message / Topic              │
└────────────────────────────────────┘
```

| 层 | 谁用 | 用它做什么 |
|---|---|---|
| **Core** | 框架作者 / 高级用户 | 设计新的 agent 类型、运行机制、传输层 |
| **AgentChat** | 应用开发者 | 大多数 multi-agent 任务 |
| **Extensions** | 所有人 | 接 LLM、接工具、接外部系统 |

**重要的"反模式警告"**：很多人卡在 Core 上写业务逻辑，又或只用 Extensions 不能解决复杂任务。三层要清楚选择。

---

## 五、解耦投递与处理：v0.4 与 v0.2 的核心差

```python
# v0.2 同步模型 —— 阻塞栈
answer = agent_a.chat(agent_b)   # 等 agent_b 完成才返回

# v0.4 解耦模型 —— 入队即返回
runtime.send(agent_b, msg)       # 入箱即返回，不等待处理
```

差别非常小，但后果是结构性的：

### 三个连锁后果

1. **故障隔离**
   v0.2：agent_b 抛异常 → 整个栈断 → agent_a 也受牵连。
   v0.4：agent_b handler 抛异常 → runtime 捕获并决定策略（重试、dead-letter、忽略）→ agent_a 继续。

2. **天然并发**
   v0.2：一次只能一对一（或手动开线程）。
   v0.4：每个 actor 独立消费 inbox，runtime 自然支持 N 路消息同时处理。

3. **分布透明**
   "投递"就是个抽象 —— 把 in-process 队列换成 NATS，actor 可以天然跨主机。**这是 actor 模型最被低估的特性**。

---

## 六、群聊拓扑（AgentChat 三种团队形态）

| 拓扑 | 何时发言 | 谁决定 |
|---|---|---|
| **RoundRobinGroupChat** | 固定轮转 | 运行时按序派发 |
| **SelectorGroupChat** | 上下文相关 | 一个 selector actor 决定下一位 |
| **Magentic-One** | 多面手 | 协调者 + 多个 specialist（web、code、files） |

- **RoundRobin** 用于讨论板、轮值批阅这类场景；可预测、成本有界。
- **Selector** 用于：下一步谁擅长需要根据上下文判断，例如"上次回答错了，应该请另一个专家"。
- **Magentic-One** 是微软给的"参考团队"，主要给 web browsing + code execution + file operations 这种复杂任务用。

---

## 七、可观测性（OTel GenAI）

AutoGen v0.4 默认开 OpenTelemetry。每条消息发一个 span，工具调用携带 `gen_ai.*` 属性 —— 完全符合 OTel GenAI 语义约定（这是第 23 章要展开的内容，本章只提名）。

这意味着：开了 OTel 收集器你就能在 tracing 后台看到 actor 间消息图、延迟分布、失败点。**对调试多 agent 比直接 print 强一个数量级。**

---

## 八、维护状态：这是工程上必须知道的真相

> **AutoGen v0.7.x 处于维护模式（截至 2026 初）**。
>
> 微软把生产开发搬到 **Microsoft Agent Framework**：
> - 公开预览：2025-10-01
> - 1.0 GA：原目标 2026 Q1 末
>
> AutoGen 模式可平滑迁移 —— **actor model 思想是耐用品**。

这是个非常重要的"软知识"：选 actor 模型不意味着绑定到微软的实现。学的是模式，绑定的是平台。

---

## 九、代码实现（`code/main.py`）

`code/main.py` 用 stdlib 实现了一个迷你 actor runtime：

```python
@dataclass
class Message:
    sender: str
    recipient: str
    topic: str
    body: Any

class Actor:                           # 抽象基类
    async def receive(self, msg, runtime): ...

class Runtime:                        # 事件循环 + 投递 + 故障隔离
    async def send(self, to, msg): ...
    async def run(self): ...
```

配套 Demo：`ReviewerAgent` + `ChecklistAgent` 双向通信达成共识；故意在某一步抛异常来演示 **一个 actor 抛错不影响另一个** —— 这是 actor 模型相对同步模型的核心交付。

运行：
```bash
python3 code/main.py
```

---

## 十、Use It：什么时候选 actor 模型

```
任务："我需要 N 个 agent 协作"
  ├─ 任务是线性 / 树形 → 用 LangGraph（chapter 13）状态图即可
  ├─ 任务需要自然并发 + 故障隔离 → 选 actor 模型（本章）
  └─ 任务只需角色模板 → 用 CrewAI（chapter 15）
```

actor 模型特别擅长：
- **长时多 agent 任务** —— web 浏览 + 代码执行 + 文件处理类。
- **跨进程 / 跨主机部署** —— transport 抽象使然。
- **每个 agent 能力差异大** —— actor 之间用消息而非共享内存协作。

---

## 十一、课后练习 5 题核心思路

1. **死信队列（DLQ）**：handler 抛出时把消息 park 起来供人查。本章 toy 大概每 5–20 次跑会触发一次 —— DLQ 不是 unusual，是日常。
2. **`SelectorGroupChat` 自实现**：额外加一个 selector actor，输入是当前 conversation state，输出是 `next_actor`。这其实是个"小 LLM + 投票模式"。
3. **分布式 transport 切换**：把 in-process 队列换成 JSON-over-HTTP。**这是把 actor model 思想变现成生产力的关键一步** —— 一旦有 transport，立刻分布式。
4. **OTel span wire**：每条消息一个 span，属性 `gen_ai.agent.name`、`gen_ai.operation.name`。哪怕只是 stub 也比没 trace 强。
5. **真把 toy 端口到 `autogen_core`**：你会发现现实少了什么 —— **消息序列化、跨进程 capability、重试策略、订阅与 topic 路由** —— 这些都是"想清楚后才知道缺"的工程能力。

---

## 十二、本章给我的工程启示

1. **同步调用是分布的敌人**。一旦代理数量上去、跨进程上云，agent 之间的同步栈立刻变成死结。actor 模型从设计上解决了这一点。
2. **三层 API 是组织复杂性的胜利**：Core/AgentChat/Extensions 让"发明新 agent 类型"和"用现成 agent 完成任务"分离。
3. **故障隔离是分布式 agent 的第一需求**。当 actor 多到某个数，"一个坏掉其他的都不受影响"从 nice-to-have 变成 must-have —— actor 模型就是为此而生。
4. **绑定的是平台，思想是资产**。AutoGen 进入维护模式这件事告诉我：**actor model 概念会在 Microsoft Agent Framework / 其他生态持续存在**；我不该押宝到具体 API，应押宝到思想。
5. **从本地到分布 = 换 transport，不改模型**。这意味着今天用 stdlib 写 actor runtime，明天切到 NATS/gRPC，你的业务逻辑一行不变 —— 这是 actor 模型最诱人的长期价值。

---

## 十三、横向对比（与前后章节）

| 章 | 模型 | 何时用 |
|---|---|---|
| 14-12 Workflow | 工程师写死图 | 步骤可枚举 |
| 14-13 LangGraph | 状态图 + 节点 + 边 | 持久状态、长流程 |
| **14-14 AutoGen** | **Actor / 消息 / 异步** | **天然并发 + 故障隔离 + 分布** |
| 14-15 CrewAI | 角色 + 任务模板 | 多个角色"按剧本" |

记一个口诀：**图 → 状态图 → actor 消息 → 角色剧本**，从最静态到最动态。如果你正在设计一个新系统，从静态往动态方向探索 —— 这种顺序决定了你能多快固化不必要的复杂度。
