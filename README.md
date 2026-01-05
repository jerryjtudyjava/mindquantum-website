# MindQuantum Website

Astro + Jupyter Book monorepo for the MindQuantum website and bilingual documentation.

## Overview

- Astro powers the homepage and overall site shell.
- Jupyter Book builds bilingual tutorials only (EN+ZH).
- Jupyter Book also builds the self-contained course notebooks under `/courses`.
- Sphinx builds the API reference as two projects (EN+ZH) using the internal `mqdocs` extension.
- Shared design tokens keep visual consistency across both.
- GitHub Pages workflow builds and deploys both outputs together.

## Local Development

Prerequisites: Node 18+ and Python 3.9+.

1. Install Node deps

```bash
npm install
```

2. (Optional) Create a Python venv and install Jupyter Book

```bash
python -m venv .venv
source .venv/bin/activate
pip install jupyter-book==1.0.4 sphinx-copybutton sphinx-design sphinx-thebe mindspore mindquantum
```

3. Sync docs content (optional during dev)

The repo can auto-clone the upstream sources into a local cache on first run. To sync both tutorials and API docs:

```bash
npm run sync:all            # clone if missing, reuse cache if present
# or update to the latest upstreams before syncing
python scripts/sync_all.py --update
```

4. Build docs (tutorials + API)

```bash
npm run build:docs   # auto-syncs upstreams, builds JB (tutorials) + Sphinx (API)
```

5. Run the site

```bash
npm run dev
```

## Content Syncing

- Auto-clone: Upstreams are cached under `.upstreams/` using `scripts/upstreams.json`.
- Tutorials: `scripts/sync_mindquantum_from_msdocs.py` vendors Sphinx sources from the cached `mindspore-docs` clone.
- API: `scripts/sync_mindquantum_api.py` vendors API sources from the cached `mindquantum` clone. It also syncs into the standalone Sphinx projects at `docs/api-en/api_python_en` and `docs/api-zh/api_python`.

### API (Sphinx) Projects

In addition to the Jupyter Books, the API reference builds as two Sphinx projects using a clean internal extension (`mqdocs`):

- `docs/api-en/` – English API, uses `ms*autosummary` directives that derive third columns from docstrings
- `docs/api-zh/` – Chinese API, uses `mscnautosummary` directives that read summaries from the local RST pages

Shared assets live under `docs/_ext`, `docs/_templates`, and `docs/_static`.

Local build example:

```bash
# Ensure sources are synced into docs/api-*/
python scripts/sync_mindquantum_api.py

# Build EN API (centralized under docs/_build)
sphinx-build -b html docs/api-en docs/_build/api/en

# Build ZH API (centralized under docs/_build)
sphinx-build -b html docs/api-zh docs/_build/api/zh
```

The extension avoids monkey-patches and implements stable `mscnautosummary`/`ms*autosummary` directives plus a minimal stub generator for those directives only. Standard autosummary remains untouched.

## Build and Deploy

- `npm run build:all` builds Jupyter Books (tutorials + courses) and Sphinx into `public/{docs,courses}/**`, then builds Astro into `dist/`. Temporary artifacts are centralized under `docs/_build/` and `courses/_build/`.
- GitHub Actions workflow `.github/workflows/deploy.yml` builds both and deploys the `dist/` folder to GitHub Pages. It auto-detects committed custom domains via `public/CNAME`, exporting `ASTRO_BASE` and `SITE_URL` before the Astro build.

## Docs Routing

- Tutorials (Jupyter Book): `/docs/en` and `/docs/zh` (from `public/docs/en` and `public/docs/zh`).
- Courses (Jupyter Book): `/courses` (from `public/courses/zh`).
- API (Sphinx): `/docs/api/en` and `/docs/api/zh` (from `public/docs/api/en` and `public/docs/api/zh`).
- The site header should link to Tutorials and API for both languages.

## Theming

- Shared CSS tokens live in `src/styles/tokens.css`.
- A small build step copies tokens to `docs/_static/mq-variables.css` so Jupyter Book can consume them.
- Jupyter Book loads `mq-variables.css` and `mq-theme.css` to style pages in line with the homepage.

## Structure

- `src/` – Astro pages, layouts, and styles.
- `public/` – static assets. Built docs are copied to `public/docs` (ignored by Git).
- `docs/` – Documentation workspace:
  - Jupyter Book projects: `docs/en` and `docs/zh` (tutorials only)
  - Sphinx API projects: `docs/api-en` and `docs/api-zh`
  - Shared assets: `docs/_ext`, `docs/_static`, `docs/_templates`
- `scripts/` – helper scripts (token sync, upstream sync, local build).

## Interactive Tutorials (Thebe + Binder)

Tutorial pages are now Thebe-enabled so users can click Run on code cells directly on the page. This uses Binder to start a live Jupyter kernel per user session.

- Toggle: A “Launch/Activate” button appears on each tutorial page to initialize Thebe. Once activated, each code cell shows a Run button.
- Scope: Execution is client-initiated; notebooks do not auto-execute. This keeps builds deterministic and scales better for traffic.
- Environment: The Binder environment is configured and controlled by the repository https://github.com/MindQuantum-HiQ/mq-env.

### Enhanced UX additions

- Per-cell Run button: Implemented in `docs/_static/mq-thebe.js` and styled by `docs/_static/mq-thebe.css`. It overlays a small “Run” button on Python code cells and notebook cells. On first click, it activates Thebe automatically and then runs the clicked cell.
- Page banner: A compact banner at the top of tutorial pages advertises that the page is runnable in the browser. It provides “Activate” and “Run all examples” actions and mirrors the built-in Thebe status line.
- Inclusion: Both assets are registered in `docs/en/_config.yml` and `docs/zh/_config.yml` under `sphinx.config.html_css_files` and `sphinx.config.html_js_files`.
- Accessibility: Buttons include `aria-label`s; colors use shared tokens for contrast. The overlay avoids the copy button by default and falls back gracefully if Thebe isn’t available on a given page.

Customization & maintenance:

- To disable the banner: remove `mq-thebe.css`/`mq-thebe.js` from `html_css_files`/`html_js_files` in the two Jupyter Book configs.
- To limit per-cell buttons: adjust the selector logic in `mq-thebe.js` (function `attachRunButtons`) to match only your preferred patterns (e.g., only `.cell` or only `code.language-python`). Cells inside `.thebe-ignored` are skipped automatically.
- The script avoids private Thebe internals: it triggers the standard launch button and clicks the cell’s built-in `.thebe-run-button`. This keeps it resilient across Thebe/Jupyter Book updates.

## Quantum Graphical Programming (Homepage)

The homepage includes an interactive circuit builder that enables quantum graphical programming directly in the browser.

- Drag-and-drop gates, pick controls/targets, and view simulated results live.
- Implemented as a standalone web component `mq-circuit-builder` with:
  - UI logic in `src/components/circuit/QuantumCircuitElement.ts`
  - Styles in `src/components/circuit/styles.css`
- Wrapped for Astro via `src/components/QuantumCircuitBuilder.astro` and rendered on
  `src/pages/index.astro` and `src/pages/[lang]/index.astro`.
- Labels/i18n come from `src/locales/home.ts` under the `builder` section.
- Runs fully client-side; no server or Python environment required.
