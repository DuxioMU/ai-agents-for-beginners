---
chapter: Multi-Agent Debate and Collaboration
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/25-multi-agent-debate
source_doc: phases/14-agent-engineering/25-multi-agent-debate/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-25 · 多 Agent 辩论与协作

> Du et al. (ICML 2024, "Society of Minds") run N model instances that independently propose answers, then iteratively critique each other over R rounds to converge. Improves factuality, rule-following, reasoning. Sparse topology beats full mesh on token cost.

## 一、本章在整体中的位置

第 14-05 章讲了 **Self-Refine**（一个模型自我批评）和 **CRITIC**（带外部工具的批评）。本章是**第三条路：多实例互相批评，意见不一致→收敛**。

> **Self-Refine 是"一个 model 自我批评"= groupthink 风险。**
> **CRITIC 是"带工具批评"= 不是所有场景都有工具。**
> **Debate 是"多实例互批"= 凭"不同意见"收敛。**

---

## 二、Society of Minds（Du et al., ICML 2024）

### 协议

```
N 个独立模型实例对同一问题给初稿
   ↓
R 轮：每个模型读其他模型的提议，批评它们，更新自己的答案
   ↓
R 轮后：返回收敛答案
```

### 数字（基线 N=3, R=2）

| 任务 | 提升 |
|---|---|
| **MMLU** | 准确率提升 |
| **GSM8K** | 准确率提升 |
| **Chess Move Validity** | 提升（一个模型漏规则，其他抓到） |
| **Biography Generation** | 提升（多个 framing 收敛到更准） |

### 关键发现

- **N↑、R↑ 在难问题上继续有效**（成本敏感则需 sparse topology，见下）。
- **跨模型组合 > 单模型辩论**：ChatGPT + Bard 一起 > 各自单用。**多样性是辩论的灵魂**。

---

## 三、Sparse Topology（稀疏拓扑）

> **arXiv:2406.11776, 2024-2025**——**Full-mesh 不是最优**。

### 数字对比

| 拓扑 | 提议数 | 批评 op 数 |
|---|---|---|
| **Full mesh N=5, R=3** | 5×3=15 | 15×4 = **60** |
| **Star N=5, R=3**（hub + 4 spokes） | 15 | **12** |

**Star 比 Full mesh 少 5x 批评 ops，准确率常可追平**。

### 拓扑选择

- **Full mesh**：小 N、要求最高准确率。
- **Star（hub-and-spoke）**：成本敏感，1 个 hub 协调 N-1 spokes。
- **Ring**：折中。
- **Hierarchical**：超大 N。

---

## 四、什么时候 debate 有用

- **事实性**：N 个独立提议，**互相校验降低幻觉**。
- **规则遵守**：chess move validity — 一个 model 漏规则，其他抓到。
- **开放推理**：多个 framing 收敛到正确答案。

## 五、什么时候 debate 没用

- **延迟敏感 UX**：N×R 串行轮 = 你可能付不起的延迟。
- **成本敏感规模**：N×R tokens 一次问。
- **简单事实查询**：一次查询比 5 个 debate 便宜。

---

## 六、2026 实战形态

- **Anthropic orchestrator-workers**（14-12）= debate + 合成步。
- **LangGraph supervisor**（14-13）= 中心路由 + 专家 agent。
- **OpenAI Agents SDK handoffs**（14-16）= handoff 来回迭代批评。
- **Multi-agent evals** = debate + evaluator-optimizer 配对。

---

## 七、本章的失败模式（明文警告）

1. **Convergence collapse** —— 全 agent 收敛到第一个错答案。**强制 disagreement rounds**。
2. **Hub failure** —— Star 拓扑，bad hub 带坏所有人。**轮转或多 hub**。
3. **Prompt homogenization** —— 全部 agent 用同一 prompt，产出同质化。**diverse prompts + diverse models**。

---

## 八、代码（`code/main.py`）速览

```python
class Debater: ...                 # 脚本化 LLM，每个有 opinion drift
class FullMeshDebate: ...          # 全互连
class SparseDebate: ...            # 稀疏拓扑

# 3 道题：factual / rule-based / reasoning
# 度量：收敛答案、收敛轮数、批评 op 总数
```

Demo 输出：每协议准确率 + 成本；sparse 在 2/3 题上追平 full mesh 成本更低。

---

## 九、Use It 决策树

```
任务需要多视角？
  ├─ N=2-3 worker debate → Anthropic orchestrator-workers
  ├─ 多轮 stateful debate → LangGraph
  ├─ 高准确率要求 → full mesh N↑ R↑
  └─ 成本敏感 → sparse topology（star / ring）
```

---

## 十、课后练习 5 题核心思路

1. **强制 disagreement 规则**：round 1 每个 debater 必须出不同提议。**测收敛速度**。
2. **置信度加权聚合**：debater 返回 (answer, confidence)，aggregator 按 confidence 加权。**是否提升**？
3. **换不同脚本化 LLM**：heterogeneity 提升准确率吗？**多样性验证**。
4. **测 full mesh vs sparse token 成本**：3 题上画 cost vs accuracy 散点。
5. **N=5, R=3 端口到 toy**：什么崩？什么好？

---

## 十一、本章给我的工程启示

1. **多样性是辩论的灵魂**。**同 prompt 多实例 ≠ 多 agent**。**真多样性 = 多 prompt 或多 model**。
2. **稀疏拓扑常常够用**。**Full mesh 是浪费**——**先 star / ring 试**。
3. **N 和 R 是超参**。**baseline N=3, R=2；难题才加**。**成本先算**。
4. **Debate 不替代 Self-Refine**。**Self-Refine 适合单模型改进，Debate 适合多视角校准**。
5. **强制 disagreement 防 convergence collapse**。**没有这个 guard，辩论变 groupthink**。

---

## 十二、横向对比

| 方法 | N | 谁批评 | 何时用 |
|---|---|---|---|
| **Self-Refine** | 1 | 自己 | 单模型可改进 |
| **CRITIC** | 1 | 自己 + 外部工具 | 需事实核查 |
| **Debate (full mesh)** | N | 全部互批 | 高准确率 |
| **Debate (sparse)** | N | 子集互批 | 成本敏感 |
| **Star (hub-and-spoke)** | N | 仅 hub | 协调式 |

**统一口诀**：

> **多样性必装（多 prompt / 多 model）；稀疏常够用；强制 disagreement 防 collapse；N×R 成本先算。**
