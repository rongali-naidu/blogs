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




### ✅ Let’s clarify:

What is showed earlier was a **simplified conceptual view** of what is inside a `LOP_BEGIN_CKPT` log record — *not* the exact physical format that SQL Server writes to the transaction log. The **actual log records in SQL Server** are structured **by type**, and **each type has its own internal format and metadata fields**.



## 📘 Examples of Log Record Types and Formats

| Log Record Type     | Description            | Contains                                                  |
| ------------------- | ---------------------- | --------------------------------------------------------- |
| `LOP_BEGIN_XACT`    | Start of a transaction | Transaction ID, timestamp                                 |
| `LOP_MODIFY_ROW`    | A row change (DML)     | Page ID, slot, before/after values                        |
| `LOP_COMMIT_XACT`   | Commit marker          | Transaction ID, Commit LSN                                |
| `LOP_BEGIN_CKPT`    | Checkpoint marker      | Serialized: Checkpoint LSN, Dirty Page Table, Active Txns |
| `LOP_HOBT_DELTA`    | Index metadata update  | HoBT (Heap or B-Tree) changes                             |
| `LOP_SET_BITS`      | Bitmap update          | For things like GAM/SGAM/BCM pages                        |
| `LOP_ALLOCATE_PAGE` | Page allocation        | File ID, Page ID, etc.                                    |

Each record has a **header** (LSN, type, PrevLSN) and a **payload** (contents specific to the type).


## So When You See This:

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

This is just a **logical representation** of the data **embedded inside the `LOP_BEGIN_CKPT`** record’s **payload**. It’s not how the log record would appear in raw form or in `fn_dblog`.



## 🔧 Internally, SQL Server log records have:

* **Record header**:

  * `Current LSN`
  * `PrevLSN` (for log chain traversal)
  * `Transaction ID` (if applicable)
  * `Type Code` (e.g., `LOP_COMMIT_XACT`, `LOP_MODIFY_ROW`)
* **Record payload**:

  * Depends on `Type Code`
    For example:

    * `LOP_MODIFY_ROW` → has row versions
    * `LOP_BEGIN_CKPT` → has serialized DPT + ATT structures



## 📘 Want to See Actual Log Record Types?

You can use `fn_dblog` to peek at the active transaction log:

```sql
SELECT [Current LSN], Operation, Context, [Transaction ID], [Transaction Name], Description
FROM fn_dblog(NULL, NULL)
WHERE Operation IN ('LOP_BEGIN_CKPT', 'LOP_COMMIT_XACT');
```

You’ll see entries like:

```text
| Current LSN | Operation       | Context  | Transaction ID | Description              |
|-------------|-----------------|----------|----------------|--------------------------|
| 00000026:00000120:0012 | LOP_BEGIN_CKPT | LCX_NULL | NULL           | checkpoint started      |
| 00000026:00000140:0001 | LOP_COMMIT_XACT| LCX_NULL | 000000000007   | commit txn 7            |
```

> But `fn_dblog` does **not** show you the entire payload — it only gives you metadata. The actual serialized contents are deeply internal and accessible only via debugging tools or undocumented APIs.


Absolutely — let’s revise and enhance the explanation by including **what the master file is**, how it relates to **Page ID**, and regenerate everything as a comprehensive, beginner-friendly explanation.

---

# 🧱 Understanding Page ID in SQL Server

When you're diving into SQL Server internals — transactions, logging, and recovery — one term you'll encounter often is the **Page ID**.

Let’s break it down step by step.

---

## 📁 1. What Is the Master File (`.mdf`)?

Every SQL Server database has at least one **primary data file** — this is typically the `.mdf` file.

### 📌 Key Characteristics:

| Attribute           | Description                                           |
| ------------------- | ----------------------------------------------------- |
| **Extension**       | `.mdf`                                                |
| **Role**            | Contains startup system info + user data              |
| **Default File ID** | Always **File ID = 1**                                |
| **Page ID Example** | `Page ID (1:347)` points to page 347 inside this file |

👉 Think of the `.mdf` as the **main warehouse** — where most of the data is initially stored unless split into other files (`.ndf`).

---

## 📦 2. What Is a Page?

SQL Server stores all data (rows, indexes, metadata) in **8 KB units** called **pages**.

| Attribute       | Value                                     |
| --------------- | ----------------------------------------- |
| **Page size**   | 8 KB = 8192 bytes                         |
| **Content**     | Holds rows for tables, indexes, metadata  |
| **Access unit** | Every I/O operation happens at page level |

---

## 🆔 3. What Is a Page ID?

A **Page ID** uniquely identifies a specific page inside a database.

### 🔹 It has two parts:

```
Page ID = (File ID : Page Number)
```

| Component       | Description                                |
| --------------- | ------------------------------------------ |
| **File ID**     | Points to a specific `.mdf` or `.ndf` file |
| **Page Number** | Position of the page within that file      |

📌 Example:

* `Page ID = (1:347)`
* Means: Page 347 in **File 1**, which is usually the primary `.mdf` file

---

## 🔗 4. How Does Page ID Map to the Actual File?

Each SQL Server database can have multiple data files:

* `.mdf` (primary data file)
* `.ndf` (secondary data files)

Each file is assigned a **File ID**, stored in system views like:

```sql
SELECT name, file_id, physical_name FROM sys.master_files WHERE database_id = DB_ID();
```

To locate a page physically:

1. Multiply the **Page Number × 8192 (8KB)** to get the **byte offset**
2. Apply that offset inside the corresponding file (`.mdf`, `.ndf`)

---

## 🔍 5. How Does Page ID Help Locate a Record?

Each page contains multiple rows, and SQL Server uses a **slot array** at the end of the page to track row positions.

So a row is identified by:

```
Record Address = (File ID : Page Number : Slot Number)
```

### Example:

```
Page (1:347)
 ├─ Slot 0 → Row: AccountID = 1
 ├─ Slot 1 → Row: AccountID = 2
 └─ Slot 2 → Row: AccountID = 3
```

So SQL Server knows:

* **Where** a row lives (Page ID)
* **Which row** (Slot number)

---

## 🧾 6. How Page ID Is Used in Logs

SQL Server logs row changes using log records like `LOP_MODIFY_ROW` that include:

* **Transaction ID**
* **Page ID** (File ID + Page Number)
* **Slot number**
* **Before/After image of the row**
* **LSN** (Log Sequence Number)
* **PrevLSN** (for undo)

This allows SQL Server to **REDO** and **UNDO** changes with precision during recovery.

---

## 🧪 7. How to View Page Mappings

You can find page info for a table using:

```sql
DBCC IND('YourDatabaseName', 'YourTableName', 1);
```

Then, inspect page contents using:

```sql
DBCC TRACEON(3604);
DBCC PAGE('YourDatabaseName', 1, 347, 3);
```

---

## ✅ Summary

| Term               | Description                                                          |
| ------------------ | -------------------------------------------------------------------- |
| **.mdf file**      | Primary data file (File ID = 1)                                      |
| **Page**           | 8KB block of storage                                                 |
| **Page ID**        | `(File ID : Page Number)` uniquely identifies a page                 |
| **Record locator** | `(File ID : Page Number : Slot)` maps to a row                       |
| **Used in logs**   | Log records track changes by Page ID + Slot                          |
| **Why it matters** | Critical for transaction logging, crash recovery, performance tuning |



## 1. Default Behavior: One Big File

When you create a database in SQL Server:

* It creates:

  * One **primary data file** (`.mdf`)
  * One **log file** (`.ldf`)

All user tables, system tables, indexes, and metadata are stored inside the `.mdf` file unless you add more files.

### So yes — **multiple tables share the same file**.

> It's like one warehouse (`.mdf`) with different shelves (tables/pages) inside.

---

## 📂 2. Inside That One File…

SQL Server **organizes data** using:

* **Filegroups** (logical container)
* **Files** (physical storage: `.mdf`, `.ndf`)
* **Pages** (8KB units that hold actual rows)

Each table is broken down into:

* **Data pages** (hold rows)
* **Index pages**
* All stored within the `.mdf` file unless otherwise configured.

So:

* Table A might be on pages (1:100)-(1:110)
* Table B on pages (1:200)-(1:210)
* Still within the same file (File ID = 1)

---

## 🧱 3. Can You Split Tables Across Files?

Yes — but only **indirectly**, using **filegroups**.

### Example:

```sql
CREATE DATABASE MyDb
ON PRIMARY (
    NAME = N'MyDb_Data1',
    FILENAME = N'C:\Data\MyDb1.mdf'
),
FILEGROUP FG_Second (
    NAME = N'MyDb_Data2',
    FILENAME = N'D:\Data\MyDb2.ndf'
)
LOG ON (
    NAME = N'MyDb_Log',
    FILENAME = N'C:\Logs\MyDb.ldf'
);
```

Then assign a table or index to that filegroup:

```sql
CREATE TABLE LargeTable (
    ID INT,
    Data VARCHAR(1000)
) ON FG_Second;
```
                                                                   

# 🧱 SQL Server Storage: One File vs Multiple Files vs Disks

When SQL Server stores all tables in **one big `.mdf` file**, and that file is on a **single disk**, it can limit **I/O parallelism**.

Let’s break this down using practical layers:

---

## 🔁 1. **Disk-Level I/O vs Table Reads**

| Concept             | Description                                                                          |
| ------------------- | ------------------------------------------------------------------------------------ |
| **Disks (or LUNs)** | Physical or virtual storage where files are written to/read from                     |
| **Files**           | `.mdf`, `.ndf` data files reside on disks                                            |
| **Tables**          | Are **not tied to a disk** — they live **inside files**, which live on disks         |
| **Reads/Writes**    | SQL Server **reads pages (8KB)** from files, which translates to **reads from disk** |

🔄 So:

* Tables → live in files → files → live on disks
* **All table data access is file I/O**, and all file I/O is **disk I/O**

---

## 🚫 Problem: One File on One Disk

```text
Disk A
└── MyDatabase.mdf
    ├── Table A
    ├── Table B
    ├── Table C
```

* All table reads/writes go through **one file**
* All file I/O hits **one disk**
* **Limited throughput**: only so many MB/s or IOPS per disk
* Even with SSDs/NVMe, heavy concurrency can overwhelm a single path

---

## ✅ Solution: One File Per Disk (or Multiple Files Across Disks)

```text
Disk A              Disk B              Disk C
└── MyDatabase.mdf  └── MyDatabase2.ndf └── MyDatabase3.ndf
    ├── Table A         ├── Table B         ├── Table C
```

### Benefits:

| Advantage               | Why It Matters                                                   |
| ----------------------- | ---------------------------------------------------------------- |
| **Parallel I/O**        | SQL Server can read/write from multiple files **simultaneously** |
| **Reduced contention**  | TempDB or heavily written tables don’t block each other          |
| **Improved throughput** | Each disk handles a portion of total I/O                         |
| **Scalability**         | As your workload grows, you can add more disks/files             |

---

## 📚 Table-to-File Mapping: Can I Put a Table on a Specific Disk?

Not directly. But you can:

1. **Create filegroups mapped to files on different disks**
2. **Place a table or index onto a specific filegroup**

```sql
CREATE TABLE SalesHistory (
    SaleID INT,
    SaleDate DATE,
    Amount MONEY
) ON FileGroup_History;
```

Where `FileGroup_History` contains files located on `Disk D:\`

> So you **don’t bind a table to a disk**, but you **indirectly bind it via filegroups and file placement**.

---

## 🔧 Example: Spreading Tables Across Disks

Let’s say you have 3 hot tables:

* `Orders` (write-heavy)
* `Customers` (read-heavy)
* `AuditLogs` (append-only)

You can do:

| Table       | Filegroup     | Files          | Disk |
| ----------- | ------------- | -------------- | ---- |
| `Orders`    | FG\_Orders    | orders1.ndf    | E:\\ |
| `Customers` | FG\_Customers | customers1.ndf | F:\\ |
| `AuditLogs` | FG\_Logs      | auditlogs1.ndf | G:\\ |

Then assign each table to its own filegroup:

```sql
CREATE TABLE Orders (...) ON FG_Orders;
CREATE TABLE Customers (...) ON FG_Customers;
CREATE TABLE AuditLogs (...) ON FG_Logs;
```

Now:

* SQL Server can read/write them in parallel
* Each disk carries part of the load
* No disk becomes a bottleneck for all data

---

## 🧠 Summary: Disk ↔ File ↔ Table Mapping

| Layer                 | Role                                                    |
| --------------------- | ------------------------------------------------------- |
| **Disk**              | Physical/virtual storage unit                           |
| **File (.mdf, .ndf)** | Maps to a specific disk                                 |
| **Filegroup**         | Logical group of one or more files                      |
| **Table**             | Created on a filegroup → maps to underlying files/disks |

---

## 🚦 Default vs Scalable Design

| Design                          | Suitable For                                       |
| ------------------------------- | -------------------------------------------------- |
| **One .mdf on one disk**        | Small DBs, dev/test, low concurrency               |
| **Multiple files across disks** | OLTP systems, high concurrency, data warehouses    |
| **Table per filegroup/disk**    | Critical for I/O tuning, backups, partial restores |


