# Share attribution — the model

How a share becomes a payment. This is the contract; the implementation lives in the
Makesbridge Java API, not in this repository.

## The split

Every hosted automation fee a copied ReadyStack generates is divided three ways:

| Share | Who | Basis |
|---|---|---|
| **20%** | the **origin founder** — whoever created the ReadyStack | lifetime, for as long as the copy runs |
| **40%** | the **immediate sharer** — whoever shared the link that led to this copy | lifetime, single hop |
| **40%** | the platform | the remainder |

The platform takes the *remainder*, not a computed 40%. That is what guarantees the three
parts always sum to the fee exactly, with no rounding drift, and it means a suppressed leg
(see below) falls to the platform rather than vanishing.

## Track the whole chain, pay one hop

The full path is recorded — `J → Faneem → Mansoor` — but only the **last** sharer before the
copy earns. This is deliberate and matches the shipped product promise: *lifetime
attribution, single-hop chain, no MLM nonsense.* The chain exists for audit, lineage
display, and so that policy can change later without losing history.

Stated against the requirement: the 40% goes to the **second-last node in the chain**, the
last node being the copier themselves.

## The trigger is Copy, not Share

Sharing is **recorded** (minting a link is an event, and clicks may be logged), but no money
is decided until someone copies. At the moment of the copy the attribution is frozen:
founder, sharer, and the chain path are written to an immutable record.

**Why freeze identity?** Because recomputing later means a resident's lifetime earnings can
move — or disappear — when someone upstream deletes or re-parents a workflow. Rates are *not*
frozen: the record stores a `ratePlanId`, so a wrongly recorded rate can be corrected
centrally without rewriting rows that are meant to be immutable.

## Who does not get paid

All identity comparisons are trimmed and case-insensitive — `Faneem` and `faneem` are one
person.

| Rule | Effect |
|---|---|
| Copier **is** the origin founder | No founder fee, no referral fee. The copy and chain are still recorded. |
| Sharer **is** the copier | No referral fee — and **no fallback** to the person above them in the chain. |
| Copier already appears **anywhere** in the chain | No referral fee. Catches the loop the rule above would miss. |
| Sharer already in the inbound chain, at mint time | The link re-roots at their existing node instead of adding a hop. Cycles collapse rather than extend. |

The "no fallback" point matters: the pre-existing behaviour fell back to the parent
workflow's owner whenever an explicit sharer was absent, which is precisely how a self-share
leaked a payment to whoever was above.

## The worked scenario

Share 1 `J → Faneem`, share 2 `Faneem → Mansoor`, share 3 `Mansoor → J`, then **J copies**.

Recorded:

```
chainPath      J > Faneem > Mansoor
chainDepth     3
sharerUserId   Mansoor        <- second-last node, the immediate sharer
founderUserId  J
payFounder     false          <- copier is the founder
paySharer      false          <- copier is the founder
blockReason    COPIER_IS_FOUNDER
```

Paid on a $100.00 fee: founder **$0.00**, sharer **$0.00**, platform **$100.00**.

Now the same chain with an unrelated copier, Priya:

```
sharerUserId   Mansoor
founderUserId  J
payFounder     true
paySharer      true
```

Paid on a $100.00 fee: **J $20.00**, **Mansoor $40.00**, platform **$40.00**. Faneem earns
nothing — she is in the chain but not the last hop.

Both cases are covered by automated tests against the real attribution code.

## Settlement

Attribution says who is *owed*. Settlement is separate and runs on the billing cycle:

- Amounts are **integer cents** end to end.
- Each leg is claimed in a ledger before any transfer, under a unique
  `(workflow, payee, role, period)` key. A re-run collides and skips instead of paying twice.
- The founder and sharer legs settle **independently** — one payee lacking a payout account
  no longer suppresses the other's money.
- A payee with no connected account is recorded as owed-but-undeliverable, so the product can
  ask them to connect one instead of showing a silent zero.
- Payouts are gated off by default and arm only on explicit configuration.

## Related

- [`API-SHARE-CHAIN.md`](API-SHARE-CHAIN.md) — endpoint contracts
- [`DB-MIGRATIONS.md`](DB-MIGRATIONS.md) — schema
- [`FRONTEND-SHARE-WIRING.md`](FRONTEND-SHARE-WIRING.md) — what the studio and publisher must do
- [`KNOWN-BLOCKERS.md`](KNOWN-BLOCKERS.md) — what was broken and why nothing had ever paid out
