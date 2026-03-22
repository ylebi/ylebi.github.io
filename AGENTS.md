# AGENTS.md — Coding Agent Guidelines

This is a **Hugo** static site (personal blog) deployed to GitHub Pages.
No JavaScript/TypeScript framework. No test suite. Go module: `lebi.me`.

---

## Stack

| Layer       | Technology                             |
| ----------- | -------------------------------------- |
| SSG         | Hugo 0.124.1 (extended)                |
| Theme       | `github.com/onweru/newsroom` (Go module) |
| Runtime     | Go 1.21.6 (`go.mod`)                  |
| Styles      | Sass (`.sass` indented syntax)         |
| Templates   | Go HTML templates                      |
| Formatting  | Prettier (HTML/CSS/YAML/MD), Black (Python in docs) |
| Linting     | pre-commit hooks                       |

---

## Commands

```bash
# Install all local dependencies (brew packages, python venv)
make init

# Start local dev server (hot-reload, production env)
make devserver
# Equivalent: hugo server --disableFastRender -e production --bind 0.0.0.0 --ignoreCache

# Run all linters / formatters
make lint
# Equivalent: pre-commit run --all-files

# Update theme submodule + pre-commit hooks
make update

# Production build (output to ./public/)
hugo --gc --minify

# Build with explicit base URL (used in CI)
hugo --gc --minify --baseURL "https://lebi.me/"
```

> **No test commands.** There is no test suite in this project.

---

## Directory Structure

```
.
├── archetypes/         # Content templates (hugo new)
├── assets/sass/        # Custom Sass overrides (_syntax.sass)
├── config/_default/    # Hugo config split into TOML files
│   ├── hugo.toml       # Site-level settings (title, theme, paginate)
│   ├── markup.toml     # Goldmark/highlight settings
│   └── params.toml     # Theme params (GA, keywords, blogDir)
├── content/
│   ├── _index.md       # Home page front matter
│   ├── pages/          # Static pages (about.md, etc.)
│   └── posts/          # Blog posts (Markdown)
├── data/
│   └── menu.yml        # Navigation menu items
├── layouts/
│   ├── _default/       # Base templates (baseof.html, single.html)
│   └── partials/       # Reusable partials (head.html, footer.html)
├── static/             # Copied verbatim to /public (images, JS, ads.txt)
└── themes/             # Git submodule for theme overrides (if any)
```

---

## Content Authoring (Markdown Posts)

### Front Matter

All content uses **TOML** front matter (between `+++` delimiters):

```toml
+++
title = "Your Post Title"
date = "2024-04-20"
image = "/images/your-image.webp"
tags = [
  "tag-one",
  "tag-two",
]
categories = [
  "Development",
]
+++
```

- `draft = true` hides posts from production builds
- `image` path is relative to `static/`
- Use `.webp` for post cover images where possible
- `date` format: `"YYYY-MM-DD"` or RFC3339 for pages

### Creating a New Post

```bash
hugo new content posts/my-post-title.md
```

This uses `archetypes/default.md` as the template.

### Code Blocks in Posts

Use fenced code blocks with language identifiers. Tab width is **2 spaces**:

````markdown
```go
func main() {
  fmt.Println("Hello")
}
```

```bash
hugo server
```

```yaml
key: value
```
````

Python code blocks are formatted by `blacken-docs` at 79 chars line length.

---

## Hugo Templates (layouts/)

Templates use **Go HTML template syntax**. Key conventions:

```html
<!-- Cached partials (no page-specific data needed) -->
{{ partialCached "nav" . }}

<!-- Non-cached partials (page-specific data) -->
{{ partial "opengraph" . }}

<!-- Block/define pattern in baseof.html -->
{{ block "main" . }}{{ end }}
```

- Partial files live in `layouts/partials/`
- Override theme partials by placing a file at the same relative path under `layouts/partials/`
- Use `partialCached` for partials that don't vary per page (nav, footer, styles)
- Theme overrides: copy the theme file to the same path under `layouts/` and modify it

---

## Config Files (TOML)

Hugo config is split under `config/_default/`. Use TOML format:

```toml
# hugo.toml — site-level
baseURL = "https://lebi.me/"
theme = ["github.com/onweru/newsroom"]
paginate = 6
```

**Do not** inline all config into a single `hugo.toml` at the root — keep it split.

---

## Data Files (YAML)

Data files under `data/` use YAML:

```yaml
# data/menu.yml
- item: About
  url: about/
```

---

## Sass (assets/sass/)

Uses **indented Sass syntax** (`.sass`), not SCSS:

```sass
.highlight
  margin: 1.5rem 0
  padding: 0 !important

  pre
    padding: 1rem
    border-radius: 4px
```

- No braces, no semicolons
- 2-space indentation
- CSS custom properties (`var(--bg)`, `var(--accent)`, `var(--text)`) for theme colors

---

## Formatting & Linting

Managed by **pre-commit**. Always run before committing:

```bash
make lint
# or directly:
pre-commit run --all-files
```

### Active Hooks

| Hook                  | What it checks/fixes                              |
| --------------------- | ------------------------------------------------- |
| `check-yaml`          | Valid YAML syntax                                 |
| `end-of-file-fixer`   | Files end with a single newline                   |
| `trailing-whitespace` | No trailing spaces                                |
| `black`               | Python code style                                 |
| `blacken-docs`        | Python in markdown code blocks (max 79 chars)     |
| `prettier`            | HTML, CSS, Sass, YAML, Markdown formatting        |

**Prettier** handles indentation, line length, and quote style for all markup files.
Do not manually reformat files — let `make lint` do it.

---

## CI / Deployment

- **CI**: GitHub Actions (`.github/workflows/ci.yml`)
- Deploys on push to `main`
- Build command in CI: `make init && hugo --gc --minify --baseURL "$BASE_URL"`
- Artifact: `./public/` directory → GitHub Pages

Do not commit the `public/` directory (it's in `.gitignore`).

---

## Key Constraints

- **No TypeScript or JavaScript build step** — inline `<script>` tags only
- **No npm / package.json** — dependencies managed by Homebrew + Go modules
- **Theme is a Go module** — update via `make update`, not by editing files inside `themes/`
- **Override theme** by mirroring the file path under `layouts/` or `assets/`, not editing the theme directly
- **Images** go in `static/images/`; prefer `.webp` format
- **Never commit** without running `make lint` first (pre-commit will catch issues)
