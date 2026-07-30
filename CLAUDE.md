# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Personal Jekyll blog (v0dro.in) hosted on GitHub Pages, using the `github-pages` gem and the minima theme. There is no build/test/lint pipeline beyond Jekyll itself — GitHub Pages builds the site automatically on push to `master`.

## Commands

```bash
bundle install                 # install dependencies
bundle exec jekyll serve       # local dev server at http://localhost:4000 (restart after editing _config.yml)
bundle exec jekyll build       # build to _site/ (gitignored)
bundle exec jekyll compose "Post Title"   # scaffold a new post (jekyll-compose gem)
```

## Structure and conventions

- Posts live in `_posts/` as `YYYY-MM-DD-title.md` (older posts use `.markdown`). Front matter is minimal: `layout: post`, `title`, and `date` with `+0900` timezone offset.
- Permalinks follow `/blog/:year/:month/:day/:title/` (set in `_config.yml`).
- Images and other static files go under `assets/images/`, typically in a per-post subdirectory.
- Markdown is kramdown with GFM input; syntax highlighting uses pygments.rb/coderay (configured in `_config.yml`).
- There are no custom layouts or includes — the site relies entirely on the minima theme defaults. `index.md` and `about.md` are the only pages.
- `_posts/migrate.rb` is a one-off script from an Octopress migration (rewrites old `{% img %}` tags); not part of the site build.
