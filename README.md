# Kino Notes

Hugo + PaperMod powered technical notes for Kubernetes source reading, cloud native systems, and Linux troubleshooting.

## Writing Checklist

Use this front matter for normal posts:

```yaml
---
title: "文章标题"
date: 2026-06-12T09:00:00+08:00
draft: false
tags: ["Kubernetes", "源码分析"]
categories: ["云原生"]
series: ["系列名称"]
summary: "一句话说明文章解决什么问题，尽量控制在 60-90 个中文字符。"
weight: 1
---
```

## Series Rules

- `series` must match the title in `content/series/<series-slug>/_index.md`.
- `weight` controls reading order inside a series. Use `1, 2, 3...`.
- Add or update a series `_index.md` when starting a new topic.
- Keep each series description focused on what readers can learn from the sequence.

## Summary Rules

- Write `summary` for every post. It powers home cards, search results, related posts, and social metadata.
- Prefer concrete nouns: component names, problem type, workflow, or source-code path.
- Avoid vague summaries such as "学习 Kubernetes 基础知识".

## Markdown Notes

- Put placeholders such as `<pid>`, `<pod>`, `<container-id>` inside backticks: `` `/proc/<pid>/status` ``.
- In fenced code blocks, placeholders can stay unescaped.
- Prefer Markdown tables over raw HTML tables.
- Avoid raw HTML unless there is a strong reason; Hugo's Goldmark renderer filters unsafe HTML by default.

## Local Commands

```bash
hugo server --disableFastRender
hugo --gc --minify
```
