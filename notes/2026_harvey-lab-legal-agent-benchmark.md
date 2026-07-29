# Harvey LAB：把法律 Agent 评测做成文件系统里的真实交付任务

- Year: 2026
- Date: 2026-07-22
- Source type: open-source benchmark / agent harness implementation note
- Project: harveyai/harvey-labs
- Reviewed revision: [`845a088`](https://github.com/harveyai/harvey-labs/commit/845a08840869b21a5c11958aae58bf5f00a7b775)
- Links:
  - Repository: https://github.com/harveyai/harvey-labs
  - README: https://github.com/harveyai/harvey-labs/blob/845a08840869b21a5c11958aae58bf5f00a7b775/README.md
  - Architecture: https://github.com/harveyai/harvey-labs/blob/845a08840869b21a5c11958aae58bf5f00a7b775/docs/architecture.md
  - Evaluation methodology: https://github.com/harveyai/harvey-labs/blob/845a08840869b21a5c11958aae58bf5f00a7b775/docs/eval-strategies.md
  - Tutorial: https://github.com/harveyai/harvey-labs/blob/845a08840869b21a5c11958aae58bf5f00a7b775/docs/tutorial.md
- License: MIT
- Tags: legal-agent, agent-benchmark, professional-work, rubric-evaluation, synthetic-data, sandbox, document-generation
- Status: read
- One-line takeaway: Harvey LAB 最有价值的不是又收集了一批法律问答，而是把评测单位改成“在隔离工作区里阅读一组 matter documents、调用工具、交付 DOCX/XLSX/PPTX，再由逐项 rubric 检查完整性”；但它的 all-pass 分数高度受 rubric 长度与粒度影响，而 synthetic matters、单次评测对一个 LLM judge 的依赖和仍在迁移的 task schema 也意味着它更适合做 agent workflow 压力测试，而不是直接当作法律能力的绝对标尺。

## 先说结论

Harvey LAB（Legal Agent Benchmark）评测的不是模型能否回答一个法律知识问题，而是它能否完成一次接近真实工作流的文件交付：

```text
task instructions + synthetic matter documents
  -> sandboxed agent + file tools + authoring skills
  -> DOCX / XLSX / PPTX / Markdown deliverables
  -> criterion-scoped LLM judging
  -> all-pass score + diagnostic reports
```

这个转变很重要。专业服务工作的失败经常不是“答案整体不够像”，而是漏掉一个 red flag、没有把一个事实交叉引用到另一份文件、没有按指定格式交付，或 redline 与 issues memo 对不上。LAB 把这些要求拆成明确的 pass/fail criteria，并保留完整工具轨迹、文档覆盖率、token、时延和成本。

它仍然不是生产质量认证：输入文件是大规模合成数据，rubric 本身就是唯一标准，judge 也是模型。更准确的定位是：一个大规模、可运行、可审计的 legal agent engineering benchmark。

## Revision 快照：规模比文档更新得更快

对 reviewed revision 直接扫描得到：

- `1,760` 个 `task.json`；
- `111,826` 条 rubric criteria；
- 单任务 criterion 数最少 `23`、中位数 `57`、最多 `1,114`；
- `26` 个顶层 task collections，其中既有 legal practice areas，也有 `contracts`、`diligence` 这类集合；
- 任务输入包含大量 DOCX、XLSX、PPTX、EML 和纯文本文件。

这些数字与同一 revision 的公开说明并不完全一致：README badge 写 `1,671` tasks，evaluation 文档写 `1,660` tasks / 约 `101,000` criteria。仓库显然正在快速扩充，引用规模时应该同时记录 revision，而不是只写一个会漂移的数字。

另一个需要注意的兼容层是：其中 `498` 个 legacy BLB-imported contracting tasks 没有新的 `work_type` 和 criterion-level `deliverables` 字段。测试代码会把它们识别为 legacy schema，评测器则退回到读取全部 output files。也就是说，文档描述的是目标 schema，但当前数据集还不是完全同构的。

相关实现：

- [`tests/test_task_integrity.py`](https://github.com/harveyai/harvey-labs/blob/845a08840869b21a5c11958aae58bf5f00a7b775/tests/test_task_integrity.py)
- [`evaluation/scoring.py`](https://github.com/harveyai/harvey-labs/blob/845a08840869b21a5c11958aae58bf5f00a7b775/evaluation/scoring.py)

## 核心对象：task 不是问题，而是一份 matter file

标准任务目录由 `task.json` 和 `documents/` 组成：

```text
tasks/<practice-area>/<workflow>/<optional-scenario>/
  task.json
  documents/
```

`task.json` 的关键字段是：

- `title`：任务标题；
- `instructions`：给 Agent 的 directional prompt；
- `work_type`：`analyze / draft / review / research`；
- `deliverables`：必须交付的文件名；
- `criteria`：内联的二元评分标准；
- `tags`：检索和分析元数据。

一个 criterion 通常包含：

```json
{
  "id": "C-001",
  "title": "Identifies key contract as requiring consent",
  "match_criteria": "PASS if ... FAIL if ...",
  "deliverables": ["red-flag-memo.docx"],
  "sources": ["customer-agreement.docx"]
}
```

这让 benchmark 的真值不再是一份理想答案，而是一组专家认为交付前必须逐项检查的条件。对于 issue spotting，它可以要求识别事实、交叉引用来源、量化风险并提出处置建议；对于 drafting / redline，它可以要求特定条款修改、track changes 形式以及 memo 与 redline 的一一对应。

仓库教程里的 M&A red-flag task 就包含 60 份 synthetic matter documents 和 68 条 criteria。它要求 Agent 同时读 CIM、QofE、合同、许可、保险、员工福利等文件，发现跨文件矛盾并生成 memo / tracker。这比单文档问答更能测出检索纪律、覆盖率、文件操作和长程状态管理问题。

## 一次评测的三个阶段

### 1. Run：简单 Agent loop，复杂文件工作

Harness 先加载任务 instructions、共享 system prompt 和默认 file-authoring skills，再根据 model ID 选择 provider adapter。当前 revision 支持 Anthropic、OpenAI、Google、Mistral、Fireworks 和 Baseten 等适配器。

Agent 只有六个 workspace tools：

| Tool | 作用 |
|---|---|
| `bash` | 在隔离工作区执行 shell 命令 |
| `read` | 读取 DOCX、XLSX、PPTX、PDF 和文本 |
| `write` | 写入 output 下的文件 |
| `edit` | 精确修改已有文件 |
| `glob` | 枚举文件 |
| `grep` | 正则搜索文件内容 |

循环本身刻意保持简单：模型响应包含 tool calls 就顺序执行并把结果送回；没有 tool call 就结束；达到 `max_turns` 或 context overflow 也会停止。没有 planner、memory service 或显式 `finish` tool。换句话说，LAB 主要固定环境、工具和评测协议，把 planning strategy 留给被测模型。

每次 run 会保存：

- `config.json`：模型、task、reasoning effort、skills 等配置；
- `transcript.jsonl`：模型与工具的逐轮轨迹；
- `metrics.json`：token、时延、文件读取覆盖率和工具次数；
- `output/`：最终交付文件。

相关实现：

- [`harness/run.py`](https://github.com/harveyai/harvey-labs/blob/845a08840869b21a5c11958aae58bf5f00a7b775/harness/run.py)
- [`harness/agent_loop.py`](https://github.com/harveyai/harvey-labs/blob/845a08840869b21a5c11958aae58bf5f00a7b775/harness/agent_loop.py)
- [`harness/tools.py`](https://github.com/harveyai/harvey-labs/blob/845a08840869b21a5c11958aae58bf5f00a7b775/harness/tools.py)

### 2. Evaluate：每条 criterion 单独交给 judge

评测器不会把整份 rubric 和全部交付物一次性塞给 judge。标准任务中，每条 criterion 声明自己关联的 deliverables，评测器只读取这些文件并转成文本，再调用一次 LLM judge：

```text
task title
+ scoped deliverable text
+ criterion title
+ match_criteria
  -> {verdict: pass | fail, reasoning: ...}
```

默认 judge 是 `claude-sonnet-4-6`，temperature 为 0，并使用结构化 JSON schema 约束输出。DOCX 通过 Pandoc 读取，XLSX 通过 pandas，PPTX 通过 MarkItDown，PDF 通过 pdfplumber；criterion 还可以选择是否让 judge 看 DOCX tracked changes。

如果 Agent 文件名与期望不完全相同，评测器会先做 extension / fuzzy matching，必要时再调用 LLM 匹配文件。这提高了对命名漂移的容忍度，但也弱化了“精确按合同交付文件名”本身的失败信号，并额外引入一次模型判断。

相关实现：

- [`evaluation/run_eval.py`](https://github.com/harveyai/harvey-labs/blob/845a08840869b21a5c11958aae58bf5f00a7b775/evaluation/run_eval.py)
- [`evaluation/judge.py`](https://github.com/harveyai/harvey-labs/blob/845a08840869b21a5c11958aae58bf5f00a7b775/evaluation/judge.py)
- [`evaluation/prompts/rubric_criterion.txt`](https://github.com/harveyai/harvey-labs/blob/845a08840869b21a5c11958aae58bf5f00a7b775/evaluation/prompts/rubric_criterion.txt)

### 3. Report：all-pass 是主指标，criterion pass rate 是诊断指标

单任务最终分数非常严格：

```text
score = 1.0  if every criterion passed
        0.0  otherwise
```

理由来自法律生产场景：漏掉一个 material red flag 的 memo 不能被解释为“95% 正确”。因此 comparison dashboard 用重复 runs 中的 all-pass rate 排名，同时保留 pooled criterion pass rate 解释“离完整交付还有多远”。报告还展示 criteria heatmap、document coverage、token、latency 和 estimated cost。

`utils/sweep.py` 可以按 task / workflow / practice area / all 解析任务集，跑 model matrix，再并行评测并生成静态 HTML dashboard。整个系统没有数据库或服务端，tasks 在 `tasks/`，runs 在 `results/`，报告是本地静态文件。

## Sandbox：不只防模型，也防恶意文档

每个 run 使用独立 Podman container：

```text
--network=none
--cap-drop=ALL
--security-opt=no-new-privileges
```

文件系统只暴露一个工作区：

- `/workspace/documents`：只读 matter documents；
- `/workspace/output`：可写交付目录；
- `/workspace`：可写 scratch space 和 skill scripts。

六个工具都经过同一个 Sandbox interface，文档解析也在容器内执行。这一点很实用：benchmark 输入不是纯文本，而是可能包含复杂 XML、宏关系和压缩包结构的 Office 文档；隔离层同时降低 Agent shell 和 crafted documents 对 host 的风险。

系统 prompt 明确告诉 Agent 不得读取 `task.json` / rubric，而且 rubric 并未作为文档挂载到它的可见工作区。这样可以避免 Agent 直接针对评分项反向生成答案。

相关实现：

- [`sandbox/README.md`](https://github.com/harveyai/harvey-labs/blob/845a08840869b21a5c11958aae58bf5f00a7b775/sandbox/README.md)
- [`sandbox/sandbox.py`](https://github.com/harveyai/harvey-labs/blob/845a08840869b21a5c11958aae58bf5f00a7b775/sandbox/sandbox.py)
- [`harness/system_prompt.md`](https://github.com/harveyai/harvey-labs/blob/845a08840869b21a5c11958aae58bf5f00a7b775/harness/system_prompt.md)

## 最值得复用的设计

### 1. 把 benchmark unit 从 answer 改成 deliverable

真实 Agent 价值往往落在文件，而不是聊天窗口。LAB 直接要求可交付的 DOCX、XLSX 和 PPTX，并把文档生成 skill、格式验证、tracked changes 与 spreadsheet 公式纳入执行环境。这个抽象可以迁移到投行、咨询、审计、合规和研究工作。

### 2. 用 criterion-to-deliverable binding 控制 judge context

长任务可能产生多个大文件。逐 criterion 只加载相关 deliverable，既降低 judge context，又避免无关文件“替 Agent 补答案”。这种 scoped evaluation 比把所有输出一次性打分更可追踪。

### 3. 把轨迹指标和结果指标一起保存

`documents_read / skipped`、tool counts、tokens、latency 和 transcript 让研究者能区分：

- 模型不知道答案；
- 模型没读到关键文件；
- 模型找到了事实但没写进交付物；
- 模型交付正确但成本不可接受。

这比只有一个最终 accuracy 更适合诊断 Agent 系统。

### 4. Filesystem-first 降低复现门槛

不依赖数据库或远程 orchestration service，task、run artifact 和 report 都是普通文件。它牺牲了一些在线实验管理能力，但换来了可检查、可版本化和容易扩展的基线。

## 主要限制与风险

### 1. All-pass 不天然可跨任务比较

若一条 criterion 独立通过的概率是 `p`，含 `n` 条 criteria 的 all-pass 概率近似为：

```text
P(all-pass) = p^n
```

即使 `p = 0.99`，中位任务 `n = 57` 的 all-pass 概率也只有约 `56%`；当 `n = 1,114` 时几乎为零。现实 criteria 并不独立，但这个计算揭示了同一个问题：rubric 越长、拆得越细，任务越难 all-pass。

因此 all-pass 很适合同一任务上的 model / agent A/B test，却不能不经校准就跨不同 rubric 长度的 task 排总榜。criterion 的拆分粒度还隐式决定了权重：一个风险被拆成“发现、量化、引用、建议”四条，影响就比只写一条的风险大四倍。

### 2. Rubric text 同时扮演规格和真值

没有独立 gold deliverable。优点是 authoring 成本低、适配多种工作产品；缺点是 rubric 的遗漏、歧义、错误或过度具体都会直接变成 benchmark truth。尤其法律任务可能存在多个合理 drafting choices，二元 criterion 容易把“与 rubric 作者选择不同”误判为能力不足。

### 3. Judge 是测量链路中的单点

每条 criterion 都依赖 LLM semantic judgment。temperature 0 和结构化输出改善了稳定性，但不等于校准：judge 仍可能漏看表格、误解 tracked changes、被 deliverable 文本干扰，或对边界案例前后不一致。仓库文档和代码在当前 revision 没有给出大规模 human-vs-judge agreement、跨 judge sensitivity 或 prompt-injection robustness 结果。

### 4. Synthetic realism 不等于真实分布

教程明确说明文档是在律师指导与 review 下批量合成，具有真实工作的 substantive complexity，但仍包含 imperfections，不能视为执业律师从零起草的完美文件。合成任务适合控制 ground truth、扩大覆盖，但可能留下模板痕迹、异常密集的 red flags 或与真实 data room 不同的噪声结构。

### 5. 全量评测成本很高

每条 criterion 对应一次 judge call。按当前 `111,826` 条 criteria 计算，一个模型跑完整 revision 后需要同量级的 judge calls，还不包括 Agent 自己的长轨迹推理。实际研究更可能需要分层抽样、固定小型 suite 或先用便宜 judge 筛选再做人审。

### 6. Benchmark 仍在快速迁移

README、evaluation 文档和实际 task tree 的计数已经出现漂移，新旧 schema 也并存。复现实验必须记录 commit、task IDs、judge model、skills、sandbox image 和 reasoning effort；只写“在 Harvey LAB 上测试”远远不够。

## 对 Agent 研究与工程的启发

Harvey LAB 最适合研究以下问题：

- 长 matter file 中，document coverage 与最终遗漏率之间是什么关系？
- Agent 应该一次读完全部文件，还是先建 manifest / issue map 再按需深入？
- 针对 DOCX、XLSX、PPTX 的 authoring skills 能带来多少增益？
- 更强模型、更多 turns、显式 planning、外部 memory，哪个最能提高 all-pass？
- 失败发生在 retrieval、reasoning、cross-document synthesis，还是 final-mile document production？
- 如何把 rubric 从事后 judge 规格升级成可验证的 typed assertions、交叉引用图和确定性检查？

对自己的工作流，最值得借鉴的是一个四层评测框架：

```text
任务层：instructions + source documents + expected deliverables
执行层：sandbox + tools + skills + transcript
评测层：criterion -> scoped deliverables -> verdict
分析层：all-pass + criterion pass rate + coverage + cost
```

它可以直接迁移到 research agent：输入论文与代码，要求输出笔记、证据表、复现实验和 slides；然后把“结论是否有出处”“图表数字是否一致”“限制是否覆盖”“引用是否可定位”拆成 criterion，而不是只让另一个模型给整篇笔记打一个模糊总分。

## Questions

- all-pass leaderboard 应如何按 rubric 长度、criterion 相关性和任务难度校准？
- 是否应同时报告 material-issue all-pass、格式合规率与普通 criterion pass rate？
- rubric 是否可以分成 deterministic checks、source-grounded checks 和 subjective quality checks，并为三者使用不同 evaluator？
- 怎样构造一套有真实律师交付物、隐私已处理的小型 calibration set，验证 synthetic benchmark 的外推性？
- 如何测量 judge 对 prompt injection、错误法条和有意误导的 Office 文档的鲁棒性？
- Agent 不读取 rubric 是必要约束；但在真实工作里，review checklist 通常可见。是否应该同时评测 rubric-hidden 与 checklist-visible 两种设定？
