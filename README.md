# Articles

Source for [portfolio blog posts](https://sanyam121r.vercel.app/blogs). Each published `.md` file is one **article**, fetched from this repo at build time.

## File naming and routes

The **filename** (without `.md`) is the blog **route**: `how-cloudflare-carries-a-request.md` → `/blogs/how-cloudflare-carries-a-request`.

Name files after **this article**, not the whole topic:

| Don't | Do |
| --- | --- |
| `cloudflare.md` (implies everything about Cloudflare) | `how-cloudflare-carries-a-request.md` |
| `Redis.md` | `rate-limiting-with-redis.md` |

Use **kebab-case** only. You can publish many articles on one topic with different filenames.

`title` in frontmatter is for the **card and page heading** only — it does not change the URL. If you want a specific route, set the filename to match (kebab-case version of the title is fine).

## Publishing checklist

```yaml
---
title: "Human title for the card and page"
date: "YYYY-MM-DD"
description: "One or two sentences for the blog card."
tags: ["tag-one", "tag-two"]
link: ""          # optional external canonical URL
readTime: ""      # optional; auto-estimated from word count if omitted
draft: true       # optional; omit or false to publish
---
```

- Prefer **no duplicate `# H1`** in the body when `title` is set.
- Put images beside the markdown; use relative paths (e.g. `![](./cloudflare-request-flow.png)`).
- WIP / private: `draft: true` or keep the file out of this repo.

After push, the Articles webhook revalidates the `articles` cache on Vercel (or wait up to 1h).

## Index

| Route | File |
| --- | --- |
| `/blogs/how-cloudflare-carries-a-request` | [how-cloudflare-carries-a-request.md](how-cloudflare-carries-a-request.md) |
| `/blogs/database-pages-fundamental-unit-of-storage` | [database-pages-fundamental-unit-of-storage.md](database-pages-fundamental-unit-of-storage.md) |
| `/blogs/rate-limiting-with-redis` | [rate-limiting-with-redis.md](rate-limiting-with-redis.md) |
| `/blogs/database-sharding-from-first-principles` | [database-sharding-from-first-principles.md](database-sharding-from-first-principles.md) |