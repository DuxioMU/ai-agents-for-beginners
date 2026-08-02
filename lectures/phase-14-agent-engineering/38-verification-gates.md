---
chapter: Verification Gates
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/38-verification-gates
source_doc: phases/14-agent-engineering/38-verification-gates/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-38 · 验证门禁

> The agent does not get to mark its own work as done. A verification gate reads the scope contract, the feedback log, the rule report, and the diff, and answers a single question: is this task actually complete? If the gate says no, the task is not done, no matter what the chat says.

## 一、本章在整体中的位置

14-37 feedback runner 给"事实"。**14-38 verification gate 答一个问题**——**"task 实际完成吗？"**。

> **Three failure shapes dominate:**
> - **"Looks good."** Model 读自己 diff 决定正确。
> - **"Tests passed."** 自信。**无测试实际跑的记录**。
> - **"Acceptance met."** 解释得宽松到 = "任何像 done 的"。

**workbench fix = 一个 verification gate 读 artifacts 出 call**。**确定性、版本控制、CI 接线、agent 贿赂不了**。

---

## 二、Gate 检查项

| Check | Source artifact | Severity |
|---|---|---|
| 所有 acceptance 命令跑过 | `feedback_record.jsonl` | **block** |
| 所有 acceptance 命令退 0 | `feedback_record.jsonl` | **block** |
| Scope check 无 forbidden writes | `scope_report.json` | **block** |
| Scope check 无 off-scope writes | `scope_report.json` | **block or warn** |
| 所有 block-severity rules 过 | `rule_report.json` | **block** |
| 无 `null` exit codes | `feedback_record.jsonl` | **block** |
| 触的文件 match `scope.allowed_files` | both | warn |

**warn 注释 verdict；block 阻止 `passed: true`**。

---

## 三、确定性，不是概率

> **The gate must produce the same verdict for the same artifact set every time. No LLM judges. LLM judges belong on the reviewer side (Phase 14 · 39).**

**LLM judge = reviewer 那侧**（14-39），**不是 gate**。

---

## 四、One report, one path

`outputs/verification/<task_id>.json` —— 单一路径，CI + reviewer 都读。**多路径多 gate = 分叉真相**。

---

## 五、Refuse without exception

**Block-severity 不可被 agent 覆盖**。**只能被人覆盖**——**带 `override_reason` + `overridden_by` user id**。**覆盖是 signed change 不是 agent 决定**。

---

## 六、生产模式

### Defense in depth 不是单 gate

pre-commit hook → CI status check → pre-tool authz hook → pre-merge gate。**每层确定性 = 一层失败下层接**。**microservices.io 2026-03 说法**：pre-commit hook **non-bypassable**，因为不像 model-side skill 依赖 agent 听话。**Verification gate 坐 CI / pre-merge 层**。

### Defense by deterministic check，model-judge only for nuance

**Anthropic 2026 Hybrid Norm**：
- **可验证奖励**（unit test / schema check / exit code）答"代码解了问题吗？"
- **LLM rubrics** 答"代码可读、安全、风格吗？"
- **Gate 跑第一类；reviewer（14-39）跑第二类**。**混了崩信号**。

### Signed override log，不是 Slack thread

每覆盖 emit 一行 `outputs/verification/overrides.jsonl`：
- timestamp
- finding code
- reason
- signing user
- current HEAD commit

**Runtime 拒任何无 signature override；audit trail git-tracked**。**覆盖政策 vs 覆盖 theater 的差**。

### Coverage floor as first-class check

`coverage_report.json` feed `coverage_floor`（默认 80%）check。**Gate fail 如果 measured coverage 落 floor 或落前一 merge floor 1pp+**。**没这条 = agent 静默删失败的 test，verification report 还绿**。

### `--strict` mode promotes warns to blocks

release branch / ship-blocking PR / post-incident triage——**`--strict` 让每 warning 成 hard fail**。**按 branch opt-in，不全局默认**（严格到处腐蚀日常 flow）。

---

## 七、代码（`code/main.py`）速览

```python
# 每输入 artifact 的 loader（stub 本地）
def load_scope(): ...
def load_rules(): ...
def load_feedback(): ...
def load_diff(): ...

# verify(task_id, artifacts) -> VerdictReport 纯函数
def verify(task_id, artifacts): ...

# printer 显示 per-check + final pass/fail
# demo: 3 task 场景 — clean pass / scope creep / missing acceptance
```

---

## 八、Use It 决策树

```
你的 verification gate 怎么接？
  ├─ CI step：verify_agent job 跑 gate vs 最终 artifacts，merge protection 拒没过
  ├─ Pre-handoff hook：runtime 调 gate 在 handoff 前 — 不绿不出 handoff
  └─ Manual triage：operator 读 report 当 agent 说 success 怀疑时
```

**Gate 是 workbench flow 的判定边**。**其他 surface 全是它上游**。

---

## 九、课后练习 5 题核心思路

1. **`coverage_floor` check**（默认 80%）。
2. **`--strict` mode**：每 warn → block。**何时 default 对**？
3. **Gate 输出 Markdown summary + JSON**。**哪些字段属于 summary**？
4. **`time_since_last_human_touch`**：60s 内 human 编辑的 file 免 off-scope flag。
5. **真 agent diff 跑 gate**：多少 finding 是真，多少是噪？

---

## 十、本章给我的工程启示

1. **Agent 不批自己作业**。**Gate 是判定边**。
2. **确定性 check 在 gate；语义 check 在 reviewer**。**别混**。
3. **Block severity 不可覆盖**——**只能人带 signature 覆盖**。**Override audit trail git-tracked**。
4. **Defense in depth**：pre-commit / pre-tool / pre-merge 多层 deterministic 闸。
5. **`coverage_floor` 防 agent 静默删 test**。**没它 verification 报告是假的绿**。

---

## 十一、横向对比

| Check | Source | Severity |
|---|---|---|
| acceptance 跑过 + 退 0 | feedback | block |
| scope forbidden / off-scope | scope_report | block / block-or-warn |
| block rules | rule_report | block |
| 无 null exit | feedback | block |
| allowed_files 触 | both | warn |

**统一口诀**：

> **Agent 不批自己；deterministic 在 gate；block 不可覆盖；defense in depth；coverage floor 防假绿。**
