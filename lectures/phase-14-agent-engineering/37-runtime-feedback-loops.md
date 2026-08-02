---
chapter: Runtime Feedback Loops
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/37-runtime-feedback-loops
source_doc: phases/14-agent-engineering/37-runtime-feedback-loops/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-37 · 运行时反馈环

> Agents that do not see real command output guess. A feedback runner captures stdout, stderr, exit code, and timing into a structured record the next turn can read. Then the agent reacts to facts instead of to its own prediction of facts.

## 一、本章在整体中的位置

14-35 init 解决"开始"——**14-37 解决"运行"**。**每条命令的输出进结构化 record，下一 turn 读它**。

> **The agent says "running tests now." The next message says "all tests pass." The reality is that no test ran.**

**Three failure shapes**:
1. model 想象输出
2. 跑了命令没读结果
3. 读了结果悄悄截断失败行

**Feedback runner 把这 gap 关掉**。

---

## 二、Feedback record 字段

| 字段 | 为什么 |
|---|---|
| `command` | 精确 argv，无 shell expansion 意外 |
| `stdout_tail` | 末 N 行，确定性截断 |
| `stderr_tail` | 末 N 行，与 stdout 分开 |
| `exit_code` | 明确成功信号 |
| `duration_ms` | 慢 probe / runaway process 暴露 |
| `started_at` | 重放时间戳 |
| `agent_note` | agent 写的"我预期"一行 |

---

## 三、确定性截断

> **A 50 MB log destroys the loop. The runner truncates head and tail with a `...truncated N lines...` marker, deterministic so the same output always produces the same record.**

**agent 需看的（最后错、最后 summary）在 tail**。**不采样**。

---

## 四、Feedback vs Telemetry

- **Telemetry**（14-23 OTel GenAI）= 跨时间给人 ops 看。
- **Feedback** = 本 run 下一 turn 看。
- **共享字段，不同文件，不同 retention**。

---

## 五、No exit, no progress

> **If the runner errors before capturing exit, the record carries `exit_code: null` and `error: <reason>`. The agent loop must refuse to claim success on a `null` exit.**

**`null` exit = 不前进**。

---

## 六、生产模式

### Redact at write, not at read

Runner 写前 redaction：strip `^Bearer `、`password=`、`api[_-]?key=`、`AKIA[0-9A-Z]{16}`（AWS）、`xox[baprs]-`（Slack）。**读时 redact 是 foot-gun**——**磁盘上的文件是攻击者摸的**。**季度审计 redaction patterns vs 生产 secret 格式**。

### Rotation policy

Cap `feedback_record.jsonl` 1 MB/file；溢出 rotate `.1 .2`，drop `.5`。**Agent loop 只读 current，runtime cost 有界**。**CI artifact 存完整 rotated**。

### Parent-command id for retry chains

每 record 有 `command_id`；retry 带 `parent_command_id` 指上 attempt。**Reviewer 的"failed attempts" + verification gate 的 audit 都跟链**。**无 link = retries 像独立成功，audit 藏失败历史**。

---

## 七、代码（`code/main.py`）速览

```python
# run_with_feedback(command, agent_note)
def run_with_feedback(command, note):
    proc = subprocess.run(command, capture_output=True, text=True)
    record = {
        "command": command,
        "stdout_tail": tail(proc.stdout, N),
        "stderr_tail": tail(proc.stderr, N),
        "exit_code": proc.returncode,
        "duration_ms": ...,
        "started_at": ...,
        "agent_note": note,
    }
    append_jsonl("feedback_record.jsonl", record)
    return record

# demo: 3 命令（success / failure / slow）→ 3 records
# tail file across re-runs 看 loop 累积
```

---

## 八、Use It 决策树

```
你的 feedback loop？
  ├─ Claude Code Bash tool：已捕获 stdout/stderr/exit/duration。Runner = framework-agnostic 等价。
  ├─ LangGraph nodes：包任何 shell node 在 runner 里，record 持久到 graph state 外。
  └─ CI logs：pipe JSONL 进 CI artifact store；reviewer 不重跑就 replay。
```

**Runner 是 thin wrapper，跨 framework 迁移存活**。

---

## 九、课后练习 5 题核心思路

1. **加 `cwd` 字段**——同命令从不同目录跑可区分。
2. **加 `redaction` step**——strip `^Bearer ` / `password=`。
3. **Cap `feedback_record.jsonl` 1 MB** by rotating to `.1 .2`。
4. **`parent_command_id`**——retry chains 可见。
5. **Tiny TUI 突出最新 non-zero exit**——**8 key features for review**。

---

## 十、本章给我的工程启示

1. **没有反馈 loop 的 agent 猜事实**。**Feedback = 反应事实，不是预测事实**。
2. **Redact at write 不是 read**。**磁盘上的文件是攻击者摸的**。
3. **Rotation policy 必装**。**1 MB cap + rotated files**。
4. **Parent-command id 让 retry audit 可见**。**没它 = 失败历史被藏**。
5. **Telemetry vs Feedback 不可混**——**不同读者，不同 retention**。

---

## 十一、横向对比

| 项 | 角色 |
|---|---|
| **Feedback record** | 下一 turn 读 |
| **OTel span** | ops 跨时间看 |
| **Redaction** | 写时 strip secret patterns |
| **Rotation** | 1 MB cap + .1 .2 .3 .4 .5 |
| **parent_command_id** | retry chain 可追 |

**统一口诀**：

> **每命令一 record；redact 写时；rotation 必装；parent id 链 retry；no exit = no progress。**
