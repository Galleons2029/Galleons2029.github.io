# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A personal academic homepage / digital garden for Jialong Liu (柳佳龙), built on **Quartz 5** (the `@jackyzha0/quartz` static site generator) and deployed to GitHub Pages at <https://galleons2029.github.io>. The author's Obsidian notes in `content/` are published as the website.

Almost all files under `quartz/` are upstream Quartz framework code — **the actual personalization lives in `content/` and `quartz.config.yaml`.** Most tasks here are content/config edits, not framework changes.

## Commands

Requires Node.js ≥ 22 and npm ≥ 10.9.2 (`engine-strict=true` in `.npmrc` enforces this).

```bash
npm ci                       # install deps (runs `install-plugins` via prebuild)
npx quartz plugin install    # fetch external plugins (REQUIRED before first build — see below)
npx quartz build --serve     # build + live-preview at http://localhost:8080 (watches content/)
npx quartz build             # one-off production build into ./public
npm run check                # tsc --noEmit + prettier --check (CI-equivalent lint)
npm run format               # prettier --write
npm run test                 # tsx --test (runs *.test.js/ts, e.g. quartz/cli/helpers.test.js)
```

Run a single test file: `npx tsx --test quartz/cli/helpers.test.js`

## Plugin architecture (the key thing about Quartz 5)

Unlike Quartz 4, plugins are **not** configured in a `quartz.layout.ts` file. Instead:

- All plugins are declared in **`quartz.config.yaml`** under `plugins:`, each sourced from a git repo (`source: github:quartz-community/<name>`) with `enabled`, `order`, `options`, and `layout` (position/priority/group) fields.
- `npx quartz plugin install` clones each enabled plugin into **`.quartz/plugins/`** (git-ignored cache) and symlinks peer deps into the host `node_modules`. This step is mandatory before any build and runs automatically in CI (`.github/workflows/deploy.yml`) and via the `prebuild` npm hook.
- `quartz.lock.json` pins plugin versions.
- `quartz/plugins/loader/` (config-loader.ts, gitLoader.ts) is the loader that resolves and fetches these. To add/remove/reconfigure a plugin, edit `quartz.config.yaml`, not TypeScript.

To change site appearance, navigation, or behavior, **edit `quartz.config.yaml`** (theme colors, fonts, `pageTitle`, locale, and the enabled-plugin list). `quartz.config.default.yaml` is the upstream default for reference.

## Content & publishing model

- **Explicit-publish mode is on.** A note is published only if its frontmatter contains `publish: true`. Notes without it (including the `示例笔记/未发布笔记.md` demo) are excluded from the built site. This filter applies to Markdown only — static assets like images in `附件/` folders are always copied to output, so **do not put private images under `content/`.**
- Content is authored as Obsidian vaults: drag a topic folder (with its `附件/` attachments folder) into `content/`; the folder hierarchy becomes the site's URL structure. Obsidian syntax is supported: wikilinks `[[note]]`, embeds `![[demo.png]]`, KaTeX math, and callouts.
- `content/private/`, `content/templates/`, and `.obsidian/` are excluded via `ignorePatterns` in `quartz.config.yaml`.
- Post dates: frontmatter `date` takes priority; otherwise the date is derived from the file's git commit history (the `created-modified-date` plugin, priority `frontmatter > git > filesystem`). This is why CI checks out with `fetch-depth: 0`.
- Bilingual homepage: `content/index.md` (zh-CN, default) and `content/index.en.md` (English), cross-linked manually.
- Content lives in `content/`: `blog/`, `paper/`, `report/`, `cv.md`, plus `示例笔记/` which is the canonical example of expected structure/syntax.

## Deployment

Push to `master` → `.github/workflows/deploy.yml` runs `npm ci` → `npx quartz plugin install` → `npx quartz build` (output `public/`) → deploys to GitHub Pages. There is no separate staging branch.

## Development notes

To learn more about Quartz 5's architecture and plugin system, see the [Quartz 5 documentation](https://quartz.jzhao.xyz/). For content editing, refer to the `示例笔记/` folder as a template for frontmatter, Obsidian syntax, and directory structure.