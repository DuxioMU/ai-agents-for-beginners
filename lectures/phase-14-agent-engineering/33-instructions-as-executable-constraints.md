---
chapter: Agent Instructions as Executable Constraints
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/33-instructions-as-executable-constraints
source_doc: phases/14-agent-engineering/33-instructions-as-executable-constraints/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-33 · Agent 指令作为可执行约束

> Instructions written as prose are wishes. Instructions written as constraints are tests. The workbench turns each rule into something an agent can check at runtime and a reviewer can verify after the fact.

## 一、本章在整体中的位置

14-32 给了 3 文件起步。**`AGENTS.md` 之外是规则**——**本章把 prose 升级成 executable constraints**。

> **A typical `AGENTS.md` reads like onboarding documentation. It tells the agent to "be careful" and "test thoroughly" and "ask if unsure."**

**3 天后，agent 交了没测试的修改、写了禁止目录、没问**。**prose 是 wish，constraint 是 test**。

---

## 二、5 类规则（覆盖大多数）

| 类别 | 规则回答什么 | 例 |
|---|---|---|
| **Startup** | 开始前必须什么真 | "state file 存在且新鲜" |
| **Forbidden** | 什么绝不能发生 | "不要 edit `scripts/release.sh`" |
| **Definition of done** | 什么证明完成 | "pytest 退 0 + acceptance line 过" |
| **Uncertainty** | agent 不确定时怎么办 | "开 question note 不猜" |
| **Approval** | 什么需人工批 | "新依赖、prod 写入" |

> **A rule that does not fit one of these five usually wants to be two rules. Force the split.**

**装不进这 5 类的 = 应该拆成 2 条**。

---

## 三、Rules 是 machine-readable + diff-friendly

- 每条 rule 有 **slug、category、one-line description、`check` field**（指向 `rule_checker.py` 里的函数）。
- 加 rule = 加 check；checker 跟 workbench 一起长。
- **每条一节标题**，renames 显现在 diff。
- 新 rule 在分类顶部。
- **Stale rule 删，不注释**。**workbench 是真相，不是 chat log**。

---

## 四、Progressive disclosure（分层而非百科）

```
AGENTS.md                  < 50 行：仓库是什么、看哪里、5 硬规则
docs/
  agent-rules.md           完整规则集（本章）
  architecture.md          任务涉及模块边界时加载
  testing.md               任务涉及 test 时加载
  deploy.md                仅 release 工作，approval 后门
```

| Tier | 位置 | 何时读 | 大小预算 |
|---|---|---|---|
| **Router** | `AGENTS.md` | 每 session | < 50 行 |
| **Rules** | `docs/agent-rules.md` | 每 session 启动 | 一屏/类 |
| **Topic docs** | `docs/<topic>.md` | 任务触及该 topic | 需要多深就多深 |

### 两个测试保分层诚实

- **Reachability**：agent 至多 2 跳到任何 rule——**router 必须 link 路径不描述 prose**。
- **Freshness**：router 短到 reviewer 每次 PR 都重读。**断了的 link = startup check violation**。

---

## 五、生产模式

### Severity 标签 at write time

每条 rule 带 `severity: block | warn | info`。Checker 全报，runtime 只在 `block` 拒。**避免 deadline 时悄悄 weaken**。

### Rule expiry as forcing function

每条 rule 带 `expires_at`（默认 90 天）。Checker 警告"60 天无 violation 的 unexpired rule"——下次季度 review justify / weaken / delete。**Cloudflare 2026-04 数据（131,246 review runs / 5,169 repos / 30 days）**：有 expiry 的 rule set 保持 < 30 条/repo；无 expiry 长到 80+ 大多数永不触发。

### Markdown-as-source, JSON-as-cache

`agent-rules.md` = authored。`agent-rules.lock.json` = checker 热路径读。Pre-commit 重新生成。**Markdown diff 可审，JSON parsing 不进每 turn**。**`package.json` / `Cargo.toml` 同样的 shape**。

---

## 六、Rules vs Framework Guardrails

- Framework guardrails（OpenAI Agents SDK、LangGraph interrupts）runtime 强制。
- 这里的 rule set = human-readable、reviewable 的 contract。
- **两个都要**：runtime catch 违规 during turn，rule set 证明 runtime 做了对的事。

---

## 七、代码（`code/main.py`）速览

```python
# agent-rules.md parser
rules = parse_rules("docs/agent-rules.md")

# rule_checker.py 风格：每条 rule 一个 check 函数
def check_<rule_slug>(ctx): ...

# demo: 一个 agent run 违反 2 条
report = check_all(rules, run_trace)
# 输出：parsed rule set + pass/fail + rule_report.json
```

---

## 八、Use It 决策树

```
你的规则？
  ├─ Claude Code / Codex / Cursor：session 启动读规则，拒时引用。CI re-run。
  ├─ OpenAI Agents SDK guardrails：同 check 注册 input/output guardrail。
  └─ LangGraph interrupts：in-flight node 违规则 interrupt，handler 问人。
```

**rule set 跨 3 个可移植**——**只是 markdown + 函数名**。

---

## 九、课后练习 5 题核心思路

1. **加第 6 类（如真需要）**——**为什么不能并入 5 类？**
2. **加 severity 字段**：checker 聚合 block/warn/info。
3. **接入 CI**：block-severity fail = fail build。
4. **加 expiry 字段**：90 天无 fail = 标记为 review。
5. **真实 `AGENTS.md` 改写为 5 类**：多少行 operational？多少 aspirational？

---

## 十、本章给我的工程启示

1. **Prose 是 wish，constraint 是 test**。**5 类规则覆盖大多数**。
2. **Layered 不是 encyclopedia**。**Router < 50 行，topic docs 按时加载**。
3. **Severity at write time** 防止 deadline 弱化。**Expiry 防止规则腐烂**。
4. **Markdown-as-source + JSON-as-cache**：**diff-friendly + hot-path 快**。
5. **Framework guardrails + rule set 双轨**：**runtime 拒 + contract 证明**。

---

## 十一、横向对比

| 工具 | rule 形式 | 强 enforce |
|---|---|---|
| **Claude Code** | 启动读规则 + 拒时引用 | session 内 |
| **OpenAI Agents SDK** | input/output guardrail | runtime |
| **LangGraph** | interrupt + handler | in-flight |
| **5 类 rule set** | markdown + check 函数 | CI / runtime |

**统一口诀**：

> **5 类覆盖大多数；severity at write time；expiry 防腐；markdown 源 + JSON cache；rule + guardrail 双轨。**
