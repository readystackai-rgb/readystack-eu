# readystack-web

Static front end for **ReadyStack** — a build-less collection of self-contained HTML
pages plus one sample data file. The main application (`index.html`) is a single-file
React app that transpiles its own JSX in the browser; there is no build step, bundler,
package manager, or test suite.

> **Heads-up:** the live site at `https://district.readystack.ai/studio` runs a
> *different, newer* bundled build (with Firebase) whose source is **not** in this
> repository. This repo is the earlier CDN-React lineage. See
> [`CLAUDE.md` §2](CLAUDE.md) before assuming parity with production.

## Quick start

No build. Serve the folder and open a page:

```bash
python -m http.server 8000
# open http://localhost:8000/index.html
```

## Stack

- React 18 + ReactDOM 18 (UMD, via `unpkg.com`)
- `@babel/standalone` — in-browser JSX (`<script type="text/babel">`)
- Tailwind CSS (Play CDN) + `lucide` icons
- Backend: a single Google Cloud Function (`advisor`); no server in this repo

## Layout

- **`index.html`** — canonical main app ("ReadyStack Engagement Utility")
- `landing.html`, `pricing*.html` — marketing pages
- `tour-*.html`, `walkthrough*.html` — onboarding tours
- `privacy-policy.html`, `terms-of-service.html` — legal
- `learning-district.json` — sample **District** document (the data-model reference)
- other pages are theme variants / prototypes / snapshots — see the inventory in `CLAUDE.md`

## Documentation

- **[`CLAUDE.md`](CLAUDE.md)** — agent & contributor guide: what to edit, how to run,
  backend contract, data model, and the traps to avoid. **Start here.**
- **[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)** — runtime, backend, and data-model detail.
