# 📚 paper-search — Claude Research Skill

一个**工程化**的学术论文检索 Skill，为 Claude.ai 提供可复用的研究助手能力模块。

不是简单的"搜索 Prompt"，而是一个完整的结构化研究流程：关键词扩展 → 多源检索 → 相关性评分 → 分类排序 → 标准化输出。

---

## ✨ 功能特性

- **意图解析**：自动从自然语言中提取检索参数，无需手动填表
- **关键词扩展**：自动生成同义词、上下游概念，覆盖不同表达方式
- **多源检索**：同时搜索 Arxiv、Semantic Scholar、Google Scholar、Papers With Code
- **相关性评分**：综合语义匹配、时效性、引用影响力、会议质量多维度打分
- **结构化输出**：支持列表、表格、综述报告、BibTeX 四种输出格式
- **质量自检**：内置 Checklist，防止捏造引用数和论文 ID

---

## 🚀 安装方法

### 方法一：直接上传（推荐）

1. 下载本仓库中的 [`paper-search/SKILL.md`](./paper-search/SKILL.md) 文件
2. 打开 [Claude.ai](https://claude.ai)
3. 进入 **Settings → Skills**
4. 点击 **Upload Skill**，选择下载的 `SKILL.md` 文件
5. 安装完成后，在任意对话中直接用自然语言触发即可

### 方法二：Clone 整个仓库

```bash
git clone https://github.com/rechun/claude-research-skills.git
```

然后按方法一上传 `paper-search/SKILL.md`。

---

## 💬 使用示例

安装后，直接用自然语言对话即可触发：

**示例 1：快速找论文**
```
帮我找一下最近关于 RAG 的重要论文，要有代码的
```

**示例 2：写综述**
```
我需要写一篇关于 diffusion model for video generation 的 related work，
帮我找近两年的代表性工作，按方向分类整理
```

**示例 3：指定格式**
```
找10篇 transformer interpretability 的论文，输出 BibTeX 格式
```

**示例 4：对比调研**
```
我想比较 GAN、VAE、Diffusion 三种图像生成方法的代表论文，生成对比表格
```

---

## 📋 支持的输出格式

| 格式 | 说明 | 触发关键词 |
|------|------|-----------|
| `list` | 带摘要的论文列表（默认） | 默认 |
| `table` | 标题/年份/引用/相关性表格 | "表格"、"对比" |
| `report` | 结构化综述报告 | "综述"、"survey"、"related work" |
| `bibtex` | 可导入的 `.bib` 文件 | "bibtex"、"参考文献" |

---

## ⚙️ 检索深度控制

| 深度 | 返回数量 | 适用场景 |
|------|----------|---------|
| `quick` | ~5 篇 | 快速了解某个方向 |
| `standard` | ~15 篇 + 摘要 | 一般调研（默认） |
| `deep` | ~30 篇 + 精读 Top 5 | 写综述、开题报告 |

可在对话中直接说明，例如："快速找几篇就行" 或 "帮我做一个深度调研"。

---

## 🏗️ 架构说明

本 Skill 是完整 Research Agent 架构中的检索层模块，详细架构设计见 [`ARCHITECTURE.md`](./paper-search/ARCHITECTURE.md)。

完整架构包含三个协作 Skill：

```
paper-search  →  paper-filter  →  paper-extract
   检索              过滤排序          深度提取
```

目前本仓库提供 `paper-search` 核心模块，`paper-filter` 和 `paper-extract` 持续开发中。

---

## ⚠️ 注意事项

- 需要在 Claude.ai 中**开启 Web Search** 功能（Settings → Features → Web Search）
- 引用数来自 Semantic Scholar，有约 1-4 周延迟，标注 `[估算]` 的为推断值
- Arxiv 预印本未经同行评审，结果中会标注 `[Preprint]`
- 付费数据库（IEEE Xplore、ACM DL）仅能获取元数据，无法访问全文

---

## 🗺️ Roadmap

- [x] `paper-search` — 多源检索 + 结构化输出
- [ ] `paper-filter` — 独立的相关性评分模块
- [ ] `paper-extract` — 单篇论文深度结构化提取
- [ ] `paper-cite` — 引用关系图构建
- [ ] `paper-compare` — 多论文方法对比矩阵

---

## 📄 License

MIT License — 自由使用、修改和分发，保留原始署名即可。

---

## 🤝 Contributing

欢迎提 Issue 和 PR！特别欢迎：
- 新的检索来源（如 PubMed、SSRN）
- 输出格式扩展（如 Notion、Obsidian 格式）
- 特定领域的调优版本（如生物医学、法律、经济学）
