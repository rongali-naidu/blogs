# 🛡️ SQL Server Backup Explained: Full + Log + Ola Hallengren Script

Backing up a SQL Server database isn’t just about copying files — it’s about maintaining **transactional consistency**, **recoverability**, and **minimal downtime**. Let’s walk through what’s really happening during a backup and how to automate it safely.


Great! Let's expand your blog post with a clear explanation of **SQL Server recovery models**, how they affect **transaction log handling**, and what happens **with or without log backups**. This is essential knowledge for understanding backup strategy and recovery planning.

---

## 🔄 Understanding Recovery Models in SQL Server

The **Recovery Model** of a SQL Server database determines:

* How **much transaction log data** SQL Server retains
* Whether you can do **point-in-time restores**
* How the **transaction log is truncated** (or not)

You can check it using this query:

```sql
SELECT name, recovery_model_desc 
FROM sys.databases;
```

---

### 🧩 Types of Recovery Models

| Recovery Model   | Log Backup Required? | Auto-Truncation? | Point-in-Time Restore? |
| ---------------- | -------------------- | ---------------- | ---------------------- |
| **FULL**         | ✅ Yes                | ❌ No             | ✅ Yes                  |
| **BULK\_LOGGED** | ✅ Yes                | ❌ No             | ✅ Yes (with caveats)   |
| **SIMPLE**       | ❌ No                 | ✅ Yes            | ❌ No                   |

---

## ✅ What Happens After You Take a Log Backup

1. SQL Server **copies all log records** (since the last log backup) into a `.trn` file.
2. The **log space is now marked as reusable** (this is called **log truncation**).
3. The **.ldf file does not shrink**, but **unused space is freed internally** for new transactions.

> 🔁 Think of it like rewinding a tape recorder — after log backup, the tape is rewound and ready to record new changes.

You can verify current log usage with:

```sql
DBCC SQLPERF(LOGSPACE);
```

---

## ❌ What Happens If You Don't Take Log Backups (in FULL Recovery)?

* The transaction log **keeps growing** — nothing is truncated.
* Over time, the `.ldf` file may consume all available disk space.
* SQL Server will eventually throw errors like:

  ```
  The transaction log for database 'YourDB' is full due to 'LOG_BACKUP'.
  ```

---

## 🔥 What Happens If You Lose the Log Backup?

This is **critical**:

* If you lose a `.trn` file and the database is in **FULL** recovery mode, you can no longer do **point-in-time recovery** past that log.
* You might only be able to restore up to the last **valid FULL backup**, depending on what log backups are available.

### 🧨 Example Scenario

| Time    | Action                      |
| ------- | --------------------------- |
| 1:00 AM | FULL backup taken           |
| 1:15 AM | Log backup 1 taken (`.trn`) |
| 1:30 AM | Log backup 2 taken (`.trn`) |
| 2:00 AM | Crash occurs                |

To restore to 1:59 AM:

* You need: **FULL backup + both log backups**
* ❌ If Log Backup 2 is **missing**, you **cannot** restore beyond Log 1

---

## 🚨 Recovery Model & Backup Strategy Are Tied Together

| If your goal is...                     | Then use...   | And do this...                                               |
| -------------------------------------- | ------------- | ------------------------------------------------------------ |
| Minimal backup effort (no PITR needed) | `SIMPLE`      | Just schedule FULL or DIFF backups                           |
| Full disaster recovery + PITR          | `FULL`        | Schedule FULL + frequent LOG backups                         |
| Large data loads with PITR             | `BULK_LOGGED` | Like FULL, but avoid minimally logged ops during log backups |

---

## 💡 How to Change Recovery Model (Be Cautious!)

```sql
-- Set to SIMPLE (logs will truncate automatically)
ALTER DATABASE YourDB SET RECOVERY SIMPLE;

-- Set to FULL (for full log retention and PITR)
ALTER DATABASE YourDB SET RECOVERY FULL;
```

> ⚠️ Switching from FULL → SIMPLE destroys the log chain — you can’t restore logs after that point.

---

## 📦 Summary: Why Log Backups Matter

| Scenario                           | Outcome                                              |
| ---------------------------------- | ---------------------------------------------------- |
| FULL recovery + log backups        | ✅ PITR possible, log space reused                    |
| FULL recovery + **no** log backups | ❌ Log file keeps growing; PITR not possible          |
| Log backups **lost**               | ❌ You lose recovery between FULL backup and lost log |
| SIMPLE recovery                    | ✅ Low maintenance, but no PITR                       |



## ✅ Why You Need a FULL Backup

A **FULL backup** is the foundation of your backup strategy.

* It includes **all data pages** and **enough transaction log records** to restore the database to the **exact point in time** when the backup completed.
* It serves as the **base image** required for restoring any future **differential or log backups**.

> Without a recent FULL backup, you cannot restore the database — even if you have all the logs.

---

## 🔁 Why You Need a LOG Backup (with FULL recovery model)

A **LOG backup** captures the changes recorded in the **transaction log (.ldf)** since the last log backup.

You need log backups because:

| Purpose                    | Why It Matters                                         |
| -------------------------- | ------------------------------------------------------ |
| ✅ Point-in-time recovery   | You can restore to any moment before failure           |
| ✅ Controls log file growth | Log backups allow SQL Server to reuse space            |
| ✅ Complements FULL backup  | You need log backups to replay changes since last full |

> Without log backups, the log will grow endlessly (under FULL recovery model), and point-in-time recovery won't be possible.

---

## 🔍 What Happens During a FULL Backup

When SQL Server performs a full backup:

1. ✅ **Checkpoint runs**:

   * Flushes dirty pages (modified in memory) to disk to get a consistent baseline.

2. ✅ **Backup starts reading data files**:

   * It copies only **allocated pages** (used data) from `.mdf`/`.ndf` files.
   * It **does NOT block transactions** or user queries.

3. ✅ **New changes are tracked via the transaction log**:

   * SQL Server notes a **Backup Start LSN**.
   * It continues backing up log records until **Backup End LSN** to capture in-flight changes.

4. ✅ **Result is a `.bak` file**:

   * Contains data pages + log records needed for transactional consistency.

> Even if data pages change while backup is reading them, **SQL Server can "correct" that** during restore using the captured log.

---

## 🚫 Does Backup Block Writes?

**No**, SQL Server allows **full concurrency** during backup:

| Component                   | Behavior During Backup                 |
| --------------------------- | -------------------------------------- |
| Inserts/Updates/Deletes     | ✅ Allowed                              |
| Lazy writer flushing pages  | ✅ Allowed                              |
| Checkpoints                 | ✅ Still occur                          |
| Structural DDL (drop table) | 🚫 Blocked briefly with metadata locks |

SQL Server ensures consistency using **Write-Ahead Logging (WAL)** and **Log Sequence Numbers (LSNs)** — not by freezing the system.

## How SQL Server Backs Up While Writes Continue

| Behavior                                                       | Supported During Full Backup? | Explanation                                                                                                                                 |
| -------------------------------------------------------------- | ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **User transactions (INSERT/UPDATE/DELETE)**                   | ✅ Yes                         | SQL Server allows full read/write concurrency during backup. Transactions are not blocked.                                                  |
| **Dirty pages flushed to data files (lazy writer/checkpoint)** | ✅ Yes                         | Backup is designed to tolerate writes happening *while* it’s reading data files.                                                            |
| **Backup reading from data files (`.mdf`, `.ndf`)**            | ✅ Yes                         | Pages are read as they exist *at the moment* — SQL Server tracks which were changed during the backup using the transaction log.            |
| **Consistency maintained despite changes**                     | ✅ Yes                         | SQL Server uses the **transaction log (LSNs)** to replay or undo changes that occurred *during* the backup.                                 |
| **Structural changes (e.g., dropping a table)**                | 🚫 Temporarily blocked        | A short-lived schema stability lock prevents changes like dropping objects during backup.                                                   |
| **Accurate, consistent `.bak` output**                         | ✅ Yes                         | Even though data may change during backup, SQL Server produces a logically consistent backup by combining **data pages** + **log records**. |


### 🔍 How It Handles Data File Changes During Backup

SQL Server uses a mechanism called a **"fuzzy backup"**:

1. A **checkpoint** flushes most committed dirty pages to disk.
2. Backup **starts reading allocated pages** from the `.mdf` and `.ndf` files.
3. As pages are read:

   * If a page was modified *after* it was read (or read with uncommitted data), SQL Server still tracks that.
   * The **transaction log is scanned** from the **Backup Start LSN** to **Backup End LSN**.
   * The log records are included in the backup and used to bring the restored database back to a **transactionally consistent state**.

> ✅ This means: even if a page is read *before* it's modified, the **transaction log replay** during restore will bring that page to the correct state — or roll back uncommitted changes.

---

### 💡 Think of It Like This

> Imagine taking a photo of a moving train:
>
> * You snap the picture while some parts are moving.
> * Later, you combine the picture with a timeline of movements (transaction log).
> * You reconstruct the **correct state** of the train at the moment of the photo.

That’s exactly how SQL Server makes backups reliable, even when the data files are being actively modified.

---

## ⚙️ How to Do Backups with Ola Hallengren's Script

Ola Hallengren’s [**Maintenance Solution**](https://ola.hallengren.com/sql-server-backup.html) is a community-trusted, highly configurable tool to automate:

* FULL, DIFF, LOG backups
* Index maintenance
* Integrity checks

---

### 💾 FULL Backup Example

```sql
EXECUTE [dbo].[DatabaseBackup]
  @Databases = 'USER_DATABASES',
  @Directory = '~\Backups\Full',
  @BackupType = 'FULL',
  @Verify = 'Y',
  @CleanupTime = NULL,
  @CheckSum = 'Y',
  @Compress = 'Y',
  @LogToTable = 'Y';
```

### 📦 What This Does

| Parameter      | Purpose                                                              |
| -------------- | -------------------------------------------------------------------- |
| `@Databases`   | `'USER_DATABASES'` backs up all user-created databases               |
| `@Directory`   | Where to store the `.bak` files                                      |
| `@BackupType`  | `'FULL'` means take a full backup                                    |
| `@Verify`      | `'Y'` performs RESTORE VERIFYONLY to validate the backup file        |
| `@CheckSum`    | `'Y'` ensures corruption detection during backup                     |
| `@Compress`    | `'Y'` enables backup compression (faster and smaller)                |
| `@LogToTable`  | `'Y'` writes the operation details to `dbo.CommandLog` table         |
| `@CleanupTime` | `NULL` means don't auto-delete old backups (can be configured later) |

---

### 🔁 LOG Backup Example

```sql
EXECUTE [dbo].[DatabaseBackup]
  @Databases = 'USER_DATABASES',
  @Directory = '~\Backups\Log',
  @BackupType = 'LOG',
  @Verify = 'Y',
  @CleanupTime = NULL,
  @CheckSum = 'Y',
  @Compress = 'Y',
  @LogToTable = 'Y';
```

> 🔄 This backs up transaction logs for all user databases using the **same engine**.

---

## 🧠 How FULL and LOG Backups Work Together

Imagine this timeline:

```
T1 ---- T2 ---- T3 ---- T4 ---- T5 ---- T6
 ^                ^              ^
 FULL           LOG 1          LOG 2
```

To restore to time `T6`, you'd:

1. Restore the **FULL backup** from `T1`
2. Restore **LOG backup 1** and **LOG backup 2**
3. Optionally use `STOPAT = 'T6'` to stop at exact timestamp

---

## 🔍 Want to See What Happened?

After running backups with `@LogToTable = 'Y'`, check:

```sql
SELECT * FROM dbo.CommandLog ORDER BY StartTime DESC;
```

This shows:

* What was backed up
* Duration, status
* File paths
* Log and data sizes

---

## 🛠️ Want to Schedule Backups?

You can hook these into **SQL Server Agent Jobs** for automation:

* Daily FULL at midnight
* Log backups every 15 minutes
* Cleanup jobs weekly

---

## 📌 Final Summary

| Component           | Purpose                                          |
| ------------------- | ------------------------------------------------ |
| **FULL backup**     | Captures entire database and log to restore base |
| **LOG backup**      | Captures all changes since last log/full backup  |
| **Ola's script**    | Automates safe, compressed, verified backups     |
| **Transaction log** | Enables point-in-time restore + crash recovery   |

> SQL Server’s backup model, combined with Ola Hallengren’s tool, gives you a **bulletproof recovery strategy**.

---

Would you like a visual workflow of how FULL and LOG backups work during restore? Or help setting up SQL Agent jobs for this strategy?
