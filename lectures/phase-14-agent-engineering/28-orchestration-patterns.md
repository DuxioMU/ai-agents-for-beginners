---
chapter: Orchestration Patterns: Supervisor, Swarm, Hierarchical
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/28-orchestration-patterns
source_doc: phases/14-agent-engineering/28-orchestration-patterns/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-28 · 编排模式：Supervisor、Swarm、Hierarchical

> Four orchestration patterns recur across 2026 frameworks: supervisor-worker, swarm / peer-to-peer, hierarchical, debate. Anthropic's guidance: "It's about building the right system for your needs." Start simple; add topology only when a single agent plus five workflow patterns is insufficient.

## 一、本章在整体中的位置

第 14-12 章讲了 **5 种 workflow 模式**（chaining、routing、parallelization、orchestrator-workers、evaluator-optimizer）。本章切到**多 agent 拓扑**——**当一个 agent 不够时**：

> **Teams reach for "multi-agent" before they need it.**

**这是本章的告诫**：**先单个 agent + workflow，不够再加拓扑**。

---

## 二、四种编排模式

### 1. Supervisor-worker

```
       Supervisor LLM
         /  |  \
        ↓   ↓   ↓
   spec  ref  bill
```

- 中心路由 LLM 派发给专家。
- 决策：loop back / 派 specialist / terminate。
- **Specialist 之间不直接对话**——**所有路由过 supervisor**。
- 框架：LangGraph `create_supervisor`、Anthropic orchestrator-workers、CrewAI Hierarchical。

#### 2026 LangChain 建议

> **Do supervision through direct tool calls rather than `create_supervisor`.**

**不用库，直接 tool call 监督**——**更细的 context engineering 控制**。

### 2. Swarm / peer-to-peer

- Agent 通过 shared tool surface **直接 handoff**。
- 无中心 router。
- 延迟低（少一跳）。
- 难推理（无单点控制）。
- 框架：LangGraph swarm、OpenAI Agents SDK handoffs（all-to-all）。

### 3. Hierarchical

- Supervisors managing sub-supervisors managing workers。
- LangGraph nested subgraphs、CrewAI nested crews。
- **大 agent 群**才用。
- 何时用：单 supervisor context 装不下所有 specialist 描述。

### 4. Debate

- 并行提议 + 迭代互批（14-25）。
- 不是真 orchestration，是 verification——但作为拓扑选项存在。

---

## 三、Crew vs Flow 维度

CrewAI 形式化两个部署模式（与四模式正交）：

| 模式 | 形态 | 何时用 |
|---|---|---|
| **Flow** | 确定性事件驱动 | **生产推荐起点** |
| **Crew** | 自助角色协作 | 探索 / draft |

**映射到拓扑**：Flow 通常是 supervisor / hierarchical；Crew 通常是 supervisor + LLM router。

---

## 四、Anthropic 的指导（核心决策序）

> **"Success in the LLM space isn't about building the most sophisticated system. It's about building the right system for your needs."**

**决策序**：

1. **单 agent + workflow 模式**（14-12）—— **起点**。
2. **Supervisor-worker** —— 2-4 个 specialist。
3. **Swarm** —— 延迟比清晰度重要。
4. **Hierarchical** —— supervisor context 装不下时。
5. **Debate** —— 准确率比成本重要。

---

## 五、本章的失败模式（明文警告）

1. **Topology-first thinking** —— "我们要 multi-agent" 在识别问题前。**问题先**。
2. **Bouncing handoffs in swarm** —— A→B→A→B。**hop counter**。
3. **Fake hierarchy** —— 3 层因为"企业"，实际 2 团队。**压平**。

---

## 六、代码（`code/main.py`）速览

```python
class Supervisor: ...           # 中心 router
class Swarm: ...                 # peer-to-peer 直接 handoff
class Hierarchical: ...          # supervisor of supervisors
class Debate: ...                # 并行提议 + 批评

# 同一 3-intent 任务（refund / bug / sales）走 4 模式
# trace 形状不同
```

Demo：每模式 trace + op 数。

- **Supervisor**：最清晰。
- **Swarm**：最短。
- **Hierarchical**：最深。
- **Debate**：最贵。

---

## 七、Use It 决策树

```
你的编排？
  ├─ 2-4 specialist + supervisor → LangGraph
  ├─ 低延迟 peer-to-peer → OpenAI Agents SDK handoffs
  ├─ 确定性事件驱动生产 → CrewAI Flow
  ├─ 大 agent 群 + 嵌套 → Hierarchical（LangGraph/CrewAI）
  └─ 准确率优先 → Debate（14-25）
```

---

## 八、课后练习 5 题核心思路

1. **Supervisor → Swarm**：去掉 router。**什么崩？什么好？**
2. **加 hop counter**：swarm 3 次后拒。**A→B→A bouncing 抓到吗？**
3. **12 specialist 2-level hierarchical**：**单 supervisor context 哪里失败？**
4. **4 模式 production-shaped profile**：每模式比 latency / cost / accuracy / debuggability。
5. **读 Anthropic "Building Effective Agents"**，把生产 flow 映射到 4 模式。

---

## 九、本章给我的工程启示

1. **拓扑是手段不是目标**。**先识别问题，再选拓扑**。
2. **Anthropic 决策序是金标准**。**单 agent + workflow 起步**。**多 agent 不到迫不得已不上**。
3. **Swarm 比 supervisor 快，难推理**。**这是 latency vs 可解释性的权衡**。
4. **Hierarchical 是 escape hatch**。**单 supervisor context 不够时才上**——**不要"为了层级而层级"**。
5. **Direct tool call supervision > `create_supervisor`**。**更细的 context 控制**。

---

## 十、横向对比

| 模式 | 中心 | 延迟 | 推理难度 | 何时用 |
|---|---|---|---|---|
| **Supervisor** | 有 | 中 | 低 | 2-4 specialist |
| **Swarm** | 无 | **低** | 高 | 延迟敏感 |
| **Hierarchical** | 嵌套 | 高 | 中 | 大 agent 群 |
| **Debate** | 无 | **高** | 中 | 准确率优先 |

**统一口诀**：

> **单 agent 起步；supervisor 加 2-4 specialist；swarm 拼延迟；hierarchical 拼规模；debate 拼准确率。**
