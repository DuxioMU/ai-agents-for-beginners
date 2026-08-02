---
chapter: Initialization Scripts for Agents
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/35-initialization-scripts
source_doc: phases/14-agent-engineering/35-initialization-scripts/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-35 · Agent 初始化脚本

> Every session that starts cold pays a tax. The agent reads the same files, retries the same probes, and rediscovers the same paths. An init script pays the tax once and writes the answers into state.

## 一、本章在整体中的位置

14-32 给 3 文件起步。**Session 开始时要做 setup 工作**——**14-35 把 setup 变成一次性的 init script**。

> **Open a session. The agent guesses the Python version. Guesses the test command. Lists the repo root five times...**

**1 万 token 烧在 setup**——**应该是 1 个脚本**。

---

## 二、Init script 探什么

| Probe | 为什么 |
|---|---|
| **Runtime versions** | Python/Node 版本错 = 静默 wrong-version bug |
| **Dependency availability** | 缺包后面 10x 成本 |
| **Test command** | agent 必知怎么验证；命令缺 = workbench 坏 |
| **Repo paths** | 硬编码路径漂；解析一次并 pin |
| **Environment variables** | 缺 `OPENAI_API_KEY` = 失败 surface，不是 mystery |
| **State + board freshness** | crashed session 留 stale state = 脚枪 |
| **Last-known-good commit** | session 末 handoff diff 的锚 |

---

## 三、Fail loud, fail fast, fail in one place

> **A probe failure means halt and surface to the human. No "the agent will figure it out."**

**Init = 拒绝启动当 workbench 坏**。**不是 fallback**。

---

## 四、Idempotent

> **Run it twice in a row. The second run should be a no-op except for a fresh timestamp.**

**幂等性让你能挂 CI / hooks / pre-task slash command**。

---

## 五、Init vs Startup Rules

- **Rules**（14-33）描述"必须什么真才能 act"。
- **Init** = 建立"rules 能被 check"的脚本。
- **Rules without init** = "be careful"。
- **Init without rules** = 抛光失败。

---

## 六、生产模式

### Last-known-good commit anchoring

Probe 当前 commit vs `LKG` file（上次成功 merge 写）。Diff 超 budget（默认 50 files）= 拒启动，要人批准新 baseline。**Cloudflare AI Code Review 用此 scope reviewer agent**。

### Lock files with TTL

第一次 probe pass 后写 `prereqs.lock`。后续 N 小时（默认 24h）trust lock 跳过贵 probe。**Init 读 lock 优先；新鲜 + dep manifest hash 匹配 = short-circuit**。**Docker layer cache 同款**。

### No network, no LLM, no surprises in hot path

Init probe = 确定性 plumbing。Probe 调 LLM 分类失败 = workflow 不是 probe。Probe > 3s = workbench smell。

---

## 七、代码（`code/main.py`）速览

```python
# init_agent.py
def probe_python_version(): ...
def probe_deps(): ...
def probe_test_command(): ...
def probe_env_vars(): ...
def probe_state_freshness(): ...

# 每个 probe → (name, status, detail)
# 写 init_report.json，block-severity fail → 退非零
```

Demo：跑 → 表 + 写 init_report.json + happy path 退 0；失败列出 probes。

---

## 八、Use It 决策树

```
init 挂哪里？
  ├─ Claude Code hooks：pre-task hook 调 init 脚本，failed 不启
  ├─ GitHub Actions：setup-agent job 跑 init，agent job depends
  └─ Docker entrypoint：容器 init 后 exec agent runtime
```

**init portable**——**不绑特定 framework**。Bash / Make / task file 都包。

---

## 九、课后练习 5 题核心思路

1. **加 LKG diff probe**：> 50 files 改 = 拒启。
2. **`prereqs.lock` + > 7 天拒启**。
3. **`--fix` flag**：自动装 dev deps，但 runtime deps 必 approval。
4. **probe 移到 YAML registry**——**trade-off**？
5. **每 probe 加 timing budget**：> 3s = smell。

---

## 十、本章给我的工程启示

1. **Init 是一次性付的税**。**Session 启动 = 0 token setup**。
2. **Fail loud**——**Init 失败 = 拒启动**，不靠"agent 想办法"。
3. **Idempotent** = 能挂 CI、hooks、slash command。
4. **LKG anchoring 防 drift 累积**。**Cloudflare 的现实做法**。
5. **No LLM in init**——**init 是 plumbing，不是 workflow**。

---

## 十一、横向对比

| 项 | 角色 |
|---|---|
| **`init_report.json`** | Init 写，agent 读 |
| **LKG commit** | 锚定 handoff diff |
| **prereqs.lock** | 跳过贵 probe 的 cache |
| **probe timing budget** | 3s = smell |

**统一口诀**：

> **Init 一次性付税；fail loud fast；idempotent；LKG 锚定；prereqs lock；no LLM in init。**
