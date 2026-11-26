# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Content repository for [Zenn](https://zenn.dev) (Japanese tech-writing platform), synced with GitHub. There is no application code, build, lint, or test step (`npm test` is a placeholder). Articles are written in Japanese.

## Commands

Environment is managed with devbox (Node.js 22); `devbox shell` runs `npm install` automatically. Scripts defined in `devbox.json`:

- `devbox run preview` — `npx zenn preview` (local preview server)
- `devbox run new:article` — `npx zenn new:article` (scaffold an article)
- `devbox run new:book` — `npx zenn new:book`

Without devbox, use `npx zenn ...` directly (zenn-cli is the only dependency).

## Structure

- `articles/<slug>.md` — one file per article. Slug is either an auto-generated hex id (`zenn new:article`) or a custom slug (12–50 chars, `a-z0-9-_`), e.g. `claude-api-public-beta-structured-outputs.md`.
- `images/<slug>/` — images for an article, grouped by the article's slug.
- `books/` — Zenn books (currently empty).

Each article starts with frontmatter:

```yaml
---
title: "..."
emoji: "🧱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Claude", "Anthropic"]
published: true # false = draft, not publicly visible
---
```

Publishing happens by pushing to `main` (Zenn's GitHub integration); `published: true` makes the article live. Commit messages follow `article: <title>` for article changes and `chore: ...` for tooling.
