# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal blog (phuwanart.github.io) built with Jekyll using the **Chirpy theme** (`jekyll-theme-chirpy` ~> 7.2). Deployed to GitHub Pages. Content is primarily Ruby on Rails development notes and tutorials.

## Development Commands

```bash
# Install dependencies
bundle install

# Serve locally with live reload
bundle exec jekyll serve --livereload

# Build the site
bundle exec jekyll build

# Run HTML validation tests
bundle exec htmlproofer _site

# Create a new post via jekyll-compose
bundle exec jekyll post "Post Title"

# Create a new draft
bundle exec jekyll draft "Draft Title"

# Publish a draft
bundle exec jekyll publish _drafts/draft-title.md
```

## Post Format

Posts go in `_posts/` with filename format: `YYYY-MM-DD-slug-title.md`

Front matter template:
```yaml
---
layout: post
title: Post Title
categories: Notes
tags:
- tag1
- tag2
date: YYYY-MM-DD HH:MM +0700
---
```

- Timezone is Asia/Bangkok (+0700)
- Category is typically `Notes`
- Drafts go in `_drafts/` (no date prefix needed)

## Architecture

- **Theme**: Chirpy gem-based theme — layouts, includes, and sass come from the gem (`bundle info --path jekyll-theme-chirpy` to inspect)
- **`_plugins/posts-lastmod-hook.rb`**: Auto-sets `last_modified_at` from git history
- **`_includes/metadata-hook.html`**: Injects Buy Me a Coffee widget
- **`_tabs/`**: Sidebar navigation pages (about, archives, categories, tags)
- **`_data/`**: `contact.yml` and `share.yml` configure sidebar contact links and post sharing options
- **Comments**: Giscus (GitHub Discussions-based)
- **Analytics**: Google Analytics (G-1Q43VKEBZ0)
