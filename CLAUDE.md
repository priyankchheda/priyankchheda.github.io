# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Personal blog built with Hugo, deployed to GitHub Pages via GitHub Actions on push to `master`. Uses the [nostyleplease](https://github.com/hanwenguo/hugo-theme-nostyleplease) theme (included as a git submodule).

## Commands

```bash
# Local development server (with drafts)
hugo server -D

# Build the site
hugo build --gc --minify

# Create a new post
hugo new posts/my-post-title.md
```

## Architecture

- `hugo.toml` — Site configuration (theme, params, markup settings)
- `content/` — Markdown content; `_index.md` is the homepage
- `data/menu.toml` — Defines homepage layout sections (info, posts list, links, rss)
- `layouts/partials/` — Template overrides on top of the theme (e.g., custom footer)
- `themes/nostyleplease/` — Git submodule; do not edit directly
- `public/` — Generated output (committed for Pages, rebuilt by CI)
- `archetypes/default.md` — Template for `hugo new` (posts start as draft by default)

## Deployment

CI is defined in `.github/workflows/hugo.yml`. Pushes to `master` trigger a build with Hugo 0.160.0 (extended) and deploy to GitHub Pages. The theme submodule is fetched recursively during checkout.
