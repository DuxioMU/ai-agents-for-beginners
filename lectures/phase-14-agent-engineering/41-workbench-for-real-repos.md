---
chapter: The Workbench on a Real Repo
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/41-workbench-for-real-repos
source_doc: phases/14-agent-engineering/41-workbench-for-real-repos/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-41 · 在真仓上跑 Workbench

> Eleven lessons of surfaces are worth nothing if they do not survive contact with a real codebase. This lesson runs the same task twice on a small sample app: prompt-only versus workbench-guided. The numbers do the arguing.

## 一、本章在整体中的位置

14-31 ~ 14-40 给了 7 surface + workbench。**14-41 把它们在真仓上对比**——**prompt-only vs workbench-guided，5 outcomes 量化**。

> **A demo on a toy task convinces no one.**

---

## 二、Sample app + Task

### Sample app
- `app.py`（`/signup` 无 validation）
- `test_app.py`（一 happy-path test）
- `README.md` + `scripts/release.sh` 作 forbidden-zone bait

### Task

> Add input validation to `/signup`: reject passwords shorter than 8 characters, return 422 with a typed error envelope. Add a test that proves the new behavior.

---

## 三、两条 pipeline

**Prompt-only**:
1. 读 README
2. 读 `app.py`
3. 改 files
4. 声称 done

**Workbench-guided**:
1. 跑 init 脚本（14-35）
2. 读 scope contract（14-36）
3. 读 state（14-34）
4. 仅改 allowed files
5. 跑 acceptance via feedback runner（14-37）
6. 跑 verification gate（14-38）
7. 跑 reviewer（14-39）
8. 生成 handoff（14-40）

---

## 四、5 outcomes

| Outcome | 为什么 |
|---|---|
| `tests_actually_run` | 多数"tests passed" 不可验证 |
| `acceptance_met` | 真证 goal 的 test 必须 = 跑的 test |
| `files_outside_scope` | Scope creep 是静默失败主流 |
| `handoff_quality` | 下 session 受益或负担 |
| `reviewer_total` | 在 gate 之上的定性判断 |

---

## 五、2026 数据

| 数字 | 来源 |
|---|---|
| **Terminal Bench 2.0**：同 model，harness 改 → top 30 外跳到 rank 5 | LangChain |
| **Vercel**：删 80% 工具 → 80% → 100% 成功率 | MongoDB |
| **Harvey**：legal agent 准确率 2x 提升（harness only） | MongoDB |
| **88% enterprise AI agent project 不到 production** | preprints.org |
| **WebAgent long-context collapse**：40-50% → < 10% | 早期 2026 |

> **False negatives still exist.** 单步事实 task / 一行 lint / formatter run / 任何 model 默背的——**这些 prompt-only 更快**。**Benchmark 老实枚举它们，workbench 不被框 overkill**。

---

## 六、代码（`code/main.py`）速览

```python
# 两 pipeline 跑同 sample app
# scripted agent（无 LLM 在 loop）= reproducible

# 输出：console table of outcomes per pipeline
#        markdown report saved next to script
#        JSON for charting
```

---

## 七、Use It

**This lesson is the case file you cite when**:
- 有人问"为什么每 PR 都有 `agent-rules.md` 和 scope contract"。
- 团队想"这次 sprint 砍掉 verification gate"。
- 新 agent 产品 launch 你要 portable benchmark 测它是否真的省时间。

**Numbers 走得更远 than explanation**。

---

## 八、课后练习 5 题核心思路

1. **加第 6 outcome：time-to-first-meaningful-edit**。**怎么 clean 测**？
2. **真 second-day task 跑对比**：workbench 数字哪里滑？
3. **"false negative" pass**：prompt-only 更快的 task + workbench overhead 是真成本。**为什么仍保留 workbench**？
4. **scripted agent → 真 LLM**：**哪个 outcome 变噪**？
5. **一页给非工程师的 summary**——**什么 cut 后活下来**？

---

## 九、本章给我的工程启示

1. **数字走更远**。**prompt-only vs workbench 量化是 case file**。
2. **5 outcomes 必测**：tests ran / acceptance met / scope / handoff / reviewer。
3. **False negatives 老实枚举**。**Workbench 不被框 overkill**。
4. **同 model 不同 harness = 25 rank 差**。**Harness 改 ≠ model 改**。
5. **88% 不到 prod = runtime 问题**。**别等 model 升级**。

---

## 十、横向对比

| 维度 | Prompt-only | Workbench-guided |
|---|---|---|
| **tests_actually_run** | 多数假 | 必有 record |
| **acceptance_met** | 解释宽松 | 真证 |
| **files_outside_scope** | 常 creep | contract 限 |
| **handoff_quality** | 散 | 7 字段 packet |
| **reviewer_total** | 无 | 5 维 rubric |

**统一口诀**：

> **5 outcomes 必测；数字走远；false negatives 老实枚举；88% 不到 prod = runtime；harness 改 ≠ model 改。**
