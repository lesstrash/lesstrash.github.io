# Jekyll 4.x upgrade — design

**Date:** 2026-04-21
**Scope:** Drop the `github-pages` v197 gem (Jekyll 3.7.4) in favor of Jekyll 4.4.x, with deploy via GitHub Actions. Template HTML/CSS/JS and `_posts/` untouched unless Jekyll 4 forces a change.

## Current state

- `Gemfile`: pins `github-pages '197'` + `jekyll-paginate`.
- `Gemfile.lock`: resolves to Jekyll 3.7.4, ~50 transitive gems (most are unused `jekyll-theme-*`).
- `_config.yml`: declares plugins `jekyll-paginate`, `jekyll-sitemap`; timezone `China/Shanghai` (legacy tzdata name).
- `.github/`: only `FUNDING.yml` and issue/PR templates — no CI.
- Deploy: GitHub Pages default build (locked to Jekyll 3.x).
- Dockerfile: `FROM jekyll/jekyll` (unpinned, outdated image).
- Local: no Ruby installed; Docker 29.4 available.

## Target state

### Gemfile (rewritten)

```ruby
source 'https://rubygems.org'

gem 'jekyll', '~> 4.4'

group :jekyll_plugins do
  gem 'jekyll-paginate'
  gem 'jekyll-sitemap'
  gem 'jekyll-feed'
  gem 'jekyll-seo-tag'
end

gem 'webrick', '~> 1.8'  # Ruby 3+ no longer ships webrick; jekyll serve needs it
```

### `_config.yml`

- `timezone: "China/Shanghai"` → `timezone: "Asia/Shanghai"` (canonical IANA name).
- Any other Jekyll 4 deprecations surfaced during build (e.g., `relative_permalinks`, `gems:` key) get fixed.
- Plugins list kept as-is (`jekyll-paginate`, `jekyll-sitemap`).

### Templates (`_layouts/`, `_includes/`, `css/`)

No changes unless Jekyll 4 build emits warnings/errors. Known possibilities:
- Sass: `@import` still works in `jekyll-sass-converter` 3.x via libsass; Dart-sass would force `@use`. Stick with default (libsass-compat) to avoid churn.
- Liquid 4 syntax changes: none expected for this template's usage.

### GitHub Actions (`.github/workflows/pages.yml`)

Official Jekyll → Pages flow using `actions/deploy-pages`:
- Trigger: push to `master`, plus manual `workflow_dispatch`.
- Ruby 3.3 via `ruby/setup-ruby@v1` (bundler-cache enabled).
- Steps: checkout → setup-ruby → configure-pages → `bundle exec jekyll build` → upload artifact → deploy.
- Concurrency: one deploy at a time per branch.
- **Repo setting change required once:** Settings → Pages → Source: "GitHub Actions" (noted in commit/PR body for the user to flip).

### Dockerfile

Refresh to a pinned, maintained base:
```
FROM ruby:3.3-slim
WORKDIR /srv/jekyll
RUN apt-get update && apt-get install -y --no-install-recommends build-essential git && rm -rf /var/lib/apt/lists/*
COPY Gemfile Gemfile.lock ./
RUN bundle install
COPY . .
EXPOSE 4000
CMD ["bundle", "exec", "jekyll", "serve", "--host", "0.0.0.0"]
```
(`jekyll/jekyll` Docker image is abandoned; the official Jekyll docs now recommend running via ruby base image.)

## Verification

1. **Build green:** `docker build` + `docker run ... jekyll build` exits 0 with no ERROR-level log lines. Warnings acceptable but reviewed.
2. **`_site/` sanity:** homepage, `/tags/`, each post permalink, `feed.xml`, `sitemap.xml` all present and non-empty.
3. **Chrome MCP live check:** `jekyll serve` container on `127.0.0.1:4000`; navigate to `/`, `/tags.html`, one post page, `/aboutus`. DevTools console: no JS errors. Layout renders with navbar + footer + post list.
4. **Visual diff:** compare key pages against a capture of current state (will build the old stack once in Docker first to capture baseline `_site/`).

## Out of scope

- Beautiful Jekyll theme template refresh.
- Post content edits.
- Adding new plugins beyond feed/seo-tag.
- Moving to Dart Sass / modern Sass module syntax.
- Chinese font / typography updates.

## Rollback

Single revert commit restores `Gemfile`/`Gemfile.lock`/`_config.yml`/`Dockerfile` and deletes `.github/workflows/pages.yml`. Repo Pages source toggle back to "Deploy from a branch". No data migrations.
