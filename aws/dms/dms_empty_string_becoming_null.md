# The Empty String That Became NULL: A DMS Full-Load vs CDC Mystery

*A debugging story about how the same value can be an empty string in one half of your data
lake and NULL in the other — and why your NOT-NULL data-quality check was doomed from the start.*

## TL;DR

We had a `NOT NULL` column that was failing a completeness (NOT-NULL) data-quality check with tens
of thousands of nulls — even though the source database reported **zero** SQL nulls for it. The
truth: the column is an **empty string (`''`)** in the source, DMS **preserves `''` during full
load** but **emits `NULL` for the same value during CDC (ongoing replication)**. The nulls were
real in the target, but they came only from the CDC slice. The value was never "missing" — it was
always empty. The fix was to stop NOT-NULL-checking a column that is legitimately empty most of the
time.

---

## The symptom

Pipeline: Aurora MySQL → AWS DMS → S3 (parquet) → downstream processing with an inline
data-quality (DQ) step.

Consider a `shop.orders` table with an optional `coupon_code` column. Most customers don't use a
coupon, but at some point someone declared the column `VARCHAR(40) NOT NULL` (with an empty string
as the "no coupon" sentinel). Our DQ ran a completeness (NOT-NULL) check on it.

The job failed:

```
DQBlockingFailure: completeness(coupon_code) — 57,269 nulls of 8,490,087 rows scanned
```

So ~57K nulls. But when we queried the **source**:

```sql
SELECT COUNT(*)                              AS total,
       SUM(coupon_code IS NULL)              AS real_nulls,
       SUM(coupon_code = '')                 AS empty_strings
FROM shop.orders;
-- total: 8,490,766   real_nulls: 0   empty_strings: 7,729,023
```

**Zero real nulls in the source. 7.7 million empty strings** (the "no coupon" orders). So two
things didn't add up:

1. The source has **no nulls**, yet the target DQ found **57,269 nulls**. Where did nulls come from?
2. If empty strings were being turned into nulls, we'd expect ~7.7 **million** nulls, not 57
   **thousand**. Why only a tiny fraction?

That mismatch is the whole mystery. The answer is that DMS's two load paths disagree.

---

## The investigation

We stopped theorizing and measured the raw parquet DMS produced, split by load phase.

### Full-load parquet

```
total: 8,430,590   null: 0   empty_string: 7,669,161   populated: 761,429
```

**Zero nulls.** The 7.67M empty strings (no-coupon orders) were preserved as `''`. So full load
does **not** convert empty → null.

### CDC parquet (ongoing replication files)

Sampling recent CDC files, every empty `coupon_code` came through as **`NULL`** — zero empty
strings. A representative CDC record (a freshly placed order with no coupon):

```
Op            = 'I'                      (insert)
order_id      = 'da46cfff-...-b33b2ae3'
coupon_code   = None                     ← NULL in the parquet
channel       = 'web'
status        = 'created'
```

### The decisive source check

We took the exact `order_id`s that landed as NULL from CDC and looked them up in the source:

```sql
SELECT order_id, coupon_code,
       LENGTH(coupon_code)  AS len,
       coupon_code IS NULL  AS is_sql_null,
       HEX(coupon_code)     AS hex_bytes
FROM shop.orders
WHERE order_id IN ('da46cfff-...', '...');
-- every row: coupon_code = '' , len = 0 , is_sql_null = 0 , hex_bytes = (empty)
```

The source rows are **empty strings** (`len = 0`, `is_sql_null = 0`, zero bytes) — **not** null.
Yet those same rows are **NULL** in the CDC parquet.

---

## Root cause

**DMS represents an empty string differently depending on the load phase for this source:**

| | source (MySQL) | DMS output |
|---|---|---|
| **Full load** | `''` (len 0, not null) | `''` — empty string preserved |
| **CDC (binlog)** | `''` (len 0, not null) | **`NULL`** |

- The **7.67M** empty strings came almost entirely through the **full load** → stayed `''` →
  non-null → **passed** the completeness check.
- The **57K** nulls came through **CDC** (recent orders) where DMS emitted `NULL` for the same
  logically-empty value → **failed** the completeness check.

That reconciles both puzzles: nulls appeared even though the source had none (they were *created*
by the CDC path), and it was ~57K rather than ~7.7M because only the CDC slice is affected — not a
uniform conversion.

### Why the column is empty so often

Worth understanding, because it explains why a NOT-NULL check was fragile. A very common way a
`NOT NULL` string column ends up full of empty strings is a bulk-load path that has no real value
to supply:

```
Upstream record has no value for the field (attribute simply absent / blank)
   └─► export/ETL serializes the record to CSV (missing field → empty field, never \N)
       └─► LOAD DATA ... INTO a VARCHAR NOT NULL column
           └─► MySQL stores an empty (non-\N) field as ''   (satisfies NOT NULL)
```

Two mechanics to internalize:
- A typical CSV **writer** renders a missing/undefined property, an empty string, and null all as
  the **same empty field** (`,,`) — it does not emit the `\N` token that MySQL needs for a real
  NULL.
- MySQL `LOAD DATA` stores an empty (non-`\N`) field into a **string** column as `''`. Only a
  literal `\N` becomes SQL NULL. So the column ends up `''`, which satisfies `NOT NULL`.

The upshot: the column is **legitimately empty for ~91% of rows** by design — most orders simply
have no coupon.

---

## Why a NOT-NULL DQ check was the wrong contract

The column is empty (`''`) most of the time in the source and arrives as either `''` (full load)
or `NULL` (CDC) in the lake. Neither is a data defect — the value is simply "no value." A NOT-NULL
completeness check on a structurally-optional column will always fail as soon as any CDC-origin
rows show up. So the fix is **not** to force the data to be non-null; it's to **stop asserting a
constraint the data was never going to satisfy**: remove that column from the NOT-NULL / completeness
checks.

That is not "relaxing DQ to hide a bug." The data is genuinely, correctly empty. The check encoded
a wrong assumption.

---

## How to detect this class of problem

- **Never diagnose from the source count alone.** The source said "0 nulls," which was true and
  misleading. Measure the **target parquet**, and split it by **full-load vs CDC** files — the
  divergence is invisible until you do.
- **When a NOT-NULL check fails but the source has no nulls, suspect empty strings + CDC.** Run:
  ```sql
  SELECT SUM(col IS NULL)  AS real_nulls,
         SUM(col = '')     AS empty_strings
  FROM schema.table;
  ```
  Many empty strings + zero real nulls is the fingerprint.
- **Confirm on the landed data** by counting `col IS NULL` vs `col = ''` in the target table, and
  by checking a CDC-origin row's key back against the source (`LENGTH`, `HEX`, `IS NULL`).

---
