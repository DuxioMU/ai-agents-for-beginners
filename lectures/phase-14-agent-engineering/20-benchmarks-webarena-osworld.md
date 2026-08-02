---
chapter: Benchmarks — WebArena and OSWorld
phase: 14-agent-engineering
chapter_dir: phases/14-agent-engineering/20-benchmarks-webarena-osworld
source_doc: phases/14-agent-engineering/20-benchmarks-webarena-osworld/docs/en.md
generated: 2026-08-02
language: zh-CN
---

# 14-20 · 基准测试：WebArena 与 OSWorld

> WebArena tests web-agent capability across four self-hosted apps. OSWorld tests desktop-agent capability across Ubuntu, Windows, macOS. At release (2023–2024) both showed a big gap between best-in-class agents and humans. The gap is narrowing; the failure modes haven't changed.

## 一、本章在整体中的位置

第 14-19 章覆盖了**通用 agent 能力评估**（SWE-bench 测代码、GAIA 测通才、AgentBench 测多环境）。本章切到**两种最具体的 agent 形态**：

> **Web agent**（浏览器里点 20 次完成购物结账）、
> **Desktop agent**（仅用键鼠配置 Linux 桌面）。

**这两类 benchmark 的设计哲学与前 19 章不同** —— 它们测的是**真实操作能力**，不是"模型输出对不对"，而是"状态改对没有"。

---

## 二、WebArena：浏览器里的 long-horizon 任务

### 基本面

- **Zhou et al., ICLR 2024**。
- **812 个 long-horizon 任务** 跨 **4 个自托管 web app**：

| App | 任务场景 |
|---|---|
| 购物站 | 选品 → 加购 → 结算 |
| 论坛 | 发帖、订阅、归档 |
| GitLab-like 工具 | 提 issue、合并 PR |
| 业务 CMS | 编辑发布、版本控制 |

外加：地图、计算器、草稿纸这些"工具"。

### 关键设计：**自托管 + 执行式评估**

- **自托管**：WebArena **不依赖外部网站**，把目标 app 自己跑起来。
  - **好处**：测试装置不抖动（无网络变化、无目标站改版）。
  - **坏处**：app 版本被钉死（必须重新选过才能比较）。
- **执行式评估（execution-based）**：通过 gym API 检查**实际状态** —— 订单下成功了吗？issue 关了吗？CMS 页面更新了吗？**不是"模型说完成了"**。

**这是和"主观打分"或"字符串匹配"评估的根本区别**。

### 当时数字

- 最佳 GPT-4 agent：**14.41% 成功率**
- 人类：**78.24%**
- **差距巨大**，且**这一差距在收窄**（Claude computer use、OpenAI CUA、Gemini 都在追）。

### 三个扩展

| 扩展 | 加了什么 |
|---|---|
| **VisualWebArena** | **视觉任务**（截图是一等观察）—— 测"看图点"能力 |
| **TheAgentCompany**（2024-12） | 加 terminal + coding → 更接近远程办公场景 |

**WebArena → VisualWebArena** 是"DOM-based → screenshot-based"的位移；**TheAgentCompany** 是"web only → 全工作环境"。

---

## 三、OSWorld：真实桌面跨平台

### 基本面

- **Xie et al., NeurIPS 2024**。
- **369 个真实桌面任务** 跨 **Ubuntu / Windows / macOS**。
- 自由键鼠控制真实应用。
- 观察：**1920×1080 截图**。

### 关键设计：**真截图，不用 accessibility API**

> **OSWorld uses real OS screenshots instead of accessibility APIs.**

| 设计 | 后果 |
|---|---|
| **用真截图** | 强制模型学 GUI grounding（像素 → 元素映射）；逼真、跨应用通用 |
| **不用 a11y API** | 不被"应用有什么 tree"绑架；任何应用都能跑（包括没 a11y 的） |

**这个设计选择 = 真生产**：computer use agent 进用户桌面就是看屏幕，不是 tree 行走。**OSWorld 直接对准真实工作负载**。

### 当时数字

- 最佳模型：**12.24%**
- 人类：**72.36%**

> *The gap is narrowing; the failure modes haven't changed.*

**这一句话是本章的题眼**：分数在涨，但**失败模式没变**。

---

## 四、OSWorld 的两个根本性失败模式

### 1. **GUI grounding**（像素 → 元素映射）

> "Pixel → element mapping. Models struggle to localize UI elements reliably in 1920×1080."

1920×1080 截图里找出"那个按钮在哪"是**基础视觉定位问题**。模型不是不会思考，是**找不到东西在哪**。

### 2. **Operational knowledge**（OS know-how）

> "Which menu has the setting, which keyboard shortcut, which preference pane. Knowledge tail that humans build over years."

**人类花多年积累的"系统操作肌肉记忆"**：

- macOS 的"系统设置 → 通用 → 登录项"在哪
- Ubuntu 的 `apt` 装完东西要去 `/etc/profile.d/` 改环境
- Windows 的"显示设置 → 缩放 → 高级缩放设置"有几次跳转

**这些知识在训练语料里稀薄（频率低、长尾），模型就是不会**。

### 失败模式重要性

**这两个失败模式是结构性的**。**分数涨是因为工具越来越强、知识越来越丰富，但结构没变**。**这就是为什么不能光看"成功率上升"就误判已解决**。

---

## 五、两个有用的扩展

### OSWorld-G（grounding 套件）

- **564 样本的 grounding-only 集** + Jedi 训练集。
- **关键作用**：**把 grounding 和 planning 拆开测**。你要分开看"是定位失败还是规划失败"才能针对性优化。

### OSWorld-Human（人工 gold 轨迹）

- **人工挑选的金标准动作轨迹**。
- **惊人发现**：**top agent 用 1.4–2.7 倍于人需要的步骤**。**trajectory efficiency gap**。

> 成功率 50% 但步骤数 2.5x = 你以为"对了一半"，但每次成功都"绕了 2.5 圈"。

---

## 六、为什么这一章和现实强相关

> **Claude computer use, OpenAI CUA, Gemini 2.5 Computer Use (Lesson 21) all train on workloads shaped by WebArena and OSWorld. The benchmarks are the target; the production models are the shipped answer.**

**这三个 production 模型的训练数据形态被 WebArena / OSWorld 决定**。**Benchmark 不仅是"考卷"，它是训练目标**。**这一条是商业和技术上都要严肃理解的事实**。

---

## 七、本章的失败模式（明文警告）

1. **Screenshot-only evals** —— OSWorld 是截图驱动；用 DOM / a11y 评估一个 agent 在 OSWorld 上"过"了，**你错失了对 grounding 挑战的覆盖**。
2. **Ignoring trajectory length** —— 只看成功率不看步骤数，**错过 OSWorld-Human 暴露的 1.4–2.7x 效率差距**。
3. **Stale self-hosted apps** —— WebArena 的 app 版本被钉死；不重新校准就**不可比**。

---

## 八、代码（`code/main.py`）速览

`code/main.py` 实现一个 toy web-agent harness：

```python
# 最简"购物 app" 状态机
state_machine = {
    "list_items": ...,
    "add_to_cart": ...,
    "checkout": ...,
}

# 3 个 gold trajectory
gold_trajectories = [...]

# scripted agent 尝试每个任务
def scripted_agent(task): ...

# 执行式 evaluator（state check）
def evaluate(state, goal): ...   # "订单真的下了吗？"

# 轨迹效率指标
def trajectory_efficiency(steps, gold): ...   # steps / gold
```

运行后输出：
- 每任务成功率
- 轨迹效率（步数比）
- 镜像 OSWorld-Human 的方法论

**这一章的代码和 14-19 一样：让你看清"评估装置在做什么"**。

---

## 九、Use It：决策树

```
你在评估 web/desktop agent？
  ├─ 浏览器内 long-horizon 任务 → WebArena（自托管）
  ├─ 视觉任务（看截图点） → VisualWebArena
  ├─ 跨 OS 真实桌面 → OSWorld（VM fleet）
  ├─ 只想测 grounding → OSWorld-G
  ├─ 想测"步骤效率" → OSWorld-Human
  └─ 你的产品 flow → 抓你自己的 top 20 任务 gold trajectory，每周跑
```

**这条最后一项是工程上的金科玉律**：

> **Capture gold trajectories for your top 20 tasks; run agents against them weekly.**

**这是把"benchmark 思路"应用到"你自家产品"的具体方法**。

---

## 十、常见踩坑（本章明文警告）

1. **Screenshot-only evals** —— 别假定 DOM / a11y 是金标准；OSWorld 暴露的 grounding 挑战你用 DOM 是看不到的。
2. **Ignoring trajectory length** —— 1.4–2.7x 步骤差是真实成本。
3. **Stale self-hosted apps** —— 升级 app 版本必须重新校准。

---

## 十一、课后练习 5 题核心思路

1. **扩 toy harness 到第二个 app**（论坛）：3 任务 + gold trajectory。**这是"我把 benchmark 思想用上"的实践**。
2. **加 trajectory efficiency 报告**：跑出 1x / 2x / 3x over gold。**生产意义直接对应"agent 贵不贵"**。
3. **加 "distractor tool"**（gold 永远不用的工具）：看 scripted agent 是否被吸引。**这是测"agent 抵抗噪音"能力的雏形**。
4. **读 OSWorld-G，分离 grounding vs planning 失败**：你的 eval 里这两种失败被混淆吗？**这是测度设计本身的反省**。
5. **读 WebArena app README，看升级破坏什么**：版本对齐是 benchmark 工程的暗物质。

---

## 十二、本章给我的工程启示

1. **"执行式评估"是 browser/desktop agent 的第一原则**。不是"模型说完成了"，是"状态真的改对了吗"。**这条原则延伸到任何"agent 操作世界"的场景**。
2. **GUI grounding 是结构性问题，不靠"更强的模型"自动解决**。**模型增强能减低失败率但失败模式不变**。要专门优化（数据集 / 工具）才能突破。
3. **Operational knowledge 是长尾、不可被简单 scaling law 覆盖**。**未来需要"user-personalized operational memory"**（记住你 Mac 上的偏好设置）。
4. **Trajectory efficiency 是被严重低估的 KPI**。**2.5x 步数 = 2.5x token = 2.5x 成本**。**只追成功率会让账单偷偷涨**。
5. **Benchmark 是训练目标**。**WebArena / OSWorld 不仅考模型，它们塑造了模型的训练数据形态**。**你的产品评估也会塑造你的 agent 训练方向**。

---

## 十三、横向对比

| Benchmark | 测什么 | 任务数 | 评估方式 | 失败模式 |
|---|---|---|---|---|
| **WebArena** | 浏览器 long-horizon | 812 | **执行式（gym API）** | 长链规划、工具使用 |
| **VisualWebArena** | + 视觉任务 | (ext) | 同上 | **screenshot grounding** |
| **OSWorld** | 跨 OS 桌面 | 369 | **state check** | **GUI grounding + op knowledge** |
| **OSWorld-G** | grounding only | 564 | state check | 像素 → 元素 |
| **OSWorld-Human** | 轨迹效率 | gold | steps/gold | **1.4–2.7x 步数浪费** |

**统一口诀**：

> **WebArena 看 web 状态改没改，OSWorld 看桌面状态改没改；都看 trajectory efficiency，不只看成功率。**
