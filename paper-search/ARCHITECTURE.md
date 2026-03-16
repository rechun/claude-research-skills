# Research Agent 完整架构设计

> 工程化视角：将论文检索能力模块化，构建可复用、可扩展的研究助手系统

---

## 一、系统设计原则

| 原则 | 说明 |
|------|------|
| **单一职责** | 每个 Skill 只做一件事，边界清晰 |
| **契约驱动** | Skill 间通过 JSON Schema 通信，不依赖自然语言传递 |
| **幂等执行** | 相同输入产生相同输出，支持缓存和重试 |
| **显式降级** | 工具失败时有明确的 fallback 路径 |
| **可观测** | 每步输出可检查，错误可定位 |

---

## 二、分层架构

```
┌─────────────────────────────────────────────────────┐
│                   用户输入层                          │
│  自然语言 / 结构化参数 / 上下文文件                    │
└────────────────────┬────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────┐
│           Research Orchestrator Agent                │
│  - 意图解析 (Intent Parser)                          │
│  - 任务拆解 (Task Decomposer)                        │
│  - Skill 调度 (Skill Dispatcher)                     │
│  - 结果综合 (Result Synthesizer)                     │
└──────┬──────────────┬─────────────────┬─────────────┘
       │              │                 │
┌──────▼──────┐ ┌─────▼──────┐ ┌───────▼──────┐
│paper-search │ │paper-filter│ │paper-extract │
│  Skill      │ │  Skill     │ │  Skill       │
├─────────────┤ ├────────────┤ ├──────────────┤
│关键词扩展   │ │相关性评分  │ │结构化提取    │
│多源搜索     │ │时效过滤    │ │方法分析      │
│结果去重     │ │领域匹配    │ │对比表格生成  │
└──────┬──────┘ └─────┬──────┘ └───────┬──────┘
       │              │                 │
┌──────▼──────────────▼─────────────────▼──────┐
│                  Tool 层                       │
│  web_search · web_fetch · LLM Scorer           │
│  JSON Parser · Citation API                    │
└────────────────────┬──────────────────────────┘
                     │
┌────────────────────▼──────────────────────────┐
│              Synthesis Node                    │
│  去重 · 排序 · 证据链构建 · 摘要生成            │
└──────────┬────────────────────────┬────────────┘
           │                        │
┌──────────▼──────┐    ┌────────────▼──────────┐
│ Research Memory │    │   Output Formatter    │
│ 已检索论文       │    │ Markdown · BibTeX     │
│ 关键词历史       │    │ 表格 · 报告           │
│ 引用图          │    └───────────────────────┘
└─────────────────┘
```

---

## 三、Skill 目录与职责

### `paper-search` Skill
**职责**：将用户意图转化为多源学术检索，返回标准化论文元数据池

**输入**：
```json
{
  "topic": "string",
  "depth": "quick|standard|deep",
  "time_range": {"from": 2022, "to": "now"},
  "filters": {"venue": [], "citation_min": 0}
}
```

**输出**：
```json
{
  "papers": [PAPER_SCHEMA],
  "queries_used": ["string"],
  "sources_searched": ["arxiv", "semantic_scholar"]
}
```

---

### `paper-filter` Skill
**职责**：对论文池进行多维度过滤与排序，输出精选列表

**核心算法**：
```
relevance_score = (
  0.4 * semantic_similarity(query, abstract) +
  0.2 * recency_score(year) +
  0.2 * citation_impact(citations) +
  0.1 * venue_quality(venue) +
  0.1 * has_code_bonus
)
```

---

### `paper-extract` Skill
**职责**：对单篇论文进行深度结构化提取，生成可复用的知识单元

**提取维度**：
- 研究问题 (Research Question)
- 核心方法 (Methodology)
- 实验设置 (Experimental Setup)
- 主要结论 (Key Findings)
- 局限性 (Limitations)
- 未来工作 (Future Work)

---

## 四、Orchestrator 决策逻辑

```python
def orchestrate(user_input: str) -> ResearchReport:

    # Step 1: 意图解析
    intent = parse_intent(user_input)
    # intent.type: "survey" | "specific_paper" | "comparison" | "background"

    # Step 2: 根据意图选择执行路径
    if intent.type == "quick_lookup":
        papers = paper_search(intent, depth="quick")
        return format_list(papers)

    elif intent.type == "survey":
        papers = paper_search(intent, depth="deep")
        filtered = paper_filter(papers, top_k=20)
        extracted = [paper_extract(p) for p in filtered[:5]]  # 精读 top5
        return synthesize_report(filtered, extracted)

    elif intent.type == "comparison":
        papers = paper_search(intent, depth="standard")
        extracted = [paper_extract(p) for p in papers]
        return generate_comparison_table(extracted)

    # Step 3: 检查 Memory，跳过已处理论文
    papers = [p for p in papers if p.id not in memory.known_ids]

    # Step 4: 写入 Memory
    memory.update(papers)

    return result
```

---

## 五、接口契约（Skill 间通信）

所有 Skill 遵循统一的调用格式：

```json
{
  "skill": "paper-search",
  "version": "1.0",
  "params": { },
  "context": {
    "session_id": "string",
    "prior_skills_output": { },
    "user_preferences": { }
  }
}
```

返回格式：

```json
{
  "status": "success | partial | failed",
  "data": { },
  "metadata": {
    "execution_time_ms": 1200,
    "sources_used": ["arxiv"],
    "warnings": ["引用数为估算值"]
  }
}
```

---

## 六、错误处理与降级策略

| 场景 | 降级策略 |
|------|----------|
| Arxiv 搜索失败 | 切换至 Google Scholar |
| 引用数无法获取 | 标注 [未知]，不影响流程 |
| 全文无法访问 | 仅用摘要，标注 [摘要模式] |
| 论文数量不足 | 放宽时间范围，降低相关性阈值 |
| LLM 评分超时 | 使用规则评分替代 |

---

## 七、扩展点设计

本架构预留以下扩展位：

1. **`paper-cite` Skill**：构建引用关系图，发现间接相关工作
2. **`paper-compare` Skill**：生成多论文方法对比矩阵
3. **`hypothesis-generator` Skill**：基于文献 gap 生成研究假设
4. **`venue-monitor` Skill**：订阅特定会议/关键词的新论文推送
5. **外部 MCP 接入**：接入 Zotero / Notion / Obsidian 进行文献管理

---

## 八、设计决策记录 (ADR)

**ADR-001**: 为何将评分逻辑放在 `paper-filter` 而非 `paper-search`？
→ 职责分离。搜索只管"找到"，过滤才管"好不好"。这样 `paper-search` 可被其他 Agent 复用而不带评分逻辑。

**ADR-002**: 为何使用 JSON Schema 而非自然语言传递 Skill 参数？
→ 可靠性。自然语言传递在 Agent 链路中容易产生歧义和遗漏字段。

**ADR-003**: Memory Store 的必要性？
→ 避免重复检索。长对话中用户可能多次问相关话题，Memory 确保增量更新而非全量重搜。
