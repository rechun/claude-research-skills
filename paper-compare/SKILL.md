---
name: paper-compare
description: >
  多论文方法对比矩阵生成模块。当用户需要横向比较多篇论文的方法、性能、设计选择时触发。
  典型场景："对比一下这几篇论文的方法"、"帮我做一个方法对比表"、"这几个方法有什么区别"、
  "哪个方法在 XX 数据集上表现最好"、"各方法的优缺点是什么"、"帮我整理 related work 的对比"、
  "这些论文用的什么 backbone"、"生成一个 comparison table"。
  输入为 2 篇以上论文，输出为结构化对比矩阵 + 分析报告。
---

# paper-compare Skill

对多篇论文执行**系统性横向对比**，生成结构化对比矩阵与差异分析报告。
职责边界：跨论文对比分析，不做单篇提取（上游：`paper-extract`），不做引用图（`paper-cite`）。

---

## 输入格式

```
INPUT_TYPES:
  A. 多篇论文的 KNOWLEDGE_UNIT（来自 paper-extract 输出）
  B. 用户提供的论文列表（URL / 标题，自动调用 paper-extract）
  C. 用户粘贴的多篇摘要文本

COMPARE_FOCUS（从用户意图推断）:
  "method"      → 技术方法对比（架构、算法、创新点）
  "performance" → 实验结果对比（指标、数据集、SOTA）
  "design"      → 设计选择对比（假设、trade-off、适用场景）
  "full"        → 全维度对比（默认，涵盖以上三个）
```

---

## 对比维度框架

根据 `compare_focus` 选择激活哪些维度：

```
COMPARISON_DIMENSIONS:

  # ── 方法维度（method / full）──────────────────────
  M1. 核心技术路线      (Paradigm)
      e.g. autoregressive / diffusion / GAN / transformer-based

  M2. 模型架构          (Architecture)
      backbone、关键模块、参数量级

  M3. 训练策略          (Training)
      预训练/微调、损失函数、数据增强

  M4. 关键创新点        (Innovation)
      相较已有工作新增了什么

  M5. 方法局限性        (Limitation)
      作者承认的约束或场景限制

  # ── 性能维度（performance / full）────────────────
  P1. 使用数据集        (Datasets)
  P2. 评估指标          (Metrics)
  P3. 主要性能数字      (Results)
      注：仅记录原文报告的数字，不做换算或推断
  P4. 与 baseline 的差距 (Gain over baseline)
  P5. 计算成本          (Efficiency)
      训练时间、推理速度、GPU 需求

  # ── 设计维度（design / full）──────────────────────
  D1. 核心假设          (Assumptions)
      方法成立的前提条件

  D2. 适用场景          (Use Case)
      最适合哪种场景/数据类型

  D3. 可复现性          (Reproducibility)
      代码可用性、超参数透明度

  D4. 与其他方法的兼容性 (Composability)
      能否作为即插即用模块

  D5. 作者评价          (Self-assessment)
      论文中作者如何定位自己的工作
```

---

## 执行流程

### Step 1 — 输入标准化

```
FOR each paper in input:
    IF type is URL/title:
        knowledge_unit = invoke paper-extract(paper, depth="standard")
    ELIF type is KNOWLEDGE_UNIT:
        use directly
    ELIF type is raw_text:
        parse into partial KNOWLEDGE_UNIT

papers = [knowledge_unit_1, knowledge_unit_2, ...]
assert len(papers) >= 2, "至少需要2篇论文"
```

---

### Step 2 — 维度对齐

不同论文可能使用不同的评测指标或数据集，对齐前需要：

```
ALIGNMENT_RULES:
  指标对齐：若两篇论文使用不同指标，标注为 [不可直接对比]
  数据集对齐：仅对在相同数据集上的结果做横向比较
  时间对齐：标注每篇论文的发表年份，避免拿新论文打旧论文
  字段缺失：若某论文某维度信息缺失，填 [原文未提及]，不猜测
```

---

### Step 3 — 矩阵生成

生成两种粒度的对比表：

#### 速览矩阵（Quick Matrix）

所有论文 × 核心维度，一张表看完主要差异：

```markdown
| 维度 | {Paper A} | {Paper B} | {Paper C} |
|------|-----------|-----------|-----------|
| 技术路线 | {M1} | {M1} | {M1} |
| 核心创新 | {M4} | {M4} | {M4} |
| 主要数据集 | {P1} | {P1} | {P1} |
| 关键指标 | {P3} | {P3} | {P3} |
| 有无代码 | ✓/✗ | ✓/✗ | ✓/✗ |
| 适用场景 | {D2} | {D2} | {D2} |
```

#### 深度矩阵（Deep Matrix，full 模式）

按维度分组，每组一张子表，附分析说明。

---

### Step 4 — 差异分析

在矩阵基础上，生成以下分析：

#### 关键差异提炼

```
KEY_DIFFERENCES:
  找出维度间差异最大的 3-5 个点
  对每个差异点：
    - 描述差异
    - 解释为何存在此差异（设计选择 / 应用场景不同 / 时间先后）
    - 评估哪种选择在什么条件下更优
```

#### 优劣势矩阵

```markdown
### 各方法优劣势

**{Paper A}**
- 优势：{strength_1}；{strength_2}
- 劣势：{weakness_1}；{weakness_2}
- 最适合：{best_for}
```

#### 选择建议

```
RECOMMENDATION:
  根据以下场景给出建议：
  - 追求最高性能 → {paper_X}，理由：{reason}
  - 快速复现/有代码 → {paper_Y}，理由：{reason}
  - 资源受限场景 → {paper_Z}，理由：{reason}
  - 工业落地 → {paper_W}，理由：{reason}
```

---

### Step 5 — 输出生成

完整输出结构：

```markdown
# {topic} 方法对比报告

> 对比论文：{n} 篇 | 对比维度：{compare_focus} | 生成时间：{date}
> ⚠️ 性能数字均来自原文，不同论文可能使用不同实验设置，直接比较需谨慎

## 速览对比矩阵
{quick_matrix}

## 关键差异分析
{key_differences}

## 各方法深度对比

### 技术路线对比
{method_analysis}

### 实验结果对比
{performance_analysis}
> 注：以下数字仅对在相同数据集上测试的论文有可比性

### 设计选择对比
{design_analysis}

## 优劣势总结
{pros_cons_per_paper}

## 选择建议
{recommendations}

## 原始数据
<details>
<summary>展开：各论文完整提取信息</summary>
{per_paper_knowledge_units}
</details>
```

---

## 质量自检

```
QUALITY_CHECKLIST:
  [ ] 所有性能数字来自原文，已标注来源论文和数据集
  [ ] 不可比较的指标已标注 [不可直接对比]
  [ ] 缺失字段使用 [原文未提及]，无捏造
  [ ] 优劣势描述有依据（来自方法分析或实验结果），非主观臆断
  [ ] 选择建议条件明确（在什么场景下），非绝对推荐
  [ ] 时间因素已考虑（不以新论文的标准苛求早期论文）
```

---

## 与 Agent 的接口契约

**输入**：
```json
{
  "skill": "paper-compare",
  "params": {
    "papers": [KNOWLEDGE_UNIT or URL or title],
    "compare_focus": "full",
    "output_format": "markdown"
  }
}
```

**输出**：
```json
{
  "comparison_matrix": {
    "quick": "markdown_table",
    "deep": {"method": "table", "performance": "table", "design": "table"}
  },
  "key_differences": [{"dimension": "...", "analysis": "..."}],
  "recommendations": [{"scenario": "...", "choice": "...", "reason": "..."}],
  "rendered_markdown": "string"
}
```

---

## 边界与限制

- **不做实验复现**：所有数字来自原文，不重新测试
- **不做价值判断**：不判断哪篇论文"更好"，只描述在特定维度的差异
- **性能对比局限**：不同论文的实验设置差异可能导致性能数字不具可比性，已标注警告
- **最多支持 10 篇**：超出时建议拆分为多次对比或使用 Research 功能

---

## 示例调用

**用户输入**：`帮我对比 LoRA、Prefix Tuning、Adapter 这三种 PEFT 方法`

**执行过程**：
1. 对三篇论文分别调用 `paper-extract`（standard 深度）
2. compare_focus = "full"（方法 + 性能 + 设计）
3. 维度对齐：发现三篇使用不同数据集，仅对 GLUE benchmark 结果做横向比较
4. 生成速览矩阵 + 三张深度子表
5. 关键差异：参数位置（输入 vs 中间层 vs 旁路）、训练开销、表达能力
6. 选择建议：资源受限→LoRA；需要模块化→Adapter；生成任务→Prefix Tuning
