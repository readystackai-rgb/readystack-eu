# CLAUDE.md — Agent & contributor guide for `readystack-web`

This file orients an AI agent (or a new human) working in this repository. Read it
before editing. It states what this repo **is**, what it **is not**, how to run it,
and the traps that will otherwise waste your time.

---

## 1. What this repository is

A **build-less, static front end** for ReadyStack — a set of self-contained HTML
pages plus one sample data file. There is **no build step, no bundler, no package
manager, and no test suite.** Every page is opened directly in the browser; the app
compiles its own JSX at runtime.

- **Stack (all via CDN, no local install):**
  - React 18 + ReactDOM 18 (UMD builds from `unpkg.com`)
  - `@babel/standalone` — JSX is transpiled **in the browser**; app code lives in
    `<script type="text/babel">` blocks
  - Tailwind CSS via the Play CDN (`cdn.tailwindcss.com`)
  - `lucide` icons via `unpkg.com`
- **Language:** plain JS/JSX inside HTML. No TypeScript, no `.jsx`/`.js` source files.
- **The main application is [`index.html`](index.html)** (~6.5k lines, title
  "ReadyStack Engagement Utility"). It is a single-file React app.

## 2. What this repository is NOT (read this — it prevents wrong work)

> **The live site at `https://district.readystack.ai/studio` is NOT built from this
> repository.** Production serves a newer, **bundled** build (`dist/bundle.js` + a
> service worker + Firebase Auth/Firestore/FCM). That source is **not committed
> anywhere in this repo.**

Consequences you must respect:

- This repo is the **earlier CDN-React lineage** (deployment marker dated
  `2026-03-01`; last commit `2026-03-25`). The live studio diverged after that onto
  Firebase + a bundler.
- Features that exist **only in the live build, not here**, include the district
  **list** view (`listDistricts`) and anything Firebase-backed. Do **not** assume
  parity with production, and do not add "fixes" here expecting them to affect the
  live site.
- If a task is about the live `/studio` behaviour, the code is elsewhere — flag that
  rather than editing this repo.

## 3. How to run / preview

No build. Serve the folder statically and open a page:

```bash
# any static server works; pick one
python -m http.server 8000
#   then open http://localhost:8000/index.html
```

Opening `index.html` via `file://` mostly works, but use a local server so that
`fetch` calls, the manifest, and relative asset paths behave normally.

There is nothing to compile, lint, or test in-repo. "Deploy" = copy the static files
to the host.

## 4. Backend contract

The front end has no server of its own. It calls one Google Cloud Function.

- **Endpoint:** `https://us-central1-gen-lang-client-0769430147.cloudfunctions.net/advisor`
  (defined as `CLOUD_FUNCTION_URL` in `index.html`).
- **Protocol:** `POST` JSON. The function **dispatches on a `task` field**. An
  unknown/absent task returns `{"error":"Unknown task: \"…\"","bodyKeys":[…]}`.
- **Tasks this repo actually calls:** `aiReviewServer`, `generateStory`,
  `generateStoryServer`, `runVisionCheck`, `publishLandingPage`.
  (The live build uses more tasks — e.g. `listDistricts` — that this repo does not.)
- **Image proxy:** YouTube thumbnails are fetched through a Google opensocial proxy
  (`images1-focus-opensocial.googleusercontent.com/gadgets/proxy`), built as
  `proxyUrl` in `index.html`.

The `advisor` function's source is **not in this repo** (it is a separate GCP
project). Server-side changes cannot be made from here.

## 5. Core data model — a "District"

The app's central domain object is a **District**. See
[`learning-district.json`](learning-district.json) for a complete real example. Shape:

| Field | Meaning |
| --- | --- |
| `districtName`, `stackLabel` | the district's name |
| `selectedVideoData`, `youtubeEmbedUrl` | the anchor YouTube video |
| `videoStory` | the generated narrative: `problem`, `resident`, `network{partners,customers,targets}`, `checklist[]`, `automations[{trigger,action,outcome}]`, `narrative` |
| `storyParticipants.residents[]` | people in the district (`name,company,specialty,email,department`) |
| `mapping.{learning,selecting,executing,expanding}[]` | the funnel grid: rows of `{ven,lab,platform,ga4Count,…}` |
| `heroFields` | landing-page hero (`headline`, `youtubeUrl`, `ctaLabel/Url`, `mindMapLink`) |
| `suggestedRoles[]` | role suggestions for participants |

Note the vocabulary is **districts / residents / automations** — this does **not**
correspond to the BridgeMail Workflow API's `workflows / steps / options / rules`.
They are different domains; there is no shared schema.

## 6. Page inventory

| File | Role | Notes |
| --- | --- | --- |
| **`index.html`** | **Canonical main app** | the one to edit for the studio UI |
| `index_pre-production.html` | Pre-prod snapshot of the app | not canonical |
| `index.txt`, `index_pre-production.txt` | HTML saved with a `.txt` extension | snapshots/backups, not served |
| `landing.html` | Marketing landing page | |
| `pricing.html`, `pricing-details.html`, `automation-pricing.html` | Pricing pages | |
| `privacy-policy.html`, `terms-of-service.html` | Legal | |
| `tour-01-define-market-story.html`, `tour-02-residents.html`, `tour-03-resident-links.html` | Guided onboarding tour | |
| `walkthrough.html`, `walkthrough-v3.html` | Interactive walkthrough | near-duplicates (both titled "v3"); reconcile before editing |
| `artist-district.html`, `artist-district-index.html`, `music-storyteller.html` | District-theme variants | prototypes |
| `district-operations-prototype.html` | Compose-view prototype | |
| `join.html` | District join/invite page | |
| `learning-district.json` | Sample District document | the schema reference |

## 7. Conventions & gotchas

- **Single-file components.** UI lives inside `<script type="text/babel">` in each
  HTML file. There is no module system — helper functions and components are defined
  in-page. Match the surrounding style; don't introduce imports/bundler syntax.
- **Edit the canonical file.** Several pages have `_pre-production`, `.txt`, `-v3`,
  or theme-variant twins. Confirm which one is live before changing, and avoid
  "fixing" a stale copy.
- **No secrets in this repo** today — keep it that way. The Cloud Function URL is
  public by design; do not add API keys or tokens to client HTML.
- **Don't add a build system casually.** The deploy model is "copy static files."
  Introducing npm/bundling is a real architecture change — propose it, don't sneak
  it in via a docs PR.
- **Production divergence** (see §2) is the single biggest source of confusion here.

## 9. Where the backend actually is (correction to §4)

§4 is incomplete and, for anything involving money, misleading. Three separate backends
exist:

1. **The `advisor` Cloud Function** — §4 lists five tasks. This repo actually calls ~22; the
   missing ones go through the `callAdvisor('name', …)` helper rather than raw `fetch`, and
   cover invites, auth, persistence and Stripe checkout.
2. **The Makesbridge/BridgeMail Java API** — where copying and money live: `WorkflowAPI.jsp`
   (`copyToUser`, `mintShareLink`), `JobContainer.billForReadyStack()` (the 20/40/40 split),
   Stripe Connect payouts. Nothing about revenue can be changed from this repo.
3. **The live studio bundle** — `district.readystack.ai/dist/bundle.js`, source not in this
   repo or the `readystackai-rgb` org, calling tasks that do not exist here
   (`mintSharerCopyLink`, `copyDistrictReadyStack`, `getReadyStackByHandle`).

If a task concerns sharing, copying, attribution or payouts, start with
[`docs/SHARE-ATTRIBUTION.md`](docs/SHARE-ATTRIBUTION.md) — not with `index.html`.

## 10. Related docs

- [`README.md`](README.md) — human-facing entry point / quick start.
- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — deeper notes on the runtime,
  the `advisor` backend contract, and the District data model.
- [`docs/SHARE-ATTRIBUTION.md`](docs/SHARE-ATTRIBUTION.md) — the 20/40/40 model, chain
  semantics, and who does not get paid.
- [`docs/API-SHARE-CHAIN.md`](docs/API-SHARE-CHAIN.md) — endpoint contracts.
- [`docs/DB-MIGRATIONS.md`](docs/DB-MIGRATIONS.md) — schema and deployment order.
- [`docs/FRONTEND-SHARE-WIRING.md`](docs/FRONTEND-SHARE-WIRING.md) — what the studio bundle
  and the page publisher must do.
- [`docs/KNOWN-BLOCKERS.md`](docs/KNOWN-BLOCKERS.md) — defects found in the settlement path.
