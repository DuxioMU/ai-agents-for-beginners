---
chapter: Voice Agents — Pipecat and LiveKit
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/22-voice-agents-pipecat-livekit
source_doc: phases/14-agent-engineering/22-voice-agents-pipecat-livekit/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-22 · Voice Agents：Pipecat 与 LiveKit

> Voice agents are a first-class production category in 2026. Pipecat gives you a Python frame-based pipeline (VAD → STT → LLM → TTS → transport). LiveKit Agents bridges AI models to users over WebRTC. Production latency targets land at 450–600ms end-to-end for premium stacks.

## 一、本章在整体中的位置

第 14-21 章讲了**屏幕**（computer use）。本章切到**声音**—— 另一个"实时"维度。

> **Voice agents are not a text loop with TTS bolted on.**

把文字 agent loop 加个 TTS 不是 voice agent。**Voice agent 是另一类工程**：

- **延迟预算 600ms 级别**（不是 30s）
- **部分音频是默认**（用户说到一半就要处理）
- **turn detection 是个模型**（不是 silence-based）
- **transport 横跨 SIP 电话到 WebRTC**

**两种主导路径**：

| 路径 | 形态 | 代表 |
|---|---|---|
| **自建** | **frame-based pipeline** | Pipecat |
| **平台** | **WebRTC + 平台抽象** | LiveKit Agents |

**商业托管**：Vapi / Retell 在这两个之上再加一层。

---

## 二、Pipecat：frame-based pipeline

### 核心抽象

- **Frame**：类型化数据单元（audio / transcript / text / tts_audio / control）。
- **FrameProcessor**：处理 frame 的 stage，有 `process(frame)`。
- **PipelineTask**：管 lifecycle，发 `on_pipeline_started` / `on_pipeline_finished` / `on_idle_timeout` 事件，挂 observers 出 metrics / tracing / RTVI。

### 两个流方向（这是 Pipecat 的灵魂）

```
   DOWNSTREAM（source → sink）           UPSTREAM（feedback / control）
   ┌────────────────────────────────┐    ┌────────────────────────────────┐
   │ VAD → STT → LLM → TTS → 出口  │    │ 取消 / 指标 / barge-in / 错误   │
   │ （音频进 → 语音出）            │    │ （从下游回传到上游）            │
   └────────────────────────────────┘    └────────────────────────────────┘
```

- **DOWNSTREAM**：音频进 → 文字 → 推理 → 合成语音 → 出口。
- **UPSTREAM**：取消、指标、barge-in、错误，**从下游回传到上游**。

**UPSTREAM 是 voice agent 与 text agent 最大的区别**。**barge-in（用户打断）就是 UPSTREAM 取消信号回到 TTS 停掉当前语音**。**没有 UPSTREAM 的 pipeline 不能叫 voice pipeline**。

### 典型 pipeline

```
VAD (Silero) → STT → LLM (user/assistant context 交替) → TTS → transport
```

### 三个常见附加

- **Pipecat Flows**：结构化对话（state machine）。
- **Pipecat Cloud**：托管 runtime。
- **Transports**：Daily / LiveKit / SmallWebRTCTransport / FastAPI WebSocket / WhatsApp。

### 关键洞察

**Frame-based design 让"加新 stage"和"换 transport"是同一种操作**。**这是 frame pipeline 范式的最大回报**。

---

## 三、LiveKit Agents：WebRTC 之上的 voice 抽象

### 核心抽象

- `Agent` / `AgentSession` / `entrypoint` / `AgentServer` 四个原语。
- 把 AI 模型通过 **WebRTC** 桥接到用户。

### 两个 voice agent 类（必须分清）

| 类 | 路径 | 控制力 | 适用 |
|---|---|---|---|
| **MultimodalAgent** | 音频直接 → 多模态模型 → 音频直接（OpenAI Realtime / 同类） | **低**（模型在中间全权处理） | 超低延迟、放弃文本层控制 |
| **VoicePipelineAgent** | STT → LLM → TTS cascade（文字层有 hook） | **高**（可截中间文本、调 prompt） | 需要可观测、可调试、改 prompt |

**关键决策点**：

> **要 text-level control → VoicePipelineAgent；追求最低延迟 → MultimodalAgent。**

**MultimodalAgent 的代价是放弃可解释性**。**VoicePipelineAgent 的代价是更高的延迟**。

### 关键特性

- **Semantic turn detection via transformer model**（不是 silence-based）：更准确地判断"用户说完了吗"。
- **Native MCP integration**：tooling 一等公民。
- **Telephony via SIP**：能进电话网络。
- **LiveKit Inference**：50+ 模型无 API key；200+ 插件可用。

---

## 四、2026 商业托管层

- **Vapi**：~450–600ms 优化 premium stack。
- **Retell**：~600ms 端到端，180 个测试通话。

**"如果你没 WebRTC 团队就上托管"** 是这里的经验。

---

## 五、延迟预算：把链加起来再 ship

### 2026 各 stage 的延迟范围

| Stage | 延迟 |
|---|---|
| **VAD** | 20–60ms |
| **STT partial** | 100–250ms |
| **LLM first token** | 150–400ms |
| **TTS first audio** | 100–200ms |
| **Transport RTT** | 30–80ms |

### 端到端体感

| 延迟 | 体感 |
|---|---|
| **450–600ms** | premium（舒服） |
| **800–1200ms** | 常见（能用） |
| **>1500ms** | **broken 感** |

**Pipecat / LiveKit 是达到 450–600ms 的工程路径**。**每一 stage 必加才知总和**。

### 关键洞察

> **Every component adds 50–200ms. Sum your chain before shipping.**

**别猜延迟**。**每一 stage 跑 profiler，5 个 stage 加起来，450 还是 1500 ms 一目了然**。

---

## 六、voice agent 必装的 4 个东西

### 1. **Barge-in 处理**

> **User interrupts; agent keeps talking. Requires UPSTREAM cancel frames in Pipecat, equivalent in LiveKit.**

**没有 barge-in 的 voice agent 是失败品**。**用户说"等一下" agent 还在讲，体验灾难**。

### 2. **STT confidence 闸门**

> **Low-confidence transcripts fed to the LLM as if gospel. Gate on confidence or request confirmation.**

**STT 不可能 100% 对**。**置信度低于阈值的 transcript 要么确认要么丢掉**。**不要把噪声当真理**。

### 3. **TTS mid-sentence cutoff**

> **When the pipeline cancels mid-utterance, TTS needs to know or cut audio.**

**barge-in 触发时，TTS 必须立刻停**。**否则用户听到的是"被掐断的尾音"**。

### 4. **Latency budget 必算**

**不是"差不多就行"，是"450ms / 800ms / 1500ms"哪个档**。**链的每一段必须有数字**。

---

## 七、本章的失败模式（明文警告）

1. **No barge-in handling** —— 用户打断 agent 不停。
2. **STT confidence ignored** —— 低置信度 transcript 喂 LLM。
3. **TTS mid-sentence cutoff 漏** —— barge-in 后还有"被掐尾音"。
4. **Latency budget ignored** —— 不知道链加起来的延迟。

---

## 八、代码（`code/main.py`）速览

`code/main.py` 实现一个 frame-based toy pipeline：

```python
class Frame: ...                        # 类型化数据单元
   AudioFrame, TranscriptFrame, TextFrame,
   TTSAudioFrame, ControlFrame

class Processor:                         # pipeline stage
   def process(self, frame): ...

# 五 stage pipeline
vad → stt → llm → tts → transport

# UPSTREAM cancel frame：barge-in
ControlFrame(type="cancel")   # ← 关键演示
```

Demo：正常流 + barge-in cancel 演示（barge-in 把 TTS 在句子中段停掉）。

运行：
```bash
python3 code/main.py
```

---

## 九、Use It：决策树

```
你的 voice agent 起步？
  ├─ 想要完整控制 / 自定义 processor / Python-first → Pipecat
  ├─ 部署在 WebRTC / 想要 SIP 电话 / 想要 MCP / 想要 LiveKit Inference → LiveKit Agents
  │     ├─ 想要 text-level 控制 → VoicePipelineAgent
  │     └─ 想要最低延迟 / 放弃文本层 → MultimodalAgent（OpenAI Realtime / Gemini Live）
  └─ 没 WebRTC 团队 / 不想自建 → Vapi / Retell
```

---

## 十、Ship It：`outputs/skill-voice-pipeline.md`

自动生成 Pipecat 形态的 voice pipeline 脚手架（VAD + STT + LLM + TTS + transport + barge-in）。

---

## 十一、课后练习 5 题核心思路

1. **加 metrics observer**：每 stage 每秒多少 frame。**你会立刻看到"哪一 stage 卡"**。
2. **Confidence-gated STT**：阈值以下就 "could you repeat that?"。**这一题把"STT 不可信"做成产品行为**。
3. **Semantic turn detection**：简单规则"transcript 结尾是 '?' 就 end of turn"。**rule-based 起步，model-based 进阶**。
4. **Pipecat transport 切换**：把 stdlib transport 换 SmallWebRTCTransport stub。**这一题让"换 transport"成肌肉记忆**。
5. **OpenAI Realtime vs cascade 测延迟**：同一个查询比"端到端"。**让你看到"放弃 text-level control 的代价"**。

---

## 十二、本章给我的工程启示

1. **Voice agent 是工程 discipline，不是加个 TTS**。**barge-in、STT confidence、TTS cutoff、latency budget** 这 4 件事没一件可以省。
2. **UPSTREAM 是 voice pipeline 的灵魂**。**没有 UPSTREAM = 没有 barge-in = 没有 voice agent**。**这一条是设计时必问的**。
3. **MultimodalAgent vs VoicePipelineAgent 的本质是 control vs speed**。**这是"我要可解释性还是极致速度"的工程权衡**。**没有两者都拿的银弹**。
4. **延迟不是"差不多"，是 budget**。**每一 stage 必有数字**。**链加起来才知道是不是 premium**。**1500ms 以上的 voice agent 是 broken 感**。
5. **Semantic turn detection 是 voice agent 的"判断力"**。**silence-based 太脆，model-based 才是 2026 的标配**。

---

## 十三、横向对比

| 框架 | 抽象 | 延迟 | 控制 | 何时用 |
|---|---|---|---|---|
| **Pipecat** | Frame + Processor | 自定 | **完全** | 自建 / 自定义 stage |
| **LiveKit VoicePipelineAgent** | STT + LLM + TTS cascade | 中 | **高** | WebRTC + text-level 控制 |
| **LiveKit MultimodalAgent** | Audio 直接进出 | **最低** | 低 | 极致速度 |
| **Vapi / Retell** | 托管 | 450-600ms | 中 | 无 WebRTC 团队 |

**统一口诀**：

> **Barge-in 必装，STT confidence 必闸，latency 必算，turn detection 必用 model。**
