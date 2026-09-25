# Graeme Newlands Personal Website - Agent Context & Guidelines

> **Project Identity**: Personal website and engineering blog for Graeme Newlands, hosted via GitHub Pages at **[graemenewlands.com](https://graemenewlands.com/)**.
> **Repository**: [`graemenewlands/graemenewlands.github.io`](https://github.com/graemenewlands/graemenewlands.github.io)
> **Engine**: Hugo (Extended) with the [Ananke](https://github.com/theNewDynamic/gohugo-theme-ananke) theme (configured as a submodule in `themes/ananke`).

---

## 1. Directory Layout & Architecture

```
.
├── config.toml                     # Primary Hugo site configuration (menu, baseURL, theme params)
├── content/
│   ├── _index.md                   # Homepage hero content & cover image configuration
│   └── posts/
│       ├── building-ops5-in-go.md  # Live blog post: OPS5 Rete rule engine implementation
│       ├── conways-game-of-life-in-ops5.md # Live blog post: Declarative cellular automata in WebAssembly
│       └── hello-world.md          # Initial placeholder post (draft = true)
├── static/
│   ├── CNAME                       # GitHub Pages custom domain (graemenewlands.com)
│   ├── .nojekyll                   # Prevents GitHub Pages from running Jekyll
│   ├── images/
│   │   ├── cover.jpeg              # Homepage hero header background image
│   │   └── ops5-cover.jpeg         # Featured header image for OPS5 blog post
│   └── life/                       # Hosted Conway's Game of Life WebAssembly App
│       ├── index.html              # Standalone Life UI with interactive rule inspector
│       ├── style.css               # Life app dark theme styling
│       ├── app.js                  # Canvas renderer, state loop, and Wasm bridge
│       ├── wasm_exec.js            # Go 1.23 WebAssembly JS runtime glue
│       └── main.wasm               # Compiled OPS5 Life engine binary (GOOS=js GOARCH=wasm)
├── themes/
│   └── ananke/                     # Ananke theme submodule
├── .github/
│   └── workflows/
│       └── gh-pages.yaml           # GitHub Actions workflow: builds & deploys to gh-pages branch
└── public/                         # Hugo build output (mirrors gh-pages deployment artifact)
```

---

## 2. Current State & Recent Work

- **Custom Domain & Persistence**:
  - `baseURL` is configured to `https://graemenewlands.com/` in `config.toml`.
  - `static/CNAME` contains `graemenewlands.com` ensuring that GitHub Action orphan deployments to `gh-pages` preserve the custom domain.
  - Live deployment verified serving HTTP 200 at `https://graemenewlands.com/`.

- **Interactive WebAssembly Apps**:
  - **Conway's Game of Life (`/life/`)**: Fully functional client-side Conway's Game of Life simulation powered by the `ops5` Rete pattern matching rule engine compiled to WebAssembly. Includes rule hot-reloading, pattern presets, step metrics, and toroidal wrapping. Source repository: [`graemenewlands/go-ops5-apps`](https://github.com/graemenewlands/go-ops5-apps).
  - Main site navigation links directly to `/life/` alongside `/posts/`.

- **Visual Assets**:
  - Homepage cover image: `static/images/cover.jpeg` (referenced in `content/_index.md` as `/images/cover.jpeg`).
  - OPS5 post image: `static/images/ops5-cover.jpeg` (referenced in `content/posts/building-ops5-in-go.md` as `/images/ops5-cover.jpeg`).
  - Duplicate/scratch image files have been cleaned up to maintain a lean repository footprint.

- **Blog Content**:
  - `content/posts/building-ops5-in-go.md`: Detailed engineering log covering the creation of [`graemenewlands/ops5`](https://github.com/graemenewlands/ops5), the Rete algorithm, and the OPT-1 through OPT-7 optimizations. Clearly marked as a *Work in Progress*, linking directly to the live `/life/` demo.
  - `content/posts/conways-game-of-life-in-ops5.md`: Deep dive explaining how `go-ops5-apps/apps/life` leverages the `ops5` engine, the 5-stage declarative pipeline (emit votes, tally accumulator, evaluate, promote, quiescence), and Wasm integration.

---

## 3. Critical Invariants & Constraints

When modifying or extending this codebase, adhere strictly to these rules:

1. **Push Constraint**: **NEVER push commits or tags to `origin` without explicit user permission.**
2. **CNAME Preservation**: Always keep `static/CNAME` containing `graemenewlands.com`. If deleted, automated orphan builds (`peaceiris/actions-gh-pages`) will drop the custom domain configuration on GitHub Pages.
3. **Static Image Paths**: Store site images in `static/images/` and reference them with an absolute root path (e.g., `featured_image: "/images/<filename>.jpeg"`). Hugo copies files in `static/` directly into the web root.
4. **Draft Status**: Hugo ignores files with `draft: true` when building for production. New posts intended for publication must set `draft: false`.
5. **Theme Submodule Integrity**: The `themes/ananke` directory is a git submodule; avoid committing untracked edits directly inside `themes/` unless explicitly intending to fork or update the submodule pointer.

---

## 4. Common Commands & Workflows

### Local Development & Preview
To start the local development server with drafts and live reloading:
```bash
hugo server -D -p 1313
```
Preview in browser at: `http://localhost:1313/`

### Production Build
To verify the site compiles cleanly without errors:
```bash
hugo --minify
```

### Creating a New Post
Create a new markdown file under `content/posts/<post-slug>.md` with front matter:
```markdown
---
title: "Your Post Title"
date: 2026-09-24T00:00:00-07:00
draft: false
featured_image: "/images/<image-name>.jpeg"
description: "Brief summary for SEO and previews."
tags: ["Tag1", "Tag2"]
---
```

---

## 5. Deployment Lifecycle

1. Commits pushed to `main` trigger the GitHub Actions workflow [`.github/workflows/gh-pages.yaml`](.github/workflows/gh-pages.yaml).
2. The action checks out `main` (including submodules), sets up Hugo Extended, runs `hugo --minify`, and deploys `./public` to the `gh-pages` branch using `peaceiris/actions-gh-pages@v3`.
3. GitHub Pages serves the `gh-pages` branch at `https://graemenewlands.com/`.
