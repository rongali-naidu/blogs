
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
