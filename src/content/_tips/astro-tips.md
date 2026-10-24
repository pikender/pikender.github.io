# AstroPaper Tips & Reference

Personal reference collated from the template docs. Not a blog post — lives in `_tips/` so the glob loader ignores it (files prefixed with `_` are excluded).

---

## Post File Placement & URL Routing

Posts go in `src/content/posts/`. Subdirectory names become part of the URL — prefix a folder with `_` to use it for organisation only without affecting the URL:

```bash
src/content/posts/very-first-post.md          -> /posts/very-first-post
src/content/posts/2025/example-post.md        -> /posts/2025/example-post
src/content/posts/_2026/another-post.md       -> /posts/another-post       # _ stripped
src/content/posts/docs/_legacy/how-to.md      -> /posts/docs/how-to        # _ stripped
src/content/posts/Example Dir/Dummy Post.md   -> /posts/example-dir/dummy-post
```

The `_` prefix also works on files — `_` prefixed files are excluded from routing entirely (e.g. `_draft-ideas.md` won't become a post).

---

## Post Frontmatter Fields

```md
---
title: "Post Title"                         # required
pubDatetime: 2026-10-25T10:00:00Z           # required, ISO format
modDatetime: 2026-10-26T10:00:00Z           # optional; if set, used for sort order
author: "Pikender Sharma"                   # optional, defaults to site.author
description: "Short description for SEO"   # required
tags: ["elixir", "ai"]                      # optional, defaults to ["others"]
featured: true                              # optional; shows in Featured section
draft: false                                # optional; true = hidden from site
slug: "custom-url-slug"                     # optional; defaults to filename
ogImage: "../../assets/images/my-og.png"   # optional; auto-generated if omitted
canonicalURL: "https://original-source"    # optional; for cross-posted content
hideEditPost: true                          # optional; hide edit link for this post
timezone: "Asia/Kolkata"                    # optional; override site timezone
---
```

**Quick ISO datetime:** Run `new Date().toISOString()` in the browser console to get a correctly formatted timestamp.

**Sort order:** Posts sorted by `modDatetime` if present, otherwise `pubDatetime`.

**Slug:** If `slug` is not specified, the markdown filename is used. Cannot be an empty string.

**Draft:** Set `draft: true` to hide from the site during development preview and production build.

---

## Table of Contents

Add `## Table of contents` as an h2 heading anywhere in the post body. The TOC will be generated and inserted at that position automatically.

```md
---
# frontmatter
---

Intro paragraph.

## Table of contents

## First section
```

---

## Headings Convention

The post `title` frontmatter field is rendered as the `<h1>`. Start all headings in the post body at `##` (h2) and below — never use `# h1` in the body. This matters for accessibility and SEO.

---

## Callouts

Supported via `rehype-callouts` (Obsidian theme). Use blockquote syntax:

```md
> [!NOTE]
> Supplementary information.

> [!TIP]
> Helpful advice or best practices.

> [!WARNING]
> Something that could go wrong.

> [!DANGER]
> Serious risk of failure or data loss.

> [!INFO]
> Neutral informational context.

> [!SUCCESS]
> Confirmation that something worked.
```

Full list of types: `NOTE`, `ABSTRACT`, `INFO`, `TODO`, `TIP`, `SUCCESS`, `QUESTION`, `WARNING`, `FAILURE`, `DANGER`, `BUG`, `EXAMPLE`, `QUOTE`. Many have aliases (`HINT`/`IMPORTANT` → `TIP`, `CAUTION` → `WARNING`).

**Collapsible callouts:**
```md
> [!WARNING]- Collapsed by default
> Hidden until the reader expands it.

> [!TIP]+ Expanded but collapsible
> Starts open, can be collapsed.
```

**Custom titles:**
```md
> [!NOTE] Did you know?
> The text after the type becomes the heading.
```

---

## Images in Posts

**Option 1 — `src/assets/` (recommended):** Astro auto-optimizes these at build time.

```md
![alt text](@/assets/images/example.jpg)
<!-- or relative path -->
![alt text](../../assets/images/example.jpg)
```

Note: `<img>` tags do not work in plain markdown for `src/assets/` images — use MDX if you need an `<img>` tag or custom styling on optimized images.

**Option 2 — `public/`:** Images served as-is, no optimization. Use absolute paths:

```md
![alt text](/assets/images/example.jpg)
```

**Compress before publishing** — especially for `public/` images. Recommended tools: [TinyPNG](https://tinypng.com/), [TinyJPG](https://tinyjpg.com/).

**OG image size:** Recommended **1200 × 640 px**.

---

## Draft Workflow

**Draft values and what they mean:**

| `draft` value | Behaviour |
|---|---|
| `true` | Hidden from production build; visible in dev mode (`pnpm dev`) |
| `false` | Published — visible everywhere |
| `first` | Special value for the git hook: on next commit, hook sets `draft: false` and clears `modDatetime` |
| omitted | Treated as `false` (published) |

**Full lifecycle:**

1. **Start writing** — create the file with `draft: true`. Stub out `pubDatetime` (any value, the hook will set it on first commit).
   ```md
   ---
   title: "My Post"
   pubDatetime: 2026-01-01T00:00:00Z
   modDatetime:
   draft: true
   description: "..."
   tags: []
   ---
   ```

2. **Preview locally** — run `pnpm dev`. Draft posts are visible at their URL in dev mode but excluded from post listings and the production build.

3. **Ready to publish** — change `draft: true` to `draft: first`.
   ```md
   draft: first
   ```

4. **Commit** — the pre-commit git hook sees `draft: first`, sets `draft: false`, and clears `modDatetime`. The `pubDatetime` is set to the commit timestamp if the file is new.

5. **Subsequent edits** — with `draft: false`, the hook auto-updates `modDatetime` on every commit.

**Without the git hook**, just set `draft: false` manually and set `pubDatetime` to the current datetime yourself.

**Key points:**
- Drafts are fully invisible in production — excluded from post listings, tag pages, RSS, and OG image generation
- You can have unlimited drafts in `src/content/posts/` without affecting the live site
- `modDatetime` requires the field to be present in frontmatter (even if blank) for the git hook's `sed` command to work

---

## Featured vs Recent Posts

- `featured: true` → appears in "Featured" section on homepage
- `featured: false` (or omitted) → appears in "Recent Posts" section
- A post is either featured or recent — never both
- Both sections are hidden if empty
- `posts.perIndex` in config controls how many recent posts show on homepage

---

## Dynamic OG Images

Enabled by default via `features.dynamicOgImage: true` in `astro-paper.config.ts`.

Auto-generates a PNG per post at build time using the post `title`, `site.author`, and `site.title`. Only runs for posts that:
- don't have `ogImage` set in frontmatter
- are not `draft: true`

**To disable** (speeds up large-site builds): set `features.dynamicOgImage: false`.

**Non-Latin characters** won't render correctly out of the box. Switch the Google font in `astro.config.ts` to one covering your script, with both `400` and `700` weights.

---

## Adding LaTeX Equations

Install plugins:
```bash
pnpm install rehype-katex remark-math katex
```

Update `astro.config.ts`:
```ts
import remarkMath from "remark-math";
import rehypeKatex from "rehype-katex";

markdown: {
  remarkPlugins: [remarkMath, remarkToc, [remarkCollapse, ...]],
  rehypePlugins: [rehypeKatex],
}
```

Import KaTeX CSS in `src/layouts/Layout.astro`:
```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@0.15.2/dist/katex.min.css" />
```

Add color fix in `src/styles/typography.css`:
```css
.prose .katex-display {
  @apply text-foreground;
}
```

Usage in posts:
- Inline: `$E = mc^2$`
- Block: `$$\int_{-\infty}^{\infty} e^{-x^2} dx = \sqrt{\pi}$$`

---

## VSCode Snippet for New Posts

AstroPaper ships with a VSCode snippet to scaffold new post frontmatter without manual copy-pasting.

To use: open a new `.md` file, type the snippet prefix, and hit Tab to expand the full frontmatter block including title, dates, tags, description, and a table of contents stub.

The snippet file lives at `.vscode/astro-paper.code-snippets` in the project root. You can edit it to match your defaults (e.g. pre-fill `author`, set your preferred `timezone`).

---

## `modDatetime` Nullable Schema

By default `modDatetime` is `optional()` in the content schema. If you use the git hook approach, the field needs to exist in frontmatter but be blank on draft posts — which requires the schema to also accept `null`:

```ts
// src/content.config.ts
modDatetime: z.date().optional().nullable(),  // allows blank/null values
```

Without `.nullable()`, Astro will fail to build when `modDatetime:` is present in frontmatter with no value (blank after the colon).

The hook writes a blank value (`modDatetime:`) when first-publishing a post, so this schema change is required if you use the git hook workflow.

---

## Git Hooks for Auto-Dating

Use a `pre-commit` hook to auto-set `pubDatetime` on new posts and `modDatetime` on edits.

**Auto-set `modDatetime` on edits** (only when `draft: false`):
```shell
git diff --cached --name-status |
grep -i '^M.*\.md$' |
while read _ file; do
  filecontent=$(cat "$file")
  frontmatter=$(echo "$filecontent" | awk -v RS='---' 'NR==2{print}')
  draft=$(echo "$frontmatter" | awk '/^draft: /{print $2}')
  if [ "$draft" = "false" ]; then
    cat $file | sed "/---.*/,/---.*/s/^modDatetime:.*$/modDatetime: $(date -u "+%Y-%m-%dT%H:%M:%SZ")/" > tmp
    mv tmp $file
    git add $file
  fi
  if [ "$draft" = "first" ]; then
    # First publish: set draft: false, clear modDatetime
    cat $file | sed "/---.*/,/---.*/s/^modDatetime:.*$/modDatetime:/" | sed "/---.*/,/---.*/s/^draft:.*$/draft: false/" > tmp
    mv tmp $file
    git add $file
  fi
done
```

**Auto-set `pubDatetime` on new files:**
```shell
git diff --cached --name-status | egrep -i "^(A).*\.(md)$" | while read a b; do
  cat $b | sed "/---.*/,/---.*/s/^pubDatetime:.*$/pubDatetime: $(date -u "+%Y-%m-%dT%H:%M:%SZ")/" > tmp
  mv tmp $b
  git add $b
done
```

Install [Husky](https://typicode.github.io/husky/) to make hooks portable across machines.

---

## Comments with Giscus

Serverless comments stored in GitHub Discussions — perfect for a static blog.

**Prerequisites:**
- Repo must be public
- [Giscus app](https://github.com/apps/giscus) installed on the repo
- GitHub Discussions enabled

**Simple embed** — add to `src/layouts/PostLayout.astro` after `<ShareLinks />`:
```html
<script
  src="https://giscus.app/client.js"
  data-repo="pikender/pikender.github.io"
  data-repo-id="[REPO ID]"
  data-category="[CATEGORY NAME]"
  data-category-id="[CATEGORY ID]"
  data-mapping="pathname"
  data-theme="preferred_color_scheme"
  data-lang="en"
  crossorigin="anonymous"
  async
></script>
```

**Light/dark theme sync** — use the `@giscus/react` component with event listeners on the theme toggle button. See original docs post for the full `Comments.tsx` component.

Get your repo/category IDs from [giscus.app](https://giscus.app/).

---

## Scheduled Posts

Posts with a `pubDatetime` in the future are hidden until that time. A grace window is configurable:

```ts
// astro-paper.config.ts
posts: {
  scheduledPostMargin: 15 * 60 * 1000, // 15 minutes before pubDatetime
}
```

---

## Updating Dependencies

Use `npm-check-updates`:
```bash
npm install -g npm-check-updates
ncu                        # preview all updates
ncu -u --target patch      # apply patch updates
ncu -i --target minor      # interactive minor updates
ncu -i                     # remaining (major) updates — check release notes first
```

---

## Layout Width

Default max-width is `768px` (`max-w-3xl`). To change it, edit `src/styles/global.css`:

```css
@utility max-w-app {
  @apply max-w-4xl xl:max-w-5xl; /* change to whatever you need */
}
```

---

## Logo / Site Title

Three options in `src/components/Header.astro`:

**Option 1 — Text** (default): update `site.title` in `astro-paper.config.ts`.

**Option 2 — SVG logo:**
```astro
---
import DummyLogo from "@/assets/dummy-logo.svg";
---
<a href="/"><DummyLogo class="scale-75 dark:invert" /></a>
```

**Option 3 — Image logo:**
```astro
---
import { Image } from "astro:assets";
import dummyLogo from "@/assets/dummy-logo.png";
---
<a href="/"><Image src={dummyLogo} alt="My Blog" class="dark:invert" /></a>
```

Replace the `{config.site.title}` expression in `Header.astro` with your chosen option.

---

## Custom Font

Three files to update:

1. `astro.config.ts` — add/change the font entry:
```ts
fonts: [{
  name: "Your Font Name",
  cssVariable: "--font-your-font",
  provider: fontProviders.google(),
  fallbacks: ["monospace"],
  weights: [300, 400, 500, 600, 700],
  styles: ["normal", "italic"],
}]
```

2. `src/layouts/Layout.astro` — update the `<Font>` component's `cssVariable`.

3. `src/styles/theme.css` — point `--font-app` at your new variable:
```css
@theme inline {
  --font-app: var(--font-your-font);
}
```

Font name must match exactly as shown on Google Fonts.

---

## Adding a New Social Icon

The `name` in `socials` must match an SVG filename in `src/assets/icons/socials/`. To add a platform not in the defaults:
1. Add `your-platform.svg` to `src/assets/icons/socials/`
2. Add `{ name: "your-platform", url: "https://..." }` to the `socials` array

Same applies to `shareLinks`.

---

## Customizing Colors

Colors live in `src/styles/theme.css` as CSS custom properties. Seven tokens:

| Token | Purpose |
|---|---|
| `--background` | Primary background |
| `--foreground` | Primary text color |
| `--accent` | Links, buttons, highlights |
| `--accent-foreground` | Text on accent backgrounds |
| `--muted` | Cards, tags, hover backgrounds |
| `--muted-foreground` | Text on muted backgrounds |
| `--border` | Borders and dividers |

Edit both blocks in `src/styles/theme.css`:

```css
:root, [data-theme="light"] {
  --background: #fdfdfd;
  --foreground: #282728;
  --accent: #006cac;
  --accent-foreground: #ffffff;
  --muted: #e6e6e6;
  --muted-foreground: #6b7280;
  --border: #ece9e9;
}

[data-theme="dark"] {
  --background: #212737;
  --foreground: #eaedf3;
  --accent: #ff6b01;
  --accent-foreground: #ffffff;
  --muted: #343f60;
  --muted-foreground: #afb9ca;
  --border: #ab4b08;
}
```

To disable dark mode entirely: set `features.lightAndDarkMode: false` in `astro-paper.config.ts`. Site will use only the light scheme.

### Predefined Color Schemes

Drop-in replacements — copy into `src/styles/theme.css`.

**Kha-Yan** (purple/warm light):
```css
:root, [data-theme="light"] {
  --background: #fefaec; --foreground: #120e01; --accent: #6e10cf;
  --accent-foreground: #fefaec; --muted: #dcdcdc; --muted-foreground: #6b7280; --border: #cdc4d6;
}
```

**Nila** (cool blue-purple light):
```css
:root, [data-theme="light"] {
  --background: #f6f6fb; --foreground: #0c0c19; --accent: #6760b4;
  --accent-foreground: #f3f3f3; --muted: #dddcea; --muted-foreground: #54515b; --border: #d8d6ec;
}
```

**Jadeite** (teal light):
```css
:root, [data-theme="light"] {
  --background: #f6fcf7; --foreground: #060b07; --accent: #027c6d;
  --accent-foreground: #ffffff; --muted: #c9e4e2; --muted-foreground: #6b7280; --border: #d4e1df;
}
```

**Paper Dark** (cyan accents dark):
```css
[data-theme="dark"] {
  --background: #2f3741; --foreground: #e6e6e6; --accent: #1ad9d9;
  --accent-foreground: #0d2b2b; --muted: #596b81; --muted-foreground: #8faabb; --border: #3b4655;
}
```

**Deep Purple** (magenta dark):
```css
[data-theme="dark"] {
  --background: #212737; --foreground: #eaedf3; --accent: #eb3fd3;
  --accent-foreground: #1a0d1a; --muted: #513f51; --muted-foreground: #c09abc; --border: #642451;
}
```

**Ember** (red/warm dark):
```css
[data-theme="dark"] {
  --background: #1a1a1a; --foreground: #f5efe4; --accent: #ff3737;
  --accent-foreground: #1a1a1a; --muted: #38342f; --muted-foreground: #a59a8c; --border: #6f5648;
}
```

**Espresso** (brown/warm dark):
```css
[data-theme="dark"] {
  --background: #2f2f2f; --foreground: #ebe5e1; --accent: #ee781e;
  --accent-foreground: #1a1a1a; --muted: #4f4b44; --muted-foreground: #ddbfa7; --border: #6f5648;
}
```

---

## Google Site Verification

Preferred: set in config (gets committed with the repo):
```ts
site: {
  googleVerification: "your-token-here",
}
```

Alternative: use `.env` (not committed):
```bash
PUBLIC_GOOGLE_SITE_VERIFICATION=your-token-here
```

When both are set, config takes precedence.

---

## MDX Support

Posts can use `.mdx` extension to embed Astro/React components and JSX expressions. The glob loader picks up both `*.md` and `*.mdx` automatically.

---

## Edit Post Links

`features.editPost.enabled: true` adds an "Edit this page" link at the bottom of each post. The link is constructed as:

```
editPost.url + <relative path to the .md file>
```

Example: `https://github.com/pikender/pikender.github.io/edit/main/src/content/posts/my-post.md`

To hide the edit link on a specific post, add `hideEditPost: true` to its frontmatter.
