---
name: paper-search
description: >
  可复用的学术论文检索能力模块。当用户需要查找、筛选、整理学术论文，或进行文献综述、
  研究背景调研、技术方案对比时，必须触发此 Skill。关键词包括但不限于：
  "帮我找论文"、"最新研究"、"文献综述"、"相关工作"、"survey"、"state of the art"、
  "找一下 XXX 方向的 paper"、"这个领域有什么代表性工作"。
  此 Skill 不是单次搜索，而是一个结构化的研究流程，输出标准化的论文列表和摘要报告。
---

# paper-search Skill

一个**工程化、可组合**的学术论文检索能力模块，设计目标是让每次检索都可复现、可评估、
可接入上下级 Agent。

---

## 使用前：明确检索意图

在执行任何搜索前，先从用户输入中提取以下结构化信息（可隐式推断，不必逐一追问）：

```
INTENT_SCHEMA:
  topic:        string          # 核心研究主题，e.g. "diffusion model for video generation"
  subtopics:    list[string]    # 子方向，e.g. ["temporal consistency", "motion prior"]
  time_range:   {from: YYYY, to: YYYY | "now"}   # 默认: 过去 3 年
  venue_filter: list[string]    # 可选, e.g. ["NeurIPS", "CVPR", "ICML"]
  citation_min: int             # 可选，最低引用数，默认 0
  depth:        "quick" | "standard" | "deep"
                # quick=5篇概览, standard=15篇+摘要, deep=30篇+全文精读
  output_format: "list" | "table" | "report" | "bibtex"
```

若用户输入不完整，使用合理默认值并在输出头部注明假设。

---

## 执行流程（标准 Pipeline）

### Step 1 — 关键词扩展

将 `topic` 转换为多组检索词，覆盖不同表达方式：

```
QUERY_EXPANSION:
  primary:    原始关键词（英文学术表达）
  synonyms:   同义词变体，e.g. "text-to-video" → ["T2V", "video synthesis from text"]
  upstream:   上游概念，用于背景论文
  downstream: 下游应用，用于应用场景论文
  ablation:   对比基线关键词，e.g. "GAN-based video generation"
```

**原则**：宁可多搜再过滤，不要漏掉核心工作。

---

### Step 2 — 多源检索

按优先级使用以下来源（每源独立搜索，结果合并去重）：

| 优先级 | 来源 | 适用场景 |
|--------|------|----------|
| P0 | Semantic Scholar API | 综合覆盖，有引用图 |
| P0 | Arxiv (abs search) | 最新预印本，CS/ML 必查 |
| P1 | Google Scholar (web_search) | 跨学科，非 CS 领域 |
| P1 | ACM DL / IEEE Xplore | 工程类、系统类论文 |
| P2 | Papers With Code | 带代码实现的论文 |

**执行方式**（使用 `web_search` + `web_fetch` 工具）：

```
FOR each query IN QUERY_EXPANSION:
  FOR each source IN [arxiv, semantic_scholar, google_scholar]:
    results = web_search(f"site:{source.domain} {query}")
    parsed  = parse_search_results(results)
    pool.add(parsed)

pool = deduplicate(pool, key="title+authors")
```

---

### Step 3 — 结构化解析

对每篇论文提取以下字段（优先从摘要/标题推断，必要时 fetch 全文）：

```python
PAPER_SCHEMA = {
  "id":           str,   # arxiv_id 或 DOI
  "title":        str,
  "authors":      list[str],   # 最多取前5位
  "year":         int,
  "venue":        str,         # 会议/期刊，无则填 "Arxiv"
  "citations":    int,         # 引用数，无法获取填 -1
  "abstract":     str,         # 原文摘要，不截断
  "contributions": list[str],  # 3-5条，从摘要中提取
  "methods":      list[str],   # 核心技术手段
  "datasets":     list[str],   # 使用的数据集
  "code_url":     str | None,
  "relevance_score": float,    # 0.0-1.0，见 Step 4
  "tags":         list[str],   # 自动打标，e.g. ["survey", "SOTA", "baseline"]
}
```

---

### Step 4 — 相关性评分

对每篇论文计算 `relevance_score`，综合以下维度：

```
RELEVANCE_RUBRIC:
  topic_match:     0-0.4   # 标题/摘要与 topic 的语义匹配程度
  recency:         0-0.2   # 近1年=0.2, 近3年=0.1, 更早=0
  citation_impact: 0-0.2   # log(citations+1) / log(max_citations+1) * 0.2
  venue_quality:   0-0.1   # 顶会=0.1, 普通会议=0.05, Arxiv=0.02
  has_code:        0-0.1   # 有代码=0.1, 无=0

THRESHOLD:
  include if relevance_score >= 0.3   (quick: 0.5, deep: 0.2)
```

---

### Step 5 — 分类与排序

按以下维度对论文分组，便于用户快速定位：

```
CATEGORIES:
  - "奠基工作 (Foundational)"   : year < 3年前 AND citations > 100
  - "当前 SOTA"                 : 近2年 AND relevance >= 0.7
  - "Survey / 综述"             : title contains survey/review/overview
  - "方法论 (Methodological)"   : 提出新模型/算法
  - "应用导向 (Application)"    : 聚焦特定场景
  - "数据集 / Benchmark"        : 主要贡献是数据或评测
```

---

### Step 6 — 输出生成

根据 `output_format` 选择对应模板：

#### `list` 模式（默认）

```
## 论文检索结果：{topic}

> 检索时间：{date} | 共找到 {total} 篇，展示 Top {n} 篇
> 检索假设：{any_assumptions}

### 奠基工作
1. **{title}** ({year}, {venue})
   - 作者：{authors}
   - 引用：{citations} | 相关性：{score}
   - 核心贡献：{contributions[0]}
   - [PDF]({url}) {[代码]({code_url}) if code}

...（按分类列出）

### 检索关键词
使用了以下查询：{queries_used}
```

#### `table` 模式

输出 Markdown 表格，列：标题 | 年份 | 会议 | 引用 | 相关性 | 代码

#### `report` 模式

生成结构化综述报告，包含：背景介绍 → 主要方向 → 方法对比 → 开放问题

#### `bibtex` 模式

输出可直接导入的 `.bib` 格式条目

---

## 输出质量控制

执行完成后，自检以下条目：

```
QUALITY_CHECKLIST:
  [ ] 结果数量符合 depth 参数要求
  [ ] 所有 arxiv_id / DOI 可验证（非捏造）
  [ ] 引用数来自可靠来源或标注为估算
  [ ] 无重复论文（按标题+第一作者去重）
  [ ] 每篇论文的 contributions 来自原文而非推断
  [ ] 检索关键词已记录（可复现）
```

如有字段无法确认（如引用数），显式标注 `[未知]` 或 `[估算]`，**不得捏造数据**。

---

## 与上级 Agent 的接口契约

当被 Research Orchestrator 调用时，本 Skill 接受以下输入格式：

```json
{
  "skill": "paper-search",
  "params": {
    "topic": "string",
    "subtopics": ["string"],
    "time_range": {"from": 2022, "to": "now"},
    "depth": "standard",
    "output_format": "list"
  },
  "context": {
    "prior_searches": ["已搜关键词列表"],
    "known_papers": ["已知论文 ID，跳过这些"]
  }
}
```

返回格式为结构化 JSON（内部传递）或 Markdown（面向用户）。

---

## 边界与限制声明

- **不做全文分析**：本 Skill 仅处理摘要层信息；全文精读由 `paper-extract` Skill 负责
- **不保证完整性**：部分付费库（IEEE、ACM）仅能获取元数据
- **引用数可能滞后**：Semantic Scholar 数据有 1-4 周延迟
- **预印本未经同行评审**：Arxiv 论文标注 `[Preprint]`

---

## 示例调用

**用户输入**：`帮我找一下最近关于 RAG 的重要论文，要有代码的`

**Skill 执行**：
1. topic = "Retrieval-Augmented Generation", depth = "standard"
2. 扩展关键词：["RAG", "retrieval augmented LLM", "dense retrieval for QA"]
3. 搜索 Arxiv + Semantic Scholar + Papers With Code
4. 过滤 `has_code = true`，按相关性排序
5. 输出 list 格式，高亮代码链接
