# SCD2 Interview Solution + Slowly Changing Dimension Overview

## The problem

```
dim_customer(customer_id, region, tier, effective_start_date, effective_end_date, is_current)
stg_customer(customer_id, region, tier, snapshot_date)   -- daily batch snapshot
```

Apply today's snapshot into `dim_customer`, correctly handling: no-change
customers (no new row), changed customers (close old row, insert new one),
and brand-new customers.

## The approach, in order

1. **Detect "current" correctly** — every comparison against `dim_customer`
   must be scoped to the current row only (`is_current = 'Y'`), never
   against the full history, or you'll diff against stale/closed rows.
2. **Change detection belongs in `WHERE`, not `ON`** — the join's `ON`
   clause should only describe how rows match (`customer_id`); which rows
   you keep afterward is a separate filtering step.
3. **New customers are `LEFT JOIN ... WHERE d.customer_id IS NULL`** — not
   `IS NOT NULL`, which would (incorrectly) only capture matched rows.
4. **Order of operations matters:** `UPDATE` (close out old rows) must run
   **before** `INSERT` (add new current rows). Reversing this creates a
   window where a changed customer has two rows both marked
   `is_current = 'Y'` simultaneously — a real correctness bug, not just a
   style preference.

## The corrected SQL

```sql
-- Step 1: close out changed customers' current rows FIRST
UPDATE dim_customer d
SET
  is_current = 'N',
  effective_end_date = CURRENT_DATE
FROM stg_customer s
WHERE d.customer_id = s.customer_id
  AND d.is_current = 'Y'
  AND (d.region <> s.region OR d.tier <> s.tier);

-- Step 2: insert new current rows — brand-new AND changed customers
INSERT INTO dim_customer (customer_id, region, tier, effective_start_date, effective_end_date, is_current)
SELECT
  s.customer_id,
  s.region,
  s.tier,
  CURRENT_DATE AS effective_start_date,
  '9999-12-31' AS effective_end_date,   -- sentinel "open" date
  'Y' AS is_current
FROM stg_customer s
LEFT JOIN dim_customer d
  ON s.customer_id = d.customer_id
  AND d.is_current = 'Y'
WHERE d.customer_id IS NULL                          -- brand-new customers
   OR d.region <> s.region OR d.tier <> s.tier;       -- changed customers
```

Unchanged customers naturally fall out of both statements untouched: they
don't match the `UPDATE`'s change condition, and the `INSERT`'s join
succeeds with no diff, so its `WHERE` excludes them too.

**Real-world edge case worth naming in an interview:** two snapshots or a
same-day correction reprocessing the same customer twice can create
overlapping or same-day boundary rows if `effective_start_date`/
`effective_end_date` are dates rather than timestamps. State the
assumption ("one row per customer per staging load") or use timestamps
instead of dates if that assumption doesn't hold.

## `is_current` vs. `effective_end_date` sentinel — redundant by design

Both identify the exact same row in a correctly-maintained table — they're
not two different pieces of information.

- **`is_current = 'Y'`** — optimized for "give me the current state of
  every customer": a cheap, indexable boolean filter.
- **`effective_end_date`** (with a real date, sentinel for current rows) —
  optimized for **point-in-time queries**: `WHERE effective_start_date <=
  :asof AND effective_end_date > :asof` works uniformly for historical and
  current rows alike, no special-casing a `NULL` end date.

Since they're redundant, they can **drift out of sync** if an `UPDATE`
touches one but not the other. Worth a standing data-quality check:
`SELECT COUNT(*) FROM dim_customer WHERE (is_current = 'Y') != (effective_end_date = '9999-12-31')`
should always return zero. Treat one as the source of truth for join logic
(e.g. `is_current = 'Y'`) and the other as a derived convenience kept in
sync by the same transaction — never updated independently.

## SCD Type 1, 2, 3 — brief overview

**Type 1 — overwrite, no history**
```sql
UPDATE dim_customer SET region = 'EAST' WHERE customer_id = 42;
```
Use when history genuinely doesn't matter (typo corrections, an attribute
where only the current value is ever meaningful). Simplest to implement
and query, but retroactively rewrites history for any fact table joined to
it — last year's sales report will show last year's sales attributed to
the customer's *current* region, not the region at the time.

**Type 2 — full history, new row per change**
What's built above: every version is its own row, with an effective date
range marking which is active. Use when accurate historical analysis
matters — cohort analysis, "what was this customer's tier when they placed
this order." Most complete and most commonly used in real warehouses, but
every fact-table join needs a range condition (or `is_current`), the table
grows over time, and close-out/insert ordering must be handled correctly.

**Type 3 — limited history, extra column per attribute**
```
dim_customer(customer_id, region, previous_region, region_changed_date, tier)
```
Use when you only ever need "current vs. immediately previous," not the
full chronological chain. Cheap, no extra rows, no range joins — but only
remembers one step back; a second change loses the value before the first
one entirely. Niche compared to Types 1 and 2 in practice.

**One-line comparison:** Type 1 = no history (overwrite), Type 2 = full
history (new row per change), Type 3 = one-step-back history (extra
column, no new rows). Type 2 is what real interviews and real warehouses
mostly use — 1 and 3 usually come up as "explain the alternative and why
you didn't pick it."

## Combining SCD2 with SCD3-style "previous value" columns

A real, legitimate pattern: keep the SCD2 table as the full audit trail,
but add `previous_region`/`previous_tier`-style columns to the *current*
row so common "what changed, from what" queries skip a self-join against
history.

**The risk:** if those columns are populated by a separate, independently
written code path from the SCD2 insert/update logic, you now have
redundant state maintained by two write paths — the same drift risk as
`is_current`/`effective_end_date`, just at the application-logic level
instead of the SQL level.

**The safer version: derive, don't hand-maintain.** Populate the
"previous value" columns from the SCD2 history itself, atomically, in the
same statement that creates the row:
```sql
LAG(region) OVER (PARTITION BY customer_id ORDER BY effective_start_date)
```
This keeps the SCD2 rows as the single source of truth — the SCD3-style
columns become a computed cache of it, not independently mutable state
that can silently drift.

**A stronger version for a live source system:** if the source is
something like DynamoDB, its change stream can supply `OLD_IMAGE`/
`NEW_IMAGE` directly and atomically from the source's own transaction
log — a stronger guarantee than deriving "previous value" via `LAG()` over
your own downstream table. This shifts the design from batch reconciliation
(diffing a daily snapshot against the dimension) to event-driven
incremental maintenance, which introduces its own concerns: idempotency
(stream delivery is typically at-least-once, so redelivered events need a
dedup key), per-key-only ordering guarantees across shards, and a limited
stream retention window (e.g. DynamoDB Streams: 24 hours) requiring a
batch-diff fallback if a consumer is down longer than that.

## SCD2 vs. a proper audit table — different questions, not competing designs

**SCD2 answers:** "what did the dimension look like at time T" — built for
correctly joining historical facts via effective date ranges. It's
optimized for point-in-time correctness in joins, not for describing the
change event itself.

**An audit table answers:** "what happened, when, and who did it" — the
change event as the unit of record:
```
customer_audit(audit_id, customer_id, field_changed, old_value, new_value,
                changed_by, changed_at, change_reason)
```

**Why SCD2 can't cleanly answer "who changed this," even with a
`changed_by` column bolted on:** SCD2 rows represent *states*, not
*actions*. A single `changed_by` column conflates "who set this state"
with whether the change was a real business event, a system correction, or
a backfill — categorically different kinds of change that a state-snapshot
row structurally can't distinguish. Example: "every inactive→active
transition and who caused it" is a direct filter in an audit table
(`WHERE field_changed = 'status' AND old_value = 'inactive' AND new_value
= 'active'`), but in SCD2 it requires diffing consecutive rows per
customer via `LAG(status)` and hoping `changed_by` was captured correctly
on the newer row.

**The practical design:** SCD2 and an audit/event log are complementary,
not competing. SCD2 stays lean, serving analytics/BI joins. A separate
event log captures every change with `changed_by`/`change_reason`/
`source_system`, serving compliance and "who did this" questions. Ideally,
**the event log is the source of truth and SCD2 is a derived, read-
optimized projection of it** — not two independently-maintained tables
each trying to serve both jobs.

## `is_active` flag vs. effective-date timestamps — two different concepts

**If "active" means "is this row the current version"** — no separate flag
needed; that's exactly the `is_current`/`effective_end_date` redundancy
above, purely a query-convenience denormalization.

**But "is this customer's account active" is a different, business-level
concept** — no different from `region` or `tier`. If `status` (active/
inactive/suspended) is a value that changes over time and needs history,
it should be **a normal tracked column subject to the same SCD2
change-detection logic as any other attribute** — not a flag layered on
top of the versioning mechanism.

**Where conflating them breaks:** if you tried to represent "customer
deactivated" by manipulating the *versioning* timestamps (e.g. setting
`effective_end_date` early to signal deactivation), you'd overload the
row-validity mechanism to also mean business-status — and lose the ability
to distinguish "this row was superseded because an attribute changed" from
"this row was closed because the account was deactivated." Keep versioning
(row validity) and business attributes (like account status) as separate
concerns, even when a status change happens to look like it could be
expressed via the date columns.

---

## Variation questions and answers

### Q1: Point-in-time lookup — "what was this customer's tier on a given date?"

```sql
SELECT region, tier
FROM dim_customer
WHERE customer_id = 42
  AND effective_start_date <= '2026-03-15'
  AND effective_end_date > '2026-03-15';
```

**Why `>` on the end date, not `>=`:** the end date marks the moment the
row stopped being valid (i.e. the day it was superseded). If a change
happened exactly on `2026-03-15`, that date belongs to the *new* row's
start, not the old row's end — using `>=` on the old row would make both
rows match on the boundary date, an off-by-one that silently returns two
conflicting answers for the same point in time. This is exactly why a
half-open interval convention (`[start, end)`) is used deliberately, not
arbitrarily.

### Q2: Find the current row without an `is_current` flag

Some schemas skip the flag entirely and rely only on dates. Two common ways
to get "current":

```sql
-- Option A: sentinel end-date
SELECT * FROM dim_customer WHERE effective_end_date = '9999-12-31';

-- Option B: ROW_NUMBER, if no sentinel/flag exists at all
WITH ranked AS (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY effective_start_date DESC) AS rn
  FROM dim_customer
)
SELECT * FROM ranked WHERE rn = 1;
```

**When you'd need Option B:** if `effective_end_date` is stored as `NULL`
for current rows instead of a sentinel, and there's no `is_current` flag —
`ROW_NUMBER()` sidesteps needing to know the storage convention at all,
at the cost of a per-customer sort instead of a flat filter.

### Q3: How many times has each customer changed tier?

```sql
SELECT customer_id, COUNT(*) - 1 AS tier_change_count
FROM dim_customer
GROUP BY customer_id;
```

**Why `COUNT(*) - 1`:** every customer has at least one row (their initial
state), which isn't a "change" — it's the baseline. `N` rows means `N - 1`
transitions occurred. A common trap here: if you only want to count *tier*
changes specifically (not region changes that also created a new SCD2
row), plain `COUNT(*)` overcounts, since a row is inserted whenever *any*
tracked attribute changes. Correct version needs to compare tier against
the previous row specifically:

```sql
WITH with_prev AS (
  SELECT customer_id, tier,
    LAG(tier) OVER (PARTITION BY customer_id ORDER BY effective_start_date) AS prev_tier
  FROM dim_customer
)
SELECT customer_id, COUNT(*) AS tier_change_count
FROM with_prev
WHERE tier <> prev_tier
GROUP BY customer_id;
```

### Q4: Data-quality check — find gaps or overlaps in a customer's history

A correctly-maintained SCD2 table should have zero gaps and zero overlaps
between a customer's consecutive rows.

```sql
WITH ordered AS (
  SELECT customer_id, effective_start_date, effective_end_date,
    LEAD(effective_start_date) OVER (
      PARTITION BY customer_id ORDER BY effective_start_date
    ) AS next_start
  FROM dim_customer
)
SELECT customer_id, effective_end_date, next_start
FROM ordered
WHERE next_start IS NOT NULL
  AND effective_end_date <> next_start;   -- should always match if clean
```

**What this catches:** if `effective_end_date <> next_start`, either
there's a **gap** (end date is earlier than the next row's start — some
period has no record at all) or an **overlap** (end date is later — two
rows both claim validity over the same period). Either indicates a bug in
the load logic, most commonly the `UPDATE`-before-`INSERT` ordering issue
from the main problem, or a batch that was skipped/re-run out of order.

### Q5: MERGE — the same UPDATE + INSERT logic as one statement

Most modern warehouses (Snowflake, BigQuery, SQL Server, Databricks) support
`MERGE`, which can express the same two-step logic atomically:

```sql
MERGE INTO dim_customer d
USING stg_customer s
ON d.customer_id = s.customer_id AND d.is_current = 'Y'
WHEN MATCHED AND (d.region <> s.region OR d.tier <> s.tier) THEN
  UPDATE SET is_current = 'N', effective_end_date = CURRENT_DATE
WHEN NOT MATCHED THEN
  INSERT (customer_id, region, tier, effective_start_date, effective_end_date, is_current)
  VALUES (s.customer_id, s.region, s.tier, CURRENT_DATE, '9999-12-31', 'Y');
```

**The catch that makes this incomplete on its own:** `MERGE`'s
`WHEN MATCHED` can only **update** the matched row — it cannot also
**insert** a new row for that same matched-but-changed customer in the
same clause. So this `MERGE` correctly closes out changed rows and inserts
brand-new customers, but **does not insert the new current row for a
changed customer** — that still needs a second statement (the `INSERT ...
WHERE d.region <> s.region OR d.tier <> s.tier` from the main solution)
run after the `MERGE`. This is a common trap: candidates present `MERGE`
as if it fully replaces the two-statement approach, when in an SCD2
context it only replaces the `UPDATE`/new-customer-insert half, not the
full three-way logic (unchanged / changed / new).

### Q6: Late-arriving snapshot — a staging row with an older `snapshot_date` than the current row's `effective_start_date`

If the batch pipeline can receive out-of-order data (e.g. a delayed
snapshot for yesterday arrives after today's snapshot already loaded),
blindly running the standard logic would insert a "new current" row with
an earlier logical date than the row it's supposedly superseding.

**Approach:** don't trust `CURRENT_DATE` for `effective_start_date` in this
case — use the staging row's own `snapshot_date`, and explicitly guard
against inserting a start date earlier than the existing current row's
start date:

```sql
INSERT INTO dim_customer (customer_id, region, tier, effective_start_date, effective_end_date, is_current)
SELECT s.customer_id, s.region, s.tier, s.snapshot_date, '9999-12-31', 'Y'
FROM stg_customer s
LEFT JOIN dim_customer d
  ON s.customer_id = d.customer_id AND d.is_current = 'Y'
WHERE (d.customer_id IS NULL OR d.region <> s.region OR d.tier <> s.tier)
  AND (d.effective_start_date IS NULL OR s.snapshot_date > d.effective_start_date);
```

**Why this matters as an interview answer:** it's a good example of stating
an assumption proactively — "I'm assuming snapshots always arrive in
order; if not, here's the guard I'd add" — rather than waiting for the
interviewer to surface the late-arrival case as a gotcha.

