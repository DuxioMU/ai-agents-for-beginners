---
chapter: The Minimal Agent Workbench
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/32-minimal-agent-workbench
source_doc: phases/14-agent-engineering/32-minimal-agent-workbench/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-32 · 最小化 Agent Workbench

> The smallest useful workbench is three files: a root instructions router, a state file, and a task board. Everything else is layered on top. If a repo cannot carry these three, no model will save it.

## 一、本章在整体中的位置

14-31 讲了"为什么需要 7 surface"。本章降到**最小可用品**——**3 个文件**。

> **Most teams reach for a workbench by writing a 3000-line `AGENTS.md` and calling it done.**

**3 个文件，每个有职责**——**不是 3000 行 prose 叫"完成"**。

---

## 二、3 个文件

### 1. `AGENTS.md` = router（不是手册）

**好的 `AGENTS.md` 是短的**——**指向 deeper files**：

- State file（你在哪）
- Task board（还剩什么）
- Deeper rules（`docs/agent-rules.md`）
- Verification command（怎么知道工作）

**长 manual 被忽略。短 router 被跟随**。

### 2. `agent_state.json` = system of record

- 活跃 task id
- 触过的 files
- 作出的 assumptions
- 阻塞
- 下一步

**State 在文件**——**chat 历史不可靠**。**Session 死、对话被截、文件不死**。

### 3. `task_board.json` = queue

每条 task 有 status：`todo | in_progress | done | blocked`。

- **id**、**goal**、**owner**（builder / reviewer / human）、**acceptance criteria**。
- 队列当 state 空时 agent 拉取。
- **小是 feature**：超过一屏 = 计划问题。

---

## 三、为什么这 3 个

**后面章节会加** scope contracts、feedback runners、verification gates、reviewer checklists、handoff packets。**这 3 个是它们都依赖的"基线"**。

---

## 四、生产模式

### 嵌套 `AGENTS.md`（nearest-wins）

- OpenAI 主仓 88 个 `AGENTS.md`，一个 per sub-component。
- Codex / Cursor / Claude Code / Copilot 都从 working file 走回 repo root，**concat 所有 `AGENTS.md`**。
- 子目录文件 extend 根文件。Codex 加 `AGENTS.override.md` 替换（Codex-specific，避免跨工具）。
- **Augment Code 测量**：**最好的 `AGENTS.md` 等于 Haiku→Opus 升级**；**最差的让输出比没文件还糟**。

### 必拒的反模式

- **冲突指令静默把 agent 从 interactive 降到 greedy mode**（ICLR 2026 AMBIG-SWE：48.8% → 28% resolve rate）。
- **不可验证的风格规则**（如"follow Google Python Style Guide" 无 lint command）= agent 编造 compliance。
- **风格领先于命令** = 验证路径被埋。
- **写给人类不是写给 agent** = 浪费 context budget。**简洁是 feature**。

### 跨工具 symlinks

```bash
ln -s AGENTS.md CLAUDE.md
ln -s AGENTS.md .github/copilot-instructions.md
ln -s AGENTS.md .cursorrules
```

**单源真相分发到每个 coding agent**。Nx `nx ai-setup` 自动化 6 个工具。

---

## 五、代码（`code/main.py`）速览

```python
# 把 minimal workbench 写入空 repo
write_workbench(workdir)   # 三个文件

# demo：单 agent turn
state = read_state()
if state.empty:
    task = pull_next_task(board)
# touch single file in scope
# write back updated state
```

`workdir/` 下生成 3 文件，跑一 turn，print diff。**重跑看第二 turn 接第一 turn**。

---

## 六、Use It 决策树

```
workbench 3 件套在哪？
  ├─ Claude Code：AGENTS.md / CLAUDE.md + state-style stores + hooks
  ├─ Codex / Cursor：workspace rules + session memory + chat sidebar queue
  └─ 自建 Python agent：你刚写的同样 3 文件
```

**名字变，shape 不变**。

---

## 七、课后练习 5 题核心思路

1. **`last_run` timestamp** —— 超过 24h 拒跑，除非操作员确认。
2. **Task board 加 `priority`** —— puller 选最高 priority todo。
3. **`task_board.json` 迁到 JSON Lines** —— 每 task 一行，diff 干净。
4. **`lint_workbench.py`** —— `AGENTS.md` > 80 行 或引用不存在文件就 fail。
5. **3 个文件哪个最痛**？**defend**。

---

## 八、本章给我的工程启示

1. **最小可用品 = 3 文件**。**不是 3000 行 prose**。
2. **`AGENTS.md` 是 router 不是 manual**。**短的被跟随，长的被忽略**。
3. **State 在文件**——**chat 是易失的**。
4. **Task board 是 queue**。**Agent 拉取而不是被告知**。
5. **嵌套 `AGENTS.md` + symlinks** = 跨工具单源真相。**Augment 数据 = 质量提升等于模型升级**。

---

## 九、横向对比

| 工具 | Router | State | Queue |
|---|---|---|---|
| **Claude Code** | `AGENTS.md` / `CLAUDE.md` | `.claude/state.json` 风格 | hooks |
| **Codex / Cursor** | workspace rules | session memory | chat sidebar |
| **自建** | `AGENTS.md` | `agent_state.json` | `task_board.json` |

**统一口诀**：

> **3 文件起步；AGENTS.md 短 + router；state 在文件；task board 是一屏；嵌套 + symlinks 跨工具。**
