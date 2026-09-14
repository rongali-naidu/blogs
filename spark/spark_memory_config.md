# Spark Executor Memory: Components and Config Parameters

## The memory layout, top to bottom

```
Executor container  (total memory requested from the cluster manager)
│
├── JVM heap                              [spark.executor.memory, default 1g]
│   │
│   ├── Reserved memory                   [hardcoded, ~300MB, not configurable]
│   │
│   ├── User memory                       [1 - spark.memory.fraction, default 0.4]
│   │     Your own objects, UDF state, RDD lineage metadata — unmanaged by Spark
│   │
│   └── Unified memory                    [spark.memory.fraction, default 0.6]
│         │
│         ├── Storage memory              [spark.memory.storageFraction, default 0.5
│         │     Broadcast variables +      of unified pool — a soft floor, not a cap]
│         │     cached/persisted RDDs
│         │
│         └── Execution memory            [1 - spark.memory.storageFraction, default 0.5]
│               Shuffle buffers, join hash
│               tables, sort buffers
│
└── Overhead (off-heap)                   [spark.executor.memoryOverhead]
      Metaspace, native shuffle-transfer    default = max(384MB,
      buffers, thread stacks, Python         executorMemory ×
      worker processes                       spark.executor.memoryOverheadFactor [0.1])
```

Execution and storage memory dynamically borrow from each other within the
unified pool. The one asymmetry: **execution can evict storage** (including
cached/broadcast blocks) under memory pressure, but **storage cannot evict
execution memory actively in use by a running task**.

## Config parameter reference table

| Component | Config parameter | Default | What it controls |
|---|---|---|---|
| Executor container (total) | `spark.executor.memory` + `spark.executor.memoryOverhead` (+ `spark.memory.offHeap.size` if enabled) | — | Total memory Spark requests per executor from YARN/K8s |
| JVM heap | `spark.executor.memory` | `1g` | Size of the on-heap JVM memory pool |
| Reserved memory | *(not configurable)* | 300MB fixed | Spark's own bookkeeping, off-limits to the job |
| Unified memory (share of heap − reserved) | `spark.memory.fraction` | `0.6` | Fraction of (heap − 300MB) given to execution + storage combined |
| User memory (remainder) | `1 - spark.memory.fraction` | `0.4` | Unmanaged memory for your own objects/UDFs |
| Storage memory (floor within unified) | `spark.memory.storageFraction` | `0.5` | Minimum share storage can retain even under execution pressure |
| Execution memory (remaining share) | `1 - spark.memory.storageFraction` | `0.5` | Shuffles, joins, sorts, broadcast hash-table builds |
| Overhead | `spark.executor.memoryOverhead` | `max(384MB, executorMemory × spark.executor.memoryOverheadFactor)`, factor default `0.1` | Off-heap: metaspace, native buffers, thread stacks |
| Overhead, Python-specific | `spark.executor.pyspark.memory` | unset (falls into shared overhead) | Dedicated ceiling for Python worker processes |
| Off-heap pool (opt-in, not in the diagram above) | `spark.memory.offHeap.enabled` + `spark.memory.offHeap.size` | disabled / `0` | A second execution+storage pool living outside the JVM heap, avoiding GC pressure |

**Related configs from adjacent topics (broadcast joins, driver collection):**

| Config | Default | Relevance |
|---|---|---|
| `spark.sql.autoBroadcastJoinThreshold` | `10MB` | Table size below which Spark auto-selects a broadcast join; the broadcast copy then lives in **storage memory** on every executor |
| `spark.driver.maxResultSize` | `1g` | Caps data pulled back to the driver — relevant to the collect step before a broadcast; also an internal ~8GB hard limit on broadcast size exists regardless |

## spark-submit flag vs. --conf

Both ultimately set the same underlying property — they differ in scope and
where you're allowed to use them.

- **Dedicated flags** (`--executor-memory`, `--driver-memory`,
  `--num-executors`, `--executor-cores`) are shorthand built into
  `spark-submit` for the most commonly set properties. `--executor-memory 8g`
  is translated internally into `spark.executor.memory=8g` before the job
  launches — it isn't a separate mechanism.
- **`--conf key=value`** is the general-purpose escape hatch that can set
  *any* Spark property, including the vast majority (like
  `spark.memory.fraction`, `spark.sql.adaptive.skewJoin.enabled`) that have
  no dedicated flag at all.
- **Precedence** (highest to lowest): values set in code via `SparkConf`/
  `SparkSession.builder.config(...)` > `--conf`/dedicated flags on the
  `spark-submit` command line > `spark-defaults.conf` on the cluster.
  Don't set the same property via both a dedicated flag and `--conf` in the
  same command — it's redundant and version-dependent which one wins.

## Task-level memory: how tasks actually draw from these pools

There is **no config that reserves a fixed slice of memory per task.**
Instead:

- **`spark.executor.cores`** sets how many task slots run concurrently in
  one executor (each task normally consumes `spark.task.cpus`, default `1`,
  core). This determines *how many* tasks compete for the execution memory
  pool simultaneously — it does not reserve memory per slot.
- **Execution memory is dynamically shared** among however many tasks are
  currently active. Spark's `ExecutionMemoryPool` guarantees each active
  task at least `1 / (2 × N)` of the pool (`N` = current active task count),
  but a task can grow beyond that share if others aren't using theirs, and
  must shrink or spill if contention increases. A per-task
  `TaskMemoryManager` allocates memory in pages and can be asked to release
  pages back under pressure.
- **Storage memory — including the broadcast copy — is not per-task at
  all.** It's one shared block per executor. Every task on that executor
  reads the *same* broadcast copy; there's no duplication per task, which
  is exactly why broadcasting once per executor (not once per task) is
  efficient.
- **When a task can't get enough execution memory,** Spark doesn't fail it
  outright — spillable structures (`ExternalSorter` for sort-based
  shuffles, `BytesToBytesMap` for hash aggregations) spill to local disk,
  trading I/O cost for staying within the pool's limits. A job with too
  many concurrent tasks per executor (high `spark.executor.cores` relative
  to `spark.executor.memory`) often shows heavy disk spill in the Spark UI
  rather than an outright OOM — the memory manager rationing at a
  performance cost, not crashing.

**The practical tension to hold in mind:** more `spark.executor.cores`
means more parallelism, but also more tasks dividing the *same* execution
memory pool — and a broadcast sitting in storage memory is a fixed tax on
that executor's total footprint regardless of how many tasks are running.
A "safe-looking" broadcast size can still hurt if cores are high enough
that execution memory is already thinly spread across many concurrent
tasks.
