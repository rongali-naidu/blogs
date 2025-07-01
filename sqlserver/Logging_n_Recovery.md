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



## ✅ At a High Level: What Happens at Checkpoint?

When SQL Server runs a **checkpoint**, it performs the following steps:

1. **Flushes all dirty data pages** to disk
2. **Flushes log buffer** to ensure log records are durable
3. Writes a **Checkpoint Log Record** to the transaction log (`.ldf`) with metadata:

   * `Checkpoint LSN`
   * **Active Transaction Table**
   * **Dirty Page Table**

This metadata is **crucial for recovery** — it tells SQL Server:

> “Here is a known-consistent state. If we crash, start scanning the log from this point forward.”

---

## 📘 What is Written in the Checkpoint Log Record?

The **Checkpoint Log Record** (internal type: `LOP_BEGIN_CKPT`) includes:

### 1. ✅ **Checkpoint LSN**

* Marks the point in the log where checkpoint begins
* Stored in **boot page** of the database (`page_id = 9`)
* Used by recovery to determine where to start scanning on restart

### 2. 📋 **Active Transaction Table**

* List of **in-progress transactions** at checkpoint time
* Includes:

  * `TransactionID`
  * `FirstLSN` (where txn began)
  * `LastLSN` (most recent log record for the txn)
* Used during crash recovery **UNDO phase** to rollback any uncommitted transactions

### 3. 💾 **Dirty Page Table**

* List of **dirty buffer pages** (in memory but not yet flushed) at checkpoint time
* Includes:

  * `PageID`
  * `Recovery LSN` (the oldest LSN that dirtied the page)
* Used during crash recovery **REDO phase** to determine what pages may need to be redone

---

## 📍 Where Are These Tables Maintained Internally?

These structures live in **SQL Server’s memory** and are used during crash recovery:

| Table                        | Maintained In                                   | Purpose                                                              |
| ---------------------------- | ----------------------------------------------- | -------------------------------------------------------------------- |
| **Active Transaction Table** | Transaction Manager (internal memory structure) | Track all open/incomplete transactions                               |
| **Dirty Page Table**         | Buffer Manager                                  | Track which pages are dirty, and which log record first dirtied them |

✅ They are **not user-accessible tables**, but SQL Server serializes them and writes them into the log record during a checkpoint.

---

## 📌 How Dirty Page Table Maps PageID to LSN

Each page in the buffer pool is associated with:

* **PageID** (file\_id + page\_number)
* **Recovery LSN** (the LSN of the **first log record** that made the page dirty)

This Recovery LSN is crucial:

> If page was dirtied at LSN 105, then to **fully redo the page**, SQL Server needs to reapply all log records starting from LSN 105.

📌 So when a checkpoint occurs, SQL Server:

* Iterates through all dirty pages in memory
* Extracts their `PageID` and `Recovery LSN`
* Serializes this into the **Dirty Page Table**, which is embedded in the **Checkpoint Log Record**

---

## 🔍 How Are These Log Entries Structured?

You can’t directly read this raw structure from the `.ldf`, but internally, SQL Server writes a `LOP_BEGIN_CKPT` log record with:

```text
{
  CheckpointLSN: 125,
  ActiveTransactions: [
    { TxnID: 51, BeginLSN: 100, LastLSN: 122 },
    { TxnID: 52, BeginLSN: 104, LastLSN: 123 }
  ],
  DirtyPageTable: [
    { PageID: (1:204), RecoveryLSN: 109 },
    { PageID: (1:237), RecoveryLSN: 113 }
  ]
}
```

> Internally, this metadata is written as a **structured blob** in the log record, not a SQL-readable format. But this is how SQL Server reconstructs state on recovery.

---

## 🧠 How Does SQL Server Use This During Crash Recovery?

### 1. 🔎 **Analysis Phase**

* Start from **Checkpoint LSN** in the `LOP_BEGIN_CKPT` record
* Rebuild:

  * Active Transaction Table
  * Dirty Page Table

### 2. 🔁 **Redo Phase**

* Use Dirty Page Table to determine:

  * Which pages may not be up-to-date on disk
  * From which LSN to start applying log records

### 3. 🔃 **Undo Phase**

* Use Active Transaction Table to find uncommitted transactions
* Walk **backward** through their log chains and **undo** changes

---

## 🧪 DMV Insight (for live systems)

You can query the recovery-related info like this:

```sql
SELECT 
  database_id,
  recovery_model_desc,
  last_log_backup_lsn,
  checkpoint_lsn,
  redo_start_lsn,
  redo_start_fork_guid,
  redo_target_lsn
FROM sys.dm_database_recovery_status;
```

This shows what LSNs SQL Server will use in recovery.

---

## ✅ Summary

| Component                    | Description                                                |
| ---------------------------- | ---------------------------------------------------------- |
| **Checkpoint LSN**           | Starting point for recovery                                |
| **Active Transaction Table** | Tracks uncommitted transactions (TxnID, BeginLSN, LastLSN) |
| **Dirty Page Table**         | Tracks PageID and earliest LSN that dirtied the page       |
| **Log Entry Written**        | Serialized into `LOP_BEGIN_CKPT` log record                |
| **Used During Recovery**     | To REDO committed and UNDO uncommitted work post-crash     |


