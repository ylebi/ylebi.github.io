# AGENTS.md — Coding Agent Guidelines

Hugo static site (personal blog) deployed to GitHub Pages.
No JavaScript/TypeScript framework. No test suite. Go module: `lebi.me`.

---

## Stack

| Layer      | Technology                                          |
| ---------- | --------------------------------------------------- |
| SSG        | Hugo 0.158.0+ extended (local) · 0.158.0 (CI-pinned) |
| Theme      | `github.com/onweru/newsroom` (Go module)            |
| Runtime    | Go 1.21.6 (`go.mod`)                               |
| Styles     | Dart Sass — `.sass` indented syntax                 |
| Templates  | Go HTML templates                                   |
| Formatting | Prettier (HTML/CSS/Sass/YAML/MD) · Black (Python)   |
| Linting    | pre-commit (`fail_fast: true`)                      |

---

## Commands

```bash
make init          # Install brew deps (incl. Dart Sass), python venv
make devserver     # hugo server --disableFastRender -e production --bind 0.0.0.0 --ignoreCache
make lint          # pre-commit run --all-files
make update        # Update theme Go module + pre-commit hooks

hugo --gc --minify                              # Production build → ./public/
hugo --gc --minify --baseURL "https://lebi.me/" # CI build
```

> **No test suite.** `make lint` is the only verification step before committing.

---

## Directory Structure

```
├── archetypes/         # hugo new templates (default.md)
├── assets/sass/        # Custom Sass overrides (_syntax.sass)
├── config/_default/    # Split Hugo config (TOML)
│   ├── hugo.toml       # baseURL, theme, paginate=6
│   ├── markup.toml     # Goldmark + highlight (lineNos=true, tabWidth=2)
│   └── params.toml     # GA, keywords, blogDir="posts"
├── content/
│   ├── _index.md       # Home page
│   ├── pages/          # Static pages (about, etc.)
│   └── posts/          # Blog posts
├── data/menu.yml       # Nav items (YAML)
├── layouts/
│   ├── index.html      # Theme override: Pager.PagerSize fix
│   ├── _default/       # baseof.html, single.html
│   └── partials/
│       ├── head.html   # Custom: GA, AdSense, GTM tags
│       └── styles.html # Theme override: css.Sass fix
└── static/             # Verbatim copy to /public (images/, js/)
```

---

## Content Authoring

### Front Matter

All content uses **TOML** front matter (`+++` delimiters):

```toml
+++
title = "Monitoring and Logging for Memcached-Operator"
date = "2024-05-16"
image = "/images/monitoring-and-logging-for-memcached-operator.webp"
tags = [
  "kubernetes",
  "operator",
]
categories = [
  "Development",
]
+++
```

- `draft = true` hides from production; remove when publishing
- `image` path is relative to `static/`; use `.webp`
- `date`: `"YYYY-MM-DD"` for posts, RFC3339 for pages
- Tags/categories: lowercase, hyphen-separated

### New Post

```bash
hugo new content posts/my-post-title.md   # uses archetypes/default.md
```

### Code Blocks

Fenced with language identifier. Tab width = **2 spaces** (`markup.toml`):

````markdown
```go
func main() {
  fmt.Println("Hello")
}
```
````

Python snippets in Markdown are auto-formatted by `blacken-docs` at 79 chars — do not manually format them.

---

## Hugo Templates

### Patterns

```html
{{ partialCached "styles" . }}   <!-- cached: no per-page data -->
{{ partial "opengraph" . }}      <!-- not cached: page-specific data -->
{{ block "main" . }}{{ end }}    <!-- baseof.html block/define pattern -->
```

### Theme Override Pattern

The `newsroom` theme loads as a **Go module** — never edit module cache or `themes/`.
Override by mirroring the path under `layouts/` or `assets/`:

```
Theme:    github.com/onweru/newsroom/layouts/partials/styles.html
Override: layouts/partials/styles.html   ← local file takes precedence
```

### Hugo API Compatibility (theme is outdated — use current API in overrides)

| Removed API              | Replacement          | Since       |
| ------------------------ | -------------------- | ----------- |
| `resources.ToCSS $opts`  | `css.Sass $opts`     | Hugo 0.128.0 |
| `$pager.PageSize`        | `$pager.PagerSize`   | Hugo 0.125.0 |
| `:filename` permalink    | `:contentbasename`   | Hugo 0.144.0 |

**Dart Sass is required** for `css.Sass`. `make init` installs it via `brew install sass/sass/sass`.

---

## Sass (assets/sass/)

**Indented `.sass` syntax** — no braces, no semicolons, 2-space indent:

```sass
.highlight
  margin: 1.5rem 0
  padding: 0 !important

  pre
    padding: 1rem
    border-radius: 4px
```

Theme CSS custom properties for colors: `var(--bg)`, `var(--text)`, `var(--accent)`, `var(--theme)`, `var(--light)`, `var(--dark)`.

---

## Config Conventions

**Hugo config** — TOML, split under `config/_default/`. Do not collapse to a single root file.

**Data files** — YAML under `data/`:

```yaml
- item: About
  url: about/
```

---

## Formatting & Linting

```bash
make lint                    # run all pre-commit hooks on all files
pre-commit run --all-files   # equivalent
```

| Hook                  | What it enforces                             |
| --------------------- | -------------------------------------------- |
| `check-yaml`          | Valid YAML syntax                            |
| `end-of-file-fixer`   | Single trailing newline                      |
| `trailing-whitespace` | No trailing spaces                           |
| `black`               | Python code style                            |
| `blacken-docs`        | Python in Markdown (79-char line limit)      |
| `prettier`            | HTML, CSS, Sass, YAML, Markdown formatting   |

**Do not manually reformat files.** Prettier owns indentation, quotes, and line length.
`fail_fast: true` — hooks stop on the first failure.

---

## CI / Deployment

- GitHub Actions: `.github/workflows/ci.yml`, triggers on push to `main`
- CI pins Hugo at **0.158.0** (`HUGO_VERSION` in `ci.yml`) — matches local dev
- CI build: `make init && hugo --gc --minify --baseURL "$BASE_URL"`
- Output: `./public/` → GitHub Pages

Do not commit `public/` (in `.gitignore`).

---

## Key Constraints

- **No JS build step** — inline `<script>` tags only; no npm/package.json
- **No editing theme source** — use `layouts/`/`assets/` overrides; update via `make update`
- **Dart Sass required** — `make init` installs it; `css.Sass` fails without it
- **Images** → `static/images/`, prefer `.webp`
- **Always run `make lint` before committing** — pre-commit will block malformatted files
