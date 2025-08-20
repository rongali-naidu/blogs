# Silent Data Traps in SQL Engines: Case, Time, Types, and Text  

Working across multiple databases sounds simple — until you run into the *silent gotchas*.  
These aren’t syntax errors. Your query may run just fine, but the results will be subtly wrong.  
Here are some common **data traps** I’ve seen in pipelines and migrations.  

## 1. Case Sensitivity  

Behavior depends heavily on whether identifiers are **quoted** when created.  

```sql
-- Postgres / Redshift folder unquoted columns to lower case
CREATE TABLE users (userid text);       -- stored as lowercase "userid"
SELECT UserID FROM users;               -- ✅ works (folds to lowercase)

CREATE TABLE users ("UserID" text);     -- stored as case-sensitive "UserID"
SELECT UserID FROM users;               -- ❌ ERROR (looks for "userid")
SELECT "UserID" FROM users;             -- ✅ works


```sql

-- SQL Server depends on collation
SELECT * FROM Users WHERE UserID = 'ABC';  -- works on CI, fails on CS collations
````

* **Postgres**: unquoted identifiers → lowercase.
* **Oracle**: unquoted → uppercase, quoted → exact.
* **SQL Server**: depends on collation.
* **Redshift**: follows Postgres (lowercase unquoted), but fewer collation options.

---

## 2. Time Zone Handling

### a) Time Zone Names

| System       | Example                 | Notes                     |
| ------------ | ----------------------- | ------------------------- |
| Windows      | `Pacific Standard Time` | SQL Server                |
| IANA (POSIX) | `America/Los_Angeles`   | Postgres, Redshift, MySQL |
| ISO Offset   | `UTC-08:00`             | Portable standard         |



### b) DST (Daylight Saving Time)

* **Spring Forward**: `2025-03-09 02:00` in `America/New_York` **does not exist**.
* **Fall Back**: `2025-11-02 01:30` exists **twice**.

**Problem:**

* ETL may skip rows or double-count.
* Redshift in particular will **accept ambiguous times silently** and normalize them to one of the possible values.

---

## 3. Timestamp Format Differences

### Default Formats

* **Postgres**: `2025-08-20 15:45:30.123456+00`
* **MySQL**: `2025-08-20 15:45:30` (TZ stored only if `TIMESTAMP`)
* **Oracle**: `20-AUG-25 03.45.30.123456 PM +00:00`
* **SQL Server**: `2025-08-20 15:45:30.123`
* **Redshift**: `2025-08-20 15:45:30.123456` (TZ optional, limited microsecond precision)

### Silent Parsing Issues

Redshift and MySQL are notorious for **silently coercing invalid formats**:

```sql
-- In Redshift
SELECT TO_TIMESTAMP('2025-13-45', 'YYYY-MM-DD'); 
-- returns: 2026-01-14 00:00:00  (unexpected!)

-- In MySQL
INSERT INTO t (ts) VALUES ('2025-08-20 25:61:61');
-- becomes '2025-08-21 02:02:01' silently
```

you think you’ve stored valid data, but downstream analytics see shifted or garbage timestamps.

---

## 4. Timestamp Functions

Different databases expose inconsistent function names and semantics:

| Engine     | Current Timestamp (no TZ) | Current Timestamp (with TZ)               | Notes                            |
| ---------- | ------------------------- | ----------------------------------------- | -------------------------------- |
| Postgres   | `NOW()` (txn start)       | `CURRENT_TIMESTAMP` / `clock_timestamp()` | `NOW()` fixed per transaction    |
| Redshift   | `GETDATE()`               | `SYSDATE` (same as GETDATE)               | No `clock_timestamp()`           |
| Oracle     | `SYSDATE` (no fractions)  | `SYSTIMESTAMP` (fractions + TZ)           |                                  |
| SQL Server | `GETDATE()`               | `SYSDATETIME()` (fractions)               |                                  |
| MySQL      | `NOW()`                   | `CURRENT_TIMESTAMP()`                     | Evaluated at statement execution |

**Inconsistency:**

* `GETDATE()` in Redshift ≠ `GETDATE()` in SQL Server (different precision + semantics).
* Postgres `NOW()` is transaction-scoped; Redshift `GETDATE()` is clock-scoped.

Result: The same code using “current timestamp” behaves differently across engines.



## 5. Data Type Size (INT vs BIGINT)

This one bites hardest when scaling.

### SQL Engines

| DB Engine  | INT (32-bit) Range               | BIGINT (64-bit) Range                        |
| ---------- | -------------------------------- | -------------------------------------------- |
| Postgres   | -2,147,483,648 → 2,147,483,647   | -9,223,372,036,854,775,808 → 9.2e18          |
| MySQL      | Same as Postgres                 | Same as Postgres                             |
| SQL Server | Same as Postgres                 | Same as Postgres                             |
| Oracle     | No native INT → `NUMBER(p)` type | Range depends on precision (up to 38 digits) |
| Redshift   | INT = 32-bit                     | BIGINT = 64-bit (same as Postgres)           |

### Programming Languages

| Language | INT (typical)                  | LONG / BIGINT                |
| -------- | ------------------------------ | ---------------------------- |
| Java     | 32-bit signed → -2.1B → 2.1B   | 64-bit signed → ±9.2e18      |
| Python   | Arbitrary precision (no limit) | But DB connectors still cast |
| C / C++  | `int` usually 32-bit           | `long long` 64-bit           |

**Real-world problems:**

* **we ofen assume int will be sufficient soon realize that we have to migrate to bigint**

  
## 6. White Spaces & Hidden Characters

```sql
'John'   vs  'John '   vs  'John\t'   vs  ''   vs  NULL
```

* **Oracle:** `'' = NULL`.
* **Postgres/MySQL/SQL Server/Redshift:** `'' ≠ NULL`.
* **CHAR(N):** `'John'` becomes `'John      '` (padded).

(`GROUP BY`, `DISTINCT`) break across DBs: Oracle treats empty string as NULL → different counts than Redshift.


## 7. Quoted Strings & Escaped Characters

```sql
SELECT 'O''Reilly';   -- ANSI SQL escape
SELECT "columnName";  -- identifier (Postgres/Redshift)
```

* **Postgres/Redshift:** double quotes = identifiers, single quotes = strings.
* **MySQL:** double quotes = strings unless `ANSI_QUOTES` enabled.
* **Oracle:** quoted identifiers become case-sensitive.

**Problem:** Queries that work in Postgres may fail in Redshift or MySQL due to quoting rules.

---

## 8. NULL vs Empty String

```sql
SELECT LENGTH('');   -- differs
SELECT LENGTH(NULL); -- always NULL
```

* **Oracle:** `'' = NULL` → `LENGTH('') = NULL`.
* **Postgres/MySQL/SQL Server/Redshift:** `''` length = 0.

**Impact:** Data validation checks (like “no empty values”) behave inconsistently between Oracle and Redshift.


