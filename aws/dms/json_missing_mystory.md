# The Case of the Missing JSON column: Why AWS DMS Silently Dropped Our JSON Field

*A debugging story about MySQL `JSON` columns, primary keys, and AWS Database Migration Service (DMS).*

## TL;DR

If a MySQL source table has a `JSON` (or `TEXT`/`BLOB`) column **and no primary key or unique
key**, AWS DMS will **not migrate that column at all** — it silently disappears from your target
(for an S3/parquet target, the column is simply absent from the output files). This is by design:
DMS treats `JSON` as a LOB, migrates LOBs in two passes (insert the row, then UPDATE the LOB), and
that UPDATE needs a key to locate the row. No key → no LOB.

---

## The symptom

We ran a DMS full-load from an Aurora MySQL source to Amazon S3 (parquet). A table we'll call
`geo.regions` looked like this in the source:

```
+-------------+---------------------------------------------------+---------------------+
| region_code | shape                                             | updated_at          |
+-------------+---------------------------------------------------+---------------------+
| WEST-01     | {"type": "Polygon", "coordinates": [[[-112.5, ... | 2026-08-20 20:09:31 |
| EAST-04     | {"type": "Polygon", "coordinates": [[[ 135.3, ... | 2026-08-20 20:09:32 |
+-------------+---------------------------------------------------+---------------------+
```

Three columns: `region_code`, `shape` (a `JSON` polygon), `updated_at`.

But the parquet DMS produced had only:

```
['Op', 'dms_timestamp', 'region_code', 'updated_at']
```

The `shape` column was **gone**. Not null — *absent from the schema entirely*. Downstream, our
processing failed with `UNRESOLVED_COLUMN: shape`, because our catalog (correctly) expected the
column to be there.

The source had the column. The target didn't. No error was raised by DMS. So where did it go?

---

## The investigation

We checked the obvious things first and ruled them out:

- **Not a column-name/casing mismatch.** The source column was exactly `shape`; the catalog
  matched.
- **Not a table-mapping exclusion.** The DMS task mappings had only selection rules (include
  everything, exclude an unrelated prefix) — no column-level transformation rules.
- **Not LOB truncation/size.** Truncation would give a shortened value, not a missing column.

That left two facts about `geo.regions`:

1. `shape` is a **`JSON`** column.
2. The table has **no primary key** (and no unique key).

Those two facts together are the whole story. Here's why, backed by the AWS docs.

---

## Root cause, in three documented steps

### 1. MySQL `JSON` is a LOB (specifically `CLOB`) in DMS

DMS maps every source type into its own internal type system. For a MySQL source, the mapping
table includes:

| MySQL data type | AWS DMS data type |
|---|---|
| `LONGTEXT`  | `NCLOB` |
| `MEDIUMTEXT`| `NCLOB` |
| `TEXT`      | `WSTRING` |
| **`JSON`**  | **`CLOB`** |
| `GEOMETRY` / `POLYGON` | `BLOB` |

`JSON` → **`CLOB`**, a character **L**arge **OB**ject. So DMS handles a `JSON` column with its LOB
machinery, not as an ordinary inline string.

> Source: [Using a MySQL-compatible database as a source for AWS DMS — "Source data types for MySQL"](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.MySQL.html#CHAP_Source.MySQL.DataTypes)

### 2. DMS migrates LOBs in two passes — insert the row, then UPDATE the LOB

LOBs are variable-length and potentially huge, so DMS doesn't write them inline with the rest of
the row. Instead it uses a two-step ("lookup") mechanism:

> "DMS initially migrates a row with a LOB column as null, then later updates the LOB column."
>
> "AWS DMS first replicates rows without the LOB column, retrieves LOB data using a **SELECT**
> command, and executes an **UPDATE** command to replicate the LOB field on the target. This
> sequential INSERT and UPDATE operation characterizes the LOOKUP behavior. ... during the CDC
> phase, AWS DMS consistently uses the Lookup method regardless of LOB settings."

> Source: [Troubleshooting migration tasks — "Tasks fail when a primary key is created on a LOB column"](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Troubleshooting.html#CHAP_Troubleshooting.General.PKLOBColumn)

### 3. That second-pass UPDATE needs a key — so no PK/unique key means no LOB

To run `UPDATE ... SET shape = ? WHERE <identify exactly one row>`, DMS must be able to uniquely
identify the row it just inserted. The only reliable way to do that is a **primary key or unique
key**. Without one, DMS can't safely perform the LOB UPDATE — so it skips the LOB. Two docs say
this outright:

> **Premigration assessment** `table-with-lob-but-without-primary-key-or-unique-constraint`:
> "Checks for the presence of source tables with LOBs but without a primary key or a unique key.
> **A table must have a primary key or a unique key for DMS to migrate LOBs.**"
>
> Source: [Assessments for all endpoint types](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Tasks.AssessmentReport.Assessments.All.html#CHAP_Tasks.AssessmentReport.Assessments.All.LOBsNoPrimaryKey)

> **Troubleshooting** "LOB changes aren't being captured":
> "Currently, **a table must have a primary key for AWS DMS to capture LOB changes.** If a table
> that contains LOBs doesn't have a primary key, there are several actions you can take ... Add a
> primary key to the table ... Create a materialized view ... Create a logical standby ..."
>
> Source: [Troubleshooting migration tasks — "LOB changes aren't being captured"](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Troubleshooting.html#CHAP_Troubleshooting.Oracle.LOBChanges)

### Putting it together

```
JSON column  ─►  DMS treats it as a LOB (CLOB)
                 └─► LOBs migrate via two-pass INSERT + UPDATE (lookup)
                     └─► the UPDATE needs a PK / unique key to find the row
                         └─► table has no PK/unique key
                             └─► DMS cannot migrate the LOB  ─►  column absent from target
```

A precise note on wording: the docs phrase this as the LOB being "not migrated / not captured."
The exact manifestation is engine/target specific. For our **MySQL → S3 (parquet)** case, it
showed up as the whole `shape` column being **omitted from the output files** — consistent with
"the LOB was not migrated." Don't assume it always lands as a null column; for S3 it can be gone
entirely.

---

## How to detect this before it bites you

- **Run a DMS premigration assessment.** The assessment
  `table-with-lob-but-without-primary-key-or-unique-constraint` flags exactly this situation.
- **Inventory your source.** Find LOB-typed columns (`JSON`, `TEXT`/`MEDIUMTEXT`/`LONGTEXT`,
  `BLOB`, spatial types) on tables lacking a PK/unique key:

  ```sql
  -- tables with a JSON/text/blob column but no primary key
  SELECT c.TABLE_SCHEMA, c.TABLE_NAME, c.COLUMN_NAME, c.DATA_TYPE
  FROM information_schema.COLUMNS c
  WHERE c.DATA_TYPE IN ('json','mediumtext','longtext',
                        'blob','mediumblob','longblob','tinyblob',
                        'geometry','point','linestring','polygon',
                        'multipoint','multilinestring','multipolygon','geometrycollection')
    AND NOT EXISTS (
          SELECT 1 FROM information_schema.STATISTICS s
          WHERE s.TABLE_SCHEMA = c.TABLE_SCHEMA
            AND s.TABLE_NAME   = c.TABLE_NAME
            AND s.NON_UNIQUE   = 0
    )
    AND c.TABLE_SCHEMA NOT IN ('mysql','information_schema','performance_schema','sys')
  ORDER BY c.TABLE_SCHEMA, c.TABLE_NAME;
  ```

- **Compare source vs. target schemas** after the first load. A column present in the source but
  missing from the target parquet is the giveaway.

---

## How to fix it

Ranked from most correct to interim workaround.

**1. Add a primary key or unique key on a non-LOB column (the real fix).**
```sql
-- if an existing non-LOB column is unique:
ALTER TABLE geo.regions ADD PRIMARY KEY (region_code);
-- otherwise add a surrogate key:
ALTER TABLE geo.regions ADD COLUMN id BIGINT AUTO_INCREMENT PRIMARY KEY;
```
The key must be on a **non-LOB** column — DMS explicitly does not support a primary key that *is*
a LOB data type (that causes the initial null-insert to fail the NOT-NULL PK). Once a key exists,
DMS can perform the two-pass LOB migration.

**2. Then (if the LOB can be large) use Full LOB mode or raise the LOB size cap.**
Adding a key is what *unblocks* LOB migration; it doesn't change the size limits. In
`LimitedSizeLobMode`, anything past `LobMaxSize` is truncated. If your JSON can exceed that (e.g.
large polygons), switch to Full LOB mode or raise `LobMaxSize` — but note Full LOB mode itself
still requires a key, so this is a follow-on to step 1, not a substitute.

**3. Bypass DMS for that table (no source schema change).**
For small reference-style tables, run a lightweight side extraction — a scheduled job that does
`SELECT <cols including the JSON> FROM geo.regions` over JDBC and writes parquet directly, or an
engine-native export. This sidesteps the DMS LOB/PK constraint entirely for that one table.

**4. Interim: drop the LOB column from your downstream schema.**
Not a real fix — it just stops the "missing column" failure so the other columns load while you
pursue #1 or #3. You lose the JSON data until then.


### Reference links

1. MySQL `JSON` → `CLOB` (LOB) mapping —
   https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.MySQL.html#CHAP_Source.MySQL.DataTypes
2. Two-pass LOB migration (insert null, then UPDATE) —
   https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Troubleshooting.html#CHAP_Troubleshooting.General.PKLOBColumn
3. LOBs require a PK/unique key or they aren't migrated —
   https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Tasks.AssessmentReport.Assessments.All.html#CHAP_Tasks.AssessmentReport.Assessments.All.LOBsNoPrimaryKey
