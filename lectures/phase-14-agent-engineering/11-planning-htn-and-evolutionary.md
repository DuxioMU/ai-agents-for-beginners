---
chapter: Planning with HTN and Evolutionary Search
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/11-planning-htn-and-evolutionary
source_doc: phases/14-agent-engineering/11-planning-htn-and-evolutionary/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-11 · HTN 与进化式搜索

> Symbolic planning handles the cases where the plan is provably correct. Evolutionary code search handles the cases where the fitness function is machine-checkable. ChatHTN (2025) and AlphaEvolve (2025) show what each unlocks when paired with an LLM.

## 一、本章在整体中的位置

`phases/14-agent-engineering` 整本涵盖了**智能体工程**这一主题。前面的章节已经讲过主流规划方案 —— ReWOO（第 2 章）、Plan-and-Execute、ReAct 等。本章解决这些方案覆盖不到的两个场景：

| ReAct/ReWOO 擅长 | 解决不了 |
|---|---|
| 大模型驱动的多步任务 | 需要**数学上可证正确**的计划 |
| 自然语言规划的灵活执行 | 需要**机器可度量**的全局最优 |

这两类问题分别由两类方法填补：**HTN（符号规划）** 和 **进化式搜索**。

---

## 二、核心概念 1：HTN（Hierarchical Task Network，分层任务网络）

HTN 是 1980 年代就提出的**老牌符号规划框架**，比 LLM 早得多。它把规划建模成"把抽象任务一层层拆解成可执行原子动作"的过程。

### 四个关键部件

| 名称 | 类比 | 角色 |
|---|---|---|
| **Task（任务）** | 目标 | 分两类：*compound task*（待拆分，如"准备早餐"）、*primitive task*（可直接执行，如"煎蛋"） |
| **Method（方法）** | 拆分规则 | 把 compound 任务拆成子任务，且附带**前置条件** |
| **Operator（操作符）** | 原子动作 | primitive 任务的具体实现，附带**前置条件** + **效果** |
| **State（状态）** | 世界事实 | 一组事实集合，operator 执行后会被更新 |

### 规划流程

```
初始状态 ──→ 目标 compound task
            ↓
       找匹配 method（必须前置条件满足）
            ↓
       拆分 → 子任务列表（compound + primitive 混在一起）
            ↓
       对子任务递归
            ↓
       直到全部是 primitive，再按顺序验证 operator 的 precondition
            ↓
       如果某 operator 的 precondition 不满足 → 回溯 / 失败
```

### HTN 的核心价值：**soundness-by-construction（构造性正确）**

只要 method/operator 的 schema 严格，每个被找到的计划**理论上一定正确**。这一性质对调度、飞行路线、合规流是刚需。

---

## 三、核心概念 2：ChatHTN（2025）

**Gopalakrishnan et al., arXiv:2505.11814**

### 动机

HTN 经典实现的痛点：手工写 method 库太贵，且遇到没写过 method 的 compound 任务就会卡住。

### 解决方案：symbolic + LLM 混合循环

```python
# 伪代码骨架
def decompose(task, state):
    methods = [m for m in method_library
               if m.matches(task) and m.precondition(state)]
    if methods:
        # 1) 正常路径：符号引擎主导
        return symbolic_search(methods, task, state)

    # 2) 兜底：询问 LLM
    sub = llm.decompose(task, state)

    # 3) 关键一步：用 schema 校验 LLM 输出
    if operator_schema.accepts(sub, state):
        return sub
    else:
        raise InvalidDecomposition
```

### 论文的中心论点（必须理解）

> LLM 只能**作为候选分解**注入，从不直接编辑最终计划。**正确性归符号层所有；LLM 只是扩展了 method 库**。

这意味着：哪怕 LLM "胡说八道"，最终产出的 plan 仍然是 sound 的 —— 因为 schema 会拒绝非法分解。这是 LLM 与符号系统结合的典型安全模式。

### 进阶：在线方法学习

2025 年 OpenReview 后续工作 `gwYEDY9j2x` 加了**在线学习器**：把 LLM 给出的分解回归泛化成新的 method，缓存复用，可减少高达 **75% 的 LLM 查询量**。

---

## 四、核心概念 3：AlphaEvolve（2025）

**Novikov et al., DeepMind, arXiv:2506.13131**

### 它解决的问题类别

不是"做一个正确的计划"，而是"找到**最好的**计划" —— 前提是最佳性可被机器打分。

### 进化循环

```
        ┌─ 种子程序 + 程序化评估器（deterministic + 快） ─┐
        │                                                 │
        ▼                                                 │
   ┌─────────────┐  mutate  ┌───────────────────┐         ▲
   │ 当前最优程序 │ ───────→ │ Gemini 2.0 ensemble│         │
   └─────────────┘          └───────────────────┘         │
                                  │ 候选变异             │
                                  ▼                     │
                            运行评估器                   │
                                  │ 分数                │
                                  ▼                     │
                            保留更优 ────────────────────┘
```

### 已发布的"战绩"

1. **Strassen 算法 56 年来首次被超越**：4×4 复数矩阵乘法从 49 → 48 标量乘。
2. **Google 数据中心调度**：通过一个 Borg 启发式回收 **0.7%** 算力。
3. **FlashAttention**：在某个前沿工作负载上加速 **32%**。

### 硬约束（也是它最容易被误用的地方）

> **fitness function 必须是机器可验证的、确定性的、且足够快**。

没有 evaluator 的 AlphaEvolve 就是退化 —— "问 LLM 这段代码是不是更好" 不是 fitness function，无法收敛。

---

## 五、怎么选：ReAct / ReWOO / HTN / AlphaEvolve

| 问题类别 | 选 | 理由 |
|---|---|---|
| 调度（硬约束） | **HTN + ChatHTN** | 需要可证正确性 |
| 编译器优化 | **AlphaEvolve** | 存在机器打分 |
| 多步任务执行 | ReAct / ReWOO | LLM 在线，不需要形式保证 |
| 用测试做代码改进 | **AlphaEvolve** | 测试就是 evaluator |
| 政策约束的自动化 | **HTN** | 前置条件可编码政策 |

### 常见踩坑（本章明确警告）

1. **HTN 没有 operator schema** → soundness 主张崩塌。LLM 一旦绕过 schema 就没意义了。
2. **AlphaEvolve 没有真 evaluator** → 退化为"靠 LLM 评分选代码"，不收敛。
3. **过度工程**：大部分 agent 任务用 ReAct/ReWOO 就够。

---

## 六、代码实现（`code/main.py`）速览

本章 `code/main.py` 用纯 stdlib 实现了两个 toy：

### Toy 1：HTN 规划器

- `Operator`/`Method` 类持有 `precondition` 和 `effect`。
- `Planner` 在 compound task 上搜索 method。
- 找不到 method 时调用 `LLMFallback`，其内部用脚本化的"假 LLM"（避免联网）返回一个分解。
- 中途还会触发 LLM 兜底，文档里说的"mid-plan LLM fallback"就是这一步。

### Toy 2：进化式搜索

- 候选程序是**算术表达式**（加减乘常量与变量）。
- 评估器计算 `(f(x) - target)^2` 在测试集的均方误差。
- 每代保留前 N 个，变异生成下一代，最小化误差即收敛到目标函数。

运行：
```bash
python3 code/main.py
```

---

## 七、本章给我的工程启示

1. **"LLM 加上 schema 校验"是一种可复用的安全模式**：LLM 给候选、符号层守门 —— 不只是 ChatHTN，工具调用、JSON 解析都可以这样设计。
2. **可证正确 ≠ 更好**：HTN 给"对"，AlphaEvolve 给"优"。两者目标不同。
3. **程序化评估器是进化搜索的灵魂**：评估器差，写得再花哨也跑不出来；评估器好，问题域可以换。
4. **别过度工程化**：本章最后专门留一句警示 —— 大部分 agent 任务够不上 HTN 或 AlphaEvolve，先 ReAct/ReWOO 起步。

---

## 八、课后练习（5 题）核心思路指引

1. **给 HTN 加回溯**：维护状态栈，postcondition 失败时退栈换下一个 method。
2. **LLM-method 缓存**：哈希 `(T, P)` 模式 → 缓存命中直接复用，省 LLM 调用。
3. **把 evaluator 换成测试套件**：让进化搜索朝着"通过全部测试"的方向变异 sort 实现。
4. **设计真实 evaluator**：评估器三要素 —— **确定性**（同一输入永远同分）、**可分性**（好坏有别）、**速度**（每秒可评上千次）。
5. **HTN ⊕ AlphaEvolve 组合**：外层 HTN 管调度骨架，内层每个 primitive 让进化搜索找最优实现 —— 工程上**利基场景**才有用，一般会"杀鸡用牛刀"。
