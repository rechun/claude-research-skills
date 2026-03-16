---
name: paper-extract
description: >
  单篇或多篇论文的深度结构化信息提取模块。当用户需要深入理解一篇论文的内容时触发。
  典型场景："帮我精读这篇论文"、"提取这篇文章的核心方法"、"总结一下这篇 paper 的贡献"、
  "帮我看看这篇论文的实验设置"、"这篇文章用了什么数据集"、"把这篇论文的结论整理一下"、
  "读一下这个 arxiv 链接"、用户直接粘贴论文 PDF/摘要/正文时。
  输出可作为 paper-compare 和 paper-cite 的标准化输入。
---

# paper-extract Skill

对单篇论文进行**全维度结构化提取**，生成可复用的知识单元（Knowledge Unit）。
职责边界：深度提取单篇内容，不做检索，不做跨论文对比。

---

## 输入格式

接受以下任意形式：

```
INPUT_TYPES:
  A. Arxiv URL / DOI 链接        → 自动 fetch 摘要页或全文
  B. 用户粘贴的摘要文本
  C. 用户上传的 PDF 内容（文本）
  D. paper-filter 输出中的单篇 PAPER_SCHEMA
  E. 论文标题（尝试搜索获取摘要）
```

同时接收**提取深度**（从用户意图推断）：

```
EXTRACT_DEPTH:
  "abstract"  → 仅摘要层，快速了解（1-2分钟阅读量）
  "standard"  → 标准结构化提取（默认）
  "deep"      → 含方法细节、公式解读、实验分析
```

---

## 提取框架（Knowledge Unit Schema）

以下为完整提取目标，根据 `extract_depth` 决定覆盖哪些字段：

```
KNOWLEDGE_UNIT:

  # ── 基本信息 ──────────────────────────────────────
  meta:
    title:       string
    authors:     list[str]       # 全部作者
    year:        int
    venue:       string
    arxiv_id:    string | null
    doi:         string | null
    code_url:    string | null
    project_url: string | null   # 项目主页

  # ── 研究定位 ──────────────────────────────────────
  positioning:
    problem:     string          # 要解决的核心问题（1句话）
    motivation:  string          # 为什么现有方法不够（1-2句）
    claim:       string          # 论文的核心声明/假设

  # ── 技术贡献 ──────────────────────────────────────
  contributions:
    main:        list[str]       # 3-5条，按重要性排序
    novelty:     string          # 与已有工作的关键差异（1句话）
    type:        "theoretical" | "empirical" | "system" | "survey"

  # ── 方法详情（standard / deep）─────────────────────
  methodology:
    overview:    string          # 方法总体描述（3-5句）
    components:  list[{name, description}]  # 核心模块逐一说明
    key_formulas: list[str]      # （deep 模式）重要公式的文字描述
    architecture: string | null  # 模型/系统架构描述

  # ── 实验设置（standard / deep）─────────────────────
  experiments:
    datasets:    list[{name, description, size}]
    baselines:   list[str]       # 对比的 baseline 方法
    metrics:     list[str]       # 评估指标
    main_results: string         # 主实验结论（2-3句）
    ablations:   list[str]       # （deep）消融实验关键发现

  # ── 结论与影响 ─────────────────────────────────────
  conclusions:
    key_findings: list[str]      # 3-5条核心发现
    limitations:  list[str]      # 作者承认的局限性
    future_work:  list[str]      # 作者指出的未来方向

  # ── 阅读辅助 ───────────────────────────────────────
  reading_aids:
    tldr:        string          # 一句话总结（≤30字）
    eli5:        string          # 用类比解释核心思想
    key_terms:   list[{term, definition}]  # 论文特有术语解释
    read_order:  list[str]       # 建议阅读顺序（针对跳读者）
```

---

## 执行流程

### Step 1 — 获取内容

```
IF input is URL:
    fetch abstract page → extract title, authors, abstract, metadata
    IF extract_depth == "deep":
        attempt fetch full PDF text or HTML version
ELIF input is pasted text:
    identify text type (abstract only / full paper / partial)
ELIF input is title only:
    web_search(title + "arxiv") → fetch top result
```

---

### Step 2 — 分层提取

按 `extract_depth` 决定执行深度：

```
abstract: 填充 meta + positioning + contributions.main + conclusions.key_findings + tldr
standard: 以上 + methodology.overview + experiments + conclusions 全部 + reading_aids
deep:     全部字段，包含 key_formulas + ablations + eli5 + key_terms
```

**提取原则**：
- 所有内容必须来自原文，不得推断或补充作者未提及的信息
- 若某字段无法从现有内容确定，填 `[原文未提及]`，不得捏造
- `tldr` 和 `eli5` 允许用自己的语言表达，但必须忠实原意

---

### Step 3 — 输出渲染

根据用户场景选择输出格式：

#### 标准阅读模式（面向用户）

```markdown
# {title}
> {year} · {venue} · [PDF]({url}) · [代码]({code_url})
> **TL;DR**：{tldr}

## 研究问题
{problem}

**动机**：{motivation}

## 核心贡献
1. {contributions[0]}
2. {contributions[1]}
...

**与已有工作的关键差异**：{novelty}

## 方法概述
{methodology.overview}

**核心模块**：
- **{component.name}**：{component.description}
...

## 实验
- **数据集**：{datasets}
- **对比基线**：{baselines}
- **主要结果**：{main_results}

## 结论与局限
**关键发现**：
- {key_findings}

**局限性**：
- {limitations}

**未来工作**：
- {future_work}

## 关键术语
| 术语 | 含义 |
|------|------|
| {term} | {definition} |
```

#### 结构化 JSON 模式（供下游 Skill 使用）

输出完整 KNOWLEDGE_UNIT JSON，供 `paper-compare` 和 `paper-cite` 消费。

---

## 批量提取模式

当输入为多篇论文时，为每篇独立执行提取，并在末尾附加跨论文速览表：

```markdown
## 批量提取摘要

| 论文 | 核心问题 | 方法类型 | 主要贡献 | 局限性 |
|------|----------|----------|----------|--------|
| {title} | {problem} | {type} | {contributions[0]} | {limitations[0]} |
```

---

## 质量自检

```
QUALITY_CHECKLIST:
  [ ] tldr 不超过 30 字且准确概括论文
  [ ] contributions 条目来自原文，非自行推断
  [ ] limitations 是作者自述，非读者批评
  [ ] 无捏造数据（实验数字、引用等）
  [ ] [原文未提及] 标注规范使用
  [ ] key_terms 仅包含论文特有术语，非通用术语
```

---

## 与 Agent 的接口契约

**输入**：
```json
{
  "skill": "paper-extract",
  "params": {
    "input": "https://arxiv.org/abs/2305.xxxxx",
    "extract_depth": "standard",
    "output_format": "markdown"
  }
}
```

**输出**：
```json
{
  "knowledge_unit": KNOWLEDGE_UNIT,
  "rendered_markdown": "string",
  "metadata": {
    "source_type": "arxiv_url",
    "content_completeness": "abstract_only | full_text",
    "missing_fields": ["key_formulas"]
  }
}
```

---

## 边界与限制

- **付费论文**：仅能提取摘要，全文字段填 `[全文不可访问]`
- **非英文论文**：提取后用中文输出，原文关键术语保留英文
- **不做评价**：本 Skill 只提取，不判断论文好坏；评分由 `paper-filter` 负责
- **不做对比**：单篇提取只描述本文；跨论文对比请使用 `paper-compare`

---

## 示例调用

**用户输入**：`帮我读一下这篇论文 https://arxiv.org/abs/2305.11206`

**执行过程**：
1. fetch arxiv 摘要页，获取标题、作者、摘要、元数据
2. extract_depth 推断为 "standard"
3. 填充所有 standard 层字段
4. 渲染为标准阅读格式输出
5. 附加 JSON 供后续 paper-compare 使用
