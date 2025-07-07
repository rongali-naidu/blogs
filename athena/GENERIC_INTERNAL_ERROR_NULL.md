
# Understanding and Fixing the Athena `GENERIC_INTERNAL_ERROR: result is null` error

If you’ve worked with Athena, which uses [Trino](https://trino.io/), you might have encountered the cryptic error:

```
GENERIC_INTERNAL_ERROR: result is null
```

I hanve encountered one today when using composite `IN` conditions like:

```sql
WHERE (col1, col2) IN ((1, 2), (3, 4),(5,6))
```


---

## The Scenario

Suppose you run a query filtering rows based on a composite key:

```sql
SELECT *
FROM my_table
WHERE (col1, col2) IN ((1, 2), (3, 4), (5, 6))
```

In some cases, Trino throws:

```
GENERIC_INTERNAL_ERROR: result is null
```

However, if you simplify the `IN` clause to only one or two tuples:

```sql
WHERE (col1, col2) IN ((1, 2), (3, 4))
```

or even:

```sql
WHERE (col1, col2) IN ((5, 6))
```

the query runs successfully.

---

## Why Does This Happen?

This [AWS Athena Blog](https://repost.aws/knowledge-center/athena-generic-internal-error), gave some details on this internal error. It doesnt suit my specific scenario.
So, had to do some more re-search on what could be happening in this case.

### Internal Evaluation of Composite `IN`

Trino internally handles composite `IN` conditions by creating an optimized hash set of tuples (using a utility called **FastutilSetHelper**) to efficiently check membership.

When your query uses multiple tuples in the `IN` clause, Trino calls an **internal `.equals()` function** to compare rows.

### The Null Problem

When Trino compares two values for equality, the expected result is a boolean: either `true` (they are equal) or `false` (they are not).

However, in SQL, the presence of `NULL` values complicates this logic. According to SQL’s three-valued logic:

* Comparing any value to `NULL` using `=` does **not** yield `true` or `false` — it yields `UNKNOWN` (which Trino represents internally as `null` in its boolean context).

  For example:

  ```sql
  NULL = 1     → UNKNOWN (null)
  NULL = NULL  → UNKNOWN (null)
  ```

When Trino internally evaluates a composite comparison like:

```sql
ROW(col1, col2) = ROW(1, 2)
```

If **either `col1` or `col2` is `NULL`**, then the equality check for that part returns `UNKNOWN` (`null`).

The **overall row equality** depends on the equality of all components, so if one part is `null`, the entire row equality evaluation can return `null` (i.e., an unknown result).


### Why This Causes the Error

Trino’s internal `.equals()` method (used in its optimized set membership checks) **expects a strict boolean `true` or `false` result** when checking equality between rows.

But because of SQL’s NULL semantics, the equality function can return `null` (unknown) when nulls are present.

When `.equals()` unexpectedly receives this `null` instead of a boolean, it **violates Trino’s internal assumption**, leading to a runtime error:

```java
Verify.verifyNotNull(...)
```

This check fails because Trino cannot handle a `null` result in a context where a boolean is required, triggering the `GENERIC_INTERNAL_ERROR: result is null` message.


## Confirming the Cause

Often, your data may include `NULL` values in one of the columns involved in the composite comparison, or implicit casts can cause unexpected nulls during evaluation.

Trino does not gracefully handle this internally when working with composite tuples in `IN` lists.


## Quick Fix: Add Explicit NOT NULL Filters

To avoid this error, the recommended quick fix is to **add explicit `IS NOT NULL` conditions** on all columns involved in the composite comparison, like so:

```sql
SELECT *
FROM my_table
WHERE col1 IS NOT NULL
  AND col2 IS NOT NULL
  AND (col1, col2) IN ((1, 2), (3, 4), (5, 6))
```
