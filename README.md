# Articles

Source for [portfolio blog posts](https://sanyam121r.vercel.app/blogs). Each `.md` file is fetched from this repo at build time (see `lib/blogs.ts` on the site). HTML case studies live only on the portfolio (`deep-customer-problem`).

## Publishing checklist

Add YAML frontmatter at the top so list cards show title, excerpt, tags, and date (not required for rendering the post body, but required for the card UI):

```yaml
---
title: "Post title"
date: "YYYY-MM-DD"
description: "One or two sentences for the blog card."
tags: ["tag-one", "tag-two"]
link: ""          # optional external canonical URL
readTime: ""      # optional; auto-estimated from word count if omitted
draft: true       # optional; omit or false to publish
---
```

- Prefer **no duplicate `# H1`** in the body when `title` is set (the site strips a leading H1 anyway).
- Put images beside the markdown file; use relative paths like `![](./my-diagram.png)`.
- Private or WIP notes: set `draft: true` or keep them out of this repo.

After push, the Articles repo webhook revalidates the `articles` cache on Vercel (or wait up to 1h).

## Index

| Slug | File | Status |
| --- | --- | --- |
| [cloudflare](cloudflare.md) | Edge path, cache vs origin, diagrams | Published |
| [DatabasePages](DatabasePages.md) | Storage pages and buffer pool | Published |
| [Redis](Redis.md) | Rate limiting from first principles | Published |
| [shard](shard.md) | Sharding reference | Published |