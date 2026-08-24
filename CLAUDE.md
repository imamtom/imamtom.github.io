# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is `imamtom.github.io` — a Jekyll-based personal academic homepage for Li Wenjie (PhD student, Xidian University, researching cryptography / privacy-preserving federated learning). It is a fork of [RayeRen/acad-homepage.github.io](https://github.com/RayeRen/acad-homepage.github.io), which itself derives from `mmistakes/minimal-mistakes` and `academicpages`.

The site is **dual-page, single-language-sectioned**: the English homepage at `/` lives in `_pages/about.md` (frontmatter `permalink: /`, `redirect_from: /about/, /about.html`); the Chinese homepage at `/zh/` lives in `_pages/about-zh.md` (frontmatter `permalink: /zh/`, `author: zh` to pull the Chinese sidebar profile from `_data/authors.yml`). Both files share the masthead, sidebar, scripts, and the `{% include publications.html %}` block.

## Common commands

- **Install dependencies** (Ruby gems for Jekyll): `bundle install`
- **Run local dev server** with live reload at http://127.0.0.1:4000: `bash run_server.sh` (this first re-runs `bibtex_build/render.sh`, then `bundle exec jekyll serve`)
- **One-off build** without serving: `bundle exec jekyll build`
- **Re-render the Publications block after editing `_data/ref.bibtex`**: `bash bibtex_build/render.sh`

The Python crawler has its own dependencies (`scholarly`, `jsonpickle`); install with `pip install -r google_scholar_crawler/requirements.txt`.

There are no tests, lint configs, or CI workflows beyond the Google Scholar crawler and the locally-run bibtex render.

## Architecture overview

### Content layer

- `_pages/about.md` — English homepage. Sections are anchored (`#about-me`, `#-news`, `#-publications`, `#-honors-and-awards`, `#-educations`, `#-internships`) and those IDs must stay in sync with `_data/navigation.yml`. The Publications section is rendered via `{% include publications.html %}`.
- `_pages/about-zh.md` — Chinese homepage. Mirror of `about.md` with `permalink: /zh/` and `author: zh`.
- `_data/navigation.yml` — masthead menu items. The first two entries are the **language toggle**: `English → /` and `🇨🇳 中文 → /zh/`. The rest are anchor links.
- `_data/authors.yml` — author profiles keyed by `page.author`. The English page uses `site.author` from `_config.yml`; the Chinese page uses the `zh:` block here.
- `_data/ref.bibtex` — **single source of truth for publications**. Order in the file is preserved by pandoc (newest first). Each entry is one paper.
- `_pages/about.md` / `_pages/about-zh.md` allow inline HTML + Markdown. The paper-citation inline widget is `<span class='show_paper_citations' data='GOOGLE_SCHOLAR_PAPER_ID'></span>`; the paper ID is the segment after `citation_for_view=` in a Google Scholar citation URL.

### Template layer

- `_layouts/default.html` — single layout used by all pages (`defaults:` in `_config.yml` forces `layout: default`, `author_profile: true`).
- `_includes/`
  - `head.html` + `head/custom.html` — meta, CSS link, favicons, MathJax (LaTeX), academicons CSS.
  - `masthead.html` — top nav (renders `_data/navigation.yml`). Adds `.masthead__menu-item--active` to the entry whose `link.url` matches `page.url` so the current language toggle is highlighted.
  - `sidebar.html` + `author-profile.html` — left sidebar with avatar, bio, and ~30 social/profile links rendered conditionally from `site.author.*` keys (or `page.author` from `_data/authors.yml`).
  - `scripts.html` — loads `main.min.js`, Google Analytics, and `fetch_google_scholar_stats.html`.
  - `fetch_google_scholar_stats.html` — client-side jQuery script that fetches `gs_data.json` from the `google-scholar-stats` branch and fills `#total_cit` + every `.show_paper_citations` element.
  - `publications.html` — **generated, do not edit by hand**. Re-rendered from `_data/ref.bibtex` by `bibtex_build/render.sh`.

**Asset paths in `_includes/` use `relative_url`** (e.g. `href="{{ '/assets/css/main.css' | relative_url }}"`) because the Chinese homepage lives at `/zh/` and bare `assets/...` URLs would otherwise resolve to `/zh/assets/...` which doesn't exist. Keep this convention when adding new includes.

### Style layer

- `assets/css/main.scss` is the entry; it imports `_sass/*.scss` (variables, base, navigation, page, sidebar, masthead, footer, etc.) plus vendored libraries under `_sass/vendor/` (breakpoint, susy, font-awesome, magnific-popup).
- Custom rules at the bottom of `main.scss`:
  - `.paper-box` / `.badge` / `.anchor` — the optional image+text card used in the commented-out Publications example.
  - `.masthead__menu-item--active > a` — bold + blue (#00369f) underline for the current language toggle.
  - `.publications .csl-entry` — hanging-indent layout for the GB/T 7714 bibliography.

### Publications pipeline (BibTeX → GB/T 7714)

1. User maintains `_data/ref.bibtex` (one `@article{...}` per paper, with `author`, `title`, `journal`, `year`, `url`).
2. `bibtex_build/render.sh` runs `pandoc --citeproc --bibliography=_data/ref.bibtex --csl=bibtex_build/china-national-standard-gb-t-7714-2015-numeric.csl` and pipes the output (a `<div id="refs" class="references csl-bib-body">…</div>` fragment) into `_includes/publications.html` wrapped in `<div class="publications">`.
3. The `<!-- This file is auto-generated -->` comment marks the include as generated. **Do not edit it by hand** — edits will be overwritten the next time `render.sh` executes.
5. The CSL file lives at `bibtex_build/china-national-standard-gb-t-7714-2015-numeric.csl` (downloaded from the [citation-style-language/styles](https://github.com/citation-style-language/styles) repo; commits should keep it in sync if upstream changes).
6. `run_server.sh` calls `bibtex_build/render.sh` before starting Jekyll, so editing `_data/ref.bibtex` and re-saving → re-render → live server refresh.

**Dependency**: requires `pandoc ≥ 2.11` (with `--citeproc`). The render script errors out clearly if pandoc is missing. GitHub Pages itself does **not** run this script — it uses the committed `_includes/publications.html`. So commit both `_data/ref.bibtex` and `_includes/publications.html` together whenever the bibliography changes.

### Citation auto-update pipeline (Google Scholar)

A scheduled GitHub Action in `.github/workflows/google_scholar_crawler.yaml` runs:

1. **Trigger**: every day at 08:00 UTC (`cron: '0 8 * * *'`) AND on every `page_build`.
2. **Step**: `python3 google_scholar_crawler/main.py` reads `GOOGLE_SCHOLAR_ID` from repo secrets, calls the `scholarly` library, and writes `gs_data.json` (full author record) and `gs_data_shieldsio.json` (just the citation count, shields.io-compatible).
3. **Output**: force-pushes the two JSON files to a branch called `google-scholar-stats`.
4. **Consumed by**: `_includes/fetch_google_scholar_stats.html` (page-load) and the shields.io badge in `_pages/about.md` and `_pages/about-zh.md` (the `img.shields.io/endpoint?url=...gs_data_shieldsio.json...` image).

CDN switch: `_config.yml` key `google_scholar_stats_use_cdn` toggles between `cdn.jsdelivr.net/gh/<repo>@` and `raw.githubusercontent.com/<repo>/` for both consumers. `true` is friendlier from mainland China but adds jsdelivr cache delay.

### Misc

- `_config.yml` `author.*` is the single source of truth for sidebar/profile links on the English page; most keys are optional and conditionally rendered.
- `Gemfile` pins `github-pages` gem group (so local build matches GitHub Pages' pinned plugin versions). Plugins enabled: `jekyll-paginate`, `jekyll-sitemap`, `jekyll-gist`, `jekyll-feed`, `jekyll-redirect-from`. The `wdm` gem is NOT included (incompatible with Ruby 3.x); Jekyll on Windows falls back to polling for file-change events. `tzinfo-data` is included so `tzinfo` 2.x finds timezone data on Windows.
- `assets/fonts/` ships Font Awesome + academicons as static assets (no CDN).
- `images/` holds favicons + avatar (`android-chrome-512x512.png`).
- The actual Ruby is at `/d/Ruby33-x64/bin/` on this machine. The Microsoft Store shim at `/c/Users/imlee/AppData/Local/Microsoft/WindowsApps/{bundle,bundler,ruby}` is broken (points to a non-existent `ruby.exe`) — `run_server.sh` prepends `/d/Ruby33-x64/bin` to `PATH` to avoid it.

## Things to know before editing

- `_config.yml` is NOT auto-reloaded by `jekyll serve` — restart the server after config edits.
- Anchor IDs in `_pages/about.md` use the kramdown-auto-generated form (e.g. `🔥 News` → `#-news` because emoji is stripped). If you rename a heading, update the matching entry in `_data/navigation.yml`.
- The crawler writes to a separate branch (`google-scholar-stats`); the action uses `--force` push. Don't run `main.py` against your local checkout expecting the output to land on `main`.
- The Chinese page (`/zh/`) requires asset URLs to use `relative_url` (or absolute `/assets/...`); bare `assets/...` will 404. This is already fixed in `_includes/`, but watch for it when adding new templates.
- `_includes/publications.html` is auto-generated — edit `_data/ref.bibtex` and run `bash bibtex_build/render.sh` instead.
- There is no automated test or build check beyond GitHub Pages' own Jekyll build — the deploy is implicit on push to `main`.