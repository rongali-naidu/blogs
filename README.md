# blogs

Notes and articles on data engineering, data architecture, AWS, and GenAI.

## Layout

Posts are grouped by topic folder (`aws/`, `data_architecture/`, `data_governance/`, `genai/`, `spark/`, `sql/`, `streaming/`, `table_formats/`, ...). Anything about a specific AWS service lives under `aws/<service>/`. Images live in `images/`.

## Post header

Every post starts with a front matter header:

```yaml
---
title: "The Tiny Files Problem: A Silent Bottleneck in Real-Time Data Lakes"
date: 2025-05-30
tags: [data-architecture]
publish: true
---
```

- `publish: true` — the post appears on the website.
- `publish: false` — the post stays in this repo but is hidden from the website (drafts, short notes, duplicates).
