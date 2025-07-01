
# Example: Transaction with Updates Followed by Rollback — Log & Buffer Pool View

---

## Step 1: Transaction Starts and Updates Data

| LSN | TxnID | Record Type | Description                     |
| --- | ----- | ----------- | ------------------------------- |
| 100 | 51    | BEGIN       | Start of transaction 51         |
| 101 | 51    | UPDATE      | AccountID=1 updated (1000→800)  |
| 102 | 51    | UPDATE      | AccountID=2 updated (1000→1200) |

**Log records are written to the log buffer (memory), not yet flushed to disk.**

---

## Step 2: Data Pages in Buffer Pool After Updates

| Page ID | AccountID | Value | Dirty? |
| ------- | --------- | ----- | ------ |
| P1      | 1         | 800   | ✅ Yes  |
| P1      | 2         | 1200  | ✅ Yes  |

* Pages loaded into memory (buffer pool)
* Modified pages marked as **dirty** because of updates
* Data files on disk **still have old values (1000, 1000)**

---

## Step 3: Transaction Is Rolled Back

| LSN | TxnID | Record Type            | Description                   |
| --- | ----- | ---------------------- | ----------------------------- |
| 103 | 51    | LOP\_ABORT\_TRANS      | Transaction 51 rollback start |
| 104 | 51    | LOP\_COMPENSATION\_LOG | Undo update for AccountID=1   |
| 105 | 51    | LOP\_COMPENSATION\_LOG | Undo update for AccountID=2   |
| 106 | 51    | LOP\_END\_XACT         | Transaction 51 rollback end   |

---

## Step 4: Buffer Pool After Rollback (Undo Applied In Memory)

| Page ID | AccountID | Value | Dirty? |                                     |
| ------- | --------- | ----- | ------ | ----------------------------------- |
| P1      | 1         | 1000  | ✅ Yes  | ← Restored original value in memory |
| P1      | 2         | 1000  | ✅ Yes  | ← Restored original value in memory |

* Undo applied to dirty pages in buffer pool (not yet flushed)
* Pages still dirty because they were modified again (restored)
* Data files on disk still hold **old values (1000, 1000)** (unchanged)

---

## Step 5: Dirty Pages Flushed to Disk (Checkpoint or Lazy Write)

| Page ID | AccountID | Value Written to Disk |
| ------- | --------- | --------------------- |
| P1      | 1         | 1000                  |
| P1      | 2         | 1000                  |

* Buffer pool writes dirty pages back to data files (`.mdf`)
* Disk now reflects the rolled-back state (original values)
* Transaction changes never made it permanent on disk

---

## Summary Table: Timeline of Data and Log

| Stage                 | Log Records          | Buffer Pool Data                          | Disk Data                        |
| --------------------- | -------------------- | ----------------------------------------- | -------------------------------- |
| After Updates         | LSN 100-102 (update) | AccountID 1 = 800 (dirty)                 | AccountID 1 = 1000 (old)         |
| Rollback Begins       | LSN 103-106 (abort)  | Undo applied → AccountID 1 = 1000 (dirty) | Still 1000 (old)                 |
| Checkpoint/Lazy Write | (no new log)         | Dirty pages flushed                       | AccountID 1 = 1000 (rolled back) |

## Rolled-back Transactions Still Cause Disk Writes — Here’s Why

Even though no new data changes are permanently committed after a rollback, SQL Server still writes to disk during the rollback process. This happens because:

1. Undoing Changes Happens in Memory First
Rollback modifies the dirty pages in the buffer pool to restore their original state.

These pages remain dirty after undo — they were changed again (restored to old values).

2. Dirty Pages Must Eventually Be Flushed to Disk
SQL Server must flush these dirty pages (now holding the rolled-back data) to the data files on disk.

This ensures the data files on disk reflect the consistent, pre-transaction state.

3. Why Disk Writes Are Needed Despite No Commit
When a transaction modifies data, changes go first into the buffer pool (memory) as dirty pages.

The transaction log records the changes immediately to ensure durability.

However, dirty pages can be flushed to the data files on disk at any time — due to checkpoints, lazy writer, or memory pressure — even before the transaction commits.

This means partial uncommitted changes might already be on disk.


Here’s a polished and rewritten version of your content. It maintains all the technical accuracy while improving clarity, flow, and readability.



## 🔄 Why SQL Server Still Writes Rolled-Back Pages to Disk

At first glance, it may seem like SQL Server could avoid writing data pages to disk when a transaction is rolled back by tracking when check points — but in reality, **it cannot safely skip that step**. Here's why:



### 🔥 Why Rolled-Back Changes Still Result in Disk Writes

#### 1. ✅ **Pages May Have Mixed Transaction Changes**

* A data page belongs to a **table or index**, but it can contain **rows modified by multiple transactions**.
* For example, Transaction A might update one row while Transaction B updates another — all on the same page.
* Even if Transaction A is rolled back, that **same page** may still hold **valid committed changes from Transaction B**.
* Therefore, SQL Server **can’t just skip writing the page** — it may contain changes that must be persisted.

---

#### 2. ⚠️ **Some Changes May Have Already Been Flushed**

* Dirty pages are flushed by **lazy writer** or **checkpoint**, even before a transaction commits or rolls back.
* These early flushes could have persisted **partial changes** from uncommitted transactions.
* SQL Server **doesn’t track which LSNs are written per page** — only that the page is dirty.
* To restore consistency, it **must write the corrected version** after rollback to **overwrite any partial data** on disk.

---

#### 3. 🔐 **Data Integrity via Write-Ahead Logging (WAL)**

* SQL Server follows WAL: **log first, then data**.
* Before flushing a page, it ensures all associated **log records are safely on disk**.
* During rollback, SQL Server:

  * Uses log to **restore original values** (undo).
  * Marks the page dirty **again**.
  * Flushes it to disk to ensure the data file reflects the correct (rolled-back) state.

---

### 🧠 Real-World Analogy

> Think of a shared notebook (data page). Multiple users write in it (different transactions). A copy (disk flush) is made before everyone finishes. To publish the final version (commit), you must use a separate record (transaction log) to undo unfinished edits and reprint the corrected version.

---

### ✅ Summary Table

| Misconception                               | Why It Doesn’t Hold Up                                          |
| ------------------------------------------- | --------------------------------------------------------------- |
| “Skip flushing rolled-back pages”           | Pages might contain other committed changes                     |
| “Track if rollback was completed in memory” | Pages may have already flushed partial data before rollback     |
| “Keep rollback changes only in memory”      | SQL Server must flush dirty pages eventually to maintain safety |

---

## 🔍 How Can Multiple Transactions Share a Page?

Even though pages are mapped to a **specific table or index**, they are **not isolated by transaction**. Here's how that works:

### Example: Page `P1` of `Accounts` Table

| Row | AccountID | Balance |
| --- | --------- | ------- |
| 1   | 1001      | 500     |
| 2   | 1002      | 800     |
| 3   | 1003      | 200     |

Now suppose:

* **Transaction A** updates `AccountID = 1001 → 700`
* **Transaction B** updates `AccountID = 1003 → 250`

Both transactions touch different rows, but the **same data page (`P1`)**.

**In memory (buffer pool)**:

* Page `P1` becomes **dirty**, containing uncommitted changes from both A and B.
* If `P1` is flushed to disk during this time, it will contain a **mix of valid and invalid data**.
* Hence, rollback of Transaction A must:

  * Undo changes in memory.
  * Mark the page dirty again.
  * **Flush the corrected version to disk**.

---

### 🔄 Flush Behavior

| Mechanism    | Can Flush Dirty Pages? | Knows About Transaction Boundaries? |
| ------------ | ---------------------- | ----------------------------------- |
| Lazy Writer  | ✅ Yes                  | ❌ No                                |
| Checkpoint   | ✅ Yes                  | ✅ Partially (uses Dirty Page Table) |
| Manual Flush | ✅ Yes                  | ❌ No                                |



