# Shares

这里放论文分享材料。建议区分不同发布场景：组会/内部分享可以偏 slides，小红书分享要更像一篇专业短文，重点是把问题定义、方法、公式、know-how 和 takeaways 讲清楚。

```text
shares/
  xiaohongshu/                 # 小红书专业图文分享
    README.md
    drafts/
    published/
    figures/
  YYYY-MM-topic/               # 组会、读书会、内部分享
    outline.md
    slides.md
    references.md
    figures/
```

## 小红书分享定位

目标读者默认是 AI 方向硕博、大厂算法/平台/应用工程师、关注前沿 agent / LLM training / RAG / multimodal 的技术读者。内容不要写成泛科普，也不要写成论文逐页复述，而是输出“读完这篇论文后我真正理解了什么，以及对做系统/做研究有什么启发”。

推荐结构：

1. `问题定义`: 这篇论文解决的技术问题是什么，为什么旧方法不够。
2. `核心方法`: 用图、公式或伪代码讲清楚方法，不只写结论。
3. `关键实验`: 只挑最能支撑主张的实验和 ablation。
4. `Know-how`: 如果自己要复现、借鉴或迁移，最需要注意什么。
5. `Takeaways`: 3-5 条高密度结论，面向研究和工程分别写。
6. `局限`: 哪些 claim 需要谨慎看，哪些条件下可能不成立。

新建小红书草稿时，可以从 [../templates/xiaohongshu-paper-share.md](../templates/xiaohongshu-paper-share.md) 复制。
