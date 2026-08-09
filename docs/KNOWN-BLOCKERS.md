# Known blockers found while building share attribution

Everything below was confirmed by reading the source, not inferred. All line numbers are
from the Makesbridge Java tree as it stood before this work.

## The headline: no ReadyStack payout has ever fired

Not "payouts were rare" — **zero**, by construction. Four defects in series:

| # | Defect | Where | Effect |
|---|---|---|---|
| A | `Connection conn = null;` then `conn.createStatement()` | `BMSQueryServerBean.java:13380-13382` (the next method, `:13402`, does it correctly) | Guaranteed `NullPointerException`, swallowed by the catch. Activity always returned 0 → fee always 0 → every split hit the `<= 0` guard and `continue`d. |
| B | `WorkflowUsage` INSERT lists 7 columns, supplies 6 values, binds 4 placeholders, then calls `setFloat(5, …)` | `BMSQueryServerBean.java:13503-13509` | Always threw. No usage/earnings row was ever written — and `moneyPaid` was never set by the caller anyway. |
| C | `moneyToSend.longValue()*100` | `JobContainer.java:590, 635, 636` | Truncates to whole dollars *before* scaling to cents. $1.99 paid $1.00; $0.60 produced a **zero-value transfer** that still passed the `> 0` guard, because the guard tested the `Double`, not the cents. |
| D | Row mapper never read `parentWfId` or `sharedByUserId` | `BMSQueryServerBean.java:1376-1398`, despite `SELECT *` | `getWorkflowById().getParentWfId()` was always 0, so `action=get` always reported `parentWorkflowId: null` and the founder-stamp guard was always true. |

### E — the one that would have hurt

`getReadyStackWorkflowActivity` counted **all activity ever**:

```sql
SELECT count(*) FROM <transmission> WHERE workflowId = ?
```

with no date bound (`:13384`), while the settlement job runs daily. The instant A was fixed,
that job would have paid 20% + 40% of *lifetime* revenue **every day, forever**. Per-period
idempotency does not save you: each day is legitimately a new period, and each period bills
all of history.

This is why A was not fixed on its own. The order was: repair the ledger and the arithmetic,
gate payouts off, bound the metering window, *then* fix A.

## Security

| Issue | Where | Why it matters here |
|---|---|---|
| `X-User-Id` accepted as identity with **no verification** | `WorkflowAPI.jsp:46-51` | Anyone who can reach the JSP is anyone they name |
| `copyToUser` did no ownership check on the source, and never validated `sharedByUserId` | `WorkflowAPI.jsp:407-464` (the comment at `:444` claiming otherwise was false) | Anyone could copy any stack into any account and name themselves as sharer to take the 40% |
| Stored SQL injection reaching the billing job | `getOwnerConnectedAccountId:13487` and `getReadyStackOwner:13427` concatenated values | The attacker-controlled `sharedByUserId` flowed into `:13487` during settlement |
| Response JSON hand-concatenated with unescaped user values | `WorkflowAPI.jsp:454-459` | Broken/injectable payloads |

An attribution system built on top of that auth would have been a payout system anyone could
drain, so the guards had to land before the chain did.

## Structural problems worked around, not fixed

- **No transactions.** `BMSQueryServerBean` takes a connection per method, and
  `createWorkflow` even closes its connection mid-method (`:330`). The copy-attribution write
  therefore *cannot* share the copy's transaction. It is best-effort plus a unique key on
  `newWorkflowId` so it is safe to retry and can be backfilled. Nothing here should be
  described as transactional.
- **`WorkflowUsage` is not a money ledger.** `moneyPaid` is a `FLOAT`, with no period, no
  role, and no unique key — it can neither reconcile against Stripe nor prevent double
  payment. It is kept for display; settlement writes to a new ledger with integer cents.
- **`transferGroup` is not idempotency.** Stripe will create any number of transfers sharing
  one, and both legs previously used the identical string, making payouts indistinguishable.
  The transfer call now takes a real idempotency key (~24h), and the durable guard is the
  ledger's unique constraint.
- **Chain walker with no visited set.** `getReadyStackFounderByParentChain:13399` terminates
  a cycle only by exhausting its depth cap and then returns `null`, which the billing job
  swallows as "no owner" — silent non-payment, the worst failure mode for a revenue-share
  product. The new walker carries a visited set and logs cycles loudly.
- **Case-sensitivity mismatch.** The billing self-check used `equals()` while the copy path
  used `equalsIgnoreCase()`. On a case-insensitive database that is one account slipping past
  the guard. All identity comparison is now normalised.

## Still open — product decisions, not bugs

1. **The `admin`/60% branch** (`JobContainer.java:568-598`) pays 60% to the resolved owner and
   **nothing to any sharer**. Minting referral links on admin/marketplace templates therefore
   promises a fee that branch cannot pay. Exclude those stacks from sharing, or convert them
   to 20/40/40.
2. **`JOIN UserBilling` in the billing candidate query** (`:13337`) is a hidden eligibility
   filter: a copier with no `UserBilling` row is never billed, so **their sharer never earns**,
   however perfect the chain.
3. **Epoch cutover.** Before payouts are armed, decide the date from which settlement counts.
4. **Settlement scope.** The job still walks every ReadyStack workflow on every daily run. The
   monthly `periodKey` plus the ledger's unique key make that safe, but scoping the query to
   the users actually being billed would be cheaper and clearer.

## Unrelated, but urgent

`StripeDefination.java:14` contains a hardcoded **live** Stripe secret key (test keys are
duplicated at `StripeManager.java:16` and `UserFeedbackRESTAPI.java:71`). It should be rotated
and moved to configuration. It is not part of this feature and was not touched.
