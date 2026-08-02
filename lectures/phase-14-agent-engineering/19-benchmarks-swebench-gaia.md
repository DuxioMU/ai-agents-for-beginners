---
chapter: Benchmarks — SWE-bench, GAIA, AgentBench
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/19-benchmarks-swebench-gaia
source_doc: phases/14-agent-engineering/19-benchmarks-swebench-gaia/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-19 · 基准测试：SWE-bench、GAIA、AgentBench

> Three benchmarks anchor agent evaluation in 2026. SWE-bench tests code patching. GAIA tests generalist tool use. AgentBench tests multi-environment reasoning. Know their composition, their contamination story, and what they do not measure.

## 一、本章在整体中的位置

phase 14 走过了：ReAct 循环 → ReWOO 规划 → HTN/AlphaEvolve → Workflow → 各种 multi-agent 框架。**所有这些都靠"能不能跑"来判断好坏吗？** —— 这一章正式进入**评估学**。

> **Leaderboards tell you which model wins on one benchmark. They do not tell you:**
> - **是否被污染**（solutions 在训练数据里、test 泄漏）。
> - **是否度量了你关心的事**（code vs browsing vs generalist）。
> - **evaluator 是否鲁棒**（AST 匹配、状态检查、人工复核）。

**这一章的总纲**：在引用任何数字之前，先知道这个数字的**测试装置、污染故事、它不测什么**。

---

## 二、SWE-bench：代码 agent 的"硬骨头"

### 基本面

- **Jimenez et al., ICLR 2024 oral**。
- **2,294 个真实 GitHub issue**，来自 12 个流行 Python 仓库。
- agent 拿到：pre-fix commit 的代码 + 自然语言 issue 描述。
- agent 输出：**一个 patch**。
- 评估器：apply patch → 跑仓库测试套件。**patch 必须把 FAIL_TO_PASS 翻过来，同时不破坏 PASS_TO_PASS**。

### 关键术语

| 术语 | 含义 |
|---|---|
| **FAIL_TO_PASS** | 之前失败的测试，patch 后必须通过（修好了 bug） |
| **PASS_TO_PASS** | 之前通过的测试，patch 后必须仍然通过（没引入回归） |

**双门控是 SWE-bench 的精髓**：单测"修好了"不够，**不能打坏其他东西**。这正是真实软件工程的真正难度。

### 历史性成绩

- **SWE-agent（Yang et al., 2024）首发 12.5%**。关键是 **agent-computer interface (ACI) 设计**：文件编辑命令、模型能理解的搜索语法。**"模型能力"不是唯一决定，interface 设计同等重要**。

### SWE-bench Verified（OpenAI 清理版）

- **2024-08** OpenAI 推出，人工挑选 **500 task** 子集。
- **去除**：含糊的 issue、不可靠的测试、修法不清的题。
- **目的**：作为"你的 agent 能不能交付真实 patch"的主基准。

### 污染故事（极其重要）

> - **94%+** 的 SWE-bench issue 时间早于多数模型 cutoff。
> - **SWE-bench+** 发现：**32.67%** 的成功 patch 存在 issue 文本里"泄漏修法"（模型在描述里看到 fix）；**31.08%** 因测试覆盖弱而可疑。
> - **Verified 更干净但不是无污染**。

### 关键启示

> **a model that scores 50% on SWE-bench may score 35% on SWE-bench+. Always report both if you claim SWE-bench performance.**

**这是报告数字的伦理底线**。报 SWE-bench 不报 Verified / SWE-bench+ = 误导。

---

## 三、GAIA：通用工具使用能力

### 基本面

- **Mialon et al., 2023-11**。
- **466 题**（公开 166 + 私人 300 leaderboard）。
- 设计哲学："**conceptually simple for humans (92%) but hard for AI (GPT-4 with plugins: 15%)**"。

### 测什么

- 推理
- 多模态
- 网页
- 工具使用

### 三档难度

- **Level 1**：单工具，单文件。
- **Level 2**：多工具，多步骤。
- **Level 3**：**长 tool chain + 跨模态**（这是 GAIA 真正的硬度）。

### 与 SWE-bench 的对照

| 维度 | SWE-bench | GAIA |
|---|---|---|
| **测什么** | 代码 patch | 通用工具使用 |
| **任务形式** | 真实 issue | 现实问题（更开放） |
| **难度来源** | 代码 / 仓库理解 | 工具编排 / 长链 / 跨模态 |
| **何时用** | code agent 评估 | 通用 agent 评估 |

> **GAIA is what you run to measure "generalist capability." Do not confuse with code-specific benchmarks.**

**别把 SWE-bench 和 GAIA 混着用**。一个考"程序员"，一个考"瑞士军刀使用者"。

---

## 四、AgentBench：多环境推理

### 基本面

- **Liu et al., ICLR 2024**。
- **8 个环境** 跨 4 大类：

| 类别 | 环境 |
|---|---|
| **Code** | Bash, DB, KG（知识图谱） |
| **Games** | Alfworld（具身家务）、LTP（语言接龙） |
| **Web** | WebShop（电商浏览）、Mind2Web（网页交互） |
| **Open** | Open-ended generation |

- 多轮，**每 split 4k–13k turn**。

### 关键发现

> **Long-term reasoning, decision-making, and instruction following are the blockers for OSS LLMs catching up to commercial.**

**OSS（开源）模型的"追赶差距"在哪？** 不是单点能力（写代码、查网页），是**长链路推理 + 决策 + 指令遵循**。**这一条对评估体系设计是核心指导**。

---

## 五、这三个 benchmark **不测什么**

| 不测的 | 重要性 |
|---|---|
| **真实运营成本**（tokens、wall-clock） | 你 production 的账单 |
| **对抗条件下的安全行为** | red-team 关注的 |
| **你自家领域的性能** | **永远要自己写 eval**（第 14-30 章） |
| **尾部失败**（benchmarks 报均值；生产关注最差 1%） | 客户感知到的体验 |

**Benchmark 是参考，不是判决书**。**最危险的是"benchmark-as-development-target"** —— 优化 benchmark 等于把生产有用性摆在次要位置。

---

## 六、本章的失败模式（明文警告）

1. **Single-number fixation** —— SWE-bench 50% 单独看不告诉你什么；要 P50 / P75 / P95 的 cost + step 分布。
2. **Contaminated claims** —— 报 SWE-bench 不提 Verified / SWE-bench+ 是误导。
3. **Benchmark-as-development-target** —— 优化 benchmark 与生产实用脱钩。

**核心判断**：

> **Know the three anchoring benchmarks and their failure modes before you quote a number.**

---

## 七、代码（`code/main.py`）速览

`code/main.py` 实现一个 toy SWE-bench-like harness：

```python
# 3 个合成 bug-fix 任务
synthetic_tasks = [...]

# scripted "agent" 给 patch
def agent_propose(task): ...

# test runner
def evaluate(patch, task):
    fail_to_pass_passed = all(test(patch) for test in task.fail_to_pass)
    pass_to_pass_passed = all(test(patch) for test in task.pass_to_pass)
    return fail_to_pass_passed and pass_to_pass_passed

# GAIA-style 难度分类器
def classify_difficulty(question): ...   # 基于问题分解深度
```

运行后输出：
- 每任务解决率
- 每档难度
- 评估规则具体演示

**这一章的代码不是让你写新框架，是让你看清"评估装置在做什么"**。

---

## 八、Use It：决策树

```
你在评估 agent 能力？
  ├─ 测"能不能交付代码 patch" → SWE-bench Verified
  ├─ 测"通用工具使用" → GAIA（用 private leaderboard split）
  ├─ 测"多环境" → AgentBench
  └─ 测"你的产品场景" → Custom evals（第 14-30 章）
```

---

## 九、常见踩坑（本章明文警告）

1. **Single-number fixation** —— 一个数字不够；要分布。
2. **Contaminated claims** —— 报 SWE-bench 必须同时报 Verified / SWE-bench+。
3. **Benchmark-as-development-target** —— 别把 benchmark 当作唯一优化目标。

---

## 十、课后练习 5 题核心思路

1. **把 toy harness 跑到真仓库**：挑一个你的代码库写 3 个 FAIL_TO_PASS test。**这是从"我懂 SWE-bench"到"我会做 SWE-bench"的台阶**。
2. **加 step-count 指标**：3 任务上各跑出多少步解决。**这一题会让你看清"agent 步数"作为 KPI 的有效性**。
3. **读 SWE-bench+ paper，实现 solution-leakage 检测**：模式匹配 issue 文本 vs diff。**这是工程上防止"自欺欺人"的 check**。
4. **从 GAIA 公开 split 下 1 题 trace GPT-4 类 agent**：看需要什么 tool。**GAIA 的难度往往在 tool 链长度**。
5. **读 AgentBench 的 per-env 拆分**：哪个环境对应你的产品？那里 SOTA 长什么样？**这一题让你把 benchmark 翻译成"我的产品的对应难度区间"**。

---

## 十一、本章给我的工程启示

1. **测试装置 = 评估学的第一性原理**。在引用任何数字前，先问"这个数字用什么 test 装置生成"。**双门控（FAIL_TO_PASS + PASS_TO_PASS）这种设计是工业级的**。
2. **污染故事决定数字的可信度**。**94% issue 早于 cutoff + 32% 修法泄漏在 issue 文本里** —— 这就是为什么任何报数字都要带 Verified 副号。**学术诚信的体现**。
3. **ACI（Agent-Computer Interface）设计能力 ≈ 模型能力**。SWE-agent 12.5% 不是因为它用更强模型，是因为它的**文件编辑命令、搜索语法更合模型"胃口"**。**这一条是"产品力"层面最重要的提示**。
4. **Benchmark 测的是均值，生产关注的是尾部**。**最差 1% 才是客户感知**。**这是为什么 Custom evals（第 14-30 章）必须做**。
5. **OSS 模型在长链路推理 / 决策 / 指令遵循落后**。这一条是 AgentBench 的最重要发现，对自建 vs 商用模型选择有直接指导。

---

## 十二、横向对比

| Benchmark | 测什么 | 任务数 | 难度来源 | 报告道德 |
|---|---|---|---|---|
| **SWE-bench** | code patch | 2,294 | 仓库理解 | 必须带 Verified |
| **SWE-bench Verified** | code patch | 500 | 同上但干净 | 必报 |
| **SWE-bench+** | contamination audit | audit | 检测 issue 文本泄漏 | 副号 |
| **GAIA** | 通用工具使用 | 466 | tool 链 + 跨模态 | 用 private split |
| **AgentBench** | 多环境 | 8 envs | 长链路推理 | per-env 拆分 |

**一个统一的口诀**：

> **报数字必带 test harness、contamination story、"它不测什么"三件套。**

少一件 = 误导。
