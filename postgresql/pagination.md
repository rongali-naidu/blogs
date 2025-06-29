## What is Pagination?

Pagination is the technique used to break a large dataset into manageable "pages" (chunks) of results. It’s essential for:

* UI rendering (e.g., showing multiple pages in tables, feeds)
* API responses
* Reports and dashboards

### 1. **OFFSET-based Pagination** (a.k.a. Page Number Pagination)

#### Example:

```sql
SELECT * FROM transactions
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;
```

#### How it works:

* Skips the first `OFFSET` rows.
* Returns the next `LIMIT` rows.
* LIMIT and OFFSET values can be dynamically calculated:

  * `LIMIT` = RecordsPerPage
  * `OFFSET` = `(PageNumber - 1) * RecordsPerPage`


#### OFFSET-based Pagination Syntax for different databases

| Database          | Pagination SQL (summary)                 |
| ----------------- | ---------------------------------------- |
| **SQL Server**    | `OFFSET 40 ROWS FETCH NEXT 20 ROWS ONLY` |
| **PostgreSQL**    | `LIMIT 20 OFFSET 40`                     |
| **Oracle (12c+)** | `OFFSET 40 ROWS FETCH NEXT 20 ROWS ONLY` |
| **Redshift**      | `LIMIT 20 OFFSET 40`                     |


#### Pros:

* Easy to implement and understand.
* Works well with UI controls like page numbers ("Go to page 5").

#### Cons:

* Becomes **slower as OFFSET increases** because the DB reads and discards previous rows. Each call still sorts the data, and the more rows it skips, the slower it gets.
  **Analogy**: Like flipping through pages in a book to get to page 501 instead of jumping straight to a bookmark.
* **Unstable** when rows are inserted/deleted during pagination — may lead to duplicates or skips.

#### Use when:

* You need jump-to-page functionality.
* Dataset is relatively small or access is infrequent.


### 2. **Keyset Pagination** (a.k.a. Seek Pagination or Cursor Pagination)

#### Example:

```sql
SELECT * FROM transactions
WHERE created_at > '2024-06-01 10:00:00'
ORDER BY created_at ASC
LIMIT 20;
```

#### How it works:

* Instead of skipping N rows, you ask for rows **after a specific value or position** (e.g., created\_at). If we refer the value  or token that represents the last item as position, it is reffered as the Cursor Pagination in some documents
* Uses an **index to jump** to the position.
* This is commonly used for CDC (Change Data Capture) based on audit-timestamp columns. 
* Note ON ORDER BY Cause+ LIMIT Cause : This is required for restricting in terms of pages. In Some scenarions, we could skip it. For ETL Batch CDC Logic, I will return skip it but track the max value of  created_at separately.

### Keyset Pagination Syntax  for different databases

| Database       | Pagination SQL (summary)                             |
| -------------- | ---------------------------------------------------- |
| **SQL Server** | `WHERE created_at > ... OFFSET 0 ROWS FETCH NEXT 20` |
| **PostgreSQL** | `WHERE created_at > ... LIMIT 20`                    |
| **Oracle**     | `TO_TIMESTAMP(...) FETCH FIRST 20 ROWS ONLY`         |
| **Redshift**   | `WHERE created_at > ... LIMIT 20`                    |


#### Pros:

* **Fast and scalable** even with large data.
* **Stable** even if new rows are inserted or deleted.
* Works well with **infinite scrolling** UIs.

#### Cons:

* You **can’t jump to arbitrary pages** (like “page 5”).
* Requires careful handling of **tie-breakers** to avoid duplicates or missing of the data. for example: if there are multiple records with same created_at and we fetched a partial  records in the previous . next fetch will either miss some records or re-fetch a few record depending on if we used `=` or `>=` in the filter condition

#### Use when:

* You need **fast pagination** at scale.
* Data is **time-based or sequential** i.e when you have a column with timestamp data type or a coulmn which gets auto incremental values and you can rely on this column to get the incremental data
* UI supports “load more” or infinite scroll.
