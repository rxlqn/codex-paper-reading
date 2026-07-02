# Contributing

这个仓库主要沉淀 AI 论文阅读笔记、主题综述和面向专业读者的论文分享稿。贡献内容时，优先保证问题定义、方法解释和 takeaway 的准确性。

## 内容要求

- 不提交第三方论文 PDF，除非已经确认许可证允许再分发。
- 引用论文时提供 arXiv、alphaXiv、OpenReview、项目页或 DOI 链接。
- 不大段复制论文原文；用自己的话总结，并在关键结论处标明来源。
- 方法部分要讲清楚输入、输出、训练信号、目标函数或算法流程。
- 实验部分要区分 paper claim、实验 evidence 和自己的判断。

## 新增论文

1. 把论文链接登记到 `index.md`。
2. 从 `templates/paper-note.md` 复制一份到 `notes/`。
3. 如果准备做小红书分享，从 `templates/xiaohongshu-paper-share.md` 复制到 `shares/xiaohongshu/drafts/`。
4. 如果属于已有主题，把它加入对应的 `topics/*.md`。

## 文件命名

```text
notes/YYYY_short-title.md
shares/xiaohongshu/drafts/YYYY_short-title.md
```

短名使用英文小写和连字符，避免空格。

