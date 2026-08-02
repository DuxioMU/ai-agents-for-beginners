---
chapter: Agent Workbench Engineering — Why Capable Models Still Fail
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/31-agent-workbench-why-models-fail
source_doc: phases/14-agent-engineering/31-agent-workbench-why-models-fail/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-31 · Agent Workbench 工程：为什么强模型还是失败

> A capable model is not enough. Reliable agents need a workbench: instructions, state, scope, feedback, verification, review, and handoff. Strip those away and even a frontier model produces work that is unsafe to ship.

## 一、本章在整体中的位置

phase 14 后 12 章（14-31 ~ 14-42）切到**"Agent Workbench"**——**强模型不等于可靠 agent**。**本章是整个 mini-track 的总纲**。

> **You drop a frontier model into a real repo and ask it to add input validation. It opens four files, writes plausible code, declares success, and stops.**

**模型对 Python 不陌生，但对"工作"陌生**。**它不知道"完成"意味着什么、在哪允许写、哪个测试权威、下次 session 怎么接**。

---

## 二、7 个 workbench 表面

| Surface | 承担 | 缺时失败 |
|---|---|---|
| **Instructions** | 启动规则 / 禁止动作 / 完成定义 | agent 猜"ship 是什么意思" |
| **State** | 当前任务 / 触过的文件 / 阻塞 / 下一步 | 每个 session 从零开始 |
| **Scope** | 允许文件 / 禁止文件 / 接受标准 | 编辑泄漏到不相关代码 |
| **Feedback** | 真命令输出被捕获进 loop | agent 在 400 上声明成功 |
| **Verification** | test / lint / smoke / scope check | "看起来好" 进了 main |
| **Review** | 第二遍不同角色 | 建设者自批作业 |
| **Handoff** | 改了什么 / 为什么 / 还剩什么 | 下次 session 重新发现一切 |

**loop 闭在 state 文件上，不在 chat 历史**。**chat 易失，repo 是记录系统**。

---

## 三、Workbench vs Prompt Engineering vs Framework

- **Prompt engineering** 告诉模型"这一轮你要什么"。
- **Workbench** 告诉模型"怎么跨轮跨 session 工作"。
- **Framework**（LangGraph / AutoGen / Agents SDK）= 运行时。
- **Workbench** = runtime 内的"工作场所"。

> **Most agent failure stories are workbench failures wearing prompt-engineering clothes.**

**多数 agent 失败 = workbench 失败穿着 prompt-engineering 外衣**。

---

## 四、从分布式系统原语翻译

phase 14 31-42 不发明新概念。**每个 workbench 表面映射到经典分布式系统原语**：

| 原语 | Agent 上承担 |
|---|---|
| **Function** | 工具调用、规则 check、verification |
| **Worker** | builder、reviewer、verifier、MCP server |
| **Trigger** | agent tick、HTTP、queue、cron、file change、hook |
| **Runtime** | Claude Code 进程、LangGraph runtime |
| **HTTP/RPC** | 工具调用协议、MCP、model API |
| **Queue** | task board、feedback log、review inbox |
| **Session persistence** | agent_state.json / checkpoint / KV |
| **Authorization policy** | 允许/禁止文件、approval、MCP capability |

### 7 surfaces → 8 原语映射

- **Instructions** = policy + function metadata。
- **State** = session persistence。
- **Scope** = authorization policy per task。
- **Feedback** = invocation log 进 queue。
- **Verification** = function，fails closed。
- **Review** = 独立 worker（read-only on builder artifacts）。
- **Handoff** = session-end 触发的 durable record。

**agent loop = 消费事件的 worker，调 function，写 record，发 trigger**。

---

## 五、Harness 词汇翻译表

| 流行术语 | 实际是 |
|---|---|
| **Ralph Loop** | 触发器：re-enqueue 任务带 clean context |
| **Plan/Execute/Verify** | 三个 worker 通过 state + queue 通信 |
| **Harness-compute separation** | 重述 control-plane / data-plane |
| **Open Agent Passport (OAP)** | 授权策略 + 签名审计 queue |
| **Guides + Sensors (Böckeler)** | 授权 + verification + 观测 |
| **Progressive compaction (5-stage)** | state-management worker cron-like |
| **Hooks / middleware** | 触发 + function 包 runtime |
| **Skills as Markdown** | 函数注册表，metadata 按时加载 |
| **Sandbox agents** | 隔离 fs/network 的 compute plane |
| **MCP servers** | Workers 暴露 function over stable RPC |

> **Every entry is the agent community arriving at a primitive that already had a name in distributed systems and giving it a new one.**

**每个"新概念" = 旧概念换名字**。**翻译回原语才能用**。

---

## 六、2026 数据

> **The harness-over-model claim has numbers behind it now.**

| 数字 | 来源 |
|---|---|
| **Terminal Bench 2.0**：同模型，harness 改，从 top 30 外跳到 rank 5 | LangChain |
| **Vercel**：删 80% 工具 → 80% → 100% 成功率 | MongoDB |
| **Harvey**：legal agent 准确率 2x 提升（harness only） | MongoDB |
| **88% enterprise AI agent project 不到 production** | preprints.org |
| **WebAgent long-context collapse**：40-50% → < 10% | 早期 2026 |

> **Today, the load-bearing engineering is around the model, not inside it, and the primitives that carry that load are the ones every production system has always needed.**

**今天的承重工程在 model 之外**。

---

## 七、本章的失败模式（明文警告）

> **Where vendor writeups stop short.**

- **LangChain "Anatomy"** 11 个 components——**没队列、worker、trigger、persistence、authz**。
- **Addy Osmani "Harness Engineering"** = stance 而非 spec。
- **Anthropic / OpenAI** 深入 surfaces 但在自己 runtime 内。
- **agentic_harness book** 把 harness 当 config object，**最强的一句是"harness 是 primary security boundary"**——**就是 authz**。
- **HN "harness belongs outside the sandbox"** = 授权作为独立 plane。

> **They are writing UX descriptions of a system that already exists. We are writing the system.**

**它们写的是已有系统的 UX 描述**。**我们写的是系统**。

---

## 八、代码（`code/main.py`）速览

> **A tiny repo task twice. First as prompt only, then with the seven surfaces wired in.**

```python
# 任务：给 /signup 加 input validation，写 passing test
# Run 1: prompt only → 失败
# Run 2: 7 surfaces wired → 通过
# 输出：side-by-side log + failure_modes.json + one-line verdict
```

**The agent is a tiny rule-based stub; the point is the surfaces, not the model.**

**agent 是规则 stub；重点是 surface，不是 model**。

---

## 九、Use It 决策树

```
workbench 表面在哪里已经存在？
  ├─ Claude Code / Codex / Cursor：AGENTS.md / CLAUDE.md = instructions
  │   slash commands = scope，hooks = verification
  ├─ LangGraph / OpenAI Agents SDK：checkpoints / session store = state
  │   handoffs = handoff
  └─ CI on real repo：tests / lint / type-check = verification
    PR template = handoff，CODEOWNERS = review
```

**workbench engineering = 把这些 surface 显式化和可重用**。

---

## 十、课后练习 5 题核心思路

1. **给自己的 agent repo 评 7 surfaces**（0/1/2）。**最弱是哪个？**
2. **扩 main.py 让 prompt-only 跑假"success"**——**验证 gate 抓到吗**？
3. **加第八 surface**（自己产品特定）——**为什么不能合并入 7 个**？
4. **重跑不同 stub agent**，会幻写一个文件——**哪个 surface 先抓**？
5. **26 章 5 行业模式 → 7 surface**：每个 mode 设计被哪个 surface 吸收。

---

## 十一、本章给我的工程启示

1. **强模型 ≠ 可靠 agent**。**workbench 是让"灵光一闪"变"可重复工程"的东西**。
2. **7 surface 来自 8 分布式原语**。**别发明新词**——**翻译回原语**。
3. **Harness = 8 原语连对**。**不是 11 个 components 的 config object**。
4. **承重工程在 model 外**。**88% 不到 production = runtime 问题不是 model 问题**。
5. **"修"workbench 比"等"模型升级更划算**。**Harness 改一下 = top 30 → top 5**。

---

## 十二、横向对比

| 工件 | 角色 | 章节 |
|---|---|---|
| **agent_state.json** | state | 14-34 |
| **AGENTS.md** | instructions | 14-32 / 14-33 |
| **scope_contract.json** | scope | 14-36 |
| **feedback_record.jsonl** | feedback | 14-37 |
| **verify_agent.py** | verification | 14-38 |
| **reviewer (separate agent)** | review | 14-39 |
| **handoff.md / .json** | handoff | 14-40 |

**统一口诀**：

> **7 surface 来自 8 原语；workbench ≠ framework；强模型不够；88% 不到 prod = runtime 问题；改 harness 比等模型升级划算。**
