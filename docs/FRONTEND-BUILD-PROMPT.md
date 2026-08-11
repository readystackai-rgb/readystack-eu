# Build brief — ReadyStack share & copy landing page

Hand this to whoever builds the frontend, human or agent. The backend is built and deployed;
everything below already exists on the API side.

---

## The flow

1. A ReadyStack is shown in the UI. The owner clicks **Share** and is given a link — for the
   creator, that's the founder link.
2. Someone (say Fazeen) opens that link and lands on a **landing page** that fetches the
   ReadyStack **by handle**.
3. The page offers **Copy** and **Share**.
4. Either action requires signing in to Makesbridge.
5. **Copy** → the MKS copy API creates his own copy, under a handle he chooses.
6. **Share** → mints *his* share link, so anyone copying through it earns him the 40%.

He does **not** have to copy in order to share. Sharing alone is enough to earn.

## The URL

```
https://pages.readystack.ai/{handle}?s={shareCode}
```

- `{handle}` is the whole path — it addresses the content and keeps the URL SEO-clean.
- `?s={shareCode}` says who sent you. **Preserve it across sign-in and any redirect.**
- If `?s=` is missing the page still works; the 40% then goes to the owner of the page that
  was copied instead of a referrer.

---

## Page 1 — the landing page

### Load the content

```
GET {BASE}/events/PublicAPI.jsp?action=getByHandle&handle={handle}
X-API-Key: <key>
```

Returns the full workflow definition: name, handle, video and button fields, headlines,
Zapier hooks, and every step with its options, rules and actions.

**Fetch this server-side** — render on the server or at build time. Two reasons, both
decisive:

1. **The endpoint is not open.** It admits a request only if one of these holds:
   a valid `X-API-Key` header; `Origin` exactly `https://district.readystack.ai`; or a
   `Referer` starting with that. And `Access-Control-Allow-Origin` is only ever returned for
   that one origin, so a browser fetch from `pages.readystack.ai` fails regardless.
2. **SEO.** A handle-based URL exists so crawlers can index the content. Fetching after paint
   gives you a clean URL wrapping an empty page.

The API key must stay server-side. Never ship it to the browser — the `Referer` check is
client-controlled and therefore not a security boundary, so the key is the only real gate.

No backend change is needed for this route.

### Show

- The ReadyStack itself — title, story, steps.
- **Copy** and **Share** buttons.
- If `?s=` is present, optionally "shared with you by …" using `sharerUserId` from the mint
  response of whoever shared it. Do not invent this from the code itself.

### Sign-in

Both actions need a Makesbridge session:

```
POST {BASE}/events/AuthAPI.jsp?action=login
{ "userId": "user@company.com", "password": "…" }
→ { "status": "success", "token": "…", "userId": "…" }
```

Send it as `Authorization: Bearer <token>` on every subsequent call.

**The `?s=` code must survive this round-trip.** If it is lost during login, the copy still
succeeds and silently credits the wrong person. This is the single most important thing to
get right, and the failure is invisible — so write a test for it, not just a manual check.

---

## Action: Copy

Ask for the copier's own handle first — it becomes their page's URL, so make it a real input
with a live availability check, not an afterthought.

```
POST {BASE}/events/WorkflowAPI.jsp?action=copyToUser&handle={sourceHandle}&s={shareCode}
Authorization: Bearer <token>
Content-Type: application/json

{ "targetUserId": "fazeen@example.com",
  "newHandle": "fazeen-cooperative-commerce",
  "newName": "optional" }
```

**Never send `sharedByUserId`.** The backend resolves the earner from `s`; a client-supplied
user id is ignored and exists only for legacy callers.

Response:

```json
{ "status": "success",
  "workflowId": "ENCODED_NEW_ID",
  "handle": "fazeen-cooperative-commerce",
  "parentWorkflowId": "ENCODED_SOURCE_ID",
  "sharedByUserId": "mansoor@example.com",
  "founderUserId": "j@example.com",
  "chainPath": "J > Faneem > Mansoor",
  "payFounder": true,
  "paySharer": true }
```

On success, send them to their own page: `pages.readystack.ai/{handle}`.

| Response | Handle it by |
|---|---|
| `409 {"code":"handle_taken"}` | Ask for another handle. Nothing was created — safe to retry |
| `400` | Missing `targetUserId`, or neither handle nor workflowId |
| `403` | Strict mode: `targetUserId` must be the signed-in user |
| `404` | Source handle doesn't exist |

If `handle` comes back empty, the copy exists but has no shareable URL — send them to set one
via `action=setHandle` before offering Share.

---

## Action: Share

```
POST {BASE}/events/WorkflowAPI.jsp?action=mintShareLink&handle={handle}&shareCode={inbound}
Authorization: Bearer <token>
```

`shareCode` is the `?s=` they arrived with — **pass it**. It makes their link a child of the
one that brought them, which is what keeps the chain intact. Omit it and the chain silently
re-roots, erasing the previous sharer's credit.

```json
{ "status": "success",
  "shareCode": "4s24hrOmWprAbo1XIbRENX",
  "handle": "cooperative-commerce",
  "depth": 2,
  "parentShareCode": "6VoPWiKLRiS4jh3Y5A2UD3",
  "originUserId": "j@example.com",
  "selfShare": false }
```

Compose and show: `https://pages.readystack.ai/{handle}?s={shareCode}` with a copy-to-clipboard
button.

- **Idempotent** — minting again returns the same code. Safe to call on every Share click; a
  link someone already posted keeps working.
- `selfShare: true` means they're the founder sharing their own stack. Still a valid link;
  just don't promise them a referral fee on it — they get the 20% founder share instead.
- `409 {"code":"no_handle"}` means the stack has no handle and therefore no shareable page.

---

## What to tell the user about money

- **20%** to the founder, for the life of every copy.
- **40%** to whoever shared the link that led to the copy.
- Only the **immediate** sharer earns — the chain is recorded in full, but it is single-hop,
  not multi-level.

Three states to render honestly rather than optimistically:

| State | Signal | Show |
|---|---|---|
| Copier is the founder | `payFounder:false, paySharer:false, blockReason:"COPIER_IS_FOUNDER"` | Copy succeeded, nothing earned. No pending payout |
| Used their own link | `blockReason:"SHARER_IS_COPIER"` | The 40% went to the page's owner, not to them |
| No payout account | `needsStripeAccount:true` from the earnings call | Earnings are accruing but cannot be paid — prompt them to connect Stripe |

## Earnings view (optional, same sprint or later)

```
GET {BASE}/events/DataAPI.jsp?action=getEarningsSummary[&startDate=YYYY-MM-DD][&endDate=…]
Authorization: Bearer <token>
```

Returns integer **cents** — divide by 100 for display, never do arithmetic in floats. Totals
split by `FOUNDER_20` / `SHARER_40`, per-payout `status` of `SENT` / `PENDING` / `DRYRUN` /
`FAILED`, and `needsStripeAccount`.

`DRYRUN` means payouts are still gated off server-side: the amount is genuinely owed but
nothing has moved. Don't render it as paid.

---

## Acceptance

1. Opening `/{handle}?s={code}`, signing in, and copying credits the 40% to the sharer behind
   that code — verify against `ShareCopyEvent.sharerUserId`, not the UI.
2. The same journey with `?s=` **stripped** still copies, and credits the page's owner.
3. Sharing after arriving with `?s=` produces a code whose `depth` is exactly one greater.
4. A taken `newHandle` returns 409 and creates nothing.
5. Signing in mid-flow does not lose `?s=`.
6. The founder copying their own stack shows "nothing earned", not a pending payout.
7. Landing page HTML contains the ReadyStack content for crawlers (if SSR was chosen).

## Not built yet

Click tracking. `ShareClick` exists as a table but has no endpoint, so link *visits* are not
recorded anywhere — only copies. If you want share → copy conversion rates, that endpoint
needs building and a decision on whether it is open or API-keyed.
