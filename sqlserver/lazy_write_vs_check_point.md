

## ✅ Lazy Writer vs Checkpoint — What's the Difference?

### 🔄 **Lazy Writer**

* **Purpose**: Frees up space in the buffer pool by writing **least recently used dirty pages** to disk.
* **When it runs**:

  * Periodically (on a background schedule).
  * When the buffer pool is under **memory pressure**.
* **Scope**:

  * Writes **only a few dirty pages at a time**.
  * Does **not** care about transaction state (committed or not).
* **Goal**: Make room in memory.

---

### 📍 **Checkpoint**

* **Purpose**: Writes **all dirty pages** (from committed transactions) to disk and **records a Checkpoint LSN** in the log.

* **When it runs**:

  * Automatically at intervals (controlled by recovery model & workload).
  * **When SQL Server is shutting down**.
  * **Before a backup starts**.
  * When **memory pressure is high**, but mainly to reduce recovery time.

* **Scope**:

  * Attempts to write **all dirty pages** since the last checkpoint.
  * Skips uncommitted transactions — but those are handled during recovery.

* **Also does**:

  * Writes a `LOP_BEGIN_CKPT` log record.
  * Logs the **Active Transaction Table (ATT)** and **Dirty Page Table (DPT)**.

* **Goal**: Minimize crash recovery time by reducing how much of the transaction log needs to be replayed.


## 🔍 Summary Table

| Feature       | Lazy Writer                        | Checkpoint                                  |
| ------------- | ---------------------------------- | ------------------------------------------- |
| Triggered by  | Memory pressure or background task | Time, memory pressure, backup, shutdown     |
| Writes pages  | Least recently used dirty pages    | All dirty pages (of committed transactions) |
| Scope         | Small, incremental                 | Broad, database-wide                        |
| Writes log?   | ❌ No                               | ✅ Yes — writes a `LOP_BEGIN_CKPT` entry     |
| Recovery role | ❌ None                             | ✅ Major role — defines where REDO starts    |
| Goal          | Free memory                        | Reduce recovery time                        |


