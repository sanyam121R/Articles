# Articles

Source for [portfolio blog posts](https://sanyam121r.vercel.app/blogs). Each published `.md` file is one **article** (one thesis), fetched from this repo at build time.

## File naming

| Don't | Do |
| --- | --- |
| Name after a **topic** (`cloudflare.md`, `Redis.md`) | Name after **this article** (`how-cloudflare-carries-a-request.md`, `rate-limiting-with-redis.md`) |
| Mix `PascalCase` and `kebab-case` | Use **kebab-case** only for `.md` files |

You can write many articles on the same topic; each gets its own file, for example `how-cloudflare-carries-a-request.md` and later `cloudflare-workers-and-consistency.md`.

The portfolio **URL** defaults to the filename without `.md`. Override with optional `slug` in frontmatter (e.g. keep `/blogs/cloudflare` while the file stays descriptive).

## Publishing checklist

```yaml
---
title: "Human title for the card and <h1>"
slug: "optional-short-url"   # default: filename without .md
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

| URL slug | File | What it is (not “all of X”) |
| --- | --- | --- |
| [cloudflare](how-cloudflare-carries-a-request.md) | Request path: anycast, edge, cache vs origin | One Cloudflare article |
| [database-pages](database-pages-fundamental-unit-of-storage.md) | Pages, buffer pool, link to sharding | One storage fundamentals article |
| [rate-limiting-with-redis](rate-limiting-with-redis.md) | Rate limiting from first principles via Redis | One Redis article |
| [database-sharding](database-sharding-from-first-principles.md) | Sharding reference / interview depth | One sharding article |