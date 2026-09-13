# Gaps-and-Islands, Window Functions, and the PARTITION BY / GROUP BY Confusion

## The problem type: gaps and islands

"Gaps and islands" is a class of SQL problems where you're given a sequence of
rows (usually ordered by date or by a numeric key) and asked to find
**contiguous runs** ("islands") separated by **breaks** ("gaps"). Common
real-world framings:

- Longest streak of consecutive active days per user
- Contiguous blocks of available inventory (find runs of in-stock days)
- Grouping consecutive log entries into "sessions" when the gap between
  events exceeds some threshold
- Finding runs of consecutive IDs in a sequence with missing numbers

The reason this comes up constantly in senior SQL interviews: it can't be
solved with a single aggregate or a single join. It requires recognizing that
**window functions can manufacture a grouping key that doesn't exist in the
raw data**, then falling back to ordinary `GROUP BY` once that key exists.

### The core trick: `value - ROW_NUMBER()`

If you have a sequence of dates (or any evenly-incrementing numeric sequence)
and you assign a `ROW_NUMBER()` ordered the same way, then:

- For **consecutive** rows, both the date and the row number increase by
  exactly 1 per row → their difference stays **constant**.
- The moment there's a **gap**, the date jumps by more than 1, but the row
  number always increments by exactly 1 → the difference **shifts to a new
  constant value**.

That means `date - ROW_NUMBER()` is constant within a streak and different
across streaks — it's a synthetic **group ID** for each island. Once you have
it, everything downstream is a normal `GROUP BY` + `COUNT(*)` /
`MIN`/`MAX` to describe each island.

```sql
WITH sess_days AS (
  SELECT DISTINCT user_id, TRUNC(session_date) AS session_day
  FROM user_sessions
),
grouped AS (
  SELECT
    user_id,
    session_day,
    session_day - ROW_NUMBER() OVER (
      PARTITION BY user_id ORDER BY session_day ASC
    ) AS grp
  FROM sess_days
)
SELECT user_id, grp, COUNT(*) AS streak_length,
       MIN(session_day) AS streak_start, MAX(session_day) AS streak_end
FROM grouped
GROUP BY user_id, grp
```

Then `MAX(streak_length) per user_id` (in an outer query) gives the longest
streak.

**Ordering direction matters.** This only produces a constant difference when
the date and the row number move in the *same* direction — i.e.
`ORDER BY ... ASC`. Ordering `DESC` breaks the arithmetic (subtracting two
values moving in opposite directions no longer stays constant), so `ASC` is
the standard convention for this pattern.

---

## LEAD / LAG: which direction do they actually look?

This is confusing because "ahead" and "behind" implicitly assume a viewing
direction, and that direction is set by your `ORDER BY`, not by some fixed
spatial convention.

- **`LAG(col)`** looks at the row that comes **before** the current row in
  the window's order — the *preceding* row.
- **`LEAD(col)`** looks at the row that comes **after** the current row in
  the window's order — the *following* row.

If you're looking at query output in a typical SQL client with
`ORDER BY some_date ASC`, rows print top-to-bottom in increasing order. In
that visual layout:

- `LAG` = the row **above** (earlier date, printed higher up)
- `LEAD` = the row **below** (later date, printed further down)

That mapping is correct **only because the ORDER BY is ascending**. If you
flip to `ORDER BY some_date DESC`, the rows print in decreasing order
top-to-bottom, and the visual mapping inverts: `LAG` now points to the row
**below** (which is chronologically earlier but printed later), and `LEAD`
points **above**.

**Takeaway:** don't memorize "LAG = up, LEAD = down" as a fixed rule — anchor
it to *preceding/following in the ORDER BY sequence*, and derive the visual
direction from whatever ordering you actually used.

---

## PARTITION BY vs. GROUP BY: they don't interact the way it feels like they should

The confusion: "if `user_id` is already in my `GROUP BY`, do I need to also
put it in `PARTITION BY` for my window function?"

The two clauses operate in **completely different, non-overlapping stages of
query execution**, and window functions are computed **before** grouping
ever happens. SQL's logical processing order (not the order you *type* the
clauses) is roughly:

```
FROM/JOIN → WHERE → GROUP BY → HAVING → window functions → SELECT → ORDER BY
```

Window functions are evaluated on the row set that exists **after** any
`GROUP BY`/`HAVING`, but a window function's own `PARTITION BY` is a
**completely independent instruction** — it doesn't inherit, share, or get
influenced by whatever grouping you did earlier in the query. They're not
layered or nested; they're two separate mechanisms that happen to both use
the word "by."

Practical implications:

1. If you need a window function's partitioning to align with `user_id`,
   you must write `PARTITION BY user_id` explicitly in the window clause —
   the presence of `user_id` in an outer or same-level `GROUP BY` gives you
   nothing for free.
2. In the gaps-and-islands query above, `ROW_NUMBER() OVER (PARTITION BY
   user_id ORDER BY session_day)` is computed in the CTE **before** any
   `GROUP BY` happens in that same CTE's query. The later `GROUP BY user_id,
   grp` in that same statement is a distinct, subsequent operation — it
   doesn't share scope with the window function's partitioning.
3. You generally **cannot** reference a window function's result directly in
   a `WHERE` or `GROUP BY` clause of the *same* `SELECT` — this is exactly
   why gaps-and-islands solutions are built as layered CTEs: compute the
   window function in one CTE, then `GROUP BY` its output in the next.

**Mental model that avoids the confusion:** `PARTITION BY` divides rows into
buckets *for the purpose of one window function's calculation only*, and
that division is thrown away once the function's value is computed per row.
`GROUP BY` collapses rows into fewer output rows entirely. They don't talk to
each other — they just happen to use similar-sounding syntax.

### The one case where they DO interact: same `SELECT`, same query block

Everything above describes `GROUP BY` and a window function living in
**different** CTEs/subqueries — in that setup they're fully independent,
which is why `user_id` being in an earlier or later `GROUP BY` has zero
effect on a window function's `PARTITION BY`.

But if a `GROUP BY` and a window function appear in the **same `SELECT`
statement**, the logical order (`GROUP BY` → `HAVING` → window functions)
means the window function no longer sees raw rows at all — it sees the
**already-aggregated output rows** of the `GROUP BY`. The grain has changed
underneath it.

```sql
SELECT
  region,
  order_month,
  SUM(amount) AS monthly_total,
  SUM(SUM(amount)) OVER (PARTITION BY region ORDER BY order_month) AS running_total
FROM orders
GROUP BY region, order_month
```

Here, `PARTITION BY region` is not partitioning the billions of raw `orders`
rows — it's partitioning the **post-`GROUP BY` rows**, i.e. one row per
`(region, order_month)`. That's why `SUM(amount)` has to be wrapped in an
outer `SUM(...)` for the window function — you're aggregating an aggregate,
because the window function's input rows are already at the grouped grain.

**The rule to hold in your head:** ask "what are the rows going into this
window function actually representing?" If the window function is in the
same query block as a `GROUP BY`, the rows are one-per-group, and your
`PARTITION BY`/`ORDER BY` need to be visualized against *that* grain, not
the original table's grain. If the window function lives in an earlier CTE
that a `GROUP BY` only happens *after* (in a separate, later query), the
window function still sees raw rows, and the later `GROUP BY` is irrelevant
to it.

---

## Nuance: redefining "active" with a duration threshold

A common follow-up on the streak problem: instead of "any row exists for
that day," require "the user was active for more than 5 minutes," given
`session_start` and `session_end` columns instead of a single
`session_date`.

**Key insight: this only changes the filter that builds the base CTE. The
streak algorithm itself (`ROW_NUMBER() - date`, then `GROUP BY`) is
completely unchanged** — it's decoupled from the business definition of
"active." That decoupling is itself a sign of a well-structured query: the
streak logic doesn't care *why* a day qualifies, only *that* it does.

```sql
user_sess_days AS (
  SELECT DISTINCT
    user_id,
    TRUNC(session_start) AS session_day   -- day assigned by start time
  FROM user_sessions
  WHERE DATEDIFF('minute', session_start, session_end) > 5   -- Redshift/Snowflake
  -- Postgres equivalent: EXTRACT(EPOCH FROM (session_end - session_start)) > 300
)
```

Two judgment calls worth stating out loud in an interview rather than
silently assuming:

- **Filter placement:** the duration check is a per-row condition, so it
  belongs in `WHERE`, evaluated before `DISTINCT` — not in a `HAVING` after
  some aggregation, since there's no aggregation being filtered here.
- **Day assignment for midnight-straddling sessions:** if a session starts
  at 11:57pm and ends at 12:04am, which calendar day does it count toward?
  There's no universally correct answer — it's a product decision, not a
  SQL problem:
  - **Start-day attribution** (used above): simple, deterministic, easy to
    audit. A session that's mostly in day 2 still counts entirely for day 1.
  - **Majority-duration attribution**: assign to whichever day contains more
    of the session's minutes — more "fair," but requires computing the
    overlap against both day boundaries and comparing them.
  - **Split attribution**: the session counts toward *both* days, if each
    day's portion independently clears the 5-minute bar — most accurate to
    real activity, but turns the day-per-session mapping from 1:1 into
    1:many, requiring a join/split step before `user_sess_days` rather than
    a plain `TRUNC`.
  Default to start-day attribution unless a requirement says otherwise, and
  say explicitly that you're making that call.
- Also watch timezone: `TRUNC(session_start)` only gives you the *user's*
  calendar day if `session_start` is already stored/converted to the user's
  local time. If it's UTC or server time, a user active every night at
  11pm local can appear to skip days simply because sessions round into
  "tomorrow" in UTC — and even a user who never travels can hit this at
  DST transitions, since their UTC offset isn't constant year-round.

---

## Nuance: shuffle cost and skew at scale (billions of rows)

`ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY session_day)` over a
50-billion-row table isn't just "a sort" — the expensive part usually
happens *before* the sort.

### Why a shuffle happens at all

Incoming rows aren't physically colocated by `user_id`. Before the engine
can compute a per-`user_id` window function, it must first **repartition
the data so all rows for a given `user_id` land on the same node/task** —
this network-and-disk data movement is the shuffle. Only after that
repartitioning can the per-partition sort and window computation run
locally.

- **Spark:** this appears in the physical plan as an `Exchange` (hash
  repartition) step immediately before the `Sort` and `Window` operators.
  If one `user_id` has 10M rows while others have hundreds, that single
  partition becomes a straggler — visible in the Spark UI as one task in a
  stage running far longer than its siblings, or as a heavily skewed
  partition-size distribution in the query plan's shuffle read metrics.
- **Redshift:** shuffle behavior is governed by `DISTKEY`/`DISTSTYLE` on the
  table, not decided per-query. If `orders` is distributed with
  `DISTKEY(user_id)`, all of a user's rows are already colocated on the
  same slice — the window function needs no network shuffle at all. If the
  table uses `DISTSTYLE EVEN` or a different key, Redshift has to
  redistribute rows by `user_id` across slices before the window function
  can run, which is the same cost as Spark's `Exchange`, just framed around
  slices instead of tasks. This also matters for joins: a join to another
  table (e.g. `customers`) only avoids a shuffle if both tables share the
  same distribution key and style — otherwise Redshift redistributes one or
  both sides to match, or broadcasts the smaller one if `DISTSTYLE ALL` is
  set on it.

### Salting, applied concretely to the streak query

Salting only helps the parts of the query that are **associative** — the
initial dedup. It cannot help the ordered `ROW_NUMBER()` step. Both halves,
worked through:

**Where it works — the `DISTINCT user_id, session_day` dedup.** The salt
column must be part of the actual shuffle key (`DISTINCT`/`GROUP BY` list),
not just carried along in `SELECT` — a column that isn't part of the key has
zero effect on how rows get hashed to partitions. This requires two phases,
since a single-pass `DISTINCT` that includes a random salt wouldn't fully
dedupe (two identical rows could draw different salts and both survive):

```sql
-- Phase 1: salt IS part of the DISTINCT key, so the shuffle actually spreads on it
sess_salted AS (
  SELECT
    user_id,
    TRUNC(session_date) AS session_day,
    FLOOR(RANDOM() * 10) AS salt          -- uniform random integer 0-9
  FROM user_sessions
),
partial_dedup AS (
  SELECT DISTINCT user_id, session_day, salt
  FROM sess_salted
),

-- Phase 2: drop salt, dedupe again on the real key
sess_days AS (
  SELECT DISTINCT user_id, session_day
  FROM partial_dedup
)
```

The whale user's raw rows spread across 10 partitions in phase 1's shuffle
(salt is genuinely part of the key there). Phase 2's shuffle re-concentrates
on `(user_id, session_day)` again — skew returns in principle — but by then
`partial_dedup` only holds at most `10 × (distinct days for that user)`
rows, not their original billions. Salting doesn't eliminate the final
shuffle on the skewed key; it makes the expensive reduction happen in a pass
where the key was genuinely spread, so the re-concentrating pass has almost
nothing left to move.

**Where it breaks — the `ROW_NUMBER()` step.** `ROW_NUMBER() OVER
(PARTITION BY user_id ORDER BY session_day)` needs one strict, correct
ordering across *all* of a user's rows to compute a valid streak. Salting
`user_id` here would produce 10 independent, non-communicating row-number
sequences — not slower, but *wrong*. The real fix for this step is
**isolating** the hot key into its own unsalted branch (see below), not
salting.

**How `FLOOR(RANDOM() * N)` produces the bucket number:** `RANDOM()` returns
a uniform float in `[0, 1)`. Multiplying by `N` stretches that to `[0, N)`.
`FLOOR()` truncates to an integer, chopping the continuous range into `N`
equal-width buckets — e.g. `[0,1) → 0`, `[9,10) → 9` for `N=10` — each
equally likely. `N` can be any positive integer (12, 15, 37 all work
identically); there's no power-of-2 requirement here, since this is
multiply-and-truncate on a uniform float, not a modulo over a hash.

**Variations on the basic random salt:**
- **Round-robin salting** — `ROW_NUMBER() OVER (PARTITION BY user_id ORDER
  BY session_date) % 10` instead of `RANDOM()`. Guarantees a perfectly even
  split of that user's rows across buckets (vs. only even *on average* with
  random assignment), and is deterministic/reproducible for debugging.
- **Selective salting** — only salt keys known to be skewed (via a cheap
  `COUNT(*) GROUP BY user_id` profiling pass, or known business context),
  leaving everything else as `user_id || '_0'`. Salting every key
  unconditionally means paying the "replicate the small side by N" cost
  even for keys that were never skewed.

### Why `HASH_BUCKET(user_id)` does NOT solve this, even though it solved
### uniform distribution across tables in Oracle

This is a real distinction, not just terminology — the two techniques solve
different problems, and it's easy to conflate them because both involve a
"bucket."

**What consistent hash bucketing (Oracle partition-wise joins, Spark
`bucketBy`, Redshift `DISTKEY`) actually guarantees:** if two large tables
are hash-partitioned on the *same* join key with the *same* function and
bucket count, matching rows from both sides land in the *same* bucket on
both tables — so the join runs **locally per bucket with no shuffle at join
time**. This is what you were using in Oracle, and it's a real, valuable
technique for avoiding repeated shuffles on tables that get joined
repeatedly on the same key.

**Why it cannot fix skew:** a hash function's defining property is
**consistency** — the same key value always maps to the same bucket, on
both sides, every time. That's required for the join to be correct at all.
But it also means: if `user_id = 42` has 2 billion rows, *every one* of
those 2 billion rows hashes to the same bucket, on both sides, no matter how
good the hash function is. Consistent bucketing guarantees **colocation and
correctness** — it says nothing about **volume**. A skewed key produces one
oversized bucket regardless of how many buckets exist or how well the hash
function distributes *different* keys from each other.

**The distinction to hold onto:** hashing (and `HASH_BUCKET`) decides
*where* a given key's rows go — always the same place, which is exactly
right for avoiding join shuffles. Salting decides *what the key even is*,
by manufacturing multiple synthetic keys out of one real value, which is
the only way to split one hot value's rows across multiple destinations.
They solve different problems and are complementary in practice: bucket the
tables for join colocation (Oracle-style), *and* separately salt or isolate
any keys known to be skewed on top of that.

### Adaptive Query Execution (Spark 3+): what the settings actually mean

AQE re-optimizes a query plan at runtime using actual shuffle statistics,
instead of relying only on the static plan built before execution. The
skew-join piece is one part of it:

- **`spark.sql.adaptive.enabled`** — master switch for AQE overall. Must be
  `true` (default in Spark 3.2+) for any of the settings below to matter.
- **`spark.sql.adaptive.skewJoin.enabled`** — turns on skew *detection and
  splitting* specifically for sort-merge joins. Without this, AQE can still
  coalesce small shuffle partitions, but won't do anything about an
  oversized one.
- **`spark.sql.adaptive.skewJoin.skewedPartitionFactor`** (default `5`) — a
  partition is flagged as skewed if its size is more than this multiple of
  the **median** partition size across that shuffle stage. E.g. with the
  default, a partition 5x larger than the median gets flagged.
- **`spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes`** (default
  `256MB`) — a **floor** below which a partition is never treated as
  skewed, even if it technically exceeds the factor above. This avoids
  flagging tiny partitions as "skewed" just because the median happened to
  be even tinier.
  A partition is only actually treated as skewed once it exceeds **both**
  the factor-over-median check and this absolute-size floor.
- **`spark.sql.adaptive.advisoryPartitionSizeInBytes`** (default `64MB`) —
  once a partition is flagged skewed, this is the *target* size AQE aims
  for when splitting it into smaller sub-partitions. A 2GB skewed partition
  with a 64MB target gets split into roughly 32 sub-partitions, each
  processed independently and in parallel.
- **`spark.sql.adaptive.coalescePartitions.enabled`** — a separate AQE
  feature (not skew-specific): merges shuffle partitions that turned out
  *too small* after a shuffle, reducing task overhead from over-partitioning.
  Frequently enabled alongside skew-join handling, but solves the opposite
  problem (too many tiny partitions vs. one oversized one).

**Net effect:** with AQE's skew-join handling on, the whale `user_id`'s
oversized partition gets automatically detected against live runtime
statistics (not a static guess) and split into several smaller
sub-partitions processed in parallel — functionally similar to what manual
salting achieves, but done by the engine, without a query rewrite, and
without the two-phase dedupe complexity salting requires for aggregations.
This is why, in a modern Spark pipeline, manual salting is increasingly a
"know it exists and know why it works" answer rather than something you'd
hand-roll by default.

### Other skew-handling techniques

- **Isolating the skewed key(s):** filter out the known hot `user_id`(s)
  into a separate branch, process them independently (often with a
  broadcast join or a manual repartition sized for just that key), then
  `UNION` the result back with the rest of the population processed
  normally. This is the correct fix for the `ROW_NUMBER()` step above,
  where salting can't apply.
- **Broadcasting the small side of a join:** if the skew is in a join
  rather than a window function (e.g. joining `orders` to a small
  `customers` table), broadcasting `customers` to every executor avoids the
  shuffle-and-sort-merge join entirely — no partitioning by `user_id` is
  needed on either side. Standard fix when one side fits in executor memory
  (Spark's `spark.sql.autoBroadcastJoinThreshold`), regardless of skew on
  the large side.
- **Bucketing (pre-partitioning at write time):** if this query pattern
  runs repeatedly, writing `orders` pre-bucketed by `user_id` (Spark's
  `bucketBy`, or Redshift's `DISTKEY(user_id)`) eliminates the shuffle for
  every future query that partitions/joins on that same key, since
  colocation is done once at write time instead of on every read. Same
  caveat as above: this guarantees colocation, not even distribution of a
  skewed key.
