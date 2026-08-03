# Architecture

Technical detail behind [`CLAUDE.md`](../CLAUDE.md). This describes **this repository**
— the build-less CDN-React lineage — not the live Firebase build (see `CLAUDE.md` §2).

## Runtime model

There is no server and no build pipeline in this repo. Each HTML page is fully
self-contained:

```
Browser loads index.html
  ├─ <script> React 18 UMD            (unpkg.com)
  ├─ <script> ReactDOM 18 UMD         (unpkg.com)
  ├─ <script> @babel/standalone       (unpkg.com)   ← transpiles JSX at runtime
  ├─ <script> cdn.tailwindcss.com     (Play CDN)    ← utility CSS at runtime
  ├─ <script> lucide                  (unpkg.com)   ← icons
  └─ <script type="text/babel">  … the entire app … </script>
```

Because Babel runs in the browser, the JSX in `<script type="text/babel">` is the
actual source — there is no compiled output to keep in sync. The cost is runtime
transpile time on load; the benefit is zero tooling.

## Backend: the `advisor` Cloud Function

The only backend is a single Google Cloud Function:

```
POST https://us-central1-gen-lang-client-0769430147.cloudfunctions.net/advisor
Content-Type: application/json

{ "task": "<taskName>", ...args }
```

- Dispatch is on the **`task`** field. Unknown/missing task →
  `{"error":"Unknown task: \"…\"","bodyKeys":[…]}`.
- Empty body is rejected by the platform with `411 Length Required` (a `POST` needs a
  body/length).
- Auth-gated tasks return `{"error":"Not authenticated."}` when called without
  credentials.

### Tasks used by this repo

| Task | Called from | Purpose |
| --- | --- | --- |
| `aiReviewServer` | `index.html` | AI review/scoring pass |
| `generateStory` | `index.html` | Generate the district narrative (client-triggered) |
| `generateStoryServer` | `index.html` | Server-side story generation |
| `runVisionCheck` | `index.html` | Vision/image analysis check |
| `publishLandingPage` | `index.html` | Publish the generated landing page |

> The **live** studio build calls additional tasks (e.g. `listDistricts`) that this
> repo does not. Those live only in the production bundle + its Firebase layer.

### Image proxy

YouTube thumbnails are routed through a Google opensocial proxy to avoid hotlink/CORS
issues:

```
https://images1-focus-opensocial.googleusercontent.com/gadgets/proxy?container=focus&refresh=2592000&url=<encoded thumbnail url>
```

## Data model: District

The central object is a **District**. `learning-district.json` is a complete,
real-world instance and is the canonical schema reference. Summary:

```
District
├─ districtName / stackLabel            name
├─ selectedVideoData, youtubeEmbedUrl   anchor YouTube video
├─ videoStory                           the generated narrative
│   ├─ problem, resident, narrative
│   ├─ network { partners[], customers[], targets[] }
│   ├─ checklist[]
│   └─ automations[] { trigger, action, outcome }
├─ storyParticipants.residents[]        { name, company, specialty, email, department }
├─ mapping                              funnel grid, keyed by stage
│   ├─ learning[]  ┐
│   ├─ selecting[] ├─ rows: { ven, lab, platform, ga4Count, price?, … }
│   ├─ executing[] │
│   └─ expanding[] ┘
├─ heroFields                           { headline, youtubeUrl, ctaLabel, ctaUrl, mindMapLink }
└─ suggestedRoles[]                     role suggestions
```

**Domain note.** This vocabulary (districts / residents / automations) is unrelated to
the BridgeMail Workflow API (`workflows / steps / options / rules`). They are separate
systems with no shared schema and no mapping layer in this repo.

## Known repository-hygiene issues

Useful for an agent to know before trusting a filename:

- **Duplicated / snapshot pages.** `index_pre-production.html` and the two `.txt`
  files (`index.txt`, `index_pre-production.txt`, which are HTML saved with a `.txt`
  extension) are snapshots of the app, not the canonical source. `index.html` is
  canonical.
- **Near-duplicate walkthroughs.** `walkthrough.html` and `walkthrough-v3.html` are
  both titled "Interactive Walkthrough v3" and differ only slightly. Reconcile before
  editing either.
- **Theme variants.** `artist-district*.html` and `music-storyteller.html` are
  themed prototypes of the district concept.

When in doubt, edit `index.html` and leave the variants alone unless the task names one
explicitly.
