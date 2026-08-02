# 课程讲解（Lectures）

本目录存放对 `phases/` 各章节的**逐章讲解**，由 Claude 在每次讲解后自动写入。

> 路径约定：原文位于 `phases/<phase>/<chapter>/docs/en.md`；讲解位于本目录下对应位置的 `.md` 文件。

## 目录结构

```
lectures/
├── README.md                          # 本文件
└── <phase-name>/                      # 与 phases/ 下的 phase 目录同名
    ├── <chapter-full-slug>.md         # 与 phases/<phase>/<chapter>/ 同名 + .md
    └── ...
```

例如：

```
lectures/
└── phase-14-agent-engineering/
    ├── 11-planning-htn-and-evolutionary.md
    ├── 12-anthropic-workflow-patterns.md
    ├── 14-autogen-actor-model.md
    └── 15-crewai-role-based-crews.md
```

## 文件格式

每个讲解文件包含两部分：

1. **YAML frontmatter**（元数据）
   - `chapter`：原始章节英文标题
   - `phase`：所属 phase
   - `chapter_dir`：原始章节路径（`phases/<phase>/<chapter>`）
   - `source_doc`：原始章节文档路径
   - `generated`：讲解生成日期（ISO 8601）
   - `language`：讲解语言（`zh-CN`）

2. **正文**：用中文撰写的逐节讲解，按"概念 → 关键论点 → 工程启示 → 与前后章节关系"展开。
   - 章节内引用一律用原始 `en.md` 章节小标题作为锚点，便于交叉检索。

## 自动写入规则

- 用户每次请求"讲解 `<phase>-<chapter>`"或等价表述时，**默认自动写入**对应路径的 `.md` 文件，并在对话中告知用户。
- 文件已存在时，**追加本次讲解为附录**而非覆盖；如需重写，单独说明。
- 章节在 `phases/` 下尚未存在时，先确认路径再写，避免凭空创建错位。

## 命名原则

- **顶层目录**：`lectures/`（英文，与现有 `phases/`、`book/`、`glossary/` 风格一致）。
- **phase 子目录**：直接复用 `phases/` 下同名目录，不带 `00-` 前缀。
- **章节文件**：直接复用 `phases/<phase>/<chapter>/` 目录名 + `.md`。

这样在 shell 里可以一行命令做交叉检索：

```bash
diff lectures/phase-14-agent-engineering/11-planning-htn-and-evolutionary.md \
     phases/14-agent-engineering/11-planning-htn-and-evolutionary/docs/en.md
```

## 不放什么

- 不放原文 `en.md` 的副本或翻译（避免与 `book/` 重复）。
- 不放临时调试笔记（使用 `outputs/`）。
- 不放代码实验结果（已有 `phases/*/outputs/` 与 `notebook/`）。
