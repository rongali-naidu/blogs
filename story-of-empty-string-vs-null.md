# Story of Empty String vs NULL

## Introduction ##

One of the sources of confusion for people working with the data and SQL is the distinction between empty strings ('') and NULL values. If you are working with the databases like Oracle, which handles Emptry Strings as NULL, you may argue that both NULL and Emptry Strings are same. If you are working with databases like Amazon Redshift, AWS Athena you will argue that they are different. 

Look at the following screenshot showing the NULL and Emptry Strings in the output returned by Athena. Visually both looks similar . 
![Alt text](images/null-vs-emptry-string.jpeg)
While both may seem similar at first glance, they have fundamental differences that can impact data quality, query results, and application logic.Misunderstanding these concepts often leads to more time spent in debugging data quality issues.In this blog, I’ll clarify the difference between empty strings and NULL values, explore common pitfalls, and provide practical tips to handle them effectively in SQL-based databases like Amazon Redshift. I also added programming context to understand what these means at the Programming Level.

## Understanding the Difference

### NULL: The Absence of a Value

- Represents **unknown or missing data**.
- Special SQL keyword, **NULL**.
- Comparisons (`=` or `!=`) do **not** work with `NULL`; use `IS NULL` or `IS NOT NULL` instead.
- Functions like `COALESCE()` `NVL()` replace `NULL` values.

### Empty String (`''`): A Valid String With No Characters

- Represents **a known but empty** value.
- It is **not** NULL and cannot be identified with `IS NULL`
- can be compared with `=` or `!=`.
- Functions like `LENGTH('')` return `0`, while `LENGTH(NULL)` returns `NULL`.

---

## Common Pitfalls and Unexpected Behaviors

### 1. **COALESCE Doesn’t Replace Empty Strings**

```sql
SELECT COALESCE('', 'default_value');
```

**Output:** `''` (empty string, not `default_value`)

✅ **Fix:** Use `NULLIF` to treat empty strings as `NULL`:

```sql
SELECT COALESCE(NULLIF('', ''), 'default_value');
```

**Output:** `default_value`

---

### 2. **Incorrect COMPARISION results Due to Empty Strings AND NULL**

```sql
SELECT 
CASE WHEN NULL=NULL THEN 1 ELSE  0 END  null_to_null_comparision,
CASE WHEN ''=NULL THEN 1 ELSE  0 END  null_to_empty_string_comparision,
CASE WHEN ''='' THEN 1 ELSE  0 END  empty_string_to_empty_string_comparision
```

**Output:** `0 0 1`

- Imagineyou are comparing the valaues in two columns match. Even if both the columns have same value (NULL), your compairision will flag it as not equal

✅ **Fix:** Use `COALESCE` AND `NULLIF` to treat empty strings and NULL values in columns before comparing them.

```sql
SELECT 
CASE WHEN COALESCE(CAST(NULL AS VARCHAR),' ')=COALESCE(CAST(NULL AS VARCHAR),' ') THEN 1 ELSE  0 END  null_to_null_comparision,
CASE WHEN COALESCE(NULLIF('',''),' ')=COALESCE(CAST(NULL AS VARCHAR),' ') THEN 1 ELSE  0 END  null_to_empty_string_comparision,
CASE WHEN ''='' THEN 1 ELSE  0 END  empty_string_to_empty_string_comparision
```
**Output:** `1 1 1`

### 3. **Aggregations Behave Differently**

- **COUNT(column_name)** **excludes** `NULL` values but includes empty strings.
- **COUNT(*)** counts all rows, including those with NULLs.
- **AVG(column_name)** ignores NULLs but includes empty numeric fields as `0` if stored as `VARCHAR`.

✅ **Fix:** Use `NULLIF` and `COALESCE` depending on how you want to treat the NULL values and Empty Strings in the aggregations



## Not All Databases handles the Emptry String and NULL the same: Oracle Vs Redshift

If you're migrating from **Oracle to Amazon Redshift**, you may face unexpected issues due to differences in how they handle empty strings and NULL values:

- In **Oracle**, `''` (empty string) is automatically converted to `NULL`.
- In **Redshift**, `''` remains an empty string and is **not treated as `NULL`**.

### Example Issue:

```sql
-- Oracle behavior ('' is treated as NULL)
SELECT COALESCE('', 'default_value') FROM dual;
-- Output: 'default_value'

-- Redshift behavior ('' remains an empty string)
SELECT COALESCE('', 'default_value');
-- Output: '' (empty string, not replaced)
```


## Programming Context: NULL vs Empty Strings in C

[Note: Following example is taken from https://c-for-dummies.com/blog/?p=2641

The above narrative helps you from the SQL user. if you wonder whats happening at the programming level, this example helps you.

The C language provides a unique perspective on empty strings vs. NULL values that data engineers should be aware of. Unlike SQL databases, where NULL represents missing data, in C:

- A **null string** is an uninitialized character array, meaning it exists in memory but has no assigned value.
- An **empty string** contains the `\0` null character, meaning it is explicitly set to a zero-length string.

### Example in C:

```c
#include <stdio.h>
#include <string.h>

int main() {
    char empty[5] = { '\0' };
    char null[5];

    if (strcmp(empty, null) == 0)
        puts("Strings are the same");
    else
        puts("Strings are not the same");

    return 0;
}
```

**Output:**
```
Strings are not the same
```

- `empty[]` is an **empty string** with a null terminator (`\0`).
- `null[]` is an **uninitialized array**, which may contain garbage values.
- `strcmp()` correctly differentiates between them.

This behavior highlights the importance of **initialization** and **explicit null handling**, which also applies when dealing with NULL and empty strings in databases.

---

## Best Practices to Handle NULL and Empty Strings

✅ **Define NULL AND Empty String Handling:** Standardize how you want to handle the  `NULL` or Empty strings early in the data processing pipeline

✅ **Use `NULLIF(column, '')` to convert empty strings to NULL where needed.**

✅ **Check for `NULL` explicitly using `IS NULL` or `IS NOT NULL`.**

✅ **Check for `EMPTY STRING ('') ` explicitly using `IS ''`

✅ **Standardize the  `NULL` or Empty strings  before joins and aggregations.**


---

## Conclusion

Both NULL and empty strings can lead to subtle yet serious data inconsistencies. Understanding their behavior helps data engineers (or whoever is dealing with the data and SQL) avoid common mistakes and write cleaner, more reliable SQL queries. By applying these best practices, you can ensure accurate reporting, smoother data transformations, and better overall data quality.

Do you have your own tips or experiences dealing with NULLs and empty strings? Share them in the comments!



