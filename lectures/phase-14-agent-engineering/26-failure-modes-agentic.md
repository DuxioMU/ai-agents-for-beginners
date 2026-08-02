---
chapter: Failure Modes: Why Agents Break
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/26-failure-modes-agentic
source_doc: phases/14-agent-engineering/26-failure-modes-agentic/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-26 · 失败模式：Agent 为何崩

> MASFT (Berkeley, 2025) catalogs 14 multi-agent failure modes in 3 categories. Microsoft's Taxonomy documents how existing AI failures amplify in agentic settings. Industry field data converges on five recurring modes: hallucinated actions, scope creep, cascading errors, context loss, tool misuse.

## 一、本章在整体中的位置

前几章讲"如何让 agent 更强"。本章切到**"agent 是怎么挂的"**——**只有命名才能监控**。

> **你的 agent 在 90% trace 上工作。10% 失败不是随机噪声**——**它们集中在少数反复出现的类别**。**一旦你能命名，你就能监控+修复**。

---

## 二、MASFT（Berkeley, arXiv:2503.13657）

> **Multi-Agent System Failure Taxonomy** —— 14 个失败模式，3 类。**Inter-annotator Cohen's Kappa 0.88**（类别可靠可分）。

### 中心论点

> **失败是 multi-agent 系统的设计缺陷，不是 LLM 限制（无法靠更好 base model 修复）**。

**这一条非常关键**：**问题不在模型，在系统**。**更好的模型 ≠ 更好的系统**。

---

## 三、Microsoft Taxonomy

- **现有 AI 失败**（bias、hallucination、data leakage）—— **在 agentic 场景下放大**。
- **新失败涌现**（autonomy 带来的）：unintended action at scale、tool misuse、mission drift。
- **白皮书是 agentic 产品的风险登记册**。

---

## 四、LLM Agent Hallucinations Survey（arXiv:2509.18970）

### 两种主要表现

1. **Instruction-following Deviation** —— agent 不跟 system prompt。
2. **Long-range Contextual Misuse** —— agent 忘 / 误用前轮 context。

### Sub-intention errors

- **Omission**（漏步）
- **Redundancy**（重复步）
- **Disorder**（乱序步）

---

## 五、五个反复出现的行业失败模式

**Arize / Galileo / NimbleBrain 2024-2026 现场数据汇聚：**

| 模式 | 描述 |
|---|---|
| **1. Hallucinated actions** | agent 调用不存在的工具或编造参数 |
| **2. Scope creep** | agent 扩展任务（创建额外 PR、发额外邮件） |
| **3. Cascading errors** | 一次错调用触发下游多系统事件 |
| **4. Context loss** | 长任务忘前轮约束 |
| **5. Tool misuse** | 对的工具错参数，或错的工具 |

### 最致命的是 cascading

> **Agents cannot distinguish "I failed" from "the task is impossible" and often hallucinate a success message on 400 errors to close the loop.**

**一个幻 SKU 触发 4 个 API call = 多系统事件**。**400 错误上 agent 假装成功**——**最危险的失败模式**。

---

## 六、缓解：每步加闸门

| 闸门 | 章节 |
|---|---|
| **Per-step safety classifier** | 14-21 |
| **Tool-call argument validation** | 14-06 |
| **Cross-check retrieved content against facts** | 14-05（CRITIC） |
| **Detect success hallucination by re-probing state** | 本章 |

---

## 七、本章的失败模式（明文警告）

1. **Tagging only crashes** —— 多数失败输出"看起来正常"。**需要 content-level 检查**。
2. **No baseline** —— 漂移检测需"上次已知好的"。**没 baseline 你说不了"在变差"**。
3. **Over-alerting** —— 每个失败发页。**聚类 + 限速**。

---

## 八、代码（`code/main.py`）速览

```python
# 5 模式合成数据集
synthetic_traces = [...]

# 每模式 detector（tool call / output / repeat 的 signature pattern）
detectors = {
    "hallucinated_actions": ...,
    "scope_creep": ...,
    "cascading_errors": ...,
    "context_loss": ...,
    "tool_misuse": ...,
}

# 标每 trace + 报分布
tagger = FailureTagger(detectors)
labels = tagger.tag_all(traces)
```

Demo 输出：每 trace 标签 + 聚合分布——**镜像 Phoenix trace clustering**。

---

## 九、Use It 决策树

```
你的失败模式检测？
  ├─ 生产漂移聚类 → Phoenix（14-24）
  ├─ Session replay + annotation → Langfuse
  └─ 自定义 domain signature → 自建 detector
```

---

## 十、课后练习 5 题核心思路

1. **加 "success hallucination" detector**：agent 返回成功但目标状态未变。**这是最危险的模式**。
2. **标 100 真 trace**：哪个模式占主导？**修它要花多少**？
3. **"cascade radius" 指标**：step N 失败影响多少下游 step？**量化 ripple**。
4. **MASFT 14 模式选 3 个对产品**：写 detector。
5. **CI 接入**：≥5% trace 标某模式 = fail build。**质量门禁**。

---

## 十一、本章给我的工程启示

1. **失败不是随机**——**它们有名有姓**。**命名 → 监控 → 修复**。
2. **Cascading 是杀手**。**一次小错传成大事件**。**detect 越早，破坏越小**。
3. **Success hallucination 特别阴险**。**400 错 + agent 说"完成"= 噩梦**。**必须 re-probe state 验证**。
4. **MASFT Kappa 0.88 = 类别可靠**。**这 14 个不只是研究分类，是工程检查清单**。
5. **更好的模型不解决这些问题**。**这是系统问题**。**别等 GPT-6**。

---

## 十二、横向对比

| 失败类别 | 来源 | 核心症状 |
|---|---|---|
| **MASFT 14 模式** | Berkeley | 三大类：specification / inter-agent / system |
| **Microsoft** | Microsoft | 现有 AI 失败在 agentic 放大 + 新涌现失败 |
| **5 行业模式** | Arize/Galileo 现场 | hallucinated / scope / cascade / context / tool |

**统一口诀**：

> **失败有名有姓；Cascading 杀手；Success hallucination 最阴险；BASELINE 必存；不靠模型救系统。**
