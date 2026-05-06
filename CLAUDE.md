# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal portfolio and technical blog for Michael Paonam (michaelpaonam.com). Jekyll site deployed to GitHub Pages from the `/docs` directory.

## Development Commands

```bash
# Install dependencies
cd docs && bundle install

# Serve locally with live reload (http://localhost:4000)
cd docs && bundle exec jekyll serve

# Build the site (output to docs/_site/)
cd docs && bundle exec jekyll build
```

Note: Changes to `_config.yml` require restarting the server.

## Architecture

- **Source directory:** `docs/` (not root — GitHub Pages is configured to publish from `/docs`)
- **Theme:** Minima with heavy custom overrides in `assets/main.scss`
- **Layout:** Custom three-column grid (left sidebar | main content | right sidebar) defined in `_layouts/default.html` with sidebar partials in `_includes/`
- **Hosting:** GitHub Pages using the `github-pages` gem (not standalone Jekyll)
- **Domain:** Custom domain via `docs/CNAME` → michaelpaonam.com

## Content

- Blog posts go in `docs/_posts/` with filename format `YYYY-MM-DD-title.md`
- Posts use Jekyll front matter (`layout`, `title`, `date`, `categories`)
- Static assets (images) in `docs/assets/images/`

## Key Layout Files

- `docs/_layouts/default.html` — Main page structure with the three-column grid
- `docs/_includes/head.html` — Meta tags, Font Awesome CDN link
- `docs/_includes/sidebar-left.html` — Left navigation/profile panel
- `docs/_includes/sidebar-right.html` — Right panel content
- `docs/assets/main.scss` — All custom CSS (grid layout, responsive breakpoints at 768px, color variables)

## Behavioral Guidelines

### Think Before Acting
- Don't assume context — if the request is ambiguous, ask.
- If a simpler approach exists, suggest it before implementing the complex one.
- For multi-step tasks, state a brief plan before executing.

### Simplicity First
- Minimum code/markup that solves the problem.
- No speculative features or abstractions for single-use code.
- No "flexibility" that wasn't requested.

### Surgical Changes
- Touch only what the task requires. Don't "improve" adjacent code, styles, or content.
- Match existing patterns and style conventions.
- Every changed line should trace directly to the user's request.

### Content Standards
- Posts are technical content for backend/systems engineers. Direct, practical tone.
- Code examples must be realistic — prefer Java/Spring Boot for backend snippets.
- Never invent fake metrics, team names, or company details.
- No dramatic storytelling or marketing language.
