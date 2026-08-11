# Frontend wiring — share chain & 40% attribution

Build instruction for the two frontend surfaces that carry share attribution. **Neither is
in this repository.** This document exists so the work is specified while access to the
studio bundle source is arranged.

| Surface | Where it lives | What it must do |
|---|---|---|
| **Landing page** | New — fetches its content by `{handle}` | Read `?s=`, keep it, pass it into the copy call |
| **Studio** | `district.readystack.ai` → `dist/bundle.js` (source not in `readystackai-rgb/readystack-eu`) | Forward `shareCode` into mint + copy; collect the copier's own handle; show real earnings |

The backend (Makesbridge Java API) is being built to the contract in
[`API-SHARE-CHAIN.md`](API-SHARE-CHAIN.md). Nothing here can be verified end-to-end until
both surfaces below are updated — the Java side is testable by direct API call in the
meantime.

---

## 1. The share URL

```
https://pages.readystack.ai/{handle}?s={shareCode}
```

- `{handle}` addresses the **content** and is the whole path — clean for SEO. It maps to
  `Workflow.handle` (unique index `idx_workflow_handle`) and resolves through
  `PublicAPI.jsp?action=getByHandle`.
- `?s={shareCode}` identifies **who sent you**. Minted per `(workflow, sharer, inbound code)`.

Two properties worth designing around:

- **Every copy has its own handle**, supplied by the copier at copy time. So each sharer's
  page is a distinct URL, and the copier is never mistaken for the founder — the founder is
  resolved by walking `parentWfId` to the root, however many handles sit in between.
- **Losing the code degrades gracefully.** If a shortener or a paste strips `?s=`, the page
  still resolves and the copy still works; the 40% simply falls to the owner of the page
  that was copied instead of the sharer who sent the link.

**The backend does not compose this URL.** `mintShareLink` returns `{shareCode, handle}` and
the caller assembles it, so the URL shape stays a frontend decision.

## 2. The landing page

A new page keyed entirely on the handle. It fetches its own content, so nothing needs to be
pre-generated per sharer:

```
GET /pms/events/PublicAPI.jsp?action=getByHandle&handle={handle}
```

What it has to do:

1. **Read `?s=` on load and keep it** — through sign-in and any redirect. If it is dropped,
   the copy still succeeds but credits the page owner instead of the sharer. That failure is
   invisible in testing: a working link that quietly pays the wrong person. Test it.
2. **Pass it into the copy call** as `shareCode` (or `s`). This is the only moment money is
   decided.
3. **Collect the copier's own handle** before copying, and send it as `newHandle` — that is
   what gives their copy a shareable page of its own. A `409 handle_taken` means the handle
   is in use; the check runs before the copy, so nothing is left half-created.
4. Optionally beacon a click for share → copy conversion metrics. Attribution never depends
   on it.

If the page is CDN-cached, read the code from the URL at runtime — never bake it into cached
HTML.

The existing published pages carry a share modal (`#rsCoopModal`) whose CTA is
`district.readystack.ai/?shareCoop=1&rs={readyStackId}&d={districtId}`. If those pages stay
in service, append `&s={shareCode}` there too so the studio receives it.

## 3. Studio bundle changes

Existing symbols in the bundle, for orientation:

| Symbol | Current behaviour |
|---|---|
| `mintSharerCopyLink` | POST `{task, sessionId, handle}` → `{success, url, handle}`; codes `self_share`, `not_logged_in`, `mint_in_progress` (client retries 3× at 2s) |
| `copyDistrictReadyStack` | POST `{districtId}` → `{success, micrositeUrl, landingPageUrl}`; guards "already have a copy" |
| `getReadyStackByHandle` | Resolves `/preview/{handle}` |
| `shareCoop` / `rs` / `d` params | Read on load; triggers `generateInviteLink{kind:'coop_share'}` + mint |
| `getDistrictEarningsSummary` | **Not implemented** — Portfolio shows hardcoded figures (153.75 / 58.13 / 16.69) behind the message *"Not yet tracked — getDistrictEarningsSummary not yet shipped."* |

Required changes:

1. **Capture** the inbound `s` param alongside `shareCoop`/`rs`/`d`, and persist it for the
   session (it must survive the OAuth round-trip — the same problem `rs_pending_invite`
   already solves in the older lineage).
2. **Mint through it.** Pass the inbound code as the parent when minting, so the chain
   extends by exactly one hop: sharer B minting from A's link produces a code whose parent
   is A's. Passing nothing silently re-roots the chain and destroys A's attribution.
3. **Copy through it.** Pass the code into the copy call. This is the only moment money is
   decided. The backend resolves the sharer from the code — the client must not send a
   sharer identity of its own (a caller-supplied `sharedByUserId` is exactly the hole the
   backend's security phase closes).
4. **Read real earnings** instead of the fixtures, and delete the "not yet shipped" message
   once wired.

### Advisor → Java mapping

The studio talks to the GCP `advisor` function; `advisor` must call the Java API:

| Advisor task | Java action |
|---|---|
| `mintSharerCopyLink` | `WorkflowAPI.jsp?action=mintShareLink` — `{handle \| workflowId, parentShareCode?}` → `{shareCode, handle}` |
| `copyDistrictReadyStack` | `WorkflowAPI.jsp?action=copyToUser` — accepts `handle` **or** the legacy `workflowId`, plus `shareCode` and the copier's own `newHandle` |
| `getDistrictEarningsSummary` | the extended `getEarnings` (per-role breakdown, period filter, chain context) |

`copyToUser` keeps accepting `workflowId`, so the currently deployed bundle continues to
work unchanged during the transition.

## 4. What "single-hop" means in the UI

Store the whole chain, pay one hop. The product copy already commits to this — *"Lifetime
attribution, single-hop chain, no MLM nonsense"* — so any lineage view should show the full
path while making clear that only the **immediate** sharer earns the 40%, and the origin
founder the 20%.

Three states the UI must handle honestly:

- **Self-copy** — the copier is the founder. The copy succeeds and the chain is recorded, but
  nothing is earned. Do not show a pending payout.
- **Re-routed fee** — the copier used their own share link, so the 40% went to the page's
  owner instead (`blockReason: SHARER_IS_COPIER`). Do not credit it to them.
- **No payout account** — the payee has no Stripe connected account. Earnings accrue in the
  ledger but cannot be transferred; surface the connect prompt rather than a silent zero.

## 5. Acceptance

1. Visiting `/{handle}?s={code}` and clicking Share produces a **new** code whose parent is
   the inbound one — verify the recorded chain depth increments by exactly 1.
2. Copying from that link attributes the 40% to the sharer who sent it, and the 20% to the
   origin founder — not to whoever last touched the stack.
3. The copier's `newHandle` becomes their page's URL, and a taken handle returns `409`
   **before** any copy is created.
4. `J → Faneem → Mansoor → J` (J copies their own piece back) records all three hops and pays
   **nothing**.
5. A link with `?s=` stripped still copies, and the 40% goes to the page's owner — not to
   nobody, and not to the wrong person.
6. Earnings shown in Portfolio match the backend ledger to the cent.
