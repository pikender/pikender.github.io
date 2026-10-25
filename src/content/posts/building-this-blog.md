---
title: "Building This Blog"
author: "Pikender Sharma"
pubDatetime: 2026-10-25T11:03:10Z
description: "How I set up this blog — from cloning a template to publishing on GitHub Pages."
tags: ["meta", "astro"]
featured: true
draft: false
---

I wanted a clean, fast blog to write about AI systems, Elixir, and distributed systems. This post covers how I got from zero to published.

## The Starting Point

My old blog at `pikender.github.io` was a Jekyll site forked years ago. It worked, but the workflow felt dated. I wanted something I could write Markdown in, deploy automatically, and not think about.

A quick search and a suggestion later, [AstroPaper](https://github.com/satnaing/astro-paper) stood out — clean, minimal, and ready to go.

## Setting Up

Clone the template and install dependencies:

```bash
git clone https://github.com/satnaing/astro-paper.git pikender-astro-blog
cd pikender-astro-blog
npm install
npm run dev
```

The dev server runs at `localhost:4321`. Everything works immediately — no configuration needed to get started.

## Personalising

All site configuration lives in a single file: `astro-paper.config.ts`. I updated:

- `site.url`, `site.title`, `site.author`, `site.description`
- `site.timezone` → `Asia/Kolkata`
- `socials` → GitHub and LinkedIn only

The homepage hero text is hardcoded in `src/pages/index.astro` — I replaced the default greeting and intro paragraph with my own.

For the About page (`src/content/pages/about.md`), I pulled content from my old Jekyll blog rather than writing from scratch.

## Cleaning Up the Template Posts

The template ships with 18 documentation and example posts. Before deleting them, I read through all of them and extracted the useful tips — configuration options, workflows, color schemes — into a reference file at `src/content/_tips/astro-tips.md`.

Files prefixed with `_` are excluded from routing by the glob loader, so this file stays in the repo as a personal reference without appearing as a blog post.

After that, I deleted all 18 template posts and the 5 template images in `src/assets/images/`.

## Writing a Post

The day-to-day workflow is simple. Start the dev server:

```bash
npm run dev
```

Create a new `.md` file in `src/content/posts/` with the required frontmatter:

```md
---
title: "Building This Blog"
author: "Pikender Sharma"
pubDatetime: 2026-10-25T11:03:10Z
description: "How I set up this blog — from cloning a template to publishing on GitHub Pages."
tags: ["meta", "astro"]
featured: true
draft: false
---

Post content here.
```

The post shows up live at `localhost:4321` immediately. When ready to publish, commit and push to `main` — the GitHub Actions workflow picks it up and deploys automatically.

## Git Hooks

I set up [Husky](https://typicode.github.io/husky/) with two hooks to automate the boring parts:

**`pre-commit`** — automatically sets `pubDatetime` on new posts and updates `modDatetime` on edits. No more manually copying ISO timestamps.

**`pre-push`** — runs `npm run build` before every push. If the build breaks, the push is blocked. Catches issues locally before they fail in CI.

The hooks live in `.husky/` and are committed to the repo. The `prepare` script in `package.json` activates them automatically after `npm install` on any machine.

## Deploy Setup

My old repo `pikender/pikender.github.io` was a Jekyll fork. I initially force-pushed `main` with the new Astro content, but then decided to rename the old repo to preserve it and create a fresh `pikender.github.io` repo for the new blog.

For CI/CD, I added a deploy workflow at `.github/workflows/deploy.yml` that triggers on every push to `main`:

1. Install dependencies
2. Run `npm build` — Astro generates static HTML into `dist/`
3. Upload and deploy to GitHub Pages

In the GitHub repo settings, I set Pages source to **GitHub Actions** instead of the legacy branch deploy.

Every push to `main` now builds and deploys automatically.

## What's Next

The blog is live. Next up: writing about the actual work — LLM evals, agent architectures in Elixir, and distributed systems patterns.
