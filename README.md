# LLM 垂直知识库 / LLM Vertical Knowledge Base

这个仓库是一个实验：如何让 LLM 帮助人长期应用某个垂直领域的知识，而不是只在一次问答里复述资料。

The repository is an experiment in using LLMs to maintain and activate domain knowledge over time, instead of treating notes as one-off summaries.

## 中文说明

### 为什么做

人在学习中会积累大量笔记、书籍框架、研报和个人思考，但很多知识长期闲置后会失去可调用性。这个项目想解决的问题是：当某个现实问题出现时，LLM 能不能把过去沉淀的领域知识重新唤醒，并按专业框架给出解释、判断和下一步行动。

这里的目标不是把文件夹变得更整齐，也不是把几本书塞进 RAG。更核心的是建立一个可维护的垂直知识库，让材料从“读过”变成“能被再次使用”。

### 基本哲学

本项目把知识库分成三层：

1. 第一层：原始材料。包括研报、原文、案例、回测输出等底层资料。公开仓库不包含这类第三方原始文件。
2. 第二层：类型化整理。包括书籍笔记、方法笔记、策略实验等经过一次压缩的材料。
3. 第三层：二次重组。不是按原书目录排列，而是把多个来源拆开后重新组织成一个领域判断系统。

LLM 的工作方式应当是渐进式披露：先读第三层主入口；如果信息不足，再查索引；仍不足时，才下钻到第二层或本地第一层材料。

### 仓库内容

- `skill/llm-wiki-workspace/`: 用于维护文件夹 wiki 和垂直知识库的 Codex skill。
- `技术分析/`: 一个技术分析垂直知识库样例，包含第三层主体、第二层材料和来源索引。

本仓库刻意不放 `examples/` 文件夹，因为“技术分析”本身就是当前的真实专题，而不是展示用样例。

### 使用方式

进入某个专题时，建议按这个顺序阅读：

1. 专题内的 `AGENTS.md`
2. 第三层主体文件
3. 索引文件
4. 第二层类型化材料
5. 本地保存的第一层原始材料

维护时，优先强化第三层主体，而不是不断制造平行摘要。只有当某份材料有独立复用价值时，才升级为新的第二层笔记。

### 声明

本项目仅用于研究、学习和知识管理实验。`技术分析/` 中的内容不构成投资建议，也不保证任何交易策略有效。

## English

### Purpose

People accumulate books, notes, reports, and personal research, but dormant knowledge is hard to reuse when a real situation appears later. This project explores whether an LLM can help reactivate that knowledge and apply it through a durable domain framework.

The goal is not merely tidying folders or stuffing books into retrieval. The goal is to turn notes into a maintainable vertical knowledge base that can support interpretation, judgment, and future action.

### Three-Layer Model

1. Layer 1: source material. Reports, originals, cases, backtest outputs, and other bottom-layer materials. Third-party source files are not included in this public repository.
2. Layer 2: type-organized notes. Book notes, method notes, and self-built experiments that have already been compressed once.
3. Layer 3: recomposed domain knowledge. Materials are reorganized around the domain's logic rather than the original table of contents.

The intended LLM behavior is progressive disclosure: read the layer-3 main note first, then consult the index, then drill into layer-2 or local layer-1 material only when needed.

### Repository Layout

- `skill/llm-wiki-workspace/`: a Codex skill for folder wiki maintenance and vertical knowledge base construction.
- `技术分析/`: a real technical-analysis vertical knowledge base, including the layer-3 body, layer-2 notes, and a source index.

There is no `examples/` directory. The Chinese `技术分析` folder is the working topic itself.

### License

Code and skill harness files are released under the MIT License. Notes and knowledge-base content are released under CC BY-NC-SA 4.0 unless otherwise stated. See `LICENSE` and `LICENSE-CONTENT.md`.

