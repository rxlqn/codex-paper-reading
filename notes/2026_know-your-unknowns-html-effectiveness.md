# Know Your Unknowns: HTML Artifacts for Surfacing Unknowns

- Year: 2026
- Date: 2026-07-05
- Source type: web examples gallery / agent workflow note
- Author / source: Thariq Shihipar site; examples marked Copyright 2026 Anthropic PBC
- Links:
  - Main page: https://thariqs.github.io/html-effectiveness/unknowns/
  - Parent gallery: https://thariqs.github.io/html-effectiveness/
  - GitHub repo: https://github.com/ThariqS/html-effectiveness
- Tags: human-agent-collaboration, agent-workflow, HTML-artifacts, prompt-engineering, code-review, product-design
- Status: read
- One-line takeaway: 这组例子不是在说“HTML 更漂亮”，而是在说：当 agent 协作里有未知时，临时 HTML artifact 可以把未知按实现前、实现中、实现后三个阶段变成可看、可点、可选择、可验证的决策界面。

## Problem

和 agent 协作时，很多失败不是模型完全不会做，而是人和模型都没有意识到哪些信息会改变结果。真实代码里的历史债、迁移状态、权限边界、领域词汇、视觉偏好、参考实现语义、reviewer 关切、merge 后的事故处理知识，都可能在一开始没有被表达出来。

普通 Markdown 适合沉淀结论，但它是线性的。这个网页展示的 11 个 HTML artifact 更像一次性的协作界面：让 agent 把问题、候选方案、风险、交互原型、选择按钮、复制回复、测验反馈放在同一页里，帮助用户在写代码前、中、后分别发现不同类型的 unknown。

## 三阶段总览

原网页把 11 个 demo 分成三个阶段：

| Stage | Count | 主要目标 | 对应 unknown |
|---|---:|---|---|
| Pre-implementation | 8 | 写代码前尽量把未知转成明确决策 | 代码库盲点、领域词汇、视觉方向、需求边界、参考语义、计划中的可变决策 |
| During implementation | 1 | 实现过程中记录计划撞上现实后的偏差 | 代码现实、旧数据、已有工具、权限细节、保守选择 |
| Post-implementation | 2 | 交付前处理他人的未知，并验证自己真的理解改动 | reviewer objections、stakeholder sign-off、merge readiness、事故时的判断 |

核心不是“生成 HTML”这个动作本身，而是每个 artifact 都有明确的决策闭环：

1. 先把未知暴露出来。
2. 让用户通过点击、选择、回答或测验表达判断。
3. 把用户判断折回下一轮 prompt、实现计划、review 文档或 merge checklist。

## 阶段一：Pre-implementation

写代码前是发现未知成本最低的时候。这个阶段的 8 个 demo 可以再分成四类：代码库盲点、领域/审美学习、方案探索、架构决策确认。

### 1. Blindspot pass

- Page: https://thariqs.github.io/html-effectiveness/unknowns/01-blindspot-pass.html
- Trigger: 要改一个自己不熟悉、但风险很高的模块，例如 auth、billing、permission、migration、data lifecycle。
- Artifact shape: 代码库扫描报告 + blindspot cards + 每个 blindspot 对应的 prompt 修正 + 最后一版完整 implementation prompt。
- Example domain: 新增 SSO auth provider。

这个 demo 的重点不是让 agent 泛泛说“注意安全和测试”，而是让它先扫真实代码、历史提交、迁移和旧 PR，找出如果直接实现会踩的具体坑。页面里列出的 blindspots 包括：

- session 写入正在 Redis / Postgres 双写迁移中，应该走桥接层而不是直接写最新的 store；
- 最容易复制的 SAML provider 其实绕过了 auth middleware，不能当模板；
- 旧 PR 已经尝试过类似功能并被 revert，原因仍然成立；
- provider interface 看不出 identity linking 的真实风险；
- dev / prod feature flag 行为不同，refresh-token 逻辑可能在生产无效；
- provider 上线需要 registry、DB migration、contract fixture 三步；
- logout 不是简单删 session，还要发布 revocation event，否则其他长连接授权不会释放。

它暴露的 unknown 是“代码库事实”和“历史事实”。人类一开始以为任务是实现接口，实际任务是穿过迁移状态、绕开坏模板、尊重 identity model 和事件边界。

可迁移做法：

- 让 agent 在写代码前产出“你以为是什么 / 实际是什么”的对照。
- 每个 blindspot 必须落到文件路径、历史原因、为什么会坑、下一轮 prompt 约束。
- 最后把所有发现折成一个新的 implementation prompt，而不是只留一份风险列表。

### 2. Teach me my unknowns

- Page: https://thariqs.github.io/html-effectiveness/unknowns/02-color-grading-explainer.html
- Trigger: 用户知道目标，但缺领域词汇，只能说“做得更专业一点”。
- Artifact shape: mental model + vocabulary ladder + 可调 demo + 好坏判断标准 + 更专业的下一轮 prompt。
- Example domain: color grading。

这个 demo 用 color grading 说明：当用户缺少领域语言时，agent 应该先把用户教到“能提出正确需求”的程度。页面不是百科式解释，而是组织成一套可操作的学习 artifact：

- mental model: ingest -> correct -> grade -> match；
- vocabulary ladder: exposure、white balance、contrast curve、lift/gamma/gain、vibrance、LUT；
- live frame controls: 通过 slider / preset 感受参数变化；
- quality checklist: 肤色、黑位、白场、shot-to-shot consistency、look 是否服务故事；
- payoff: 把“make it nicer”升级成具体专业要求。

它暴露的 unknown 是“我不知道该怎么描述我要的东西”。对 agent 工作流来说，这类页面适合放在设计、视频、法律、金融、实验分析等专业词汇密集的任务前。

可迁移做法：

- 先要求 agent 生成领域词汇阶梯，而不是直接做结果。
- 每个术语都要绑定一个“可用于 prompt 的句子”。
- 如果可能，用交互控件让用户体验参数，而不是只读定义。

### 3. Four design directions

- Page: https://thariqs.github.io/html-effectiveness/unknowns/03-design-directions.html
- Trigger: 用户无法描述想要什么设计，但看见具体方案后能判断。
- Artifact shape: 同一份数据渲染成四种互斥设计方向 + steal / skip chips + 自动生成回复。
- Example domain: review queue dashboard。

这个 demo 用同一份 review queue 数据做四种设计方向：

- Dense ops console: 密集、指标优先、适合运营监控；
- Airy editorial cards: 安静、低密度、强调阅读感；
- Kanban / timeline hybrid: 把 review 当成流转中的 pipeline；
- Brutalist mono terminal: 极简、键盘优先、信息压缩。

页面故意让四个方向“互斥”，这样用户反应的是设计哲学，而不是某个随机样式细节。每个方向下面都有可点击的 steal / skip 选项，最后能拼成回复：选一个主方向，再从其他方向拿 2-3 个细节。

它暴露的 unknown 是“审美和信息架构偏好”。这类未知靠语言很难一次说清，但靠并排原型很容易反应。

可迁移做法：

- 固定数据，只改变设计哲学，避免变量混在一起。
- 不要只给一个 mock；给几个明显不同的方向让用户排除。
- 让 HTML 收集 steal / skip，输出下一轮设计 prompt。

### 4. Mock before you wire

- Page: https://thariqs.github.io/html-effectiveness/unknowns/04-toolbar-mock.html
- Trigger: UI 要接真实代码前，布局、控件密度、交互模式还不确定。
- Artifact shape: 静态假数据 mock + 可切换布局 + 高杠杆问题 + 自动回复模板。
- Example domain: frame annotation toolbar。

这个 demo 明确强调“先不要碰真实 app”。页面用假数据做了一个 frame annotation toolbar，可以切换三种放置方式：

- floating pill；
- docked left rail；
- under seekbar。

它还把几个最容易导致返工的问题提前问出来：toolbar 放在哪里、颜色/笔触控制是常驻还是 popover、comments drawer 是 overlay 还是 push layout、需要 server-side export 的 blur tool 要不要进 v1。

它暴露的 unknown 是“真实可点击之后才知道的 UX 偏好”。如果直接 wire 到 app，返工会发生在 PR 后；用 throwaway HTML，返工发生在原型阶段。

可迁移做法：

- 静态 HTML 可以完全不接真实数据，但必须模拟真实密度和真实状态。
- open questions 应该聚焦那些一旦选错就会改结构的点。
- artifact 应该能生成“我选择 A/B/C，因为...”的回复，而不是只展示 mock。

### 5. Brainstorm the intervention

- Page: https://thariqs.github.io/html-effectiveness/unknowns/05-churn-brainstorm.html
- Trigger: 只有粗略问题，例如 churn、activation、质量下降、成本过高，但不确定该从哪里下手。
- Artifact shape: 基于代码库的 10 个 intervention，按成本从今天能 ship 到季度级项目排列，每项带代码证据、影响、可勾选反馈。
- Example domain: onboarding 后用户流失。

这个 demo 不是让 agent 直接选一个方案，而是先把干预空间展开。页面列出的 10 个方向从轻到重包括：

- 修 dead-end empty state；
- 打开已有 sample project；
- 在 app 内展示 pending invites；
- 把已有 milestones 表接成 dashboard checklist；
- 设计 first-comment moment；
- 处理 silently failed first uploads；
- 促成团队第一次 watch-together；
- 提供 review workflow templates；
- 内置屏幕/摄像头录制；
- 做 first-class client review portals。

每项都不是空想，而是绑定到已经存在的文件、flag、TODO、表或服务能力。这样用户可以根据“这是否 resonate”来选择方向。

它暴露的 unknown 是“可干预点在哪里”。很多时候系统里已有 70% 的能力，只是没有连到用户路径。

可迁移做法：

- 先扫描代码库，再 brainstorm；不要只凭产品直觉。
- 方案按成本和野心排序，帮助用户选粒度。
- 每个方案都要有代码证据和预期影响，避免变成普通脑暴。

### 6. The interview

- Page: https://thariqs.github.io/html-effectiveness/unknowns/06-interview.html
- Trigger: 需求仍然含糊，直接实现会让 agent 默默猜关键架构决策。
- Artifact shape: 一次一个问题的 interview UI + 进度 rail + radius 标签 + decision table + 生成后续 implementation prompt。
- Example domain: annotation export spec。

这个 demo 的细节很重要：它不是“列一堆澄清问题”，而是按 blast radius 排序，一次只问一个。页面中的 7 个问题覆盖：

- export generation 跑在哪里：API request、background job、client-side；
- export unit 是单个 video、整个 project，还是任意选择；
- drawings 是否导出，导出为 placeholder、rasterized PNG，还是 raw stroke JSON；
- timestamp 用 SMPTE timecode、milliseconds，还是两列都保留；
- v1 支持哪些格式：CSV、PDF、NLE marker、组合方案；
- 谁能 export：所有 viewer、project members、owner/admin；
- 文件如何命名和打包。

前四个属于 architecture / data model 决策，后几个更多是 UX / policy。artifact 最后把回答生成 decisions table 和一段 ready-to-implement prompt，并提醒后续实现要把 1-4 当固定约束。

它暴露的 unknown 是“哪些问题会改变架构”。普通澄清容易把命名问题和数据模型问题混在一起；这个 artifact 用 blast radius 排序，先解决最贵的猜测。

可迁移做法：

- 让 agent 每次只问一个问题，且说明为什么这个答案会改变架构。
- 问题要分层：architecture、data、UX、policy。
- 最后必须输出 decisions table 和 implementation prompt。

### 7. Point at a reference

- Page: https://thariqs.github.io/html-effectiveness/unknowns/07-reference-port.html
- Trigger: 有一个参考实现已经定义了正确语义，目标是 port / reimplement，而不是自由发挥。
- Artifact shape: semantics map + side-by-side source/target snippets + gotcha notes + preserved / changed / dropped table + edge-case table + sign-off prompt。
- Example domain: Rust rate limiter port 到 TypeScript API client。

这个 demo 要求 agent 在写代码前证明自己理解参考实现。页面先总结 Rust crate 的真实语义：

- token bucket admission；
- lazy refill 且整数截断；
- decorrelated jitter backoff；
- retry budget；
- failure classification 由 caller 负责。

然后 side-by-side 对照 Rust 和 TypeScript 的 proposed implementation，逐条标出容易丢的语义，例如：

- `Instant` / `performance.now()` 的 monotonic clock 关系；
- Rust 整数除法需要 JS `Math.floor` 恢复；
- `last_refill` 只能在 mint whole token 后推进；
- Rust inclusive range 要在 JS 里处理右端点；
- retry budget 的 check-and-debit 不能被未来 `await` 打断；
- budget exhausted 时应该 refuse，不应该 queue。

最后它把行为分成 preserved exactly、deliberately changed、dropped，并列 edge cases，例如 clock skew、burst at t=0、budget exhaustion、slow drip、delay at cap。

它暴露的 unknown 是“语义等价不是代码形状等价”。如果直接 port，很容易在一个小的语言差异上改变系统行为。

可迁移做法：

- port 前先要 semantics map，而不是先写代码。
- 必须明确 preserved / changed / dropped，尤其是 deliberate changes。
- 用 edge-case table 让人类签字确认语义，而不是只 review diff。

### 8. The tweakable plan

- Page: https://thariqs.github.io/html-effectiveness/unknowns/08-implementation-plan.html
- Trigger: 已经要进入实现，但计划里有些决策需要人类判断，有些只是机械活。
- Artifact shape: 按 likelihood-of-tweaking 排序的 plan；Section A 放高可变决策，Section B 放执行顺序，Section C 折叠机械任务。
- Example domain: annotation export implementation plan。

这个 demo 反过来组织实现计划：不是按执行顺序，而是按“用户最可能想改哪里”。页面顶部先展示 effort、files touched、risk、migration，再把 plan 分成：

- A: Decisions you probably want to change；
- B: Sequencing；
- C: Mechanical work。

Section A 里的关键决策包括：

- data model 是否新增 `annotation_exports`，snapshot 还是 live join；
- artifact 是 render once + store，还是 on-demand render；
- `ExportRequest` / `AnnotationSnapshot` 类型里哪些字段是 v1 scope；
- UX 是 background job + toast，还是 hybrid wait。

机械任务如迁移 fixture、注册 worker、生成 OpenAPI client、搬 timecode helper，则被放到底部。

它暴露的 unknown 是“计划里哪些点值得人类 attention”。普通 execution plan 会把关键产品/架构选择埋在步骤 3 或步骤 5；这个 artifact 先让用户改最可能要改的点。

可迁移做法：

- 把 plan 分成 human-tweakable decisions 和 trusted mechanical work。
- 每个选择都要有 alternative、cost、何时该选另一边。
- 让用户可以直接复制几条高杠杆修改意见。

## 阶段二：During implementation

实现中只有 1 个 demo，但它是整个 workflow 的中间胶水：计划再好，代码现实还是会逼出偏差；关键是不要让偏差消失在终端 scrollback 里。

### 9. Implementation notes

- Page: https://thariqs.github.io/html-effectiveness/unknowns/09-implementation-notes.html
- Trigger: 长时间实现、跨模块实现、计划和代码现实很可能冲突。
- Artifact shape: running log + plan-confirmed / discovery / deviation / todo for human 分类 + conservative choice + revisit + fold-back bullets。
- Example domain: export feature 3-hour build。

这个 demo 让 agent 在 build 过程中维护一份 implementation notes。页面不是完成后总结，而是按时间记录：

- 14:02: job model 和 queue wiring 按计划；
- 14:18: API endpoint 按计划；
- 14:29: 发现 `Review.duration_ms` 可能 stale，因此读 `MediaAsset.probe.duration`；
- 14:41: legacy annotations 缺 `frame_ts`，因此视频 burn-in 排除但 CSV sidecar 保留；
- 15:22: BullMQ 会把 `Date` payload JSON round-trip 成 string，因此 payload 类型改为 ISO string；
- 15:48: 已有 zip streaming 工具，放弃新增 `archiver` dependency；
- 16:07: 已有 websocket progress convention，复用并保留 polling fallback；
- 16:20: guest reviewer 能看 review 但不能下载资产，因此 export policy 先保守 403；
- 16:33 / 16:51: 留给 human 判断 guest export policy 和 retention window。

每个 deviation 都有四段：what plan said、what code revealed、conservative choice、revisit。最后还有 fold back into the plan，把这些发现转成下一次计划或 prompt 的三条约束。

它暴露的 unknown 是“计划遇到代码现实后的事实”。这类事实通常最值钱，因为它来自真实代码，不来自设计想象。

可迁移做法：

- 实现前要求 agent 维护 `implementation-notes`。
- 遇到偏差时允许 agent 做保守选择继续推进，但必须记录为什么。
- 最后必须把 discoveries 折回下一版 plan，而不是只作为流水账。

## 阶段三：Post-implementation

写完代码后，未知不会消失，只是从“我和 agent 不知道”变成“reviewer、stakeholder、未来 on-call 不知道”。这个阶段的两个 demo 分别处理外部 buy-in 和 merge 前自测理解。

### 10. The buy-in doc

- Page: https://thariqs.github.io/html-effectiveness/unknowns/10-pitch-doc.html
- Trigger: 功能已做完，需要让工程、设计、安全、PM 或运营相信现在可以 ship。
- Artifact shape: Slack-ready ship proposal + demo-first + pitch + reviewer objections + spec summary + risk/rollback + named sign-offs。
- Example domain: annotation export ship proposal。

这个 demo 的结构非常具体：

- 先放 90 秒内能看懂的 demo，而不是先讲背景；
- pitch 说明用户痛点、支持数据、已完成状态、flag/rollback；
- reviewer objections 提前回答权限、性能、格式选择、基础设施、合规；
- spec at a glance 总结 formats、entry point、permissions、limits、telemetry、feature flag ramp；
- risk & rollback 写清楚一键回滚、blast radius、最坏可信失败、已知 gap；
- what I need from you 指名每个 approver：工程看 worker pool，安全看 visibility/audit，设计看 modal/error states，PM 看 ramp/changelog。

它暴露的 unknown 是“别人会问什么，以及谁需要为哪一块负责”。很多 PR 描述只解释自己做了什么，但 buy-in doc 是面向决策：为什么现在可以发，谁还需要签字。

可迁移做法：

- 面向 stakeholder，而不是面向 diff。
- 先给 demo，再给证据。
- 把 objection 写成 Q/A，并链接到 spec、implementation notes、metric 或 test。
- 明确每个 sign-off 的 owner 和范围。

### 11. Quiz me before I merge

- Page: https://thariqs.github.io/html-effectiveness/unknowns/11-change-quiz.html
- Trigger: 大 diff、跨系统改动、权限/异步/数据生命周期等容易在事故中被误判的 change。
- Artifact shape: merge readiness report + mental model diagram + non-obvious behaviors + inherited assumptions + 必须通过的 quiz + checklist。
- Example domain: server-side clip export。

这个 demo 不是普通总结，而是把“我看过 diff”变成“我能在 incident / review 中做对判断”。页面先给 mental model：以前浏览器用 `MediaRecorder` 在 tab 内渲染并上传，现在客户端只发 export request，新 worker server-side render，client 轮询 job，最后拿 signed URL。

它特别列出三个 non-obvious behaviors：

- export 用原始上传素材，而不是 reviewer 看到的 720p proxy；
- worker crash 后不是立即 retry，而是等 visibility timeout 释放锁，再由其他 worker 接手；
- signed URL 24h 过期，但底层 object 保留 7 天，可以重新 mint URL。

还列了一个 inherited assumption：download endpoint 复用 workspace membership middleware；如果 guest reviewer session 规则变了，export access 也会跟着变。

quiz 的问题不是 trivia，而是模拟事故判断：

- 为什么 export 看起来和 reviewer 看到的不一样；
- worker OOM 后 job 怎么恢复；
- Slack 里的 export link 三天后过期该怎么处理；
- guest session 相关 ticket 为什么让这个 diff 更 risky；
- processing 15 分钟意味着什么；
- resolution 变了 annotation 为什么还能落在正确位置。

它暴露的 unknown 是“我是否真的理解系统行为”。如果答不对，就应该回去读对应 section，而不是凭感觉 merge。

可迁移做法：

- 大 PR 可以让 agent 生成 merge-readiness report 和 quiz。
- quiz 题应该对应 incident decision，不应该考记忆细节。
- 错题要指回具体 report section。
- checklist 应包含 CI、migration、merge note、deploy watch、已知依赖。

## 跨阶段模式

这 11 个 demo 其实形成了一个完整闭环：

| 阶段 | 人类输入弱点 | HTML artifact 的补强方式 | 输出给下一步 |
|---|---|---|---|
| Pre | 不知道要问什么、不知道怎么描述、没看见方案空间 | 扫代码、教学、并排原型、逐问 interview、semantics map、tweakable plan | 更好的 prompt、明确决策、可实现计划 |
| During | 计划和代码现实发生偏差 | 记录 discovery/deviation/conservative choice/revisit | attempt #2 prompt、计划修正、待人类判断项 |
| Post | reviewer / future maintainer 不知道系统行为 | demo-first buy-in、objection Q/A、merge quiz | sign-off、merge confidence、事故判断模型 |

## Useful For

- 复杂 repo 的 Codex / Claude Code / Cursor agent workflow。
- 高风险改动前的 blindspot pass。
- UI / 产品方向还不清楚时的 throwaway HTML prototype。
- 跨语言 port 或参考实现迁移。
- 长时间实现里的 deviation log。
- PR 前的 buy-in doc 和 merge readiness quiz。
- 论文分享或技术内容策划：可以先用 HTML 做多种讲法/图示方向，再沉淀成 Markdown。

## My Takeaways

1. 这不是“HTML 替代 Markdown”。Markdown 适合长期沉淀；HTML 适合短期决策和交互确认。
2. Pre-implementation 的 8 个模式最值得常用，因为越早把未知暴露出来，返工越便宜。
3. 最强的 artifact 都不是泛泛总结，而是能生成下一步输入：prompt、decision table、reply template、implementation plan、quiz checklist。
4. 对复杂代码库，`blindspot pass` 和 `implementation notes` 应该成为默认要求；它们把隐藏在历史和代码现实里的系统知识显性化。
5. 对大 PR，`quiz me before I merge` 比普通 summary 更有价值，因为它验证的是可操作理解，而不是阅读完成感。

## Caveats

- HTML artifact 应该是 disposable interface，不一定要长期维护。
- 如果 artifact 没有基于真实代码、真实数据或真实约束，它会变成漂亮但无用的 mock。
- 简单任务不需要这套流程；成本应该和未知程度、风险、返工成本匹配。
- 安全敏感场景要避免运行未知脚本；优先静态、自包含、无外部依赖的 HTML。

## Questions

- 是否应该在这个仓库新增 `templates/agent-workflow/`，沉淀 blindspot pass、interview、implementation notes、merge quiz 的 prompt 模板？
- 对 DeepGrounding / InformationMaster 这种复杂代码库，哪些任务应该强制先做 blindspot pass？
- 论文分享时能不能先生成 four directions：偏方法、偏工程、偏实验、偏争议，然后再选一种路线写小红书稿？

## Related

- [Human-Agent Collaboration](../topics/human-agent-collaboration.md)
