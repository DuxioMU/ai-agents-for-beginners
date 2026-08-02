---
chapter: Repo Memory and Durable State
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/34-repo-memory-and-state
source_doc: phases/14-agent-engineering/34-repo-memory-and-state/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-34 · 仓库记忆与持久状态

> Chat history is volatile. The repo is durable. The workbench stores agent state in versioned files so the next session, the next agent, and the next reviewer all read from the same source of truth.

## 一、本章在整体中的位置

14-32 引入了 `agent_state.json` 和 `task_board.json`。本章**把它们 schema 化、原子化、版本化**——**让 repo 真正成为 system of record**。

> **The workbench fix is repo memory: state lives in JSON files in the repo, written under a schema, persisted atomically, diff-friendly in code review.**

---

## 二、什么进 repo memory，什么不进

| 属于 | 不属于 |
|---|---|
| 活跃 task id | 原始 chat 转录 |
| 触过的 files | token 级 reasoning trace |
| agent 作的 assumptions | "用户看起来沮丧" |
| 开放 blockers | 采样 completions |
| 下一步 | vendor-specific model id |

**测试标准**：**3 个月后 CI 重跑还有用？** 是 → repo；否 → telemetry。

---

## 三、Schema-first state

> **Without a schema, every agent invents new fields, every reviewer learns a new shape, and every CI script has to special-case past versions.**

**JSON Schema = 契约**。**没它 = 各自发明 = reviewer 各自学 = CI 各 case**。

Schema 覆盖：

- 必填 key
- 允许 `status` 值
- 禁值（如 array 不允许 `null`）
- Pattern 约束（task id 匹配 `T-\d{3,}`）
- `schema_version` 给 migrations

---

## 四、原子写（Atomic writes）

> **State writes need to survive partial failures: write to a tempfile, fsync, rename over the target.**

```
tempfile.mkstemp (同目录)
  ↓ write
fsync
  ↓
os.replace (POSIX + Windows 原子)
```

**半写文件比没文件更糟**。**temp-rename = 唯一安全的方式**。

**Hive Issue #6263 (2026-03)**：项目用 `write_text()` + 吞异常，**partial writes 让 sessions 恢复时在 corrupt state 上跑**。**修法永远是 temp-fsync-rename**。

---

## 五、Migrations

> **Ship a migration script next to the schema bump.**

- `schema_version` 字段是契约。
- Manager 加载到不认识版本 = 拒读。
- `tools/migrate_state.py` idempotent，startup 跑。

---

## 六、Idempotency keys

> **If an agent crashes after calling a tool but before checkpointing the result, recovery retries the tool call. Safe for reads; dangerous for emails, DB inserts, file uploads.**

**模式**：每 tool call id 写入 `pending_calls.jsonl` **之前**执行。Retry 时查 id；存在则 skip，用缓存结果。**Anthropic 和 LangChain 2026 guidance 都这么说**。

---

## 七、Event sourcing for audit

```
state.events.jsonl   ← 追加（每次 mutation）
   ↓
state.json          ← 定期 snapshot
   ↓
resume：读 snapshot + 重放 snapshot 后所有 events
```

**多 disk，但能逐字重放 agent 决策**——**long-horizon run debug 必装**。**Postgres WAL 的同款**。

---

## 八、生产模式

| 模式 | 解决什么 |
|---|---|
| **Atomic temp-rename** | 必装——**不是 optional** |
| **Idempotency keys** | tool 重试时安全 |
| **Separate large artifacts** | CSV / 长 transcript / 生成文件不进 state（另存或 object storage） |
| **Event sourcing + snapshot** | 审计 + 重放 |
| **Schema migrations or refuse to load** | 升级不断 |

---

## 九、代码（`code/main.py`）速览

```python
# JSON Schemas
agent_state_schema = load_schema(...)
task_board_schema = load_schema(...)

# stdlib-only validator (subset: required, type, enum, pattern, items)
def validate(instance, schema): ...

# StateManager: load, update, commit (atomic temp-rename)
class StateManager:
    def load(self): ...
    def update(self, mutation): ...
    def commit(self): ...   # temp-rename

# demo: 2 turn mutation + persistence + reload + round-trip 验证
```

---

## 十、Use It 决策树

```
你的 state 存储？
  ├─ LangGraph checkpointers（SQLite/Postgres/自定义）
  ├─ Letta memory blocks（长期 persona + 14-08）
  └─ OpenAI Agents SDK session store（pluggable backend）
```

**state file in this lesson = local-file backend**。

---

## 十一、课后练习 5 题核心思路

1. **`last_human_touch` timestamp**——5 秒内人工编辑的 agent 写被拒。
2. **Validator 加 `oneOf`**——task 是 build 或 review，各有不同必填。
3. **`schema_version` + v1→v2 migration**（`blockers` → `risks`）。
4. **后端从文件迁 SQLite**，`StateManager` API 不变。
5. **两个 agent 50ms 写 race**：**什么崩？atomic rename 怎么救？**

---

## 十二、本章给我的工程启示

1. **Schema-first = 契约优先**。**没 schema = 各自发明**。
2. **Atomic write 不是 optional**。**Hive Issue #6263 是真实事故**。
3. **Idempotency keys = tool 重试的安全带**。**邮件 / 写库 / 上传 = 必装**。
4. **大 artifact 分离**。**state 小而快，artifact 大而独立**。
5. **Event sourcing for audit**——**Postgres WAL 的同款**。

---

## 十三、横向对比

| 存储 | 何时用 |
|---|---|
| **本地文件 + schema** | 单 agent / 小项目 |
| **SQLite** | 中型 |
| **Postgres** | 多 agent / 跨进程 |
| **LangGraph checkpointer** | 已用 LangGraph |
| **Letta memory blocks** | 长期 persona |
| **Event sourcing + snapshot** | 长期审计 |

**统一口诀**：

> **Schema-first；atomic write 必装；idempotency keys 必装；大 artifact 分离；event sourcing 为审计。**
