# Codex Paper Reading

这是一个 Codex 辅助的 AI 论文阅读工作区，用于沉淀人工筛选的论文 source、结构化阅读笔记、关联论文脉络，以及面向专业读者的小红书分享稿。

建议把它当成一个 Markdown-first 的论文资料库：PDF 只作为本地原文归档，真正可复用的理解、对比和分享材料都写在 Markdown 里。

## 目录结构

```text
PaperReading/
  index.md        # 论文总索引
  papers/         # 本地 PDF，不默认提交到 GitHub
  notes/          # 每篇论文一份阅读笔记
  topics/         # 跨论文主题整理、阅读路线、对比分析
  shares/         # 某次论文分享的提纲、讲稿、slides 素材
  assets/         # 图片、截图、表格等通用材料
  templates/      # 笔记和分享模板
```

## 命名规范

PDF 和笔记使用同一个短名，便于互相定位：

```text
papers/YYYY_short-title.pdf
notes/YYYY_short-title.md
```

例子：

```text
papers/2026_privileged-information-distillation-for-language-models.pdf
notes/2026_privileged-information-distillation-for-language-models.md
```

## 推荐工作流

1. 把 PDF 放进 `papers/`，但默认不提交到 GitHub。
2. 复制 `templates/paper-note.md` 到 `notes/`，按论文短名重命名。
3. 在 `index.md` 里登记论文状态、主题和一句话结论。
4. 如果多篇论文围绕同一问题，在 `topics/` 里写主题综述。
5. 准备分享时，在 `shares/YYYY-MM-topic/` 下维护提纲、slides 和引用材料。

## 状态字段

- `unread`: 已收集但还没读。
- `reading`: 正在读。
- `read`: 已完成基础笔记。
- `shared`: 已做过分享，且分享材料已归档。

## 开源边界

这个仓库计划开源时，默认只提交原创笔记、主题整理、分享稿和模板。第三方论文 PDF 默认保留在本地，不提交到 GitHub；如果某篇论文明确允许再分发，再单独记录许可证来源后提交。

发布前可以检查 [docs/open-source-checklist.md](docs/open-source-checklist.md)。

## License

原创文字内容按 [CC BY 4.0](LICENSE.md) 授权。第三方论文、论文 PDF、原论文图表和外部链接内容不包含在本仓库授权范围内。
