# ROW_NUMBER vs RANK vs DENSE_RANK: Ties, Determinism, and Top-N Patterns

## The three functions, side by side

All three assign a position based on `ORDER BY`, and all three give
**identical rank to tied rows**. They diverge only in what happens *after*
a tie.

Example: scores 90, 90, 80, 70, ordered descending.

| score | ROW_NUMBER() | RANK() | DENSE_RANK() |
|---|---|---|---|
| 90 | 1 | 1 | 1 |
| 90 | 2 | 1 | 1 |
| 80 | 3 | 3 | 2 |
| 70 | 4 | 4 | 3 |

- **`ROW_NUMBER()`** — ignores ties, forces a unique consecutive integer per
  row regardless. Total distinct output values always equals row count.
- **`RANK()`** — ties share a rank; the next rank **skips ahead by the
  number of tied rows** (Olympic-medal style: two golds, no silver, next is
  bronze).
- **`DENSE_RANK()`** — ties share a rank; the next rank is always **+1**
  from the previous distinct rank, no skipping.

## When to recommend each

- **`ROW_NUMBER()`** — use when you need a **fixed number of rows back**,
  full stop, and ties should be broken rather than preserved (e.g. "give me
  exactly 3 rows for a UI carousel," dedup-keep-latest-record patterns).
  Requires a **fully deterministic `ORDER BY`** to be reproducible — see
  non-determinism note below.
- **`RANK()`** — use for **competitive-cutoff / leaderboard semantics**,
  where a tie should genuinely consume slots below it, and "how many
  entities are strictly ahead of me" is the real question (medal tables,
  prize cutoffs, "how many titles outrank this one").
- **`DENSE_RANK()`** — use for **distinct-tier semantics**, where you care
  about which bucket/level of performance a row falls into, independent of
  how many rows share or precede that tier (e.g. "which of the top 3
  distinct viewership levels is this title in," pricing tiers, percentile
  buckets).

**The one-line pressure-test before writing any of these:** *"does a tie
count as one unit or split into multiple units for whatever cutoff N means
here?"* — that question, not the syntax, is what's actually being tested.
Also separate a **fixed row count** ("top 3 rows") from a **competitive
cutoff** ("top 3 positions," which can legitimately return more than 3 rows
under ties) from a **tier count** ("top 3 distinct score values") — these
are three different asks that all get called "top 3" colloquially.

### Worked example: "top 3" is three different queries

Leaderboard with a tie for 1st:

| viewer_count | RANK() | DENSE_RANK() |
|---|---|---|
| 100 | 1 | 1 |
| 100 | 1 | 1 |
| 90 | 3 | 2 |
| 85 | 4 | 3 |

Filtering `<= 3`:

- **`RANK() <= 3`** → rows with rank 1, 1, 3 → **3 rows**. Matches how real
  leaderboards/medal tables work: two tied for gold, one bronze, three
  medal winners total — the tie legitimately consumes a slot.
- **`DENSE_RANK() <= 3`** → rows with dense_rank 1, 1, 2, 3 → **4 rows**.
  Correct if the ask is "everyone in the top 3 distinct performance tiers,"
  wrong if the ask is "the top 3 competitive positions" — a 4th competitor
  now reads as tier "3" even though 3 people are strictly ahead of them.
- Neither guarantees **exactly 3 rows** in general — `RANK()` can return
  more than the nominal N when ties fall near the boundary (e.g. two tied
  for 2nd and one at 1st = 3 rows, but two tied for 1st and two tied for
  2nd = 4 rows). If the requirement is a strictly fixed row count no matter
  what, use `ROW_NUMBER()` with a fully deterministic `ORDER BY` instead.

## The canonical "Top-N per group" query shape

The most common practical use of these functions in interviews — top
products per category, most-watched title per genre, highest earner per
department:

```sql
WITH ranked AS (
  SELECT
    category,
    title,
    viewer_count,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY viewer_count DESC) AS rn
  FROM titles
)
SELECT * FROM ranked WHERE rn <= 3
```

`PARTITION BY` the group, `ORDER BY` the ranking criterion, filter the rank
in an **outer query** — you cannot filter a window function's result in the
same `SELECT`'s `WHERE` clause, since window functions run after `WHERE`
logically. Named variants of the identical pattern: "Nth highest salary per
department," "2nd most recent order per customer," "median per group" (via
`PERCENTILE_CONT` or a manual rank-based calculation).

## Special note: ROW_NUMBER() is non-deterministic under ties

This is a distinct issue from *which* function to pick — it's about
**reproducibility**, and it applies specifically to `ROW_NUMBER()`.

If the `ORDER BY` doesn't fully distinguish every row (ties exist on it),
`ROW_NUMBER()`'s contract — every row gets a *unique*, consecutive integer —
forces the engine to invent an arbitrary order among the tied rows, since
nothing in the query specified one. That arbitrary choice **can differ
between runs, engines, or even repeated executions of the identical
query**, because nothing about the data logically justifies one tied row
preceding another.

`RANK()` and `DENSE_RANK()` don't have this problem: their contract
explicitly allows tied rows to receive the *same* value, so there's nothing
left for the engine to arbitrarily decide — they're deterministic under
ties by construction.

**Concrete failure mode:** using `ROW_NUMBER()` to pick "the #1 row per
group" when a genuine tie exists silently and non-reproducibly picks a
"winner" among equals — a report's "#1 most-watched title" could change
identity on every re-run with no underlying data change.

**The fix:** add a fully deterministic tiebreaker to the `ORDER BY` —
e.g. `ORDER BY viewer_count DESC, title_id ASC` — which removes the
ambiguity for `ROW_NUMBER()` entirely (and is good practice for `RANK()`/
`DENSE_RANK()` too, even though their *output values* are already
deterministic, since which physical row's other columns get pulled back on
a tie can still matter downstream).

## The broader signal this whole pattern tests

None of this is really about SQL syntax. The senior-level signal is
**surfacing the ambiguity before being asked about it**: stating the
default choice, naming the specific case where it silently produces a
different result, and saying what you'd need to know to choose correctly —
then letting the interviewer tell you which case they actually care about,
rather than guessing silently.
