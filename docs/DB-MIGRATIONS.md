# Database migrations — share chain & settlement

Canonical DDL lives with the backend at `html/docs/sql/readystack-share-chain.sql` in the
Makesbridge Java tree. This is the review copy and the rationale.

Apply in order. **Additive only** — no existing table is altered except section 0, which
formalises a column that was applied by hand.

## 0. Formalise `Workflow.sharedByUserId`

```sql
ALTER TABLE Workflow ADD COLUMN sharedByUserId VARCHAR(255) DEFAULT NULL;
```

This column already exists in production but there is **no migration artifact for it
anywhere in the tree** — it was applied out of band. MySQL 5.x has no
`ADD COLUMN IF NOT EXISTS`, so check first:

```sql
SHOW COLUMNS FROM Workflow LIKE 'sharedByUserId';
```

## 1. `ShareLink` — the referral token

```sql
CREATE TABLE ShareLink (
    shareCode        VARCHAR(32)  NOT NULL,
    workflowId       INT          NOT NULL,
    handle           VARCHAR(255) DEFAULT NULL,
    sharerUserId     VARCHAR(255) NOT NULL,
    parentShareCode  VARCHAR(32)  NOT NULL DEFAULT '',
    originUserId     VARCHAR(255) DEFAULT NULL,
    depth            INT          NOT NULL DEFAULT 0,
    creationDate     DATETIME     NOT NULL,
    PRIMARY KEY (shareCode),
    UNIQUE KEY uk_sharelink_mint (workflowId, sharerUserId, parentShareCode),
    KEY idx_sharelink_parent (parentShareCode),
    KEY idx_sharelink_sharer (sharerUserId),
    KEY idx_sharelink_workflow (workflowId)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

This is the `referral_code → parent_referral_code, sharer_id, original_creator_id` mapping
from the requirements, in MySQL rather than Redis — the backend has no Redis, adding
Memorystore plus a VPC connector for a low-write ledger is not worth it, and this data must
be durable and auditable because it decides who gets paid for the life of a copy.

Two decisions worth reviewing:

- **`parentShareCode` is `''`, never `NULL`, for a root mint.** MySQL permits unlimited
  `NULL`s in a unique index, so `NULL` would defeat `uk_sharelink_mint` and let one user mint
  endless duplicate root links.
- **The chain is an adjacency list.** "Everyone downstream of X" is one indexed lookup per
  level, walked in Java with a visited set and a depth cap of 50. This works on MySQL 5.x,
  which has no `WITH RECURSIVE`. **Confirm the server version** before writing recursive SQL.

## 2. `ShareCopyEvent` — the immutable attribution record

```sql
CREATE TABLE ShareCopyEvent (
    id               BIGINT       NOT NULL AUTO_INCREMENT,
    newWorkflowId    INT          NOT NULL,
    sourceWorkflowId INT          NOT NULL,
    copierUserId     VARCHAR(255) NOT NULL,
    shareCode        VARCHAR(32)  DEFAULT NULL,
    sharerUserId     VARCHAR(255) DEFAULT NULL,
    founderUserId    VARCHAR(255) DEFAULT NULL,
    chainPath        TEXT,
    chainDepth       INT          NOT NULL DEFAULT 0,
    ratePlanId       VARCHAR(64)  NOT NULL DEFAULT 'RS_2026_20_40_40',
    payFounder       TINYINT(1)   NOT NULL DEFAULT 1,
    paySharer        TINYINT(1)   NOT NULL DEFAULT 1,
    blockReason      VARCHAR(64)  DEFAULT NULL,
    creationDate     DATETIME     NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uk_copyevent_workflow (newWorkflowId),
    KEY idx_copyevent_sharer (sharerUserId),
    KEY idx_copyevent_founder (founderUserId),
    KEY idx_copyevent_source (sourceWorkflowId)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

Identity is frozen; the rate is a pointer. `UNIQUE (newWorkflowId)` is what makes the write
idempotent and lets a reconciler backfill from the legacy `parentWfId` / `sharedByUserId`
columns without creating duplicates — necessary because this write cannot share the copy's
transaction (see `KNOWN-BLOCKERS.md`).

## 3. `ShareClick` — analytics only

```sql
CREATE TABLE ShareClick (
    id         BIGINT       NOT NULL AUTO_INCREMENT,
    shareCode  VARCHAR(32)  NOT NULL,
    occurredAt DATETIME     NOT NULL,
    userAgent  VARCHAR(512) DEFAULT NULL,
    ipHash     VARCHAR(64)  DEFAULT NULL,
    PRIMARY KEY (id),
    KEY idx_shareclick_code (shareCode),
    KEY idx_shareclick_time (occurredAt)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

Attribution never depends on a click being recorded — the published page is CDN-cached and a
beacon can be blocked or lost.

## 4. `ReadyStackRatePlan`

```sql
CREATE TABLE ReadyStackRatePlan (
    ratePlanId   VARCHAR(64)  NOT NULL,
    founderPct   INT          NOT NULL,
    sharerPct    INT          NOT NULL,
    description  VARCHAR(255) DEFAULT NULL,
    creationDate DATETIME     NOT NULL,
    PRIMARY KEY (ratePlanId)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO ReadyStackRatePlan VALUES
  ('RS_2026_20_40_40', 20, 40,
   '20% origin founder, 40% immediate sharer, 40% platform remainder', now());
```

Only the two paid percentages need stating — the platform takes the remainder.

## 5. `ReadyStackPayout` — the settlement ledger

```sql
CREATE TABLE ReadyStackPayout (
    id                 BIGINT       NOT NULL AUTO_INCREMENT,
    workflowId         INT          NOT NULL,
    payeeUserId        VARCHAR(255) NOT NULL,
    role               VARCHAR(24)  NOT NULL,   -- FOUNDER_20 | SHARER_40 | LEGACY_ADMIN_60
    periodKey          VARCHAR(32)  NOT NULL,   -- yyyyMM
    amountCents        BIGINT       NOT NULL,
    totalCents         BIGINT       NOT NULL,
    usageCount         INT          NOT NULL DEFAULT 0,
    windowStart        DATETIME     DEFAULT NULL,
    windowEnd          DATETIME     DEFAULT NULL,
    connectedAccountId VARCHAR(255) DEFAULT NULL,
    status             VARCHAR(16)  NOT NULL,   -- PENDING | SENT | DRYRUN | FAILED
    idempotencyKey     VARCHAR(128) DEFAULT NULL,
    transferId         VARCHAR(128) DEFAULT NULL,
    failureReason      VARCHAR(255) DEFAULT NULL,
    creationDate       DATETIME     NOT NULL,
    updationDate       DATETIME     NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uk_payout_leg (workflowId, payeeUserId, role, periodKey),
    KEY idx_payout_payee (payeeUserId),
    KEY idx_payout_status (status),
    KEY idx_payout_period (periodKey)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

Deliberately **not** an extension of `WorkflowUsage`, whose `moneyPaid` is a `FLOAT` with no
period, role, or unique key. That table stays as the display feed; this is the money record.

`uk_payout_leg` is the real double-payment guard: settlement inserts the claim *before*
calling Stripe, so a re-run collides here and skips. Stripe's own idempotency key expires in
about 24 hours and only covers a same-day retry.

`periodKey` is `yyyyMM` — the billing cycle, not the run date. The job walks every ReadyStack
workflow daily, so a run-date key would make each day a fresh payable period.

## Deployment order

1. Take a database backup and archive the current built artifacts. A `.bak` per source file
   is not a rollback plan for two deployable artifacts plus schema changes.
2. Apply this DDL.
3. Deploy the **EJB jar first**, then the WAR — the remote interface changed, and the wrong
   order gives a runtime interface mismatch.
4. Leave payouts gated (`readystack.settlement.dryRun` defaults on). Watch the `DRYRUN` rows
   for a cycle before arming.

## Rollback

```sql
DROP TABLE ReadyStackPayout;
DROP TABLE ShareClick;
DROP TABLE ShareCopyEvent;
DROP TABLE ShareLink;
DROP TABLE ReadyStackRatePlan;
```

`Workflow.sharedByUserId` stays: it predates this work and the billing path already reads it.
Because the design adds tables rather than altering existing ones, code rollback is clean.
