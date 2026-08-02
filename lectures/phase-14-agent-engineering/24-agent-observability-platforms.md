---
chapter: Agent Observability — Langfuse, Phoenix, Opik
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/24-agent-observability-platforms
source_doc: phases/14-agent-engineering/24-agent-observability-platforms/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-24 · Agent 可观测性平台：Langfuse、Phoenix、Opik

> Three open-source agent observability platforms dominate 2026. Langfuse (MIT) — 6M+ installs/month, tracing + prompt management + evals + session replay. Arize Phoenix (Elastic 2.0) — deep agent-specific evals, RAG relevancy, OpenInference auto-instrumentation. Comet Opik (Apache 2.0) — automated prompt optimization, guardrails, LLM-judge hallucination detection.

## 一、本章在整体中的位置

第 14-23 章给了"标准 schema"（OTel GenAI 约定）。**但只有 schema 不够**——**得有产品来收 span、做评估、存 prompt 版本、暴露回归**。这就是本章的主题。

> **OTel GenAI (Lesson 23) gives you the schema. You still need the platform that ingests spans, runs evaluations, stores prompt versions, and surfaces regressions.**

**三大开源平台的分工**：

| 平台 | License | 强项 |
|---|---|---|
| **Langfuse** | **MIT** | 端到端 + **prompt 管理** + session replay |
| **Arize Phoenix** | **Elastic 2.0** | **RAG 相关性** + auto-instrument + 行为漂移 |
| **Comet Opik** | **Apache 2.0** | **自动化 prompt 优化** + guardrails + LLM-judge |

**一个行业数据先记住**：

> **89% of organizations have agent observability in place; quality issues are the top production barrier (32% of respondents cite them).**

**质量问题是生产最大障碍**——**可观测性是质量问题的解药**。

---

## 二、Langfuse（MIT）

### 基本面

- **6M+ SDK installs / month**、**19k+ GitHub stars**。
- **2025-06**：原商用模块（LLM-as-a-judge / annotation queues / prompt experiments / Playground）**全部 MIT 开源**。

### 核心能力

- **Tracing**（OTel 兼容 / 自家 SDK）
- **Prompt management** with **versioning + playground**
- **Evaluations**（LLM-as-judge / 用户反馈 / 自定义）
- **Session replays**

### 强项定位

> **End-to-end observability with tight prompt-management loop.**

**"可观测性"和"prompt 工程"打通的**——**你的 prompt v3 跟 v4 在 production 表现差异直接看得见**。

### License 注意

**MIT**——**最宽松**。**闭源产品集成 / 商业 SaaS 嵌入**都没问题。

---

## 三、Arize Phoenix（Elastic 2.0）

### 基本面

- **深 agent-specific 评估**。
- **Native OpenInference auto-instrumentation**。
- 与 managed **Arize AX** 配对 production。
- **No prompt versioning**——刻意定位为"drift/behavioral-regression"工具，不重复造 prompt CMS 的轮子。

### 强项

- **Trace clustering**（相似 run 聚类，drift 检测）
- **Anomaly detection**
- **RAG relevancy**（检索内容是否真对得上查询）

### 强项定位

> **RAG relevancy, behavioral drift, anomaly detection.**

**这是"你的 agent 行为在悄悄变"的早期预警系统**。

### License 注意

**Elastic 2.0**（不是 OSI 标准开源）——**有商业限制**。**计划嵌入产品分发前要细看**。

---

## 四、Comet Opik（Apache 2.0）

### 基本面

- **Automated prompt optimization**（A/B 实验自动迭代）
- **Guardrails**（PII redaction、topical constraints）
- **LLM-judge hallucination detection**

### 强项

- **优化闭环**：自动 prompt 优化循环
- **自动化实验**：A/B 不用人盯
- **Guardrail enforcement**：PII / 越界在 log time 拦

### 强项定位

> **Optimization loop, automated experimentation, guardrail enforcement.**

**这是"agent 跑起来后还能继续变好"的引擎**。

### 一个 vendor benchmark

> **Opik logs + evals in 23.44s vs Langfuse 327.15s (~14x gap) — take vendor benchmarks as directional.**

**14x 速度差**——**但这是 Comet 自家测的**。**任何 vendor benchmark 都当方向看**。**自己 benchmark 才是真**。

### License 注意

**Apache 2.0**——**真开源**。**MIT / Apache 2.0 二选一，闭源友好**。

---

## 五、怎么选

| 需求 | 选 |
|---|---|
| **All-in-one + prompt 管理** | **Langfuse** |
| **深 RAG 评估 + 漂移检测** | **Phoenix** |
| **自动化优化 + guardrails** | **Opik** |
| **最宽松 license** | **Langfuse (MIT) / Opik (Apache 2.0)** |
| **混合 ops + ML 团队，已有 Datadog** | **Datadog LLM Observability** |
| **所有上面都想要** | **组合（Langfuse 主 + Phoenix 评测 + Opik 优化）** |

### Datadog / New Relic 怎么办

> **Any — they all export OTel.**

**你用 Datadog LLM Observability 也行**——**它们都吃 OTel**。**选了 Datadog 不代表放弃其他**。

---

## 六、本章的失败模式（明文警告）

1. **No eval strategy** —— 只有 trace 没有评估 = **贵日志**。**trace 是原料，eval 是成品**。
2. **Self-rolled LLM-judge without grounding** —— LLM-judge 也需要外部工具做事实核查（CRITIC 模式，14-05 章）。**别让 judge 凭感觉打分**。
3. **Prompt versions not tied to traces** —— **prompt v3 跟 v4 在 prod 表现差异**看不到 = **回归没法 bisect**。**prompt 版本必须跟 trace 关联**。

**这三条是"装好平台后立刻踩"的陷阱**。

---

## 七、代码（`code/main.py`）速览

`code/main.py` 实现 stdlib trace collector + LLM-judge evaluator：

```python
# 摄取 GenAI-shaped spans
ingest(span) ...

# 按 session 分组，标 failed run
session_groups = group_by_session(spans)
tag_failures(session_groups, predicates=[
    "guardrail_tripped",
    "low_confidence_eval",
    "tool_error",
])

# LLM-judge 打分（按 rubric）
def judge(agent_output, rubric): ...

# dashboard-like summary
dashboard = {
    "failure_rate": ...,
    "top_failure_reasons": ...,
    "eval_score_distribution": ...,
}
```

Demo 输出镜像 Langfuse / Phoenix / Opik 的 dashboard 形态。

运行：
```bash
python3 code/main.py
```

---

## 八、Use It：决策树

```
你的主要诉求？
  ├─ 端到端 + prompt 版本管理 → Langfuse
  ├─ RAG 评估 + 行为漂移 → Phoenix
  ├─ 自动化 prompt 优化 + guardrails → Opik
  ├─ 已有 Datadog → Datadog LLM Observability
  └─ 多个都要 → 组合（Langfuse 主，Phoenix 评测，Opik 优化）
```

---

## 九、Ship It：`outputs/skill-obs-platform-wiring.md`

挑一个平台，把 trace + eval + prompt 版本接进现有 agent。

---

## 十、课后练习 5 题核心思路

1. **一周 OTel trace → Langfuse cloud（free tier）**：哪些 session 失败？为什么？**这是"我的 agent 在 prod 实际怎么挂"的第一手观察**。
2. **写领域 LLM-judge rubric**（factual / tone / scope）：50 trace 上测。**judge 本身的 bias 和 stability 也是被测的**。
3. **Langfuse prompt 版本 vs Phoenix trace clustering**：哪个更快告诉你"什么坏了"？**两个工具提供**不同视角**：版本对比 vs 行为聚类。
4. **Opik guardrail docs 读 + PII redaction 接**：给一个 agent run 装上。**这是把"政策"做成"运行时执行"**。
5. **自己 benchmark 三个平台**：忽略 vendor 数字，自己测。**14x 这种数字在你自己数据上常常不是 14x**。

---

## 十一、本章给我的工程启示

1. **89% 组织有 agent observability 不是 over-engineering**——**是质量问题的标准回答**。**如果你没有，先装一个，不要裸跑**。
2. **三个平台分工清晰：Langfuse = 端到端 + prompt；Phoenix = 评测 + 漂移；Opik = 优化 + guardrail**。**单一工具很难包打**——**组合是常态**。
3. **License 选择不是细枝末节**。**MIT / Apache 2.0 适合闭源嵌入；ELv2 有商业限制**。**产品形态决定 license 偏好**。
4. **LLM-judge 也要 grounding**——**别让 judge 凭感觉打分**。**CRITIC 模式（14-05）适用**。**judge 本身需要被校准**。
5. **prompt 版本 ↔ trace 关联是 regression debug 的命脉**。**没有这个关联，**回归发生时你不知道"是哪个 prompt 改坏的"**。

---

## 十二、横向对比

| 平台 | License | Tracing | Prompt Mgmt | Eval 强项 | Guardrails | 何时用 |
|---|---|---|---|---|---|---|
| **Langfuse** | **MIT** | ✅ OTel + SDK | **✅ versioning + playground** | LLM-as-judge | 中 | **端到端** |
| **Phoenix** | **ELv2** | ✅ OpenInference | ❌ | **RAG relevancy + drift** | ❌ | **评测 / 漂移** |
| **Opik** | **Apache 2.0** | ✅ | 中 | **Hallucination judge** | **✅** | **优化 / guardrail** |
| **Datadog** | 商业 | ✅ OTel | 中 | 中 | 中 | **已有 Datadog 团队** |

**统一口诀**：

> **Langfuse 端到端 + prompt；Phoenix 评测 + 漂移；Opik 优化 + guardrail。**
> **Trace 是原料，eval 是成品；judge 必 grounding；prompt ↔ trace 必关联。**
