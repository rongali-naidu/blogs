# 🔍 SQL Server Internal Logging & Crash Recovery — A Deep Dive with Example

This guide explains how **LSNs (Log Sequence Numbers)**, **Transaction IDs**, **log records**, and **data buffers** interact in SQL Server during transaction processing and crash recovery. We illustrate this with an in-depth example and clear visuals.

---

## ✅ Key Concepts Refresher

| Term                          | Description                                                              |
| ----------------------------- | ------------------------------------------------------------------------ |
| **LSN**                       | A unique, ever-increasing identifier assigned to every log record        |
| **Transaction ID**            | Identifier for each transaction (can span multiple log records)          |
| **Log Record**                | Each operation (begin, update, commit, etc.) is written to the log       |
| **Data Buffer (Buffer Pool)** | In-memory copy of data pages; changes are made here first                |
| **Dirty Page**                | A data page in memory that has been modified but not yet flushed to disk |
| **Checkpoint**                | SQL Server event that flushes dirty pages and records a recovery LSN     |
| **Redo**                      | Reapplies committed changes from log to data pages (post-crash)          |
| **Undo**                      | Rolls back uncommitted changes from log to data pages (post-crash)       |

---

## 🧪 Example Scenario: Transfer Between Accounts

### Initial Table

```sql
CREATE TABLE BankAccounts (
  AccountID INT PRIMARY KEY,
  Balance INT
);

-- Initial data:
-- (1, 1000), (2, 1000)
```

### Transaction Starts

```sql
BEGIN TRANSACTION;
UPDATE BankAccounts SET Balance = Balance - 200 WHERE AccountID = 1;  -- 800
UPDATE BankAccounts SET Balance = Balance + 200 WHERE AccountID = 2;  -- 1200
-- COMMIT not yet issued
```

---

## 🧠 What Happens Internally

### Log Records Created:

| LSN | TxnID | Record Type | Description                          |
| --- | ----- | ----------- | ------------------------------------ |
| 100 | 51    | BEGIN       | Start of transaction                 |
| 101 | 51    | UPDATE      | Update on AccountID 1 (1000 -> 800)  |
| 102 | 51    | UPDATE      | Update on AccountID 2 (1000 -> 1200) |

> These are written to the **log buffer** in memory (not on disk yet).

### Data Pages in Buffer Pool:

| Page | Account | Value | Dirty? |
| ---- | ------- | ----- | ------ |
| P1   | 1       | 800   | ✅ Yes  |
| P1   | 2       | 1200  | ✅ Yes  |

At this point:

* `.mdf` (data file) = still has (1,1000), (2,1000)
* `.ldf` (log file) = may or may not have LSNs flushed, depending on pressure

---

## 💾 If COMMIT is Issued

```sql
COMMIT;
```

### What SQL Server Does:

1. Writes a **Commit log record**:

   | LSN | TxnID | Record Type | Description           |
   | --- | ----- | ----------- | --------------------- |
   | 103 | 51    | COMMIT      | Commit transaction 51 |

2. **Flushes** the log buffer to disk (`.ldf`) up to and including LSN 103

3. Data pages remain dirty in memory

At this point:

* `.ldf` has LSNs 100–103 for Txn 51 (durable)
* `.mdf` still has old values (1,1000), (2,1000)

---

## 💥 If Crash Happens Now

### What SQL Server Knows on Restart:

* Last **Checkpoint LSN** = 98
* Log file has LSNs 100–103 for Txn 51
* Data file is outdated

### Recovery Phases:

1. **Analysis**: Scans from checkpoint (LSN 98), finds committed Txn 51
2. **REDO**: Applies UPDATEs from LSN 101 and 102 to data file (800/1200)
3. ✅ DB is consistent and reflects committed state

---

## 💥 If Crash Happens Before COMMIT

Imagine crash occurs after LSN 102, but **before** LSN 103 is flushed

### What SQL Server Sees:

* No COMMIT record in log for Txn 51
* But UPDATEs may be partially flushed (by lazy writer)

### Recovery Phases:

1. **Analysis**: Txn 51 is uncommitted
2. **UNDO**: Walks backward:

   * LSN 102: Undo AccountID 2 = 1200 → 1000
   * LSN 101: Undo AccountID 1 = 800 → 1000
3. ✅ Rollback applied, data file restored to pre-transaction state

---

## 🛠️ Role of Checkpoint

### During Checkpoint:

* Flushes **all dirty pages** to `.mdf`
* Writes a **Checkpoint Record** in `.ldf` with:

  * Checkpoint LSN
  * Active Transaction Table (incomplete txns)
  * Dirty Page Table (page IDs and LSNs)

> On crash, SQL Server uses this to **start analysis phase** efficiently

---

## 🔄 Summary of Log Flow

1. DML happens → write log records to **log buffer**
2. Data changes → applied to **buffer pool** (dirty pages)
3. COMMIT → write `LOP_COMMIT_XACT` to log
4. Log buffer → flushed to `.ldf`
5. Checkpoint → flush dirty pages to `.mdf` + write checkpoint LSN to `.ldf`
6. On crash → SQL Server recovers using `.ldf`:

   * REDO committed transactions
   * UNDO uncommitted ones

---

## 📘 DMV Queries to Observe This

```sql
-- See active transactions
SELECT * FROM sys.dm_tran_active_transactions;

-- See last checkpoint info
SELECT checkpoint_lsn, database_id FROM sys.dm_database_recovery_status;

-- See dirty pages (requires sysinternals/debugging tools)
DBCC MEMORYSTATUS -- or use extended events
```

---

## ✅ Why This Matters

* **LSNs** ensure every operation is tracked and can be undone or redone
* **Dirty pages** may be flushed before COMMIT, so UNDO is critical
* **Crash recovery** is entirely driven by **log records** and checkpoint metadata

