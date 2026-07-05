# AI Coding 最大的问题：不知道自己不知道什么

- Source blog: [Know your unknowns](https://thariqs.github.io/html-effectiveness/unknowns/)
- Related note: [../../../notes/2026_know-your-unknowns-html-effectiveness.md](../../../notes/2026_know-your-unknowns-html-effectiveness.md)
- Open-source skill: [rxlqn/know-your-unknowns](https://github.com/rxlqn/know-your-unknowns)
- Target audience: AI 硕博 / 大厂 AI 工程师 / AI coding 重度用户
- Draft status: draft
- Suggested format: 8-10 张图 + 正文
- Keywords: AI coding, Claude Code, Codex, agent workflow, unknown unknowns, code review, prompt engineering

## 0. 一句话定位

这篇 blog 讲的不是“让 Claude 生成 HTML”，而是一个更重要的 AI coding 工作流：在写代码前、写代码中、写完代码后，用不同 artifact 把 unknown unknowns 暴露出来，避免 agent 默默替你猜。

## 1. 标题候选

- AI Coding 最大的问题：不知道自己不知道什么
- 别急着让 Agent 写代码：先让它暴露 unknown unknowns
- 这篇 blog 讲清了 AI coding 的 11 种“发现未知”方法
- Claude / Codex 提效：写代码前、中、后分别该让 Agent 做什么
- AI Agent 协作里，最重要的不是 prompt，而是发现未知

## 2. 开头 Hook

很多 AI coding 失败，并不是模型不会写代码。

更常见的是：需求里有关键决策没人说，代码库里有历史坑没人提醒，用户自己也不知道该怎么描述想要的东西。

这类问题最麻烦的地方在于：它们不是 known unknowns，而是 unknown unknowns。

也就是：你不知道自己不知道什么。

原始 blog `Know your unknowns` 最有价值的地方，是把这件事拆成 3 个阶段、11 种方法：

```text
Pre-implementation: 写代码前发现未知
During implementation: 写代码中记录未知
Post-implementation: 写完后验证未知是否消化
```

下面这 11 个方法，我觉得比“再写长一点 prompt”更适合提升 AI coding 的稳定性。

## 3. 总体框架：三个阶段

### 阶段一：Pre-implementation

目标：在写代码之前，把最贵的猜测提前暴露。

这是最重要的阶段，因为这时候改方向成本最低。

### 阶段二：During implementation

目标：实现过程中，计划撞上代码现实后，不要让新发现消失在终端 scrollback 里。

### 阶段三：Post-implementation

目标：代码写完后，处理 reviewer / stakeholder / 未来 on-call 的未知，并验证自己真的理解了改动。

## 4. 阶段一：Pre-implementation 的 8 种方法

### 1. Blindspot pass

适合场景：你要改一个不熟悉但高风险的模块，比如 auth、billing、permission、migration、data lifecycle。

不要一上来让 agent 写代码。先让它扫真实代码、测试、迁移、历史 PR，告诉你：

- 你可能会误以为什么；
- 实际代码/历史是什么；
- 哪些旧实现不能复制；
- 哪些边界需要特别遵守；
- 下一轮 implementation prompt 应该怎么写得更准确。

它解决的 unknown 是：代码库里的隐藏事实和历史坑。

一句话：

> 在陌生模块里，先让 agent 找“你会踩的坑”，再让它写代码。

### 2. Teach me my unknowns

适合场景：你知道自己想要一个结果，但缺领域词汇。

比如你只能说：

```text
这个视频调得更专业一点
这个页面更高级一点
这个策略更稳一点
```

但你不知道专业的人会怎么描述。

这时不要直接让 agent 做结果，而是先让它教你：

- 这个领域的 mental model；
- 从入门到专业的 vocabulary ladder；
- 好结果 / 坏结果怎么判断；
- 哪些参数可以调；
- 下一轮 prompt 应该怎么写。

它解决的 unknown 是：用户自己无法表达需求。

一句话：

> 先让 agent 教你怎么提需求，再让它完成需求。

### 3. Four design directions

适合场景：你说不清想要什么设计，但看到具体方案后能判断。

原文的做法是：不要让 agent 只给一个设计，而是用同一份内容生成 4 个互斥方向。

比如同一个 dashboard，可以做成：

- dense ops console；
- airy editorial cards；
- kanban / timeline hybrid；
- brutalist terminal style。

关键是：内容相同，设计哲学不同。

这样用户反应的不是随机细节，而是信息密度、视觉性格、工作流假设。

它解决的 unknown 是：审美和信息架构偏好。

一句话：

> 反应比想象容易。让用户从几个方向里 steal / skip。

### 4. Mock before you wire

适合场景：UI / interaction 还不确定，但一旦接真实 app，返工成本会变高。

原文例子是 frame annotation toolbar。先做一个 throwaway HTML mock，用假数据模拟真实状态，让用户点击、切换、反馈。

这个阶段要问的是高杠杆问题：

- toolbar 放在哪里？
- 控件是常驻还是 popover？
- drawer 是 overlay 还是 push layout？
- 哪些能力进 v1，哪些先不要？

它解决的 unknown 是：真实可点击之后才知道的交互偏好。

一句话：

> 先 mock，再 wire。别把设计探索写进真实代码里。

### 5. Brainstorm the intervention

适合场景：你只有一个大问题，比如 churn、activation、质量下降、成本过高，但不知道该从哪里下手。

普通做法是：agent 直接给一个方案。

更好的做法是：让它基于真实代码和产品表面，展开 8-12 个 intervention，从最便宜到最激进排序。

每个 intervention 都应该包含：

- 代码或产品证据；
- 预期影响；
- 实现成本；
- 风险；
- ship 之后能学到什么。

它解决的 unknown 是：可干预空间在哪里。

一句话：

> 先看完整 option space，再决定做哪个 solution。

### 6. The interview

适合场景：需求很模糊，直接实现会导致 agent 默默猜架构。

重点不是“列一堆问题”，而是一次只问一个，并按 architecture blast radius 排序。

比如 annotation export，真正会改变架构的问题是：

- export 在 API request、background job，还是 client-side 跑？
- export unit 是单个 video、整个 project，还是任意 selection？
- drawings 是否导出？
- timestamp 用 milliseconds 还是 SMPTE timecode？
- 谁有 export 权限？

这些不是小细节，而是会影响 data model、job queue、API contract、permission boundary。

它解决的 unknown 是：哪些需求答案会改变系统架构。

一句话：

> 模糊需求不要让 agent 猜，让它按架构影响力 interview 你。

### 7. Point at a reference

适合场景：你有一个参考实现，要 port / reimplement。

最危险的不是代码长得不一样，而是语义悄悄变了。

所以写代码前要让 agent 做 semantics map：

- reference 的行为是什么；
- 哪些语义必须 preserved exactly；
- 哪些是 deliberately changed；
- 哪些 dropped / out of scope；
- edge cases 下预期行为是什么。

典型坑包括：

- integer division；
- inclusive range；
- monotonic clock；
- retry budget；
- async boundary；
- idempotency；
- cancellation。

它解决的 unknown 是：参考实现的真实语义边界。

一句话：

> Port 之前先确认语义，不要等 diff 写完再猜有没有变。

### 8. The tweakable plan

适合场景：你需要实现计划，但里面混着“人类要拍板的决策”和“agent 可以自己做的机械活”。

普通 plan 会按执行顺序写：

```text
1. 改 schema
2. 改 API
3. 改 UI
4. 加测试
```

但用户真正需要先看的，往往是：

- data model 怎么设计；
- type interface 怎么定；
- UX flow 怎么走；
- 是否需要 background job；
- migration / rollout 风险是什么。

所以 tweakable plan 应该按“用户最可能想改哪里”排序，而不是按执行顺序排序。

它解决的 unknown 是：计划里哪些决策最该被人类注意。

一句话：

> 把高可变决策放到顶部，把机械重构埋到底部。

## 5. 阶段二：During implementation 的 1 种方法

### 9. Implementation notes

适合场景：长时间实现、跨模块实现、代码现实很可能改变计划。

这是我觉得最适合 AI coding 的方法之一。

因为真实实现中一定会发生：

- 某个字段其实 stale；
- legacy data 缺关键字段；
- 已有工具已经能做，不需要新依赖；
- permission 边界比预想复杂；
- 某个需求只能先保守处理。

如果这些发现只在终端里闪过，后面就很难复盘。

implementation notes 要记录：

```text
原计划是什么
代码现实是什么
agent 做了什么保守选择
哪些点需要人之后判断
下一版 plan 应该怎么修
```

它解决的 unknown 是：计划和代码现实碰撞后产生的新事实。

一句话：

> 不要只看最终 diff。实现过程中的 deviation，往往才是最有价值的系统知识。

## 6. 阶段三：Post-implementation 的 2 种方法

### 10. The buy-in doc

适合场景：功能已经做完，但需要工程、设计、安全、PM 或运营相信现在可以 ship。

它不是 PR 描述。

PR 描述通常解释“我改了什么”。buy-in doc 要回答“为什么现在可以发”。

结构应该是：

- 先放 demo / 用户可见结果；
- 解释问题和价值；
- 提前回答 reviewer objections；
- 给 spec at a glance；
- 写清 risk、rollback、blast radius；
- 指名谁需要 approve 什么。

它解决的 unknown 是：其他人会担心什么，以及谁负责签哪一块。

一句话：

> 写完代码后，最后一个未知是 other people。

### 11. Quiz me before I merge

适合场景：大 diff、跨系统改动、权限/异步/数据生命周期复杂，reviewer 不能只靠扫 diff。

这个方法很有意思：让 agent 生成 merge-readiness report，并在最后放一个必须答对的 quiz。

quiz 不应该考 trivia，而应该考 incident / review 中必须做对的判断。

例如：

- worker crash 后 job 怎么恢复？
- signed URL 过期但 object 还在怎么办？
- session middleware 变化为什么会影响 download access？
- 一个 job processing 15 分钟说明什么？
- 分辨率变了，annotation 为什么还能落在正确位置？

答不出来，就说明你还没真正理解这个 change。

它解决的 unknown 是：你是否真的理解了系统行为。

一句话：

> “我看过 diff”不等于“我理解这个系统会怎么运行”。

## 7. 我自己的理解

这篇 blog 最值得学的不是 HTML。

HTML 只是一个很好用的 artifact 形式，因为它能做：

- 并排对比；
- 点击选择；
- 状态切换；
- 逐步 interview；
- quiz feedback。

但核心不是 HTML，而是 artifact-first。

也就是：

```text
在正确阶段，用正确 artifact，把 agent 不该猜的东西暴露出来。
```

很多时候这个 artifact 可以只是 Markdown table、checklist、diagram、interview log、implementation notes。

真正重要的是：它能不能让 hidden assumption 变成 explicit decision。

## 8. 顺手整理了一个 Skill

我把这 11 个方法整理成了一个 Codex / Claude 可用的开源 skill，主要是方便自己以后在 AI coding 里复用。

```text
github.com/rxlqn/know-your-unknowns
```

这个不是重点，重点还是推荐大家去看原 blog。

## 9. 配图规划

| 图 | 内容 | 目的 |
|---|---|---|
| 1 | 封面：AI Coding 最大的问题：不知道自己不知道什么 | Hook |
| 2 | 三阶段框架：Pre / During / Post | 建立结构 |
| 3 | Pre-implementation 8 方法总览 | 展示写代码前的完整工具箱 |
| 4 | Blindspot pass / The interview 示例 | 展示最常用的两个 |
| 5 | Point at a reference / Tweakable plan 示例 | 展示架构和语义边界 |
| 6 | Implementation notes 模板 | 展示实现中怎么记录偏差 |
| 7 | Buy-in doc / Merge quiz | 展示写完后怎么验证理解 |
| 8 | Artifact-first, not HTML-first | 提炼方法论 |
| 9 | 原 blog + skill 链接 | 给读者可继续阅读/复用的入口 |

## 10. 正文草稿

### Copy-ready version

AI Coding 最大的问题，不是模型不会写代码。

更常见的问题是：你不知道自己不知道什么，agent 也不知道它该问什么。

比如：

- 需求里有关键架构决策没人说；
- 代码库里有历史坑 agent 没看到；
- 用户自己缺领域词汇，说不清想要什么；
- 实现过程中计划和代码现实不一致；
- PR 合并前，reviewer 其实没理解系统行为。

我最近看到一篇很有启发的 blog：`Know your unknowns`。

它把 AI coding 里的 unknown unknowns 拆成 3 个阶段、11 种方法。

### 阶段一：Pre-implementation

目标是在写代码之前，把最贵的猜测提前暴露。

1. Blindspot pass
不熟悉代码库时，先让 agent 找“你会踩的坑”，而不是直接写代码。

2. Teach me my unknowns
用户缺领域词汇时，先让 agent 教你怎么描述需求。

3. Four design directions
UI 方向不清楚时，用同一份内容生成几个互斥方向，让用户通过反应来选择。

4. Mock before you wire
复杂交互先做 throwaway mock，不要一上来接真实 app。

5. Brainstorm the intervention
问题很大时，比如 churn / activation / cost，先展开干预空间。

6. The interview
需求含糊时，让 agent 按 architecture blast radius 一次问一个问题。

7. Point at a reference
port 参考实现时，先做 semantics map，确认语义再写代码。

8. The tweakable plan
实现计划不要按执行顺序写，而要先暴露用户最可能要改的决策。

### 阶段二：During implementation

9. Implementation notes

长时间实现时，计划一定会撞上代码现实。

比如字段其实 stale、legacy data 缺字段、已有工具已经够用、permission 边界比预期复杂。

这些发现如果只留在终端 scrollback 里，很快就丢了。

implementation notes 要记录：

```text
原计划是什么
代码现实是什么
agent 做了什么保守选择
哪些点需要人之后判断
下一版 plan 应该怎么修
```

这记录的不是流水账，而是计划和真实系统碰撞产生的知识。

### 阶段三：Post-implementation

10. The buy-in doc

功能做完后，把 prototype、spec、implementation notes 包成一个给 stakeholder 看的 doc。

重点不是解释 diff，而是回答：为什么现在可以 ship？

11. Quiz me before I merge

大 PR 合并前，让 agent 生成 merge-readiness report 和 quiz。

quiz 不应该考 trivia，而应该考 incident 中必须做对的系统判断：

- worker crash 后 job 怎么恢复？
- signed URL 过期但 object 还在怎么办？
- session middleware 变化为什么会影响 download access？
- 一个 job processing 15 分钟说明什么？

如果答不出来，就说明你还没真正理解这个 change。

我觉得这篇 blog 最值得学的，不是 HTML。

HTML 很有用，因为它能做对比、点击、状态切换、interview 和 quiz feedback。

但核心是 artifact-first：

```text
在正确阶段，用正确 artifact，把 agent 不该猜的东西暴露出来。
```

这个 artifact 可以是 Markdown table、checklist、diagram、interview log、implementation notes，也可以是交互式 HTML。

一句话总结：

AI coding 提效，不只是写更长 prompt。

更重要的是：把 hidden assumptions 变成 explicit decisions。

原 blog：

```text
thariqs.github.io/html-effectiveness/unknowns/
```

我顺手把这 11 个方法整理成了一个 Codex / Claude skill：

```text
github.com/rxlqn/know-your-unknowns
```

## 11. 评论区/主页链接文案

```text
Blog: thariqs.github.io/html-effectiveness/unknowns/
Skill: github.com/rxlqn/know-your-unknowns
```
