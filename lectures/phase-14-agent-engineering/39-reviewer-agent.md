---
chapter: Reviewer Agent — Separate Builder from Marker
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/39-reviewer-agent
source_doc: phases/14-agent-engineering/39-reviewer-agent/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-39 · Reviewer Agent：把建设者和打分者分开

> The agent that wrote the code cannot grade it. A reviewer is a second loop with a different system prompt, a different goal, and read-only access to everything the builder produced. The gap between builder and reviewer is where most reliability lives.

## 一、本章在整体中的位置

14-38 verification gate 答"acceptance 跑了 / scope hold / rules 过"。**这是必要不充分**。

> **Did this solve the right problem? Did it expand scope without flagging it? Did it document assumptions that should have been questioned?**

**Reviewer 问 acceptance 不能问的问题**。

---

## 二、5 维 rubric

每维 0-2 分：

| 维度 | 问什么 |
|---|---|
| **Problem fit** | 改动了 stated task 而不是 nearby task？ |
| **Scope discipline** | 编辑限 contract，还是 contract 主动扩？ |
| **Assumptions** | 隐藏 assumptions 都写下来了？ |
| **Verification quality** | acceptance 命令真证 goal，还是弱版本？ |
| **Handoff readiness** | 下 session 能干净接？ |

**总分 10**：< 7 = soft fail；< 5 = hard fail。

---

## 三、Reviewer 是独立 role 不是独立 model

> **You can run the reviewer with the same model as the builder.**

**同样 model 可以两者**。**discipline = role separation**：不同 system prompt、不同 inputs、**对 diff 无写权限**。**posture 变 = signal 变**。

---

## 四、Reviewer 不能 edit diff

**Reviewer 读 diff / state / feedback / verdict**——**写 report**——**不打 patch**。**report 说"修这里" → 下次 builder turn 修**。**混角色 = 破 gap**。

---

## 五、Reviewer rubric vs Verification gate

- **Gate**（14-38）= 确定性 fact：acceptance 跑、rule 过、scope hold。
- **Reviewer** = 定性 judgment：对了问题、assumption 文档化、handoff 可用。
- **都要**。

---

## 六、生产模式

### 4 个模式让 reviewer 在规模下 work

**Cloudflare 2026-04 AI Code Review**：131,246 review runs / 48,095 MRs / 5,169 repos / 30 天。Median review 3min 39s。**最多 7 个 specialist reviewer**（security / performance / code quality / docs / release / compliance / Codex）并行 under **Review Coordinator**（dedupe findings + 判 severity）。**Top-tier model 仅给 coordinator**；specialist 走便宜 tier。

### 4 个模式

1. **Specialist pool 不是 one big reviewer**——**coordinator 跑 dedupe；specialist 不跑全 rubric**。**Model tier 分离自然发生**：便宜 specialist，昂贵 coordinator。
2. **Bias mitigation 当 design requirement**——**LLM judge 4 bias**（Adnan Masood 2026-04）：
   - **position bias**（GPT-4 ~40% 在 (A,B) vs (B,A) 不一致）
   - **verbosity bias**（~15% 给长输出涨分）
   - **self-preference**（judge 偏同家族）
   - **authority bias**（judge 高估引用已知作者）
   - **Mitigations**：评两 ordering，只数一致赢；用 1-4 scale 显式奖简洁；轮转 judge 跨 model 家族；打分前去作者名。
3. **Calibration set 不是 vibe**——**10-20 task 历史集有已知 correct verdict**。**每改 prompt 跑 reviewer over 它**。**一致率 < 80% = rubric 要改再 ship**。
4. **Hybrid norm with gate**——**Gate = deterministic，reviewer = semantic**。**别让 reviewer 重做 gate 已证的**。

---

## 七、代码（`code/main.py`）速览

```python
@dataclass
class ReviewerInputs: ...     # diff + state + feedback + verdict

# 5 维 scorer（每维一个函数，确定性 stub；真实现调 LLM）
def score_problem_fit(...): ...
def score_scope_discipline(...): ...
def score_assumptions(...): ...
def score_verification(...): ...
def score_handoff(...): ...

# review_report.json writer
# 5 scores + total + verdict (pass / soft_fail / hard_fail)

# 2 demo: clean change / "right tests, wrong problem"
```

---

## 八、Use It 决策树

```
你的 reviewer 怎么接？
  ├─ Claude Code subagents：reviewer subagent 在 builder 关闭 task 后跑，发 PR comment + rubric scores
  ├─ OpenAI Agents SDK handoffs：builder handoff 到 reviewer，reviewer 退回 findings 或 up 到人
  └─ Two-model pairing：builder 跑快便宜 model，reviewer 跑强 model 配小 context，专注 judgment
```

---

## 九、课后练习 5 题核心思路

1. **加第 6 维**（产品 domain-specific）——**为什么不能并入 5 维**？
2. **两 system prompt（terse / verbose）跑 reviewer**——**哪个报告人更愿读**？
3. **`confidence` per dimension**：最低维 < 0.6 拒发 report。
4. **Calibration set**：10 历史 task close-out with known correct verdicts。**哪条与历史不一致**？
5. **"request more evidence" affordance**：reviewer 可让 builder 跑特定 test 再打分。**back-off 怎么定**？

---

## 十、本章给我的工程启示

1. **建设者不打分自己作业**。**Gap = 可靠性**。
2. **Role separation > model separation**。**不同 prompt 同样 model 就够**。
3. **LLM judge 有 4 bias**——**mitigation 是 design requirement**。
4. **Calibration set 防 rubric 飘**。**10-20 task 起手**。
5. **Gate 做 fact，reviewer 做 judgment**。**Hybrid norm**。

---

## 十一、横向对比

| 维度 | 0 | 1 | 2 |
|---|---|---|---|
| **Problem fit** | 改错 task | 部分 | 改对 |
| **Scope discipline** | creep 无 flag | 部分 flag | hold |
| **Assumptions** | 全藏 | 部分 | 全写 |
| **Verification quality** | 弱证 | 部分 | 真证 |
| **Handoff readiness** | 散 | 部分 | 干净接 |

**统一口诀**：

> **建设者 ≠ 评分者；5 维 rubric；role separation；4 bias mitigation；calibration set 必建。**
