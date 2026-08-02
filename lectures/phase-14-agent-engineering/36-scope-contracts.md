---
chapter: Scope Contracts and Task Boundaries
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/36-scope-contracts
source_doc: phases/14-agent-engineering/36-scope-contracts/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-36 · Scope 合同与任务边界

> The model does not know where the work ends. A scope contract is a per-task file that says where the work begins, where it ends, and how to roll back if it spills. The contract turns "stay in scope" from a wish into a check.

## 一、本章在整体中的位置

14-33 rules 给了"什么禁"（"don't edit `scripts/release.sh`"）。**14-36 scope contract 把禁集**+ 接受标准 + 回滚**收成 per-task 合同**。

> **Agents creep. The task is "fix the login bug." The diff touches the login route, the email helper, the database driver, the README, and the release script.**

**每个 touch 在当时都有 plausible reason。一起是不同的 change**。**scope creep 是被最少监控的失败模式**。

---

## 二、Scope Contract 字段

| 字段 | 用途 |
|---|---|
| `task_id` | Link 到 task board |
| `goal` | 一句 reviewer 能验证的 |
| `allowed_files` | agent 可写的 globs |
| `forbidden_files` | agent 不能触的 globs（即使意外） |
| `acceptance_criteria` | 测命令或断言行 |
| `rollback_plan` | 运营可执行的一段 |
| `approvals_required` | 超出 scope 需人工批的 action |

> **A contract without `forbidden_files` is incomplete. The negative space is half the contract.**

**没 `forbidden_files` = 不完整合同**。**负空间是合同一半**。

---

## 三、Globs 不是 raw paths

真实仓库移动文件。Pin 到 globs（`app/**/*.py`），让 refactor 不作废合同。

---

## 四、回滚 = scope 的一部分

**列回滚迫使合同作者想"什么会出错"**。**不能回滚的合同 = 不该批的合同**。

---

## 五、Scope check = diff check

agent 写 diff。**checker 读 diff + 允许 globs + 禁止 globs + 跑了哪些 acceptance 命令**。**每 violation = tagged finding，verification gate 拒**。

---

## 六、两个高度：feature list vs task contract

- **Scope contract 限一个 task**——**不限项目**。
- **`feature_list.json`** = 项目级 backlog。
  - `active` = 当前 session 可触的单一 feature。
  - `features[].status` = `todo | in_progress | done | blocked`。
  - **"一个时刻至多一个 `in_progress`"** = startup check（14-33）。

> **"One feature at a time" stops being a line in the prompt the agent can rationalize past and becomes a value it reads off disk and a check the gate enforces.**

**"一次一 feature"** 不再是 prompt 文字 = 文件读取值 + 门 enforced。

---

## 七、生产模式

### Violation budgets 不是 binary

- `agent-guardrails`（OSS merge gate，Claude Code / Cursor / Windsurf / Codex via MCP）ship per-task `violationBudget`。
- **预算内 minor slips = warning**；**超 = merge gate 拒**。
- `violationSeverity: error | warning` 配。
- **预算是 gate 留下来 vs 被禁用 的差**。

### Severity asymmetry by path family

- Off-scope `docs/**` 通常 = `warn`。
- Off-scope `scripts/**`、`migrations/**`、`config/prod/**` 永远 = `block`。
- **必须住进 contract，不在 runtime**（项目特异 per task 变）。

### Time + network budgets

- `time_budget_minutes` 限 wall clock；runtime 超 = 重批。
- `network_egress` hostname allowlist 防 agent 偷连外部 API。
- **也是 scope 维度**。**file globs 必要不充分**。

### Multi-contract merge（least privilege）

两份合同适用时：

- `allowed_files` **intersect**（都得允许）
- `forbidden_files` **union**（任一禁）
- `time_budget_minutes` **min**
- `approvals_required` 累加
- `network_egress`：`None` defer；两 list intersect；deny-all 保持 deny-all

**在 schema 里说清 → merge 机械 + reviewable**。

---

## 八、代码（`code/main.py`）速览

```python
# scope_contract.json schema (subset JSON Schema + globs)
schema = ...

# diff parser → RunSummary
def parse_run(touched_files, run_commands): ...

# scope_check 返回 (violations, in_scope, off_scope)
def scope_check(run, contract): ...

# 2 demo runs: in scope / creep
# creep 旗具体文件 + 理由
```

输出：contract + 2 runs + per-run verdict + `scope_report.json`。

---

## 九、Use It 决策树

```
你的 scope contract 怎么部署？
  ├─ Claude Code slash command：/scope 写合同 + pin session context
  ├─ GitHub PR：合同作 JSON 在 PR body；CI 跑 scope checker vs merge diff
  └─ LangGraph interrupts：scope violation 触发 interrupt，handler 问人
```

**合同与 task 同行**。**Task 关闭，合同归档到 `outputs/scope/closed/`**。

---

## 十、课后练习 5 题核心思路

1. **`network_egress` 字段**：允许外部 host 列表。
2. **Checker fail-soft on `docs/**`**，fail-hard on `scripts/**`。
3. **`goal` 字段 + 静态规则派生 `allowed_files`**（无 LLM）——**第一个 edge case 怎么崩**？
4. **`time_budget_minutes`** + 超时拒。
5. **两份合同 vs 同 diff**——**正确 merge 语义**？

---

## 十一、本章给我的工程启示

1. **Scope creep 是最被少监控的失败**。**"stay in scope" 是 wish，contract 是 check**。
2. **`forbidden_files` 是合同一半**。**没它 = 不完整**。
3. **两个高度：feature list（项目）+ task contract（单 task）**。**单 contract 防不了"项目也加 feature"**。
4. **Violation budget 让 gate 活下来**。**binary fail = 被禁用**。
5. **Severity by path family**——**docs warn / scripts block 必进 contract**。

---

## 十二、横向对比

| 合同字段 | 作用 |
|---|---|
| `goal` | reviewer 一句验证 |
| `allowed_files` / `forbidden_files` | globs（不用 raw paths） |
| `acceptance_criteria` | 测命令 |
| `rollback_plan` | 运营 runbook |
| `approvals_required` | 需人工批 |
| `time_budget_minutes` | wall clock 限 |
| `network_egress` | 外部 host allowlist |

**统一口诀**：

> **每 task 一合同；forbidden_files 必填；feature list 限项目；violation budget 让 gate 活；severity by path family。**
