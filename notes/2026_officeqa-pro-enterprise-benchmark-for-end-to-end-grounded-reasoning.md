# OfficeQA Pro: An Enterprise Benchmark for End-to-End Grounded Reasoning

- Year: 2026
- Date: 2026-03-09
- Venue: arXiv
- Authors: Krista Opsahl-Ong, Arnav Singhvi, Jasmine Collins, Ivan Zhou, Cindy Wang, Ashutosh Baheti, Owen Oertell, Jacob Portes, Sam Havens, Erich Elsen, Michael Bendersky, Matei Zaharia, Xing Chen
- Links:
  - arXiv: https://arxiv.org/abs/2603.08655
  - PDF: https://arxiv.org/pdf/2603.08655
  - Code / benchmark: https://github.com/databricks/officeqa
  - Dataset: https://huggingface.co/datasets/databricks/officeqa
- Local PDF: [../papers/2026_officeqa-pro-enterprise-benchmark-for-end-to-end-grounded-reasoning.pdf](../papers/2026_officeqa-pro-enterprise-benchmark-for-end-to-end-grounded-reasoning.pdf)
- Tags: agent, benchmark, enterprise, grounded-reasoning, document-parsing, retrieval, quantitative-reasoning, table-understanding
- Status: reading
- Rating:
- One-line takeaway: OfficeQA Pro 把企业文档 Agent 的能力拆成 parsing、retrieval 和 analytical reasoning 三段，并表明任一环节的小误差都会沿链路放大；高质量结构化解析可显著提升准确率和速度，但即使给出 oracle 页面，最强配置仍有约三分之一问题答错。

## 先说结论

OfficeQA Pro 不是普通的 long-context QA。它要求 Agent 在近百年的美国 Treasury Bulletin 里：

```text
解析异构 PDF
  -> 找到正确文档、页面、表格、行列和脚注
  -> 判断同一统计量的时间版本与修订关系
  -> 必要时检索外部数据
  -> 执行多步统计或金融计算
  -> 按严格精度输出唯一答案
```

这条链路的价值在于它更接近企业知识工作的真实失败方式：系统可能搜索到了“看起来相关”的数字，却选错统计口径；可能读对了表格，却忽略后续公告里的修订值；也可能所有输入都正确，最后因为公式、单位或中间舍入出错而失败。

论文最重要的证据是：

- 只靠参数知识时，三个 frontier LLM 在严格匹配下都低于 `3%`；开放 web search 后最高也只有 `11.3%`。
- 在完整原始 PDF corpus 上，Claude Opus 4.6、GPT-5.4 和 Gemini 3.1 Pro Preview Agent 分别达到 `48.1%`、`36.1%` 和 `18.1%`。
- 把 PDF 预先解析成结构化文本后，三者分别达到 `54.1%`、`56.4%` 和 `29.3%`，同时快约 `4-9x`。
- 即使直接给 oracle 页面和高质量解析文本，最强 Agent 也只有 `66.9%`；所以 retrieval 并不是唯一瓶颈，解析后的证据使用和定量推理同样关键。

## Problem

很多现有 benchmark 把所需上下文直接放进 prompt，绕过了真实企业环境中的 collection navigation。OfficeQA Pro 关注的是 `Grounded Reasoning`：

> 在大规模、异构、持续修订的文档集合中定位证据，并基于证据完成可验证的分析。

作者希望同时满足三点：

1. **企业数据复杂度**：包含 prose、复杂表格、图表、扫描件、OCR 噪声、格式漂移和长期版本修订。
2. **多步推理**：不能只靠记忆或单次查表，需要定位文档、抽取数据并完成分析。
3. **高精度、可自动验证**：每题有唯一答案，避免依赖昂贵且主观的 expert judge。

## Benchmark Design

### Corpus

- 来源：1939 年至今的 U.S. Treasury Bulletins。
- 规模：约 `89,000` 页，超过 `2,600 万` 个数值。
- 单份 Bulletin 通常为 `100-200` 页，包含文本、复杂层级表格、图和脚注。
- 1996 年以前以扫描 PDF 为主，之后逐渐转为 digital-native PDF。
- 同一统计量会在后续月份或年份被修订，不能默认第一个命中就是最终值。
- 作者移除了 PDF 的 embedded text layer，让不同系统面对相同的文档可读性条件。

### Questions

OfficeQA Pro 包含 `133` 道高难题。另有 `113` 道较简单问题，二者合计为 `246` 题的 OfficeQA Full。

Pro split 的能力分布包括：

- `11%` 的问题需要三个或更多 bulletin；
- `22%` 需要 web search 补充历史汇率、CPI 等外部值；
- `3%` 需要直接理解图、chart 或 graph；
- `62%` 需要超出基础四则运算的数据分析，例如 OLS regression。

一道题可能只依赖一页，也可能跨越二十多页。作者先通过多轮 annotator 复核答案，再用 Agent 的冲突输出做两轮 end-to-end QA，消除歧义或纠正 ground truth。

为了保证任务确实需要 grounding，作者还过滤掉了 Claude Opus 4.5 和 GPT-5.1 仅凭参数知识即可回答的问题；随后把当时 GPT-5.1 与 Claude Opus 4.5 Agent 都能答对的题放进 Easy split，剩余题构成 Pro split。

### Evaluation

`99%` 的答案是数值。主指标使用 `0.0%` allowable absolute relative error，也就是严格正确；同时报告 `0.1% / 1% / 5%` 容差下的准确率。

评测器会归一化标点、数学符号和常见缩写。日期或多段答案必须同时匹配主要文本与数字；少量非数值答案使用容忍格式差异的 fuzzy match。

严格数值匹配很符合金融分析对精度的要求，但也意味着：

- 单位转换错误会造成数量级失败；
- 提前舍入可能让整题判错；
- 选到旧版本但数值很接近的答案仍是错误；
- `5%` 容差下的“Fermi estimate 能力”不能替代精确 grounding。

## Experiments

### 1. Direct LLM：搜索工具不等于完整工作流

作者比较了 Prompt Only、Web Search、Oracle PDF Pages 和 Oracle Parsed Pages。

- Prompt Only：三个模型严格准确率都低于 `3%`。
- Web Search：GPT-5.4 最高，达到 `11.3%`；Opus 4.6 约 `80%` 的错误 run 没能在 token budget 内产出最终答案。
- Oracle PDF + Web Search：GPT-5.4、Opus 4.6、Gemini 3.1 Pro Preview 分别为 `57.1%`、`36.1%`、`52.6%`。
- Oracle Parsed + Web Search：分别提升到 `65.4%`、`57.1%`、`56.4%`。

即使跳过 retrieval，原始 PDF 的复杂布局、扫描质量和表格层级仍会造成很大损失；换成高质量解析后还答错的问题，则更多来自公式、口径、单位、图表理解和精度控制。

### 2. Frontier Agent：解析影响准确率，也支配 latency

Agent baseline 使用 Codex CLI、Claude Agent SDK 和 Gemini CLI，并给予各自原生文件搜索、shell、代码执行与 web search 工具。

| Agent / model | Full PDF | Full parsed | Oracle PDF | Oracle parsed |
|---|---:|---:|---:|---:|
| Claude Opus 4.6 | 48.12% | 54.14% | 60.90% | 66.92% |
| GPT-5.4 | 36.09% | 56.39% | 54.89% | 65.41% |
| Gemini 3.1 Pro Preview | 18.05% | 29.32% | 39.10% | 46.62% |

几个值得注意的系统现象：

- Claude Agent 主要通过原生 Read tool 逐页提取；Codex 约 `90%` 的 tool calls 使用 OCR / PDF CLI。
- Codex 的 GPT-5.4 pipeline 通过 `tesseract + pdftoppm + Pillow` 批量 OCR，虽然准确率低于 Claude 的 full-PDF 设置，但速度约快 `2.4x`。
- 用 parsed text 替代 PDF 后，Agent 快 `4-9x`；GPT-5.4 的成本还下降约 `30%`。
- 从 full PDF 换成 oracle PDF，准确率提升 `13-21pp`，latency 平均下降约 `76%`，成本下降 `74-88%`。
- 即便只剩 oracle pages，Agent 仍会重复进行 PDF extraction；所以文档解析不仅是准确率问题，也是 workflow latency 的主要来源。

### 3. Parser Ablation：parser 不是前处理细节

作者用同一个 custom agent 比较三种 parser：

| Parser | 三个 Agent 平均准确率 | 完整 corpus 解析成本 |
|---|---:|---:|
| Databricks `ai_parse_document` | 50.4% | $178 |
| Docling | 38.4% | 未单列；agent 平均成本 $8.18 / sample |
| unstructured.io | 31.1% | $2,670 |

`ai_parse_document` 相比另外两种 parser 不只是更准，也让下游 Agent 更省成本。论文最终把 parser 输出中的 text elements 按 reading order 拼接为 `.txt`，没有把全部 bounding box metadata 直接交给 Agent。

这个实验说明端到端 Agent 评测不能只报告 model 与 prompt；parser、OCR、table topology reconstruction 和输出序列化都是系统能力的一部分。

### 4. Table Representation

作者比较 HTML 和 Hierarchical Markdown：

- HTML 在 `11` 个 Agent 中有 `7` 个准确率更高；
- 总体差异不大，且明显依赖 model family；
- Markdown 更省 token，但不原生表达 nested headers；
- Hierarchical Markdown 需要把多级表头折叠进单列名，例如 `Sept 30, 1990 > Currency > Total`。

因此没有一种 serialization 对所有模型稳定占优。表格格式应该和目标模型共同评测，而不是假设 Markdown 永远是最佳 Agent 输入。

### 5. Retrieval

custom agent 比较了：

- file search；
- 标准 vector search；
- 加入文档名、日期、页码、页眉、表名和 section title 的 contextual vector search；
- file search + contextual vector search。

标准 vector search 比 file search 平均低 `27%`，因为切块常把表格内容和 header / section context 分开。加入 contextual metadata 后，性能平均提升 `21%`，tool calls、latency 和 cost 分别下降约 `44%`、`38%` 和 `44%`。

混合 file search 与 contextual vector search 又比单独 contextual vector search 提升 `15%`。对三个模型中的两个，这是最高准确率配置；对 GPT-5.4 和 Opus 4.6，它还比 file search 单独使用便宜 `6-13%`。

核心启发是：企业 retrieval 不只是 embedding model 的问题。chunk 必须携带原文位置、时间、表格归属和结构上下文；同时，Agent 仍需要 grep-style exact search 来寻找特定年份、指标和数值。

### 6. Test-Time Scaling

作者对每题运行最多 `4` 次 rollout，并用 plurality voting 聚合。收益稳定但有限，而且和单次准确率负相关：较弱、方差较大的 Agent 获益更多；强 Agent 的输出更一致，因此 consensus 的边际收益更小。

这也揭示了一个上限：如果多个 rollout 都受同一个 parser error、错误统计口径或 retrieval blind spot 影响，多跑几次不会修复系统性错误。

## Failure Modes

### Temporal revision verification

Agent 经常命中某个数值后过早停止，没有继续确认后续 Bulletin 是否修订。强制它反复检查修订又容易形成 retrieval loop，逐渐填满上下文并丢失已经确认的中间状态。

这类问题需要显式维护：

- statistic definition；
- observation date 与 publication date；
- source issue / page；
- preliminary / revised 标记；
- supersedes / superseded-by 关系；
- 当前选用值及其理由。

### Parsing faithfulness

原始 PDF Agent 有 `40-50%` 的失败和解析错误有关，包括：

- OCR 读错单个数字；
- 表格行列错位；
- nested header 丢失；
- subheader、description 或 footnote 被剥离；
- scanned / digital-native 文档的 reading order 不一致。

parser 输出一旦出错，后续 retrieval 和 reasoning 通常不会意识到自己看到的“现实”已经被改写。

### Visual understanding

预解析文本会遗漏 figure；直接给图时，Agent 又难以从密集金融图表中精确读取 trend line、local maxima 等信息。视觉内容既可能无法被 retrieval 找到，也可能在找到后无法被高精度解释。

### Analytical reasoning

剩余错误包括：

- 统计口径接近但不等价；
- 用错 CPI 或其他外部历史值；
- sample variance 与 population variance 混淆；
- 单位对齐错误；
- 忽略题目指定的 rounding procedure；
- 本应调用 Python 的计算留在自然语言推理中完成。

## Strengths

- 把 parsing、retrieval、revision checking 和 computation 放进同一端到端评测，而不是分别用独立 benchmark 代替系统能力。
- 原始文档公开、答案主要为数值，能够自动且严格复核。
- 同时报告准确率、latency、tool calls 和 cost，能看出不同 Agent 的系统策略差异。
- Oracle pages、parser、table format、retrieval 和 test-time scaling 的 ablation 比只给总榜更有诊断价值。
- long-running historical corpus 自带格式漂移和 revision history，比静态、清洁的企业 QA 数据更接近真实 document collection。

## Weaknesses

- 领域集中在美国财政与金融统计；企业 grounded reasoning 是否能泛化到法律、生命科学、内部 wiki、邮件和数据库仍需验证。
- Pro split 是根据特定历史 frontier agents 的表现筛出来的，难度定义会随模型进步而漂移。
- questions / answers 已公开或 gated release；具备 web access 的 Agent 可能通过 benchmark leakage 找答案，论文中虽未观察到这种行为，但长期评测需要 held-out set。
- `99%` 数值答案便于自动评测，却低估了企业任务中 narrative synthesis、引用完整性、冲突解释和可交付文件质量。
- 不同 Agent framework 与 model 绑定，不能完全把模型能力、工具设计、默认 prompt 和执行策略拆开比较。
- Databricks parser 在 Databricks benchmark 上表现最好，结论有产品相关性；虽然实验差距很大，仍需要独立 corpus 和更多 parser 复核。
- paper 的“enterprise”主要指 corpus 结构复杂度和分析过程，并不包含权限、动态数据库、协作、审计、合规或真实组织流程。

## Useful For

- 设计企业文档 Agent 的端到端 evaluation suite。
- 决定优先优化模型、parser、table serialization 还是 retrieval。
- 为 long-running corpus 设计 revision-aware evidence schema。
- 评估 PDF-to-text pipeline 对 accuracy、latency 和 cost 的共同影响。
- 设计 exact search + vector search 的混合 retrieval。
- 建立“最终答案错误”到 parsing / retrieval / computation 的 failure attribution。

## Questions

- 如何构造真正 held-out、又能持续复现的企业语料和问题，避免公开 benchmark leakage？
- parser error 应该如何向 Agent 暴露 uncertainty，而不是只交付一个看似确定的文本结果？
- 是否可以对关键数值保留页面 crop、OCR alternatives、cell coordinates 和 table topology，让 Agent 在低置信度时回看原图？
- revision-aware retrieval 是否应该把文档 collection 建模成 temporal database，而不是普通向量库？
- 若把唯一数值答案扩展成 memo / spreadsheet 等 deliverable，怎样同时评估结论、证据、公式、引用和格式？
- plurality voting 只能聚合最终答案；能否先对 source cells、统计口径和计算程序做独立 verification，再组合答案？

## Share Notes

适合的分享主线：

1. 企业 Agent 的真正任务不是“读一份 PDF”，而是跨一个持续变化的文档集合完成 grounded analysis。
2. OfficeQA Pro 如何把近百年 Treasury Bulletins 变成 133 道可严格验证的问题。
3. Full corpus / oracle pages 与 PDF / parsed text 的二维实验，定位 retrieval 和 parsing 的贡献。
4. 为什么 parser 可带来 `20.3pp` 的单模型提升和 `4-9x` 的速度提升。
5. vector search 为什么会在复杂表格上输给 grep，以及 contextual metadata + hybrid search 如何补救。
6. 剩余失败为什么需要 revision-aware evidence state 和 computation verification，而不是只换更大的模型。
