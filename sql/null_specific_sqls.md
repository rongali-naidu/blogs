# SQL NULL-Handling Traps: Complete Scenario List

## The one rule underneath all of these

SQL uses **three-valued logic**: any comparison can resolve to `TRUE`,
`FALSE`, or `UNKNOWN`. `NULL` means "absence of a value," not "a value
called null" — so any ordinary comparison operator (`=`, `<>`, `<`, `>`)
applied to a `NULL` always evaluates to `UNKNOWN`, never `TRUE` or `FALSE`,
**even when comparing `NULL` to `NULL`**. A `WHERE`/`HAVING`/`JOIN ON`
clause only keeps rows where the condition is `TRUE` — `UNKNOWN` rows are
silently dropped, with no error and no warning.

Every scenario below is this same rule showing up in a different piece of
SQL syntax.

---

## 1. `= NULL` and `<> NULL` never match anything, ever

```sql
WHERE discount_pct = NULL    -- WRONG: always UNKNOWN, matches zero rows
WHERE discount_pct <> NULL   -- WRONG: also always UNKNOWN
WHERE discount_pct IS NULL   -- correct
WHERE discount_pct IS NOT NULL  -- correct
```

`IS NULL`/`IS NOT NULL` aren't comparisons — they're a distinct test that
explicitly asks "is this the absence-of-value marker," and that question
always resolves to a real `TRUE`/`FALSE`, never `UNKNOWN`. This is the
single most common real-world SQL bug: a `WHERE col = NULL` clause that
runs without error and just silently returns nothing.

## 2. `NOT IN` with a subquery that can return `NULL` breaks the entire query

```sql
SELECT * FROM customers
WHERE customer_id NOT IN (SELECT customer_id FROM blacklist);
-- if blacklist has even ONE row with a NULL customer_id, this returns
-- ZERO ROWS overall — not just rows related to the NULL
```

`NOT IN (a, b, NULL)` expands to `<> a AND <> b AND <> NULL`. The last
term is `UNKNOWN`, and `UNKNOWN` anywhere in an `AND` chain makes the
*entire* condition `UNKNOWN` for every row being tested, not only rows
near the NULL. Fix: use `NOT EXISTS`, which doesn't have this failure
mode, or explicitly filter `WHERE customer_id IS NOT NULL` inside the
subquery.

```sql
-- safe version
SELECT * FROM customers c
WHERE NOT EXISTS (
  SELECT 1 FROM blacklist b WHERE b.customer_id = c.customer_id
);
```

## 3. `UNIQUE` constraints allow multiple `NULL`s (in most engines)

```sql
CREATE TABLE t (email VARCHAR UNIQUE);
INSERT INTO t VALUES (NULL);
INSERT INTO t VALUES (NULL);  -- succeeds in Postgres/MySQL/SQLite — not a violation
```

A `UNIQUE` constraint rejects duplicates by comparing values with `=`, and
`NULL = NULL` is `UNKNOWN`, not `TRUE` — so the engine never considers two
`NULL`s "the same value" worth rejecting. People assume uniqueness blocks
all duplicates including `NULL`; in most engines it doesn't. (SQL Server
historically differs; modern versions support filtered unique indexes to
control this explicitly.)

## 4. Arithmetic and string concatenation: one `NULL` poisons the whole expression

```sql
SELECT amount + discount_pct AS net FROM transactions;
-- if discount_pct is NULL, the ENTIRE result is NULL, not "amount unchanged"
```

Unlike aggregate functions (`AVG`, `SUM`), which skip `NULL`s, ordinary
arithmetic and string concatenation do **not** — `anything + NULL = NULL`,
always. People trip on this because they've internalized how aggregates
behave and assume it's universal. Needs an explicit
`COALESCE(discount_pct, 0)` to treat missing as zero in the expression
itself.

## 5. `DISTINCT`/`GROUP BY`/`ORDER BY` treat all `NULL`s as equal — the opposite convention from `=`

```sql
SELECT DISTINCT discount_pct FROM transactions;
-- multiple rows with NULL collapse into a single NULL row in the output
```

This directly contradicts scenario 3's logic (`NULL <> NULL` for
comparison purposes) — but `DISTINCT`, `GROUP BY`, and `ORDER BY` all
special-case `NULL`s as mutually equal for grouping/sorting, even though
they're "not equal" under ordinary comparison. This inconsistency across
SQL's own clauses is one of the most-cited facts about SQL not being
fully logically uniform — worth knowing cold rather than re-deriving live.

## 6. Joins on a nullable key silently drop rows

```sql
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;
-- orders with o.customer_id = NULL (e.g. guest checkouts) vanish entirely
```

Same root cause as scenario 1, showing up in a join predicate instead of
`WHERE`. `NULL = anything` is never `TRUE`, so an `INNER JOIN` drops those
rows with no error. If guest/anonymous orders need to be preserved, they
need a `LEFT JOIN` plus explicit handling, not a plain inner join.

## 7. `ORDER BY` NULL placement is engine-dependent, not standardized

```sql
SELECT * FROM transactions ORDER BY discount_pct ASC;
```

Postgres/Oracle default to `NULL`s sorting **last** in `ASC` (first in
`DESC`); SQL Server/MySQL default to `NULL`s sorting **first** in `ASC`.
If pagination or "top N" logic depends on NULL placement, an unqualified
`ORDER BY` behaves differently across engines. Be explicit:

```sql
ORDER BY discount_pct ASC NULLS LAST   -- Postgres/Oracle syntax
```

## 8. `COUNT(*)` vs `COUNT(column)` — one counts rows, the other counts non-NULL values

```sql
SELECT COUNT(*) AS total_rows, COUNT(discount_pct) AS non_null_discounts
FROM transactions;
```

`COUNT(*)` tallies rows and never looks at any column's value — a row
that's entirely `NULL` still counts as one row, because the row exists.
`COUNT(column)` counts only the rows where that specific column is
non-`NULL`, exactly like `AVG`/`SUM`/`MIN`/`MAX`. The general rule: every
aggregate skips `NULL`s in the column it operates on; `COUNT(*)` is the
one exception because it isn't operating on a column at all. A quick,
common check for "does this column have any NULLs in this group":
`COUNT(*) = COUNT(column)` (equal → no NULLs; less → some NULLs present).

## 9. Empty string (`''`) is not `NULL` — and `IS NULL` will never catch it

```sql
SELECT * FROM customers WHERE region IS NULL;
-- a row with region = '' (empty string) does NOT match this — it's a
-- real, non-null value that just happens to be zero-length
```

`''` and `NULL` represent two genuinely different facts: `NULL` means "no
value was ever recorded," `''` means "a value was recorded, and it's an
empty string." `IS NULL` tests specifically for the absence marker — it
has no special awareness of empty strings, since they're not `NULL` at
all under standard SQL semantics. If "missing data" in your business logic
could show up as either (e.g. an ETL pipeline that sometimes writes `''`
instead of leaving a column unset), a check for `IS NULL` alone will
silently miss the empty-string rows:

```sql
-- catches BOTH "truly absent" and "recorded as blank"
WHERE region IS NULL OR region = '';
-- or, more concisely:
WHERE NULLIF(region, '') IS NULL;
```

**One major, commonly-cited exception worth knowing:** **Oracle treats an
empty string as `NULL` for `VARCHAR2` columns** — `''` and `NULL` are the
same thing in Oracle, unlike Postgres/MySQL/SQL Server/Snowflake, where
they're distinct. This is a real portability trap: code that correctly
distinguishes `''` from `NULL` on one engine can silently behave
differently after a migration to or from Oracle. Worth naming explicitly
if asked about writing engine-portable SQL.

---

## The pattern to hold onto

Whenever a column can be `NULL`, ask explicitly **"what should happen if
this value is missing here"** for every operator touching it — comparison,
arithmetic, join, aggregate, sort, uniqueness — rather than assuming
standard operators do something uniformly sensible with `NULL`. They
don't: aggregates skip it, arithmetic propagates it, `DISTINCT`/`GROUP BY`
treat it as equal to itself, plain comparisons treat it as incomparable to
anything including itself, and empty string isn't even in the same
category as `NULL` at all (outside Oracle). Naming which of these applies
to the specific clause you're writing — rather than treating `NULL` as a
single uniform behavior — is the actual skill being tested.
