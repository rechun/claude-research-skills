---
name: paper-filter
description: >
  独立的论文相关性评分与精筛模块。当用户拿到一批论文需要筛选、排序、去噪时触发。
  典型场景："这些论文哪些最值得读"、"帮我从这堆结果里挑最相关的"、"按重要性排个序"、
  "去掉不相关的"、"只保留顶会的"、"我只有时间读5篇，帮我选"。
  也作为 paper-search 的下游模块被 Research Orchestrator 自动调用。
  输入可以是论文列表、搜索结果文本、或粘贴的论文标题/摘要集合。
---

# paper-filter Skill

对论文池执行**多维度评分、过滤、分层排序**，输出精选列表。
职责边界：只做筛选排序，不做检索（上游）也不做内容提取（下游）。

---

## 输入格式

接受以下任意格式的论文集合：

```
INPUT_TYPES:
  A. paper-search 输出的结构化 JSON（含 PAPER_SCHEMA 字段）
  B. 用户粘贴的论文列表（标题 + 摘要，自由格式）
  C. Markdown 表格（标题 | 年份 | 摘要）
  D. BibTeX 文件内容
```

同时接收**过滤意图**（从用户输入中推断）：

```
FILTER_INTENT:
  query:         string        # 用户真正关心的研究问题
  top_k:         int           # 保留数量，默认 10
  must_have:     list[str]     # 必须包含的关键词/特征
  must_not:      list[str]     # 必须排除的关键词/特征
  venue_tier:    "top" | "any" # top = 仅 CCF-A / 顶会
  require_code:  bool          # 是否要求有代码
  time_range:    {from, to}    # 时间窗口
  reading_goal:  "overview" | "methodological" | "application" | "sota"
```

---

## 评分引擎

### 维度定义

```
SCORING_DIMENSIONS:

  D1. 语义相关性 (0.00 – 0.40)
      将 query 与 title + abstract 做语义匹配
      - 核心概念直接出现在标题：+0.15
      - 核心概念出现在摘要前两句：+0.10
      - 解决的问题与 query 高度一致：+0.15
      - 仅间接相关（背景/对比）：≤ 0.10

  D2. 时效性 (0.00 – 0.20)
      距今 ≤ 12 个月：0.20
      距今 13–24 个月：0.15
      距今 25–36 个月：0.10
      距今 37–60 个月：0.05
      60 个月以上：0.00（除非引用数 > 500，视为奠基作）

  D3. 引用影响力 (0.00 – 0.20)
      score = min(log(citations + 1) / log(500), 1.0) × 0.20
      若 citations = -1（未知）：按 0.05 计，标注 [估算]

  D4. 发表质量 (0.00 – 0.10)
      CCF-A / NeurIPS / ICML / CVPR / ICLR / ACL / EMNLP 等顶会：0.10
      CCF-B / 次级会议：0.06
      Workshop / 期刊：0.04
      Arxiv 预印本：0.02

  D5. 实现可用性 (0.00 – 0.10)
      有官方代码且可运行：0.10
      有非官方复现：0.06
      无代码：0.00

TOTAL = D1 + D2 + D3 + D4 + D5   # 满分 1.00
```

### 加权调整（根据 reading_goal）

```
GOAL_WEIGHTS:
  overview:        D1×1.0, D2×1.5, D3×1.2, D4×1.0, D5×0.5
  methodological:  D1×1.2, D2×0.8, D3×1.0, D4×1.2, D5×1.5
  application:     D1×1.3, D2×1.2, D3×0.8, D4×0.8, D5×1.2
  sota:            D1×1.0, D2×2.0, D3×0.8, D4×1.0, D5×1.0
```

---

## 执行流程

### Step 1 — 解析输入

将任意格式输入统一转换为内部 PAPER_SCHEMA，缺失字段填 `null`。
若标题或摘要缺失且可通过 `web_fetch` 补全，则尝试获取；否则标注 `[信息不完整]`。

---

### Step 2 — 硬过滤（Hard Filter）

在评分前排除明显不符合要求的论文：

```
HARD_FILTERS (按顺序执行，任一命中即排除):
  1. 年份不在 time_range 内
  2. venue 不符合 venue_tier 要求（若指定）
  3. require_code=true 且 code_url=null
  4. 标题/摘要包含 must_not 关键词
  5. 标题/摘要不含任何 must_have 关键词（若指定）

记录每篇被排除的原因，输出时可选显示。
```

---

### Step 3 — 软评分（Soft Scoring）

对通过硬过滤的论文执行评分，计算 `final_score`：

```python
for paper in filtered_pool:
    raw_score = score(paper, query, dimensions)
    adjusted  = apply_goal_weights(raw_score, reading_goal)
    paper.final_score = round(adjusted, 3)
    paper.score_breakdown = {D1, D2, D3, D4, D5}
```

---

### Step 4 — 分层排序

```
TIER_A (Must Read):    final_score >= 0.70
TIER_B (Should Read):  0.45 <= score < 0.70
TIER_C (Optional):     score < 0.45

若 top_k 被指定，从 Tier A 开始取满为止。
```

---

### Step 5 — 多样性修正

```
DIVERSITY_RULE:
  同一第一作者 → 最多保留 2 篇
  同一研究组（机构 + 相似方向）→ 最多保留 3 篇
  某子方向论文占比 > 40% → 截断，补入其他子方向
```

---

### Step 6 — 输出模板

```markdown
## 论文精选结果

> 输入：{total_input} 篇 | 硬过滤后：{after_hard} 篇 | 最终展示：{top_k} 篇
> 过滤条件：{applied_filters} | 阅读目标：{reading_goal}

### ⭐ Tier A — Must Read ({n} 篇)

1. **{title}** ({year}, {venue})
   综合评分：{final_score} | 相关性 {D1} · 时效 {D2} · 引用 {D3} · 质量 {D4} · 代码 {D5}
   一句话理由：{why_this_paper_matters}
   [PDF]({url})  [代码]({code_url})

### 📖 Tier B — Should Read ({n} 篇)
...

### 📂 Tier C — Optional ({n} 篇)
...

### 被排除的论文（{n} 篇）
| 标题 | 排除原因 |
|------|----------|
```

---

## 质量自检

```
QUALITY_CHECKLIST:
  [ ] 每篇论文的 score_breakdown 有据可查（非随机打分）
  [ ] "一句话理由"来自摘要内容，非捏造
  [ ] 多样性规则已执行，无明显子方向垄断
  [ ] 被排除论文有明确原因记录
  [ ] top_k 数量与用户要求一致
```

---

## 与 Agent 的接口契约

**输入**：
```json
{
  "skill": "paper-filter",
  "params": {
    "papers": [PAPER_SCHEMA],
    "query": "string",
    "top_k": 10,
    "reading_goal": "sota",
    "require_code": false,
    "venue_tier": "any"
  }
}
```

**输出**：
```json
{
  "tier_a": [PAPER_SCHEMA + {"final_score": 0.82, "score_breakdown": {}, "reason": "..."}],
  "tier_b": [...],
  "tier_c": [...],
  "excluded": [{"title": "...", "reason": "超出时间范围"}],
  "metadata": {"total_input": 30, "after_hard_filter": 22}
}
```

---

## 边界与限制

- **不做检索**：输入必须是已有论文集合；若需先检索，使用 `paper-search`
- **评分为估算**：语义相关性基于 LLM 判断，不保证完全客观
- **不读全文**：仅基于标题 + 摘要评分；全文分析请使用 `paper-extract`
