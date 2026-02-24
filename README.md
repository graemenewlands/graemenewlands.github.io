# Hugo-powered GitHub Pages Site

This repository will host a simple Hugo site built on Ubuntu Linux and published via GitHub Pages.  
These instructions assume you're running Ubuntu 20.04+.

---

## 1. Install Hugo (Ubuntu)

The version packaged by Ubuntu's APT is often quite old. Many themes — including
[Ananke](https://github.com/theNewDynamic/gohugo-theme-ananke) — require **Hugo
0.146 or later**.

Options to get a recent release:

```bash
# (1) snap: installs the latest stable, extended build, and is easy to update
sudo snap install hugo --channel=extended

# (2) download a prebuilt binary from https://github.com/gohugoio/hugo/releases
#     and put it on your PATH (e.g. /usr/local/bin). Example:
# curl -LO https://github.com/gohugoio/hugo/releases/download/v0.1XX.X/hugo_extended_0.1XX.X_Linux-64bit.tar.gz
# tar -xzf hugo_*.tar.gz && sudo mv hugo /usr/local/bin/

# (3) apt (only use if you don't care about staying on the latest version):
sudo apt update
sudo apt install hugo        # may install v0.123.x on Ubuntu 22.04/24.04
```

Run `hugo version` to confirm you have at least 0.146; bump the install method
if you see something older.

> ⚠️ Older versions result in module compatibility warnings like:
> ```
> WARN  Module "ananke" is not compatible with this Hugo version: Min 0.146.0;
> ```

> ⚠️ If you prefer another OS, adjust the install step accordingly.

---

## 2. Create a new Hugo site

```bash
cd /path/to/this/repo      # e.g. ~/git/graemenewlands.github.io
hugo new site .            # scaffolds the directory structure
```

Add a theme (example uses [Ananke](https://github.com/theNewDynamic/gohugo-theme-ananke)):

```bash
# initialize git if your directory isn't a repository yet
git init

# recommended: add the theme as a submodule so updates are easy
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke

# or simply clone the theme manually if you prefer
# git clone https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke
```

If you see an error like this when running `hugo new site .` or `hugo server`:

```
Error: failed to load modules: module "ananke" not found in "/path/to/your/site/themes/ananke"; either add it as a Hugo Module or store it in "/path/to/your/site/themes".: module does not exist
```

it means the theme directory is missing.  Re-run the submodule/clone command above to pull down the theme and commit so the site can build.

Edit `config.toml` and set the theme:

```toml
title = "My Hugo Site"
baseURL = "https://<your‑username>.github.io/"
theme = "ananke"
languageCode = "en-us"
```

Create your first content page:

```bash
hugo new posts/hello-world.md
# then edit content/posts/hello-world.md
```

Start a development server:

```bash
hugo server -D
```

Browse to `http://localhost:1313`.

---

## 3. Publish with GitHub Pages

The easiest way is to use GitHub Actions to build and push the generated `public/` folder to the `gh-pages` branch.  
Create the workflow file in `.github/workflows/gh-pages.yml` (see below).

### Workflow example

```yaml
name: "Deploy Hugo site to GitHub Pages"

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
        with:
          submodules: true  # fetch themes

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
          publish_branch: gh-pages
```

> Adjust `branches` and `baseURL` as appropriate.

Once pushed, GitHub Pages can serve from the `gh-pages` branch (repository settings → Pages → branch).

---

## 4. Additional tips

* Edit `config.toml` to customize menus, social links, etc.
* Use `hugo new` to add more pages/sections.
* Test locally before pushing by running `hugo server`.
* You can also configure the Pages source to be `main`/`docs/` if you prefer a different setup.

---

Happy blogging with Hugo and GitHub Pages! 🎉
