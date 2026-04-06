# AGENTS.md — Coding Agent Guidelines

**Generated:** 2026-04-06 · **Commit:** 6d03c2b · **Branch:** main

Hugo static site (personal blog) deployed to GitHub Pages.
No JavaScript/TypeScript framework. No test suite. Go module: `lebi.me`.

---

## Stack

| Layer      | Technology                                           |
| ---------- | ---------------------------------------------------- |
| SSG        | Hugo 0.158.0 extended (local + CI-pinned)            |
| Theme      | `github.com/onweru/newsroom` (Go module, `go.mod`)   |
| Runtime    | Go 1.21.6                                            |
| Styles     | Dart Sass 1.98.0 — `.sass` indented syntax           |
| Templates  | Go HTML templates                                    |
| Formatting | Prettier v4.0.0-alpha.8 · Black 24.4.0               |
| Linting    | pre-commit (`fail_fast: true`)                       |

---

## Commands

```bash
make init          # macOS only: brew deps + Dart Sass + uv venv. No-op in CI (CI env var guard).
make devserver     # hugo server --disableFastRender -e production --bind 0.0.0.0 --ignoreCache
make lint          # pre-commit run --all-files
make update        # git submodule update --remote (legacy/no-op) + pre-commit autoupdate -j 4

hugo --gc --minify                               # Production build → ./public/
hugo --gc --minify --baseURL "https://lebi.me/"  # CI build (baseURL from Pages action)
```

> **No test suite.** `make lint` is the only verification step before committing.

---

## Directory Structure

```
├── archetypes/default.md         # hugo new template
├── assets/sass/_syntax.sass      # Syntax highlight + copy-code-button CSS overrides
├── config/_default/              # Split Hugo config (TOML — do not collapse to single file)
│   ├── hugo.toml                 # baseURL, theme, paginate=6, permalinks
│   ├── markup.toml               # Goldmark + highlight (lineNos=true, tabWidth=2)
│   └── params.toml               # GA ID, keywords, blogDir="posts"
├── content/
│   ├── _index.md                 # Home page
│   ├── pages/                    # Static pages (about, etc.)
│   └── posts/                    # Blog posts
├── data/menu.yml                 # Nav items (YAML)
├── i18n/en.toml                  # Theme i18n string overrides (archive, share, etc.)
├── layouts/
│   ├── index.html                # Override: PagerSize fix + home post list
│   ├── _default/
│   │   ├── baseof.html           # Override: adds GTM noscript + vanilla-back-to-top CDN
│   │   └── single.html           # Override: Disqus, copy-code-button, custom date format
│   └── partials/
│       ├── head.html             # Override: GA verify meta, AdSense script, GTM script
│       ├── styles.html           # Override: resources.ToCSS → css.Sass fix; uses ExecuteAsTemplate
│       └── footer.html           # Override: comments out attribution; JS minify via resources.Minify
└── static/
    ├── images/                   # Post images — prefer .webp
    └── js/copy-code-button.js    # Manually maintained; loaded only when <pre> found in content
```

---

## WHERE TO LOOK

| Task                        | Location                              | Notes                                      |
| --------------------------- | ------------------------------------- | ------------------------------------------ |
| Add/edit blog post          | `content/posts/`                      | `hugo new content posts/slug.md`           |
| Change nav links            | `data/menu.yml`                       | YAML                                       |
| Change site title/GA/logo   | `config/_default/params.toml`         |                                            |
| Change paginate/baseURL     | `config/_default/hugo.toml`           |                                            |
| Change code highlight style | `assets/sass/_syntax.sass`            | Indented `.sass` syntax                    |
| Add `<head>` tags           | `layouts/partials/head.html`          |                                            |
| Override theme template     | Mirror path under `layouts/`          | Never edit `themes/` or module cache       |
| Add i18n string             | `i18n/en.toml`                        | Overrides theme strings                    |
| Change footer               | `layouts/partials/footer.html`        |                                            |
| Tweak copy-code button      | `static/js/copy-code-button.js`       | Loaded conditionally by `single.html`      |

---

## Content Authoring

All content uses **TOML** front matter (`+++` delimiters):

```toml
+++
title = "Post Title"
date = "2024-05-16"
image = "/images/my-image.webp"
tags = ["kubernetes", "operator"]
categories = ["Development"]
+++
```

- `draft = true` hides from production; remove when publishing
- `date`: `"YYYY-MM-DD"` for posts, RFC3339 for pages
- `image`: path is relative to `static/`; prefer `.webp`
- Tags/categories: lowercase, hyphen-separated
- Python snippets auto-formatted by `blacken-docs` at 79 chars — do not manually format them

---

## Hugo Templates

### Theme Override Pattern

The `newsroom` theme loads as a **Go module** — never edit module cache or `themes/`.
Override by mirroring the path under `layouts/` or `assets/`:

```
Theme:    github.com/onweru/newsroom/layouts/partials/styles.html
Override: layouts/partials/styles.html   ← local file takes precedence
```

### Partial Caching Rule

```html
{{ partialCached "styles" . }}   <!-- no per-page data → cacheable -->
{{ partial "opengraph" . }}      <!-- page-specific data → NOT cached -->
```

### Hugo API Compatibility (theme is outdated — use current API in overrides)

| Removed API             | Replacement        | Since        |
| ----------------------- | ------------------ | ------------ |
| `resources.ToCSS $opts` | `css.Sass $opts`   | Hugo 0.128.0 |
| `$pager.PageSize`       | `$pager.PagerSize` | Hugo 0.125.0 |
| `:filename` permalink   | `:contentbasename` | Hugo 0.144.0 |

---

## Sass (assets/sass/)

**Indented `.sass` syntax** — no braces, no semicolons, 2-space indent.
Theme CSS custom properties: `var(--bg)`, `var(--text)`, `var(--accent)`, `var(--theme)`, `var(--light)`, `var(--dark)`.

`styles.html` uses `resources.ExecuteAsTemplate` before `css.Sass` — allows Hugo template vars inside `.sass` files.

---

## Formatting & Linting

**Do not manually reformat files.** Prettier owns indentation, quotes, and line length.
`fail_fast: true` — hooks stop on the first failure.

Pre-commit hooks: `check-yaml`, `end-of-file-fixer`, `trailing-whitespace`, `black`, `blacken-docs` (errors suppressed via `|| true`), `prettier`.

---

## CI / Deployment

- GitHub Actions: `.github/workflows/ci.yml`, triggers on push to `main`
- CI pins: `HUGO_VERSION=0.158.0`, `DART_SASS_VERSION=1.98.0` (installed via curl, not brew)
- CI: `make init` is a no-op (CI env var guard); Hugo + Dart Sass installed explicitly before build
- Output: `./public/` → GitHub Pages. **Do not commit `public/`.**

---

## Key Constraints

- **No JS build step** — no npm/package.json; inline `<script>` or `static/js/` only
- **No editing theme source** — `layouts/`/`assets/` overrides only; update via `make update`
- **Dart Sass required** — `make init` installs it; `css.Sass` fails without it
- **Images** → `static/images/`, prefer `.webp`
- **Config** → TOML only, split under `config/_default/`; do not collapse to single root file
- **Always run `make lint` before committing** — pre-commit will block malformatted files

---

## Gotchas

- `vanilla-back-to-top` loaded from unpkg CDN in `layouts/_default/baseof.html` — not a local file
- Disqus: disabled by default (`disqusShortname` commented out in `hugo.toml`); enabled via `_internal/disqus.html` template in `single.html`
- `make update` runs `git submodule update --remote` which is **legacy** (theme is a Go module, not a submodule); the submodule step is harmless but unused
- `blacken-docs` hook uses `|| true` — Python code formatting errors in Markdown are suppressed, not blocking
- Prettier pinned to `v4.0.0-alpha.8` (alpha); may have rough edges
