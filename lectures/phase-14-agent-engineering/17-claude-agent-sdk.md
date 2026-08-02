---
chapter: The Harness as a Library — Subagents and Session Store
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/17-claude-agent-sdk
source_doc: phases/14-agent-engineering/17-claude-agent-sdk/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-17 · The Harness as a Library：Subagents 与 Session Store

> A harness you can import: built-in tools, subagents for context isolation, hooks, W3C trace propagation, session persistence. The Claude Agent SDK is the reference example — the library form of the Claude Code harness — and Claude Managed Agents is the hosted alternative for long-running async work.

## 一、本章在整体中的位置

phase 14 的"框架巡礼"走到了**具体厂商 SDK 的另一极** —— 与上一章 OpenAI Agents SDK 对称：

| 章 | 厂商 | 核心抽象 |
|---|---|---|
| 14-16 OpenAI Agents SDK | OpenAI | Agent + Handoff + Guardrail |
| **14-17 Claude Agent SDK** | **Anthropic** | **Harness + Subagent + Hook + Session** |

**核心二分在第一句就摆出来了**：

> **Client SDK**（`anthropic`）—— 原始 Messages API，**你写循环**。
> **Agent SDK**（`claude-agent-sdk`）—— 内置工具 + MCP + Hooks + Subagent + Session，**循环由库实现**。

一句话：**Claude Agent SDK = 把 Claude Code 的命令行 harness 暴露成 Python 库**。理解这一点是理解整章的钥匙。

---

## 二、为什么需要"harness-as-a-library"

读完前 16 章你应该有印象：每个框架都给出一组漂亮的多 agent 抽象，但**没有人包办**这些生产必须的"苦活"：

- 工具执行（文件、shell、grep、glob、web fetch）
- MCP server 连接
- 生命周期 hooks
- Subagent 派生
- Session 持久化
- 跨进程 trace propagation

把这些一并打包暴露成库，就是 Claude Agent SDK 的设计意图。**Anthropic 把"Claude Code 怎么 work"开放出来，让你能用同样的方式搭自己的 agent。**

---

## 三、Subagent 的两个目的（必须分清）

文档明文写了 Subagent 的两个目的：

### 1. Parallelization（并发）

> "Run independent work concurrently. 'Find the test file for each of these 20 modules' is 20 parallel subagent tasks."

天然适合：N 个独立子任务，**没有相互依赖**。

### 2. Context Isolation（上下文隔离）

> "Subagents use their own context window; only results return to the orchestrator. The orchestrator's budget is preserved."

**这是更微妙也更关键的目的**。如果主 agent 一次性读 20 个文件，orchestrator 的 context 就被占满；用 subagent：

```
main agent (orchestrator)
  ├─ subagent_1: 读 file_1..file_5 → 返回 200 字摘要
  ├─ subagent_2: 读 file_6..file_10 → 返回 200 字摘要
  └─ subagent_3: 读 file_11..file_15 → 返回 200 字摘要
```

主 agent 的 context 只增长 **600 字摘要**，而不是 15 个文件的全文。**这是 token 预算工程化的关键**。

### 失败模式（本章明文警告）

> **Subagent over-spawn**：100 个微任务开 100 个 subagent，开销压垮收益。**批处理**（batching）比裸派发好。

### 几个 Python SDK 的近期加成

- `list_subagents()` —— 列出全部 subagent。
- `get_subagent_messages()` —— 读 subagent 的完整 transcript。

这些"读 subagent 内部"的能力让你能做**事后审计**—— production 必备。

---

## 四、Session Store：5 个原语 + 1 个 CLI 魔法

### 5 个原语（与 TypeScript 同构）

```python
session_store.append(session_id, message)     # 加一轮
session_store.load(session_id)                # 恢复完整对话
session_store.list_sessions()                 # 枚举
session_store.delete(session_id)              # 删除（级联到 subagent sessions）
session_store.list_subkeys(session_id)        # 列出 subagent 键
```

| 操作 | 何时用 |
|---|---|
| `append` | 每轮对话结束 |
| `load` | 进程重启、跨设备恢复 |
| `list_sessions` | 列出全部活跃 session（清理 / 监控） |
| `delete` | 用户主动删除 / 过期回收（**必须级联到 subagent sessions**） |
| `list_subkeys` | 调试"为什么这个 session 这么长" |

### `--session-mirror` CLI 魔法

> **Writes the transcript to an external file as it streams.**

边跑边镜像 transcript 到外部文件 —— 本地调试神器。生产里等价于"实时出口到 S3 / 集中日志"。

### 失败模式：**Session bloat**

> **Sessions accumulate; size grows. Use `list_sessions` + expiry policy.**

必须定期清理，否则 storage 爆炸。**这是为什么 `list_sessions` 被列为原语，不是事后工具**。

---

## 五、Hooks：Lifecycle 回调的 7 个挂载点

```
┌────────────────────────────────────────────────────┐
│  Lifecycle Hooks                                  │
│                                                    │
│  SessionStart ──→ UserPromptSubmit ──→ 模型推理     │
│       ↑                              ↓              │
│  SessionEnd                       PreToolUse       │
│       ↑                              ↓              │
│     Stop ←── PostToolUse ←── 工具执行                │
│                                                    │
│  旁路：PreCompact（压缩前）、Notification（旁路通报）│
└────────────────────────────────────────────────────┘
```

| Hook | 用途 |
|---|---|
| **PreToolUse / PostToolUse** | 工具调用的前置闸门 / 后置审计 |
| **SessionStart / SessionEnd** | 会话级 setup / teardown |
| **UserPromptSubmit** | 在模型看到用户输入前做处理（注入上下文 / 改写 / 拒答） |
| **PreCompact** | 上下文压缩前做事（备份、记录） |
| **Stop** | agent 退出时清理 |
| **Notification** | 旁路告警（不阻断主流程） |

### 为什么 Hooks 是生产必需

> **Hooks are how pro-workflow ... and similar systems add cross-cutting behavior.**

- 权限检查（PreToolUse）
- 审计日志（PostToolUse）
- 数据脱敏（PreCompact）
- 告警（Notification）

**这些是横切关注点（cross-cutting concerns）= 在不污染主业务逻辑的前提下加行为**。和 AOP 思想同源。

### 失败模式：**Hook creep**

> **Every team adds hooks; startup time balloons. Review hooks quarterly.**

每加一个 hook 都是冷启动开销。**每季度审计一次 hook 列表**，把不用的摘掉。

---

## 六、W3C Trace Context：跨进程的 trace 传播

> **OTel spans active on the caller propagate into the CLI subprocess via W3C trace context headers. The whole multi-process trace shows up as one trace in your backend.**

W3C Trace Context 是 OTel 的标准 propagation 协议。SDK 把当前活动 span 通过环境变量 / header 传到子进程 —— **你在 trace 后端看到的是一棵树，不是 N 个离散 trace**。

这是 multi-agent 调试的命脉。**没有它，跨 agent 调试等于盲调**。

---

## 七、Claude Managed Agents：托管版

> **The hosted alternative (beta header `managed-agents-2026-04-01`).**
> **Long-running async work, built-in prompt caching, built-in compaction.**

**这是一段必读工程决策**：

| 维度 | 自建 SDK | Managed Agents |
|---|---|---|
| **控制** | 高 | 低（Anthropic 托管） |
| **冷启动** | 自己配置 | 0（Anthropic 兜底） |
| **长期运行** | 自己保活 / 重启 | 平台 guarantee |
| **prompt caching** | 自己实现 | 内置 |
| **compaction** | 自己实现 | 内置 |
| **何时用** | 严合规 / 自定义运行 | 异步长任务 / 运维想让出 |

**关键决策点**：你的产品是"长任务驱动"还是"短请求驱动"？长任务（批处理、夜间 ETL、长对话）→ Managed；短请求（chatBot、IDE 自动化）→ 自建 SDK。

---

## 八、代码实现（`code/main.py`）

`code/main.py` 用 stdlib 实现 Claude Agent SDK 的全要素：

```python
class Tool: ...                       # 工具基类
class ToolRegistry:                   # 工具集合
    read_file, write_file, list_dir   # 内置工具

class Subagent:                       # 独立 context + 独立 run
    async def run(task) -> summary    # 只返回摘要给主 agent

class SessionStore:                   # 5 个原语
    append, load, list_sessions, delete, list_subkeys

class Hooks:                          # 7 个 lifecycle 挂载点
    pre_tool_use, post_tool_use,
    session_start, session_end,
    user_prompt_submit, pre_compact, stop
```

Demo：主 agent 派 3 个 subagent 并行读各自的目录，**只返回摘要**；主 agent 汇总；session 持久化。

运行：
```bash
python3 code/main.py
```

trace 重点：
- **Orchestrator context size bounded**（big win，证明 context isolation 起效）。
- Hook 执行可见。
- Session 持久化跨调用。

---

## 九、Use It：决策树

```
你要做 Claude-first 产品？
  ├─ 短请求 / 强控制 → Claude Agent SDK（本 chapter）
  ├─ 长异步 / 让出运维 → Claude Managed Agents（本 chapter）
  ├─ OpenAI 优先 → 第 14-16 章 OpenAI Agents SDK
  └─ 显式状态机 → 第 14-13 章 LangGraph + 自定义工具
```

---

## 十、常见踩坑（本章明文警告）

1. **Subagent over-spawn** —— 100 个微任务开 100 个 subagent，开销压垮。
2. **Hook creep** —— 每加一个 hook，冷启动慢一点。每季度审计。
3. **Session bloat** —— storage 爆炸，定期清理。

---

## 十一、课后练习 5 题核心思路

1. **Subagent batching**：20 任务 → 5 批 × 4 并发。**测量 orchestrator context size vs 1-per-task**。这会让你看到 context isolation 的真正价值。
2. **`PreToolUse` 限流**：每 session 每分钟 5 次 `write_file` 限流。Hook 是配权限的最佳兵器。
3. **`list_subkeys` 树状可视化**：深度嵌套 subagent 长什么样？**你会发现"主 agent 派 5 个 subagent，每个 subagent 又派 5 个"是真实失败模式**。
4. **真把 toy 端口到 `claude-agent-sdk` Python**：你会发现 SDK 多了**工具 schema 注册、subagent message streaming、trace 自动 propagation**等。
5. **重读 Managed Agents 文档**：决定什么时候从自建切到托管。**最常见的迁移时刻是：会话超时阈值 > 5 分钟时**。

---

## 十二、本章给我的工程启示

1. **"Harness as a library" 是 SDK 演化的高级形态**。从"原始 API"到"循环库"再到"harness 库"，是 agent 工程的成熟路径。**用户要的不是 SDK，是"我不用从零搭这些"**。
2. **Subagent 的两大目的被并列列出，但隐含优先级**：context isolation 高于 parallelization。**真正的 token 预算工程是 subagent 隔离**。
3. **Hooks 是 cross-cutting concerns 的正解**。权限、审计、告警、压缩、脱敏——这些都不该污染主业务逻辑。**漫不经心地塞在 hooks 里，会变成性能黑洞（每季度审计）**。
4. **W3C Trace Context 是分布式 debug 的命脉**。没有它，多 agent 调试等于盲调；有了它，你能在 trace 树里看到完整因果链。**任何自研 agent 框架必须实现这一条**。
5. **自建 vs 托管不是 0/1 决策**。让出运维的同时也意味着让出控制权。**门槛是"你的产品是否容忍 Anthropic 调度中断"**。

---

## 十三、横向对比（接前几章）

| 章 | 抽象 | Subagent 的等价物 |
|---|---|---|
| 14-13 LangGraph | 节点图 | sub-graph（子图） |
| 14-14 AutoGen | Actor | nested actor |
| 14-15 CrewAI | Role + Task | 任务可派给子 crew |
| 14-16 OpenAI Agents SDK | Handoff | `transfer_to_X` 工具 |
| **14-17 Claude Agent SDK** | **Subagent** | **独立 context + 独立 run** |

**最关键的分歧**：

- LangGraph / AutoGen / CrewAI 的"子单元"都和主单元**共享某种状态机**。
- **Claude Agent SDK 的 Subagent 是真隔离** —— 独立 context、只返回摘要、不共享状态。

**这是 token 预算工程上的差异**：Claude Agent SDK 的 subagent 设计对长任务最友好。
