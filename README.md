# erdincka.github.io

Personal site, built with Jekyll and published by GitHub Pages from `main`.

## Writing a post

Add a Markdown file to `_posts/` named `YYYY-MM-DD-slug.md`:

```markdown
---
layout: post
title: The title, in sentence case
date: 2026-08-29
description: One sentence shown on the home page, in link previews and in search results.
tags: [kubernetes, ai]
image: /assets/img/social-card.png   # optional, 1200x675, used for link previews
---

Body goes here.
```

Anything in `_drafts/` is ignored by the published build. To publish a draft,
move it into `_posts/` and give the filename a date prefix.

## Previewing locally

Optional — GitHub builds the site on push, so this is only for checking layout
before you publish.

```bash
bundle install
bundle exec jekyll serve --drafts
```

Then open <http://localhost:4000>. The `--drafts` flag includes `_drafts/`.

## Layout

| Path | Purpose |
|---|---|
| `_config.yml` | Site title, description, permalink format, plugins |
| `_layouts/` | `default.html` is the shell; `post.html` wraps each post |
| `_posts/` | Published posts |
| `_drafts/` | Unpublished work in progress |
| `assets/css/style.css` | All styling; light and dark both defined here |
| `assets/img/` | Images referenced from posts |
