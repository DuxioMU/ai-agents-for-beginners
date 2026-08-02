---
chapter: Capstone — Ship a Reusable Agent Workbench Pack
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/42-agent-workbench-capstone
source_doc: phases/14-agent-engineering/42-agent-workbench-capstone/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-42 · Capstone：交付可复用 Agent Workbench Pack

> The mini-track ends with a pack you drop into any repo. Eleven lessons of surfaces compressed into a directory you can `cp -r` and have an agent working reliably the next morning. The capstone is the artifact this curriculum trades on.

## 一、本章在整体中的位置

14-31 ~ 14-41 给了所有 surface。**14-42 是包**——**一个能 `cp -r` 进任何仓的目录**。

> **A workbench that lives in a Google Doc, a chat history, and three half-remembered scripts is a workbench that gets rebuilt every quarter.**

**cure = versioned pack**——**仓或目录含 surfaces + schemas + scripts + one-command installer**。

---

## 二、Pack layout

```
outputs/agent-workbench-pack/
├── AGENTS.md
├── docs/
│   ├── agent-rules.md
│   ├── reliability-policy.md
│   ├── handoff-protocol.md
│   └── reviewer-rubric.md
├── schemas/
│   ├── agent_state.schema.json
│   ├── task_board.schema.json
│   └── scope_contract.schema.json
├── scripts/
│   ├── init_agent.py
│   ├── run_with_feedback.py
│   ├── verify_agent.py
│   └── generate_handoff.py
├── bin/
│   └── install.sh
└── README.md
```

---

## 三、什么 in / 什么 out

### In
- **Surface schemas**——**合同**。
- **4 scripts**——**runtime**。
- **4 docs**——**rules + rubric**。

### Out
- **项目特定 tasks**——**进 target repo board，不在 pack**。
- **Vendor SDK calls**——**pack framework-agnostic**。
- **Onboarding prose**——**pack 住团队已有 onboarding，不在里面**。

---

## 四、Installer

`bin/install.sh`（或 `bin/install.py`）：

1. **拒绝覆盖现有 pack**（除非 `--force`）。
2. 复制 pack 进 target repo。
3. **接 CI 如果 `.github/workflows/` 存在**。
4. **打印下一步**：填 board、设 acceptance 命令、跑 init。

---

## 五、Versioning

- **`VERSION` 文件**。
- **Major bump**：schema / script 改要 migration。
- **Minor bump**：checker re-run。
- **Patch bump**：doc-only。
- **target repo `agent_state.json`** 记初始 pack version。

---

## 六、生产模式

### `VERSION` 是合同不是 marketing

Major bump 要 state migration。Minor bump 要 checker re-run。Patch = doc-only。**Installer 写 `.workbench-version` 进 target repo 每次安装**。**`lint_pack.py` 拒 ship 如果 target lock 与 pack `VERSION` 不符**。**`npm` / `Cargo` / `pyproject.toml` 10 年生存的规则**——**agent 也不变**。

### Single source for cross-tool distribution

**Nx `nx ai-setup`** 一份配置出 `AGENTS.md` / `CLAUDE.md` / `.cursor/rules/` / `.github/copilot-instructions.md` + MCP server。**Pack 应如此**——**installer emit symlinks**（`ln -s AGENTS.md CLAUDE.md`）让单源真相分发到每个 coding agent。**为某一工具 fork pack = failure mode**。

### `uninstall.sh` 拒 on non-trivial state

**Uninstall 不删 user 的 `agent_state.json` / `task_board.json` / `outputs/`**。**Uninstaller 移除 schemas / scripts / docs / `AGENTS.md`（`--keep-agents-md` opt-out）**——**state files 有 uncommitted changes 时拒**。**State 属 user；pack 不拥有**。

### Skill-as-publishable / SkillKit-style

**Pack ship as SkillKit skill**：`skillkit install agent-workbench-pack` 跨 32 AI agents。**Pack repo = source of truth；SkillKit = distribution channel**。**Vendor lock-in 崩；7 surfaces 不变**。

---

## 七、代码（`code/main.py`）速览

```python
# 把 pack 装到 outputs/agent-workbench-pack/
# 用之前 lesson 的 schemas + scripts
# 写 README、print pack tree、exits 0
# 重跑 idempotent
```

---

## 八、Use It 决策树

```
Pack 怎么 ship？
  ├─ 目录 cp 进 repo：cp -r outputs/agent-workbench-pack /path/to/repo
  ├─ 公共 template repo：fork-and-customize，VERSION 控 drift
  └─ SkillKit skill：单命令跨 32 agents 装
```

**Pack = 配方**。**每次安装 = 一份**。

---

## 九、课后练习 5 题核心思路

1. **决定哪个 optional 第 5 文档进 pack**——**defend cut**。
2. **Installer 用 Python 重写 + `--dry-run`**。**vs bash ergonomics**？
3. **`bin/uninstall.sh`**：state files 有 non-trivial history 时拒。**什么算 non-trivial**？
4. **`lint_pack.py`**：pack drift vs `VERSION` fail。**接 CI for pack 自己的 repo**。
5. **从手工 workbench 迁 pack 的 runbook**——**最小 downtime 序**？

---

## 十、本章给我的工程启示

1. **Pack 是 artifacts 的版本**——**doc+chat 不是 workbench**。
2. **`VERSION` 是合同**——**Major/Minor/Patch 语义**。**`npm` 规则适用**。
3. **Symlinks = 跨工具单源**——**forks 工具是 failure mode**。
4. **Uninstall 安全**——**state 属 user**。
5. **SkillKit-style 跨 32 agents 分发**——**pack 不绑 vendor**。

---

## 十一、横向对比

| 形态 | 何时用 |
|---|---|
| **目录 cp 进 repo** | 一次性 |
| **公共 template repo** | 长期 fork |
| **SkillKit skill** | 跨 32 agents 分发 |

**统一口诀**：

> **Pack 版本化；VERSION 是合同；symlinks 跨工具；uninstall 安全；SkillKit-style 分发；cp -r 起步。**
