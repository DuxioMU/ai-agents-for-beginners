---
chapter: Multi-Session Handoff
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/40-multi-session-handoff
source_doc: phases/14-agent-engineering/40-multi-session-handoff/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-40 · 多 Session 交接

> The session is going to end. The work is not. The handoff packet is the artifact that turns "the agent worked for an hour" into "the next session is productive in the first minute." Build it on purpose, not as an afterthought.

## 一、本章在整体中的位置

14-31 7 surface 最后一个 = handoff。**本章把它做成 artifact**，**不是事后 prose**。

> **The cost of a bad handoff is paid every session for the life of the task.**

---

## 二、7 字段（每个 handoff 必带）

| 字段 | 答什么 |
|---|---|
| `summary` | 一段做了什么 |
| `changed_files` | diff at a glance |
| `commands_run` | 实际跑了什么 |
| `failed_attempts` | 试了什么为什么不行 |
| `open_risks` | 下一 session 可能咬什么 + severity |
| `next_action` | 下一 session 第一步 |
| `verdict_pointer` | 验证 + review reports 路径 |

> **`next_action` is the load-bearing one. A handoff with everything except `next_action` is a status report, not a handoff.**

**缺 `next_action` = status report，不是 handoff**。

---

## 三、Handoff 是 generated，不是 written

> **A hand-written handoff is a handoff that gets skipped on a hard day.**

**Generator 读 workbench artifacts 出 packet**。**Agent 的工作 = 让 workbench 在 generator 能 summary 的状态**。

---

## 四、两种形式

- `handoff.md` —— 人读。
- `handoff.json` —— 下一 agent 加载。
- **同源 artifacts**。**分歧 = JSON 赢**。

---

## 五、Feedback log trimming

> **The full `feedback_record.jsonl` may be hundreds of entries. The handoff carries only the last K plus every entry with a non-zero exit.**

**下一 session 加载完整 log 如需**——**packet 保持小**。

---

## 六、Leave a clean state

**Handoff 描述工作**。**Clean state 让工作 resumable**——**不是一回事**。

| Check | Clean 含义 | Dirty 阻塞原因 |
|---|---|---|
| **Working tree** | 每改 commit 或显式 stash + note | 半应用 diff 像 intentional work |
| **Temp artifacts** | 无 `*.tmp` / scratch / debug print / 注释块 | 散文件污染 diff |
| **Tests** | 绿，或红带 `open_risks` 命名 | 静默红 test = 陷阱 |
| **Feature board** | `feature_list.json` 反映现实 | stale board 送下 session 去做已完成的 |
| **Branch** | 在 expected branch，无 detached HEAD，无 orphan | 错 branch = 下次 commit 落错地 |

**Cleanup phase 发 `clean_state.json` 空 list = handoff generator 写的 precondition**。

---

## 七、生产模式

### Compaction strategies vary；packet schema 不变

- **Codex CLI** `POST /v1/responses/compact` 服务端 opaque AES blob；fallback local handoff summary as `_summary` user-role message。
- **Claude Code** 5-stage progressive compaction @ 95% context。
- **OpenCode** timestamp-based 消息隐藏 + 5-heading LLM summary。
- **3 机制，同需求**：**packet = 把"压缩后幸存的"序列化成可移植 artifact**。

### Fresh-session handoff ≠ compaction

> **Compaction extends a session; handoff closes one cleanly and starts the next.**

**In-place compression 降质时**——**写 compact handoff、关 session、fresh context 恢复**。**packet 让那 transition 便宜**。**错 = 一直压到质量崩；fix = 早 clean handoff**。

### One active handoff per branch and topic

> **Multi-agent coordination breaks down on stale handoffs more than on bad model output.**

总是 include `branch`、`last_known_good_commit`、status `active | superseded | archived`。**Stale archived；仅 active 驱动下 session**。

### Wrap up at 50-75% context, not at the wall

**Hand-written-pattern playbook (CLAUDE.md + HANDOVER.md)**：**50-75% context budget 收尾结果最好**。**Generator 在 compression artifact 污染源 state 前 clean 写**。

---

## 八、代码（`code/main.py`）速览

```python
# loader: state + verdict + review + feedback → WorkbenchSnapshot
def gather_snapshot(): ...

# generate_handoff(snapshot) -> (markdown, payload)
def generate_handoff(snapshot): ...

# filter: 末 K feedback entries + 所有 non-zero exit
# demo: 写 handoff.md 和 handoff.json
```

---

## 九、Use It 决策树

```
你的 handoff 怎么接？
  ├─ Session-end hook：用户关 chat 时 runtime 触 generator，packet 进 outputs/handoff/<session_id>/
  ├─ PR template：generator 的 markdown 作 PR body，reviewer 不开 5 文件
  └─ Cross-agent handoff：Claude Code 建 + Codex 续。packet = 通用语
```

---

## 十、课后练习 5 题核心思路

1. **`assumptions_to_validate` 字段**：surface 每个 builder 记的但 reviewer 评分 ≤ 1 的 assumption。
2. **Trim feedback 不同 for fail vs pass run**。**defend 不对称**。
3. **"questions for the human" list**——**什么阈值进 packet vs chat**？
4. **Idempotent generator**：跑两次出同 packet——**什么需稳定**？
5. **"next session prereqs" section**——**下次 session 必 load 的 artifacts**。

---

## 十一、本章给我的工程启示

1. **坏 handoff 的成本每 session 付一次**。**Generator 必装**。
2. **`next_action` 是承重字段**。**缺它 = status report**。
3. **Clean state ≠ handoff 写好**。**两个都做**。
4. **Compaction ≠ handoff**——**前者延 session，后者开新**。
5. **50-75% context 收尾**——**留 buffer 给 generator clean 写**。

---

## 十二、横向对比

| 形态 | 何时用 |
|---|---|
| **In-place compaction** | 短延长 |
| **Fresh-session handoff** | 压缩降质时 |
| **Hand-written prose** | 易跳 |
| **Generator + packet** | 必装 |

**统一口诀**：

> **Generator 必装；7 字段含 `next_action`；clean state + handoff 双做；50-75% context 收尾；one active per branch+topic。**
