# Case Sensitivity and Data Type Mismatches: The Silent Friction in Cross-System Data Integration

## Scenario

Today our team debugged an issue whose root cause boiled down to **case sensitivity and data type mismatches**. AWS Glue accepted a column data type defined as `STRING` in uppercase, but a downstream platform rejected the same metadata because it only recognized lowercase `string`. This small inconsistency caused an unexpected pipeline failure and highlighted how subtle differences in case handling can silently break integrations.

---

## 1. Case Sensitivity in Data Values

In ETL pipelines, we often perform **data standardization** (or data curation) to bring values into an agreed-upon format. SQL provides simple workarounds for case differences using functions like `UPPER()` or `LOWER()` to make string comparisons case-insensitive.

While this approach works for **data values**, it only addresses part of the problem. Case sensitivity also impacts **schema definitions**, which can silently disrupt cross-system data flows.

---

## 2. Case Sensitivity in Metadata (Schema)

Metadata includes **column names** and **data types**, both of which can cause compatibility issues:

### Column Names
- Lowercase vs uppercase
- Presence of special characters
- Maximum length restrictions

Different systems handle these constraints differently. What works in one system may fail in another, creating subtle bugs that are hard to trace.

### Data Types
Different platforms define and enforce **data types** in their own ways. Even for simple text values, you may encounter:

- `string` / `STRING` / `String`
- `char` / `varchar` / `text`

Additional complexities include:

- **Case sensitivity:** Some systems accept uppercase type names, while others require lowercase.
- **Length constraints:** `char(10)` vs `varchar(10)` may behave differently across platforms.
- **Encoding differences:** ASCII vs UTF-8 vs UTF-16 can introduce hidden incompatibilities.

**Example:** AWS Glue accepted `STRING` in uppercase, but a downstream platform rejected it because it only recognized lowercase `string`. Even minor inconsistencies in data type casing can cause unexpected failures.

---

## 3. General Challenges in Cross-System Data Migration

Beyond case sensitivity and type mismatches, cross-system data migration introduces several other challenges:

### Timestamp / Date Handling
- Different systems store timestamps in different formats (`YYYY-MM-DD`, `MM/DD/YYYY`, `ISO 8601`).
- Time zones may be treated differently: UTC vs local time.
- Some systems truncate milliseconds while others preserve them, causing subtle inconsistencies in historical data.

### Currency and Numeric Precision
- Currency fields may have different scales or rounding rules.
- Decimal precision can vary (`DECIMAL(10,2)` vs `NUMERIC(12,4)`).
- Different locales use different decimal separators (`.` vs `,`), which can break data ingestion.

### Encoding and Special Characters
- ASCII vs UTF-8 vs UTF-16 differences can corrupt special characters.
- Systems may normalize characters differently (e.g., accents in European languages).

### Nulls, Defaults, and Constraints
- Some systems treat empty strings as `NULL`; others as valid strings.
- Default values may be applied differently during migrations.
- Constraints (foreign keys, unique indexes) may fail silently if data violates rules.
