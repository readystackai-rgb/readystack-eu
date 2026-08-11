# Share chain API

Endpoints on the Makesbridge Java API. Base path is the deployed `WorkflowAPI.jsp` /
`DataAPI.jsp` (e.g. `https://mks.bridgemailsystem.com/pms/events/…`).

## Authentication

Identity resolves in this order: `Authorization: Bearer <base64(userId:token)>`, then
`X-User-Id`, then session.

`X-User-Id` is an **unverified assertion** — whoever can reach the JSP becomes whoever they
name. It is now gated behind a shared secret:

| Config | Behaviour |
|---|---|
| `readystack.api.internalKey` unset | `X-User-Id` accepted as before, and every use logged as `AUTH_WARN` |
| set, and `X-Internal-Key` matches | accepted (server-to-server callers) |
| set, and header missing/wrong | `401` |

`readystack.api.strictAuth=true` additionally enforces that `targetUserId` equals the
authenticated user and that the source stack is copyable. Both default to today's behaviour
so the deployed client cannot break; run first, read the `AUTH_WARN` / `COPY_WARN` lines to
learn how the trusted callers actually authenticate, then enforce.

---

## `mintShareLink`

Issue (or return) this user's referral code for a stack.

```
POST /WorkflowAPI.jsp?action=mintShareLink&handle={handle}[&shareCode={inbound}]
POST /WorkflowAPI.jsp?action=mintShareLink&workflowId={encoded}[&shareCode={inbound}]
```

Body may carry `handle` / `shareCode` instead of query params.

| Field | Meaning |
|---|---|
| `handle` | the stack's handle. Required unless `workflowId` is given |
| `shareCode` | the code **the sharer arrived through**. Omit for a root share |

```json
{
  "status": "success",
  "shareCode": "4s24hrOmWprAbo1XIbRENX",
  "handle": "cooperative_commerce_a6IlkhgWShmgzm35EYWb",
  "workflowId": "ENCODED",
  "sharerUserId": "mansoor@example.com",
  "depth": 2,
  "parentShareCode": "6VoPWiKLRiS4jh3Y5A2UD3",
  "originUserId": "j@example.com",
  "selfShare": false
}
```

**No URL is returned.** The caller composes it — the district slug lives on the publishing
side, not in this API. Canonical form:

```
https://pages.readystack.ai/{handle}?s={shareCode}
```

The handle is the whole path — clean for SEO — and attribution rides in the query string.
A link with `?s=` stripped still resolves and still copies; the 40% falls to the owner of the
page that was copied.

Notes:
- **Idempotent** per `(workflow, sharer, inbound code)` — the same person sharing the same
  stack through the same link always gets the same code, so a posted link keeps working.
- Passing the inbound code is what extends the chain by one hop. Omitting it silently
  re-roots the chain and destroys the previous sharer's attribution.
- If the sharer already appears in the inbound chain, the response is their **existing**
  node (cycles collapse, `depth` does not grow).
- An inbound code belonging to a different stack is ignored and the mint is treated as root.
- `409 {"code":"no_handle"}` if the stack has no handle — it has no shareable page, so a code
  would point nowhere. Set one with `action=setHandle` first.

---

## `copyToUser`

The conversion event. Copies a stack into a user's account and records the attribution.

```
POST /WorkflowAPI.jsp?action=copyToUser&handle={handle}
POST /WorkflowAPI.jsp?action=copyToUser&workflowId={encoded}      (still supported)
```

```json
{ "targetUserId": "priya@example.com", "newName": "optional", "shareCode": "4s24hr…" }
```

`shareCode` may also be passed as a query param. **Do not send a sharer identity** — the
backend resolves it from the code. The legacy `sharedByUserId` body field is still accepted
for clients that have not migrated, but it is validated, ignored when a `shareCode` resolves,
and dropped entirely if the user does not exist.

```json
{
  "status": "success",
  "workflowId": "ENCODED_NEW",
  "targetUserId": "priya@example.com",
  "parentWorkflowId": "ENCODED_SOURCE",
  "sharedByUserId": "mansoor@example.com",
  "shareCode": "4s24hrOmWprAbo1XIbRENX",
  "founderUserId": "j@example.com",
  "chainPath": "J > Faneem > Mansoor",
  "chainDepth": 3,
  "payFounder": true,
  "paySharer": true
}
```

When nobody earns, `payFounder` / `paySharer` are `false` and `blockReason` is present
(`COPIER_IS_FOUNDER`, `SHARER_IS_COPIER`, `COPIER_IN_CHAIN`). **Do not present a blocked copy
as earning.**

Errors: `400` missing `workflowId`/`handle` or `targetUserId`; `404` source not found;
`403` (strict mode) target mismatch or unpublished source; `500` copy failed.

---

## `getEarningsSummary`

The endpoint behind the studio's *"getDistrictEarningsSummary not yet shipped"* placeholder.
Reads the settlement ledger, so amounts reconcile against Stripe to the cent.

```
GET /DataAPI.jsp?action=getEarningsSummary[&startDate=YYYY-MM-DD][&endDate=YYYY-MM-DD]
```

Defaults to the current month.

```json
{
  "status": "success",
  "userId": "mansoor@example.com",
  "totals": {
    "founderCents": 2000, "sharerCents": 4000, "legacyAdminCents": 0,
    "totalCents": 6000,
    "sentCents": 4000, "pendingCents": 0, "unpayableCents": 2000, "dryRunCents": 0
  },
  "payouts": [
    { "workflowId": "ENCODED", "role": "SHARER_40", "periodKey": "202608",
      "amountCents": 4000, "totalCents": 10000, "usageCount": 812,
      "status": "SENT", "transferId": "tr_…", "failureReason": null,
      "creationDate": "…" }
  ],
  "needsStripeAccount": false
}
```

- `role` is `FOUNDER_20` (you created it) or `SHARER_40` (you shared it).
- `status`: `SENT` paid · `PENDING` claimed, transfer in flight · `DRYRUN` recorded while
  payouts were gated off · `FAILED` see `failureReason`.
- `needsStripeAccount: true` means money is owed but there is no connected account — prompt
  them to connect one rather than showing zero.
- The older `action=getEarnings` still returns the usage-based view and is unchanged.

---

## Event recording summary

| Moment | Recorded | Money |
|---|---|---|
| Link minted | `ShareLink` row (the share event) | none |
| Link visited | `ShareClick` row (optional, analytics only) | none |
| **Stack copied** | `ShareCopyEvent` — frozen founder, sharer, chain, pay flags | decided here |
| Billing cycle | `ReadyStackPayout` claim, then transfer | moves here |
