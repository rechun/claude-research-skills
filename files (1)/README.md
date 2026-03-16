# 📚 Claude Research Skills

一套**工程化**的学术研究 Skill 合集，为 Claude.ai 提供完整的研究助手能力模块。

五个 Skill 各司其职、可独立使用、也可协同构成完整的 Research Agent 流水线：

```
paper-search → paper-filter → paper-extract → paper-cite → paper-compare
   检索           过滤排序        深度提取       引用图谱       横向对比
```

---

## 📦 Skill 目录

| Skill | 功能 | 典型触发 |
|-------|------|----------|
| [`paper-search`](./paper-search/) | 多源学术检索，关键词扩展，结构化输出 | 「帮我找关于 RAG 的论文」 |
| [`paper-filter`](./paper-filter/) | 相关性评分，多维过滤，分级标注 | 「筛出最重要的几篇」 |
| [`paper-extract`](./paper-extract/) | 单篇深度解析，知识卡片生成 | 「帮我读这篇论文」 |
| [`paper-cite`](./paper-cite/) | 引用关系图，研究脉络分析 | 「这个方向怎么发展的」 |
| [`paper-compare`](./paper-compare/) | 多论文横向对比，生成对比矩阵 | 「比较这几种方法的优劣」 |

---

## 🚀 安装方法

每个 Skill 独立安装，按需选用：

1. 进入对应 Skill 目录，下载 `SKILL.md` 文件
2. 打开 [Claude.ai](https://claude.ai) → **Settings → Skills**
3. 点击 **Upload Skill**，上传 `SKILL.md`
4. 重复以上步骤安装其他 Skill

> **推荐**：安装全部五个，Claude 会根据对话自动选择合适的 Skill 组合调用。

> **前提**：需在 Settings → Features 中开启 **Web Search** 功能。

---

## 💬 使用示例

### 独立使用

**快速找论文**（触发 `paper-search`）
```
帮我找一下最近关于 diffusion model for video generation 的重要论文
```

**筛选过滤**（触发 `paper-filter`）
```
刚才找到的 20 篇里，只保留 2022 年后、顶会发表、有代码的
```

**读懂一篇**（触发 `paper-extract`）
```
帮我读一下 https://arxiv.org/abs/1706.03762，重点是架构和实验结果
```

**梳理脉络**（触发 `paper-cite`）
```
BERT 到 GPT-4 是怎么一步步发展的？帮我梳理引用脉络
```

**方法对比**（触发 `paper-compare`）
```
比较 LoRA、Adapter、Prefix Tuning 三种方法，生成对比表格，我要选基线
```

### 组合使用（Research Agent 模式）

安装全部 Skill 后，可以直接说：

```
我要写一篇关于「参数高效微调」的综述，帮我：
1. 找近三年的代表性论文
2. 筛出最重要的 15 篇
3. 梳理发展脉络
4. 生成方法对比矩阵
```

Claude 会自动按顺序调用各 Skill 完成完整流程。

---

## 🏗️ 架构设计

完整架构文档见 [`paper-search/ARCHITECTURE.md`](./paper-search/ARCHITECTURE.md)。

```
┌──────────────────────────────────────────┐
│         Research Orchestrator            │
│   意图解析 · 任务拆解 · Skill 调度        │
└────┬──────────┬──────────┬───────────────┘
     │          │          │
  paper-     paper-     paper-
  search     filter     extract
     │          │          │
     └──────────┴──────────┘
                │
           paper-cite
           paper-compare
                │
         综合报告输出
```

---

## 📋 各 Skill 详细说明

### `paper-search`
**功能**：将用户意图转化为多源学术检索，返回标准化论文元数据。

- 自动扩展关键词（同义词/上游概念/下游应用）
- 同时搜索 Arxiv、Semantic Scholar、Google Scholar、Papers With Code
- 支持 `quick` / `standard` / `deep` 三档检索深度
- 输出格式：列表 / 表格 / 综述报告 / BibTeX

### `paper-filter`
**功能**：对论文池进行多维评分，输出精选子集。

- 五维评分：语义相关性(40%) + 时效性(20%) + 引用影响力(20%) + 会议质量(10%) + 可复现性(10%)
- 硬过滤：撤稿论文、超出时间范围、低引用等自动排除
- 输出分级标签：⭐⭐⭐ Must-Read / ⭐⭐ Recommended / ⭐ Worth-Noting
- 附过滤报告：保留/排除数量及原因

### `paper-extract`
**功能**：单篇论文深度结构化提取，生成知识卡片。

- 支持输入：PDF 附件 / Arxiv 链接 / DOI / 标题 / 摘要文本
- 提取 30+ 字段：研究问题、方法组件、实验结果、局限性、未来工作等
- 四档深度：`tldr` / `standard` / `deep` / `code_focus`
- 支持批量提取（多篇同时处理）
- 输出可直接传入 `paper-compare`

### `paper-cite`
**功能**：构建引用关系网络，揭示研究脉络。

- 上游扩展（找奠基作）+ 下游扩展（找后续影响）
- 识别 Hub 论文（被大量引用）和 Bridge 论文（连接子领域）
- 输出主脉络链 + 推荐阅读顺序
- 支持 `shallow` / `standard` / `deep` / `timeline_only` 四档深度
- JSON 输出可用于外部可视化工具

### `paper-compare`
**功能**：多篇论文系统性横向对比。

- 三层对比维度：通用维度 / 方法论维度 / 实验维度
- 特殊模式：同一任务多方法 / 纵向演进 / 基线选择 / 优缺点速查
- 可生成 Related Work 写作草稿
- 输出：对比矩阵 + 差异分析 + 场景推荐

---

## ⚠️ 通用注意事项

- 所有 Skill 均需开启 Claude.ai 的 **Web Search** 功能
- 引用数来自 Semantic Scholar，有约 1-4 周数据延迟
- 付费数据库（IEEE、ACM）仅能获取元数据
- Arxiv 预印本未经同行评审，结果中会标注 `[Preprint]`
- 数值结果均来自原文，Skill 不做推断或估算

---

## 🗺️ Roadmap

- [x] `paper-search` — 多源检索 + 结构化输出
- [x] `paper-filter` — 独立的相关性评分模块
- [x] `paper-extract` — 单篇论文深度结构化提取
- [x] `paper-cite` — 引用关系图构建
- [x] `paper-compare` — 多论文方法对比矩阵
- [ ] `venue-monitor` — 订阅特定会议/关键词新论文推送
- [ ] `paper-write` — 基于文献自动生成 Related Work 初稿
- [ ] `hypothesis-gen` — 基于文献 gap 生成研究假设

---

## 📄 License

MIT License — 自由使用、修改和分发，保留原始署名即可。

---

## 🤝 Contributing

欢迎提 Issue 和 PR！特别欢迎：
- 新的检索来源（PubMed、SSRN、法律/经济学数据库）
- 输出格式扩展（Notion、Obsidian、Zotero 格式）
- 特定领域调优版本（生物医学、法律、社会科学）
- 多语言支持改进
