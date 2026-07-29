# OvisOCR2 Technical Report

- Year: 2026
- Date: 2026-07-15
- Venue: arXiv
- Authors: Shiyin Lu, Yinglun Li, Yu Xia, Yuhui Chen, An-Yang Ji, Jun-Peng Jiang, Qing-Guo Chen, Jianshan Zhao, En Lin, Haijun Li, Cheng Qin, Zhao Xu, Weihua Luo
- Links:
  - arXiv: https://arxiv.org/abs/2607.13639
  - PDF: https://arxiv.org/pdf/2607.13639
  - Model: https://huggingface.co/ATH-MaaS/OvisOCR2
- Local PDF: [../papers/2026_ovisocr2-technical-report.pdf](../papers/2026_ovisocr2-technical-report.pdf)
- Tags: document-parsing, OCR, multimodal, synthetic-data, reinforcement-learning, on-policy-distillation
- Status: reading
- Rating:
- One-line takeaway: OvisOCR2 把 Qwen3.5-0.8B 训练成单次生成整页 Markdown 的端到端文档解析器，通过真实数据清洗、HTML 同源合成、4B GRPO teacher、top-k 反向 KL 的 on-policy distillation 和模型融合，在 OmniDocBench v1.6 以 96.58 首次让端到端模型登顶。

## Problem

文档解析不只是识别文字，还要同时恢复公式、表格结构、视觉区域、自然阅读顺序和页面级完整性。主流 pipeline 方法先做布局检测，再裁剪区域并分别识别，虽然性能强，但部署链路复杂，而且前级漏检无法被后级修复；已有端到端方法只用一个模型把整页图像直接转成 Markdown，系统更简单，却通常落后于 pipeline。

这篇论文要回答的是：能否用一个足够小、可部署的端到端模型，在不拆分布局分析与内容识别的情况下，达到甚至超过领先 pipeline 的整页文档解析性能？

## Core Idea

论文提出 `OvisOCR2`，一个由 Qwen3.5-0.8B post-training 得到的端到端 page-to-Markdown 模型。它的重点不是新的视觉 backbone，而是围绕长、结构化输出设计数据和训练 recipe：

1. 真实文档先由专业 OCR parser 产生结构化候选，再统一序列化和保守过滤；
2. 合成数据从真实失败样例抽象 HTML 模板，由 agent 扩展内容与布局，并从同一 HTML 同时生成页面图像和 Markdown 标签；
3. 先在 4B 分支上用文本、公式、表格的可验证奖励做 GRPO；
4. 让 0.8B student 在自己的 rollout 上接受 4B RL teacher 的 token distribution 监督；
5. 对不同数据和训练配置的候选 checkpoint 做加权参数平均，得到最终模型。

## Method

### 1. Real-world Data Pipeline

真实页面先经过 `PaddleOCR-VL-1.5` 或 `MinerU2.5-Pro`，但 parser 输出只作为候选标签，不直接进入训练集。数据引擎读取结构化 JSON，并用 source-specific rules 统一转成 Markdown：

- 严格校验 block category，未知类型直接拒绝；
- 清理空文本、乱码、尾部重复和重复图片；
- 统一标题层级、行内与块级 LaTeX 公式格式；
- 要求表格包含合法 HTML `<table>` 结构并清除多余样式；
- 把图、chart 等视觉区域编码为归一化坐标的 HTML image tag；
- 按 parser 提供的 block order 用双换行拼接。

随后按数据源、parser、文档领域和转换配置划分 subset，人工抽样检查文本、公式、表格、视觉区域和阅读顺序。高错误率 subset 整体剔除；这一阶段强调保守过滤，而不是用自由生成去“修复”标签。

### 2. Source-aligned Synthetic Data

合成管线遵循 `source-of-truth` 原则：页面图像和 Markdown target 都从同一 HTML 源生成，从而避免“先渲染、再 OCR 回标”带来的标签噪声。

流程如下：

1. 从模型失败样例中挖掘难例，如复杂表格、不规则布局、页眉页脚干扰、手写区域、多栏阅读顺序和超长输出；
2. 将相似失败模式聚类，并用多模态模型把代表性样例转换成可编程 HTML 模板；
3. 用 agent 在保持模板视觉与结构意图的前提下，改变文本、数值、公式、表格拓扑、层级和页面组织；
4. 直接从 HTML DOM 序列化 Markdown，按文档类型确定单栏或多栏阅读顺序；
5. 用 Playwright 渲染文档图像，并从 DOM 获取视觉区域的精确坐标；
6. 先小批量预览迭代，再规模化生成并过滤渲染失败、空标签、结构错误、定位错误和重复样本。

这使真实失败模式可以被转换成一族可控、标签干净的训练页面，而不仅是对单个 hard case 做过拟合。

### 3. Two-branch Training

#### Supervised Fine-tuning

- Backbone: Qwen3.5-0.8B 与 Qwen3.5-4B。
- Data: 同一套真实 + 合成 page-to-Markdown 数据。
- Optimization: 两个模型都做 full-parameter SFT。
- Schedule: 0.8B 训练 2 epochs；4B 只训练 0.2 epoch，以控制较大分支成本。
- Context: 最大序列长度 16K，并使用动态图像分辨率预算。

0.8B SFT checkpoint 初始化最终 student；4B SFT checkpoint 则进入 RL，成为后续蒸馏 teacher。

#### Reinforcement Learning on the 4B Branch

RL 使用 GRPO。训练数据以标签精确的合成页为主，保留少量高质量真实文档。作者先用当前 policy rollout 并打分，重点选择“偶尔能明显做对、但平均表现仍有改进空间”的页面，过滤过易、过难或组内 reward 几乎无差异的样本。

页面奖励只平均 ground truth 中实际存在的组件：

```text
text score    = 1 - normalized edit distance
formula score = CDM
table score   = TEDS

R(y, y*) = sum_c a_c(y*) * s_c(y, y*) / sum_c a_c(y*)
```

截断和不可解析结构作为 guard，把受影响的可用组件置零。训练系统还并行化 reward worker，对公式 exact match、无效输出和表格 exact match 走 shortcut，仅把必要样本送去耗时的 CDM 渲染或 TEDS；高分辨率视觉 tensor 通过 object-store handle 传递；actor update 用 common-prefix mask 避免相同 prompt 的共享前缀主导梯度。

#### On-policy Distillation into 0.8B

作者发现直接对 0.8B 做相同 RL 会在后期产生更大的 KL drift，复杂表格质量也更不稳定，因此改用 4B RL model 作为 teacher。

student 先在当前 policy 下生成完整页面 Markdown，teacher 再在同一 student trajectory prefix 上打分。每个位置只保留 student 自己概率最高的 top-k token，并在这个 support 内归一化 teacher 与 student 分布，最小化反向 KL：

```text
S_t = TopK_k(pi_student(. | c_t))

L_OPD = mean_t KL(
  p_student(. | c_t, S_t)
  ||
  q_teacher(. | c_t, S_t)
)
```

因为监督发生在 student 实际访问的状态上，不要求 teacher 与 student 生成序列对齐。反向 KL 具有 mode-seeking 倾向，会惩罚 student 把概率放在 teacher 低概率 token 上。top-k support 把长输出蒸馏的主要张量规模从 `O(TV)` 降为 `O(Tk)`；实现上还使用 selected-logit gathering 和 chunked projection 降低峰值显存。

#### Model Fusion

作者改变数据 mixture 和训练配置得到多个候选 OvisOCR2 checkpoint，最后用 weighted parameter averaging 融合为最终 0.8B 模型。

## Experiments

### OmniDocBench v1.6

该 benchmark 含 1,651 个 PDF 页面，覆盖 10 类文档、5 类布局和 5 类语言，评估文本、公式、表格与阅读顺序。

| Model | Type | Params | Overall ↑ | TextEdit ↓ | Formula CDM ↑ | Table TEDS ↑ | ROEdit ↓ |
|---|---|---:|---:|---:|---:|---:|---:|
| PaddleOCR-VL-1.6 | pipeline | 0.9B | 96.33 | 0.033 | 97.49 | 94.76 | 0.127 |
| MinerU2.5-Pro | pipeline | 1.2B | 95.75 | 0.036 | 97.45 | 93.42 | 0.120 |
| HunyuanOCR-1.5 | end-to-end | 1B | 94.74 | 0.039 | 94.50 | 93.67 | 0.129 |
| **OvisOCR2** | end-to-end | **0.8B** | **96.58** | **0.025** | **97.53** | **94.76** | **0.111** |

OvisOCR2 比此前最好的端到端方法高 1.84 分，并以 96.58 超过领先 pipeline；它的 Table TEDS 与 PaddleOCR-VL-1.6 并列，但在文本、公式、reading order 和忽略结构的 TEDS-S 上领先。

### PureDocBench

PureDocBench 从 HTML 同源生成页面与标签，包含 Clean、Digital degradation 和 Real recapture 三条 track，共 1,475 页、4,425 张图。

| Model | Params | Clean ↑ | Digital ↑ | Real ↑ | Avg3 ↑ |
|---|---:|---:|---:|---:|---:|
| Qwen3.5-122B-A10B | 122B/10B | 76.14 | 76.34 | **69.85** | 74.11 |
| FD-RL | 4B | 78.38 | 76.33 | 67.04 | 73.92 |
| **OvisOCR2** | **0.8B** | **81.55** | **77.09** | 66.56 | **75.06** |

OvisOCR2 的 Avg3、Clean 和 Digital 最好，但 Real track 仍落后于大型通用 VLM，说明手机翻拍、复印、拍屏和压缩截图等真实退化场景仍是明显短板。

### In-house Benchmark

作者还构建了超过 1,000 页的内部 benchmark，覆盖表单、扫描报告、印刷模板中的手写标注和复杂合并单元格：

- 全集 Overall 85.54，高于 PaddleOCR-VL-1.6 的 82.88；
- hard tier 为 78.99，高于第二名的 75.06；
- handwriting subset Overall 72.28；
- complex-table subset Overall 83.97，table missing rate 为 7.96%，而对比 pipeline 为 13.27%-17.26%。

最后一项支持了端到端方法的核心论点：pipeline 的 layout stage 一旦漏掉表格，后续 recognizer 无法恢复。

### Ablations

论文没有给出标准的逐组件 ablation。与训练设计直接相关的证据主要是 Figure 4：相同 RL stage 下，0.8B direct RL 后期 policy divergence 更大且 validation table TEDS 下降，而 4B RL 更稳定，因此最终采用 4B RL teacher + 0.8B OPD。

## Strengths

- 用 0.8B 单模型在 OmniDocBench v1.6 超过 pipeline，直接支撑“端到端解析可以兼顾性能与系统简洁性”的主张。
- 合成数据不是随机拼模板，而是从线上失败模式反推可复用 HTML generator；同源生成图像和标签也解决了结构化 OCR 中常见的 synthetic label noise。
- RL reward 与最终任务结构匹配：文本、公式和表格分别使用可验证指标，而不是只用统一字符串相似度。
- 把较不稳定的 compact-model RL 转换为“大模型吸收 reward、student 在自身状态上蒸馏”的两分支训练，适合长结构化输出。
- 同时在两个公开 benchmark 和更困难的内部数据上评测，并明确暴露 Real track 的不足。

## Weaknesses

- 缺少逐组件 ablation，无法判断真实数据清洗、hard-case synthesis、4B RL、OPD 和 model fusion 各自贡献多少。
- 没有披露训练数据规模与语言分布、真实/合成比例、GRPO group size、top-k、训练步数、fusion 权重和总算力，recipe 很难完整复现。
- “compact deployment” 只有参数量支撑，没有报告 latency、吞吐、显存、不同图像分辨率下的成本，也没有与 pipeline 做端到端系统效率对比。
- 内部 benchmark 不公开，构建过程与开发过程相互关联，可能存在 sample selection 或 tuning bias；其泛化结论需要独立数据验证。
- PureDocBench Real 只有 66.56，低于若干大型通用 VLM，真实拍摄退化仍未解决。
- 真实数据标签来自已有 OCR parser 再经过规则清洗与 subset spot-check，仍可能继承 teacher parser 的系统性错误；论文没有量化过滤前后标签质量或人工检查强度。
- model fusion 仅用一句话描述，既缺配置也缺与单一 checkpoint 的对照。

## Useful For

- 设计 page-to-Markdown 或 OCR post-training pipeline 时，把数据问题拆成真实数据规范化、subset-level QA、同源合成和失败模式驱动的 hard-case expansion。
- 训练 compact multimodal model 时，参考“较大模型做可验证 RL，小模型对自己的 rollout 做 OPD”的 reward transfer 方案。
- 处理超长结构化输出的蒸馏时，参考 student top-k support、selected-logit gathering 和 chunked projection。
- 对比 pipeline 与 end-to-end 架构时，不只看 leaderboard，还要重点分析 layout 漏检这种不可恢复错误。

## Questions

- 如果去掉 4B RL、OPD 或 model fusion，OmniDocBench 和 PureDocBench 分别会下降多少？
- hard-case mining 是否持续在线更新？新失败样例怎样去重、聚类并判断值得变成一个 synthesis family？
- student top-k 之外的 teacher 概率质量完全不参与训练，会不会忽略 teacher 知道但 student 尚未探索到的更优 token？
- 真实数据中两个 parser 的标签如何混合？是否存在同页多 teacher 一致性过滤或 disagreement-based sampling？
- 在多页 PDF 上，逐页解析后的跨页标题、表格续接、页眉页脚去重和全局 reading order 怎样处理？
- 0.8B 模型在 CPU、消费级 GPU 和服务端 GPU 上的实际吞吐、首 token 延迟和峰值显存是多少？
- PureDocBench Real 的主要错误来自视觉退化、分辨率、布局定位还是长输出稳定性？

## Share Notes

适合的分享主线：

1. OCR 已经不只是转文字，而是把整页文档恢复成可计算的 Markdown。
2. pipeline 的上限很高，但 layout 漏检不可恢复；端到端方法过去的问题是性能不足。
3. OvisOCR2 的数据飞轮：真实 parser 候选 + 保守清洗；真实失败样例 + HTML 同源合成。
4. 训练主线：0.8B/4B SFT → 4B GRPO → 0.8B on-policy distillation → model fusion。
5. 结果：0.8B 首次让端到端模型登顶 OmniDocBench，但真实退化图像、复现信息和效率账本仍不完整。
