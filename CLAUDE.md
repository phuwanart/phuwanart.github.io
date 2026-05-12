# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal blog (phuwanart.github.io) built with Jekyll using the **Chirpy theme** (`jekyll-theme-chirpy` ~> 7.2). Deployed to GitHub Pages via the `.github/workflows/pages-deploy.yml` Actions workflow (push to `main`/`master`). Content is primarily Ruby on Rails development notes and tutorials, often in Thai.

Ruby version is pinned to **3.3** (`.ruby-version`, matched by the CI workflow).

## Development Commands

```bash
# Install dependencies
bundle install

# Serve locally with live reload (or use tools/run.sh)
bundle exec jekyll serve --livereload

# Build the site
bundle exec jekyll build

# Run HTML validation — match the CI flags so local results match deploy
bundle exec htmlproofer _site \
  --disable-external \
  --ignore-urls "/^http:\/\/127.0.0.1/,/^http:\/\/0.0.0.0/,/^http:\/\/localhost/"

# Build + htmlproofer in one shot (production env, mirrors CI)
bash tools/test.sh

# Create a new post / draft via jekyll-compose
bundle exec jekyll post "Post Title"
bundle exec jekyll draft "Draft Title"
bundle exec jekyll publish _drafts/draft-title.md
```

Note: a bare `bundle exec htmlproofer _site` checks external links and can fail on transient network/redirect issues; the deploy workflow disables external checks, so local runs should too unless you specifically want external link validation.

## Post Format

Posts go in `_posts/` with filename format: `YYYY-MM-DD-slug-title.md`.

Front matter template:
```yaml
---
layout: post
title: Post Title
categories:
- Posts
tags:
- tag1
- tag2
date: YYYY-MM-DD HH:MM +0700
---
```

- Timezone is Asia/Bangkok (+0700); set in `_config.yml`.
- Category for blog posts is **`Posts`** (this is what existing posts use and what the post tooling/linter enforces — do not use `Notes`).
- Drafts live in `_drafts/` (no date prefix). Comments are auto-disabled for drafts via `_config.yml` defaults.
- Permalinks are `/:categories/:slug/` — changing a post's category or slug changes its URL.

## Code Blocks in Posts

Use fenced code blocks with a language tag — Rouge highlights them and Chirpy renders line numbers by default (configured in `_config.yml` under `kramdown.syntax_highlighter_opts.block.line_numbers: true`).

Language tags used across existing posts (prefer these to keep consistency):
`sh` (shell — not `bash`), `ruby`, `js`, `erb`, `yaml`, `diff`, `css`, `nginx`, `html`, `vue`, `sql`, `json`, `ini`.

Chirpy-specific attribute annotations go on the line **immediately after** the closing fence:

````markdown
```ruby
spec.add_dependency 'importmap-rails'
```
{: file="blorgh.gemspec"}
````

- `{: file="path/or/label"}` — renders a filename header above the block. Use the real file path (e.g. `config/routes.rb`, `~/.ssh/config`) or a context label (e.g. `Local Machine`, `ubuntu@primary`) for terminal sessions where there is no file.
- `{: .nolineno}` — hides line numbers for that block. Useful for short inline snippets, terminal output, or `diff` blocks where line numbers add noise. Not currently used in any post, but valid.

Both attributes can be combined: `{: file="..." .nolineno}`.

## Architecture

- **Theme**: Chirpy is consumed as a gem. Layouts, includes (most of them), and sass live in the gem — run `bundle info --path jekyll-theme-chirpy` to inspect. Anything not present locally is being pulled from the gem.
- **`_plugins/posts-lastmod-hook.rb`**: Sets `last_modified_at` on posts from `git log` (only when a post has more than one commit). This means the timestamp is derived from history, not the front matter, so committing a fix to an old post will update its "last modified" date.
- **`_includes/metadata-hook.html`**: Injects the Buy Me a Coffee widget into every page (Chirpy looks for this include name as an extension point).
- **`_tabs/`**: Sidebar navigation pages (about, archives, categories, tags). Outputs to `/:title/` permalinks.
- **`_data/contact.yml`, `_data/share.yml`**: Configure sidebar contact links and post sharing options.
- **Comments**: Giscus (`comments.provider: giscus` in `_config.yml`, repo `phuwanart/phuwanart.github.io`).
- **Analytics**: Google Analytics (`G-1Q43VKEBZ0`).
- **PWA**: Enabled (`pwa.enabled: true`); the site is installable and has an offline cache.
