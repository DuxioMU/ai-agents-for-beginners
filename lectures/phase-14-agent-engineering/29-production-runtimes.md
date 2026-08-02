---
chapter: Production Runtimes: Queue, Event, Cron
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/29-production-runtimes
source_doc: phases/14-agent-engineering/29-production-runtimes/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-29 · 生产运行时：Queue、Event、Cron

> Production agents run on six runtime shapes: request-response, streaming, durable execution, queue-based background, event-driven, and scheduled. Pick the shape before you pick the framework. Observability is load-bearing at every shape.

## 一、本章在整体中的位置

> **Pick the shape before you pick the framework.**

**"运行时形态"决定哪些失败能存活**。**Jupyter notebook 不会暴露**：第 37 步网络超时、用户挂断、cron 死、worker OOM。

---

## 二、6 种生产运行时形态

| 形态 | 适用 | Stacks | 可观测性 |
|---|---|---|---|
| **1. Request-response** | < 30s 短任务 | Agno + FastAPI / Mastra + Express | HTTP + OTel span |
| **2. Streaming** | 渐进式输出 | 任何 + SSE/WS | per-chunk timing / first-token latency |
| **3. Durable execution** | 步数未知 / 恢复代价高 | LangGraph / AutoGen v0.4 | checkpoint + auto-resume |
| **4. Queue-based / background** | 长任务（数十~数百步） | Celery / BullMQ / SQS+Lambda | queue depth / per-job latency / DLQ |
| **5. Event-driven** | 触发式 | Claude Managed / CrewAI Flow | trigger source / event-to-start latency |
| **6. Scheduled (cron)** | 周期任务 | K8s CronJob / Render cron / Vercel cron | tick + last-known-good |

### 关键差异

- **Durable execution = LangGraph / AutoGen 的核心差异化**——**每步后 checkpoint，失败自动恢复**。
- **Queue-based 是长任务标准答案**（Anthropic computer use 公告："dozens-to-hundreds of steps per task"）。
- **Event-driven + Scheduled = Claude Managed Agents 一手覆盖**。

---

## 三、2026 部署模式

- **CrewAI Flows** —— 事件驱动生产。
- **Agno** —— 无状态 FastAPI Python 微服务。
- **Mastra** —— server adapters（Express / Hono / Fastify / Koa）嵌入。
- **Pipecat Cloud / LiveKit Cloud** —— 托管 voice（14-22）。
- **Claude Managed Agents** —— 托管长异步。

---

## 四、Observability 是 load-bearing

> **Without OpenTelemetry GenAI spans (Lesson 23) plus a Langfuse/Phoenix/Opik backend (Lesson 24), you cannot debug a multi-step agent that failed at step 40.**

**这不是 optional**。**这是"快速 debug"和"从零重放加 logging"的差别**。

---

## 五、本章的失败模式（明文警告）

1. **Wrong shape choice** —— 5 分钟任务用 request-response。**用户挂；worker 堆；重试叠加**。
2. **No DLQ** —— 队列 worker 无 dead-letter。**失败 job 蒸发**。
3. **Opaque background work** —— 后台 agent 不发 trace。**失败看不见直到用户报告**。
4. **Skipping durable state** —— 任何 > 30s 跑、不能重启 = 需要 durable execution。

---

## 六、代码（`code/main.py`）速览

```python
# 5 形态 demo（durable 在 14-13 LangGraph）：
def request_response(): ...    # 普通函数
def streaming(): ...           # generator
def queue_worker(): ...        # + DLQ
def event_trigger(): ...       # 注册表
def cron_scheduler(): ...      # 周期调度
```

同一 agent 逻辑 5 种外壳——**trace 形状对比**。

---

## 七、Use It 决策树

```
你的运行时形态？
  ├─ 短 chat 风格 → Request-response
  ├─ 渐进式响应 → Streaming
  ├─ 长任务（步数未知） → Durable
  ├─ 批 / 异步 / 长跑 → Queue（+ DLQ）
  ├─ 触发式 → Event-driven
  └─ 周期 housekeeping（memory 整合 / evals / 成本） → Cron
```

---

## 八、课后练习 5 题核心思路

1. **14-01 ReAct 端口 6 形态**：哪种合哪种产品面？
2. **Queue demo 加 DLQ**：模拟 10% job fail；surface DLQ 大小。
3. **Cron-triggered eval agent**：每晚跑昨天 top 20 trace。
4. **Streaming + backpressure**：客户端慢则暂停。**turn budget 怎么配合？**
5. **读 Claude Managed Agents 文档**：什么时候把自建长任务迁到托管？

---

## 九、本章给我的工程启示

1. **Shape first, framework second**。**形态错 = 后面全错**。
2. **DLQ 不是 optional**。**没有 DLQ = 失败 job 蒸发**。
3. **Durable execution 是 long-horizon 的命**。**没有它，长任务 = 重启 + 丢失**。
4. **Background work 必发 trace**。**看不到 = 等用户来告诉你挂了**。
5. **Cron + Durable = housekeeping 的黄金组合**。**每晚合并 / 评估 / 成本**。

---

## 十、横向对比

| 形态 | 何时用 | 关键 Stacks |
|---|---|---|
| **Request-response** | < 30s | Agno / Mastra |
| **Streaming** | 渐进式 | SSE / WS / WebRTC（LiveKit） |
| **Durable** | 长任务 | LangGraph / AutoGen |
| **Queue** | 异步 / 长跑 | Celery / BullMQ / SQS |
| **Event** | 触发式 | Claude Managed / CrewAI Flow |
| **Cron** | 周期 | K8s CronJob / Render / Vercel |

**统一口诀**：

> **Shape 选对 = 失败可活；DLQ 必装；durable 必装；trace 必发；cron 跑 housekeeping。**
