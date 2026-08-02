---
chapter: Eval-Driven Agent Development
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/30-eval-driven-agent-development
source_doc: phases/14-agent-engineering/30-eval-driven-agent-development/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-30 · Eval 驱动的 Agent 开发

> Anthropic's guidance: "start with simple prompts, optimize them with comprehensive evaluation, and add multi-step agentic systems only when needed." Evaluation is not the last step. It's the outer loop that drives every other choice in Phase 14.

## 一、本章在整体中的位置

> **Evaluation is not the last step. It's the outer loop that drives every other choice in Phase 14.**

**Eval 不是收尾**。**Eval 是外层循环**——**驱动 phase 14 每个选择**。

> **Agents pass demos. They fail in production in ways demos cannot predict.**

**Benchmarks 答"模型普遍能力强吗"**——**不答"我的 agent 在我的产品上 ship 对 patch 吗"**。

---

## 二、三层评估

### 1. 静态基准（Static benchmarks）

| Benchmark | 用途 | 章节 |
|---|---|---|
| **SWE-bench Verified** | code agent | 14-19 |
| **WebArena / OSWorld** | browsing / desktop | 14-20 |
| **GAIA** | generalist | 14-19 |
| **BFCL V4** | tool use | 14-06 |

**用途**：跨模型对比、回归门禁。**污染是真**：SWE-bench+ 32.67% 修法泄漏。**必报 Verified**。

### 2. 自定义离线（Custom offline evals）

- **LLM-as-judge**（Langfuse / Phoenix / Opik，14-24）。
- **执行式**（跑 patch，check test）。
- **轨迹式**（vs gold；OSWorld-Human 暴露 1.4-2.7x 步数浪费）。

### 3. 在线（Online evals）

- **Session replays**（Langfuse）。
- **Guardrail-triggered alerts**（14-16 / 14-21）。
- **Per-step cost / latency tracking**（14-23 OTel）。

---

## 三、Evaluator-optimizer 紧循环（Anthropic）

```
Proposer 生成
   ↓
Evaluator 判
   ↓
   精修到 pass
```

**这是 14-05 Self-Refine 的推广**。**任何你关心的 agent flow 都可包**。

---

## 四、2026 最佳实践

- **Evals 与代码同住**。
- **每个 PR 在 CI 跑**。
- **门禁 merge**（如"vs main 不退化 > 5%"）。
- **每条 guardrail 映射到一个 eval case**。
- **每条 learned rule（Reflexion / pro-workflow）映射到一个失败 case**。

---

## 五、Phase 14 全章映射到 eval case

| 章 | Eval case |
|---|---|
| 01 Agent Loop | budget 耗尽 / 无限循环 guard |
| 02 ReWOO | planner 在工具失败时重 plan |
| 03 Reflexion | 学到的 reflection 重试时生效 |
| 05 Self-Refine/CRITIC | judge 接受精修输出 |
| 06 Tool Use | 参数 coercion 生效；未知工具被拒 |
| 07-10 Memory | 检索引用匹配源；过时事实作废 |
| 12 Workflow Patterns | 每模式产生正确输出 |
| 13 LangGraph | 恢复状态完全一致 |
| 14 AutoGen Actors | DLQ 接住 crashed handler |
| 16 OpenAI Agents SDK | guardrail 在正确输入 trip |
| 17 Claude Agent SDK | subagent 结果回到 orchestrator |
| 19-20 Benchmarks | SWE-bench V / WebArena / OSWorld efficiency |
| 21 Computer Use | per-step safety 抓住 injected DOM |
| 23 OTel | spans 发必要属性 |
| 26 Failure Modes | detector 标已知失败 |
| 27 Prompt Injection | PVE 拒 poisoned retrieval |
| 28 Orchestration | supervisor 路由到正确 specialist |
| 29 Runtime Shapes | DLQ 处理 N% 失败 |

> **如果你的 eval suite 每条都覆盖了，你就覆盖了 phase 14**。

---

## 六、本章的失败模式（明文警告）

1. **No baseline** —— Eval 无 last-known-good = 不可读。**存 baseline**。
2. **LLM-judge without grounding** —— Judge 也会幻觉。**CRITIC 模式（14-05）**——**judge 用外部工具**。
3. **Over-fitting to evals** —— 优化 eval 偏离 production 实用性。**轮换 case**。
4. **Flaky evals** —— 不确定 case 假警报。**固定 seed，snapshot state**。

---

## 七、代码（`code/main.py`）速览

```python
# case 注册（benchmark / custom / online 三类）
cases = [...]

# scripted agent under test
agent = ScriptedAgent(...)

# evaluator-optimizer 循环
def eval_optimize(agent, case, max_rounds=3): ...

# CI gate：聚合 pass rate + 回归 vs baseline
verdict = ci_gate(results, baseline)
```

Demo 输出：每 case pass/fail、回归 flag、CI gate 裁决。

---

## 八、Use It 决策树

```
你的 eval 体系？
  ├─ Static benchmark：SWE-bench V / WebArena / OSWorld / GAIA
  ├─ Custom offline：LLM-judge + 执行式 + 轨迹式
  ├─ Online：session replay + guardrail alert + cost/latency
  └─ CI gate：每 PR 跑、> 5% 退化 fail
```

---

## 九、课后练习 5 题核心思路

1. **从生产失败取 1 条**：写 eval case 复现它。**现在过吗？**
2. **领域 LLM-judge rubric**（factual / tone / scope）：评 50 session。
3. **Eval suite 接入 CI**：≥ 5% 退化 fail。
4. **轨迹效率指标**：agent 步数 vs gold。
5. **Phase 14 每章映射到 eval case**：缺哪个 = gap to close。

---

## 十、本章给我的工程启示

1. **Eval 是外层循环**。**不是最后一步**。**它驱动每个选择**。
2. **三层评估互补**：benchmark 防退化、custom 测产品、online 看真生产。
3. **每 guardrail = 一条 eval case**。**这是"防御可验证"**。
4. **Phase 14 全章可映射到 eval case**。**"你覆盖了 phase 14" = "你的 eval suite 完整"**。
5. **CI gate 让 eval 不可绕**。**不是开发者记得跑**——**PR 不通过不能 merge**。

---

## 十一、横向对比

| 层 | 何时用 | 工具 |
|---|---|---|
| **Static** | 跨模型 / 回归门禁 | SWE-bench V / WebArena / OSWorld |
| **Custom offline** | 测产品 | LLM-judge / exec / trajectory |
| **Online** | 测生产 | session replay / guardrail alert / OTel |

**统一口诀**：

> **Eval 是外层循环；三层互补；每 guardrail 一条 case；CI gate 不可绕；phase 14 每章必映射。**
