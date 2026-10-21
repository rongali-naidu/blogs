---
title: "From Disk Pages to Memory Heap Objects: Why Apache Arrow is Required"
date: 2026-10-18
tags: [data-architecture, arrow]
publish: true
---

# From Disk Pages to Memory Heap Objects: Why Apache Arrow is Required

In modern data engineering, two open-source technologies sit at the heart of almost every analytical pipeline: [Apache Parquet](https://parquet.apache.org/), the de facto standard for storing columnar data efficiently on disk, and [Apache Arrow](https://arrow.apache.org/), the standard for representing that data in memory for high-performance analytics. (Table-format layers like [Apache Iceberg](https://iceberg.apache.org/) sit *on top of* Parquet files to manage schemas, transactions, and partitioning at scale — but the foundational split between disk storage and in-memory compute stays the same, so we'll set Iceberg aside for this exploration.)

You can already read a Parquet file in one line:

```python
import pandas as pd
df = pd.read_parquet("employees.parquet")

```

So a natural question is: **if a library can turn a file into usable data in one line — the way it does for CSV — why does anything called "Apache Arrow" need to exist?**

The official [Apache Arrow FAQ](https://arrow.apache.org/faq/) offers a gist of this: *"Parquet is how columnar data sits on disk; Arrow is how it sits in memory."*

Here, I will expand it using the low-level mechanisms to answer *why* in-memory needs its own standard at all in **three steps**:

* **Part 1 — Row formats & database engines.** How a row is stored and read when a real database engine owns the data.
* **Part 2 — CSV read by an application.** What happens when your program reads a plain file directly, with no engine in the middle.
* **Part 3 — Parquet & Arrow.** What happens when Python reads Parquet *like a CSV*, why that falls apart at scale, and what Arrow is actually for.

---

> **TL;DR**
> * Programs hold data as **objects joined by pointers**; storage systems hold it as **flat bytes found by offset arithmetic**. Pointers can't be persisted or shared; offsets can.
> * A database **engine** decodes its on-disk pages into a private in-memory area (the **buffer pool**) and computes over it. A plain **CSV** skips all of that and parses straight into heap objects — fine because it's small and untyped.
> * **Parquet** has an efficient, typed, *columnar* layout like a database page — but no engine owns it, and traditionally, every tool decoded it into its own custom memory layout (heap objects, NumPy arrays, etc.). **Arrow is the shared, standard in-memory columnar layout** that lets any tool decode once and pass columns around with zero copying. That's why we need Arrow.
> 
> 


# Part 1 — How row formats and database engines work

## The two worlds

There are two separate universes, each with its **own memory**. They never share memory directly; they talk through a **driver**.

```
   APPLICATION WORLD                         DATABASE WORLD
   (language heap)                           (engine: its own memory + disk)
   objects, pointers          ◄──driver──►   byte pages, offsets, catalog
```

Most confusion comes from using the word "memory" for both. They are not the same memory — often not even the same machine.

Our example row throughout the whole blog:

```
id = 1,  name = "John",  age = 42,  city = NULL
```

```sql
CREATE TABLE employees (
  id    INT,        -- fixed width, 4 bytes
  name  VARCHAR,    -- variable width
  age   INT,        -- fixed width, 4 bytes
  city  VARCHAR     -- variable width (NULL in our row)
);
```

## 1.1 The application view: objects and pointers

In your program the row is an object. On the language **heap**, the pieces are **scattered**, stitched together by **pointers** — saved memory addresses the object follows to reach each field:

```
APPLICATION HEAP
  row object @ 0x7A00
   ├─ id   → 0x9120 → [ 1 ]
   ├─ name → 0x9480 → [ "John" ]
   ├─ age  → 0x9640 → [ 42 ]
   └─ city → null
```

A **pointer** means *"the real data is over there at address X; go fetch it."* Flexible for programming, but impossible to persist: `0x9480` is valid only inside this one running program, right now. Write it to disk → garbage. So before storage, the object must be **serialized** into a flat, position-based form.

## 1.2 Writing to the database: the driver flattens the object

Your app inserts through a **driver** (JDBC, psycopg, the MySQL connector, or the cross-language **ODBC** standard):

```python
cursor.execute(
    "INSERT INTO employees (id, name, age, city) VALUES (%s,%s,%s,%s)",
    (1, "John", 42, None)
)
```

The driver **serializes** your heap object into the database's **wire protocol** — the agreed byte format a client and a database use to exchange data — producing a flat byte stream it ships to the engine (over TCP for a client-server database like PostgreSQL or MySQL; through a direct in-process call for an **embedded** engine like SQLite or DuckDB, which skips the network entirely). Either way the destination is the same: the database's internal byte format. From here on there are no pointers and no objects — only bytes and the rules to interpret them.

## 1.3 How the database stores the row on disk

Data is written in fixed-size **pages** (commonly 8 KB). A page is not just rows jammed together — it has an internal directory:

```
PAGE (8 KB) — fills from BOTH ends toward the middle
┌───────────────────────────────────────────────────────────────┐
│ PAGE HEADER (~24 bytes): free-space markers, checksum, etc.     │
├───────────────────────────────────────────────────────────────┤
│ SLOT ARRAY (line pointers) — grows DOWNWARD ↓                   │
│   slot[0] = (offset=8160, length=32)   ── points to Row 0       │
│   slot[1] = (offset=8128, length=32)   ── points to Row 1       │
│                                                                 │
│                   ← free space in the middle →                  │
│                                                                 │
│   Row 1 bytes @ 8128 ...                        grows UPWARD ↑   │
│   Row 0 bytes @ 8160 [hdr|nullbmp|id|name-mkr|age|var-data]     │
└───────────────────────────────────────────────────────────────┘
```

**Two levels of "where is it", both arithmetic, zero pointers:**

**Which row?** The **slot array** near the top of the page. `slot[0]` = "Row 0 starts at byte 8160, length 32." To find row *N*, read `slot[N]`. If a row moves inside the page, only its slot entry changes; indexes reference a row as **(page number, slot number)**, never a raw memory address.

**Which field inside the row?** The row's byte layout (offsets relative to row start):

```
offset:  0          4             8         12            16          20
         ┌──────────┬─────────────┬─────────┬─────────────┬───────────┬──────────┐
         │ row hdr  │ null bitmap │  id = 1 │ name marker │  age = 42 │ var-data │
         │ 4 bytes  │  4 bytes    │ 4 bytes │ (off=20,    │ 4 bytes   │ "John"   │
         │          │  i n a c    │         │  len=4)     │ (0x2A)    │          │
         └──────────┴─────────────┴─────────┴─────────────┴───────────┴──────────┘
```

- **Null bitmap** (`i n a c`, one bit per column) — `city` is NULL, so its bit is set and **city occupies zero data bytes**. A NULL costs **1 bit**.
- **Fixed-width** fields (INT) sit inline at known offsets.
- **Variable-width** fields (VARCHAR) store a small **(offset, length) marker**; the characters live in the variable area at the end. That marker is relative to the row, so it survives disk and network — unlike a pointer.

**How is an offset like 16 computed?** By summing the widths of preceding columns:

```
preamble: header (4) + null bitmap (4)   → fields start at offset 8
id   : INT fixed 4B                      → offset 8
name : VARCHAR → fixed 4B marker         → offset 8  + 4 = 12
age  : INT fixed 4B                      → offset 12 + 4 = 16
```

But the data page carries **no column names or widths** — so where does that knowledge come from?

## 1.4 Running SQL: catalog + buffer pool + arithmetic

```sql
SELECT name FROM employees WHERE age > 40;
```

**Planning.** The raw bytes are unlabeled. Column names, types, widths, and order live in the **system catalog** — the database's own internal tables (PostgreSQL's `pg_class`/`pg_attribute`, etc.). The planner looks up `employees`, computes each column's offset **once**, and builds a **tuple descriptor** ("age → offset 16, INT"). The catalog is itself cached in RAM, so this is nearly free.

**Loading.** The engine copies the whole 8 KB page off disk into its own RAM — the **buffer pool** — **byte-for-byte unchanged**:

```
DB DISK                          DB MEMORY (buffer pool)
┌───────────────┐  read page   ┌───────────────────────────────┐
│ Page (8 KB)   │ ───────────► │ Page (8 KB): identical bytes   │
│ header+slots+ │              │ header + slot array + rows,    │
│ rows          │              │ now at RAM address 0x5000      │
└───────────────┘              └───────────────────────────────┘
```

Nothing becomes an object. This buffer pool is the database's memory, completely separate from your application heap.

**Execution.** For each row the executor fuses the two sources of truth:

| Question | Answered by | Where |
|---|---|---|
| Where does **row N** start? | the **slot array** | inside the page |
| Where does field **age** sit, and its type? | the **tuple descriptor** | from the **catalog** (cached) |
| Is the field **NULL**? | the **null bitmap** | inside the row |

```
1. row_start = page_base(0x5000) + slot[0].offset(8160)        = 0x6FA0
2. age_addr  = row_start(0x6FA0) + age_offset(16, from catalog) = 0x6FB0
3. read 4 bytes → 42 ; 42 > 40 → qualifies ; read name → "John"
```

Pure arithmetic, no pointer-chasing. The descriptor is built once and reused for millions of rows.

## 1.5 Reading back through ODBC/JDBC

The result leaves as a flat byte stream. On the client, the **driver** does the reverse of 1.2 — it **deserializes** the bytes and rebuilds native objects on the application heap:

```
buffer-pool bytes → executor tuple ["John"] → wire → DRIVER → heap object {"name":"John"}
                                                                   (pointers are BACK)
```

The database's buffer pool and the client's object memory are two different memories, bridged only by the driver.

**Part 1 takeaway:** a database keeps data as flat **byte pages** (on disk *and* in the buffer pool) and reads fields by **computing offsets**, guided by the **catalog**. Objects with pointers exist only at the very ends — in your application.

---

# Part 2 — What happens when an application reads a CSV

Now remove the entire database world. No engine, no pages, no slots, no catalog, no driver. Just your program reading a text file into its own heap.

## 2.1 A CSV is just text

```
employees.csv  (raw bytes on disk)

i d , n a m e , a g e , c i t y \n
1 , J o h n , 4 2 , \n
2 , J a n e , 3 9 , L A \n
```

- `42` is stored as the **characters** `'4''2'`, not a 4-byte integer.
- There are **no types, no widths, no null bitmap**. The empty `city` is just "nothing between two commas" — an empty string, not NULL.
- A field is found by **scanning for the next comma**, not by computing an offset.

## 2.2 Reading it lands straight in heap objects

```python
import csv
rows = list(csv.DictReader(open("employees.csv")))
```

```
APPLICATION HEAP (after parsing)
  rows → list @ 0x4000
          ├─ [0] → dict @ 0x7A00
          │         ├─ "id"   → 0x9120 → "1"    (still a STRING)
          │         ├─ "name" → 0x9480 → "John"
          │         ├─ "age"  → 0x9640 → "42"   (still a STRING)
          │         └─ "city" → 0x9700 → ""     (empty string, not NULL)
          └─ [1] → dict @ 0x7C00 → ...
```

This is exactly the **object-with-pointers** picture from Part 1.1. CSV reading drops you *directly* into the application world — no engine, no intermediate memory format.

## 2.3 Why CSV never needed anything like Arrow

Two reasons:

1. **It's trivial.** Split on commas, make strings. The output is ordinary language objects and that's the end. There's nothing to standardize or share.
2. **You usually read a CSV into one tool and stay there.** It's small, so nobody cares about the cost.

The catch: **types are now your problem.** `age` is the string `"42"`; `rows[0]["age"] + 1` is an error until *you* convert it. The job the catalog + engine did in Part 1 (splitting fields, assigning types) now falls on you or on a library's guesswork.

**Part 2 takeaway:** for small, simple data, "read file → heap objects" is fine. CSV proves you don't always need a fancy in-memory format. The trouble starts when data is **big, typed, and shared across tools** — which is exactly Parquet's world.

---

# Part 3 — Parquet, and finally, Arrow

## 3.1 Parquet has a database-like layout — but no engine

Parquet is binary, **self-describing**, and **columnar**:

```
Parquet file (conceptually)
  id   column chunk:   [1, 2, 3]           ┐ each column stored together,
  name column chunk:   ["John","Jane","Al"] │ dictionary/RLE-encoded + compressed
  age  column chunk:   [42, 39, 51]        ┘
  footer: schema (id INT, name STRING, age INT...), row counts, min/max stats
```

Compare the three formats:

| | CSV | Database page | Parquet |
|---|---|---|---|
| Layout | text rows | binary rows | binary **columns** |
| Types | none | in the catalog | **embedded in the footer** |
| Encoded/compressed | no | lightly | **heavily** |
| Who reads it | your CSV lib | the DB **engine** | ??? (no engine owns it) |

Parquet is like the database page — typed, offset-addressed, efficient — but it's **just a file**. There is **no engine and no buffer pool** that owns it. And its columns are heavily encoded, so **you cannot compute on Parquet bytes directly; they must be decoded first.** That word *decoded* is the whole reason Arrow exists.

## 3.2 First, let's read Parquet "like a CSV" in Python — without Arrow

Pretend neither `Apache Arrow` nor `fastparquet` exists  — the libraries that actually do the work behind the scenes when we call `pd.read_parquet` . We will decode Parquet straight into Python heap objects, CSV-style:

```python
# conceptual: Parquet → Python objects, no Arrow
rows = parquet_decode_to_dicts("employees.parquet")
```

What that memory looks like:

```
APPLICATION HEAP (Parquet decoded into objects, the naive way)

  rows → list @ 0x4000
   ├─ [0] → dict @ 0x7A00
   │         ├─ "id"   → 0x9120 → int(1)
   │         ├─ "name" → 0x9480 → str("John")
   │         ├─ "age"  → 0x9640 → int(42)
   │         └─ "city" → None
   ├─ [1] → dict @ 0x7C00 → ...
   └─ [2] → dict @ 0x7E00 → ...
```

Now look at what we did to a *columnar* file:

- Parquet stored `age` as a tight, contiguous column `[42, 39, 51]`. We **exploded it back into per-row objects**, scattering each value across the heap behind pointers.
- A Python `int` object is ~28 bytes with overhead; a dict per row adds more. For millions of rows this is **enormous and pointer-chasing** — the opposite of cache-friendly.
- To do `sum(age)`, the CPU must hop pointer→pointer across the heap. No contiguous scan, no SIMD.
- And this decoded pile is **private to Python**. Hand it to Spark or DuckDB and everything must be re-serialized and copied.

We threw away the single best property of Parquet — its tight columnar layout — the instant we decoded it into objects. **That** is the problem.

## 3.3 Enter Arrow: a standard *in-memory columnar* layout

Arrow defines **one standard way to lay out decoded columnar data in RAM**: a contiguous, typed buffer per column plus a null bitmap — essentially the same shape Parquet has on disk, but **raw (uncompressed) and ready to compute on**.

```
Parquet  = standard ON-DISK  columnar format   (compressed, for storage)
Arrow    = standard IN-MEMORY columnar format  (raw, for computing)
```

Decoding Parquet into Arrow keeps the column a column:

```
ARROW in memory (age column)
  values buffer : [ 42 | 39 | 51 | ... ]   contiguous int32, back-to-back
  null  bitmap  : [ 1 1 1 ... ]            1 bit per value

  name column
  values buffer : J o h n J a n e A l      all chars packed together
  offsets buffer: [0, 4, 8, 10]            where each string starts
```

Finding `age[2]` = `base + 2*4` — offset arithmetic, exactly like the database engine, no per-value pointers. Strings use a values+offsets pair — the same "offset, not pointer" trick from Part 1. And because the column is contiguous, `sum(age)` runs at near hardware speed — the next section explains *why*.

## 3.3a Why a contiguous column is fast: cache lines + SIMD

It's easy to say "columnar is faster," but it's worth seeing *why* at the hardware level, because that is the real reason Arrow's layout matters. Two facts about how a CPU works.

**Fact 1 — The CPU reads memory in chunks, not one value at a time.**
When the CPU touches memory it doesn't fetch a single number — it pulls a whole **cache line** (typically 64 bytes) into fast on-chip cache. So one fetch of an `int32` column brings in **16 values at once** (64 ÷ 4), effectively for free:

```
one 64-byte cache-line fetch:
[ 42 | 39 | 51 | 18 | 27 | 60 | 33 | ... ]   ← 16 ages arrive together
```

If the ages are **scattered** across the heap as separate objects (the naive Parquet-to-objects case in 3.2), each one lands on a different cache line behind its own object header. Summing them then costs 16 separate fetches instead of 1, and most of each fetched line is wasted on object overhead rather than data. That gap alone is often 10x+ — before any arithmetic happens.

**Fact 2 — One CPU instruction can do math on many values at once (SIMD).**
SIMD stands for **Single Instruction, Multiple Data**. Modern CPUs have wide registers (128 / 256 / 512 bits) that hold several values side by side, plus instructions that operate on all of them in a **single step**:

```
Scalar (one at a time):            SIMD (eight at a time):
  total += age[0]                    load [a0 a1 a2 a3 a4 a5 a6 a7]
  total += age[1]                    add  [b0 b1 b2 b3 b4 b5 b6 b7]
  total += age[2]        vs.         → one ADD instruction yields
  ... one add per value                8 results at once
```

An 8-wide SIMD add does the work of eight scalar adds in roughly the time of one.

**The catch: SIMD only works on values that are contiguous and uniform.** The hardware fills a wide register by grabbing *a run of adjacent bytes of the same type*. A tight `int32` column is exactly that, so the engine can feed it straight into SIMD instructions. A pile of pointer-linked objects is not: the values aren't adjacent, aren't uniformly sized, and each needs its pointer followed first — so SIMD can't apply at all.

**So "columnar in memory" unlocks both wins together:**

```
age as a contiguous Arrow buffer:   [42 39 51 18 27 60 ...]
   ✔ one cache-line fetch brings ~16 values
   ✔ one SIMD instruction adds ~8 at a time
   → sum(age) streams through memory at near hardware speed

age as scattered heap objects:      obj→42   obj→39   obj→51 ...
   ✗ a cache miss per value
   ✗ a pointer chase per value
   ✗ SIMD impossible
   → orders of magnitude slower
```

That is the concrete payoff of Arrow's layout: it isn't merely "columnar is tidy" — a contiguous, typed column is the *only* shape the CPU's cache and SIMD units can run at full speed. It's the same offset-addressed, no-pointers principle the database engine used in Part 1, now put to work for raw compute throughput.

## 3.3b Arrow is the buffer pool for the file-based world

Notice Arrow is, in effect, the **buffer pool for the file-based world**:

```
Part 1 (database):  DB page (disk) → engine BUFFER POOL (decoded bytes) → driver → objects
Part 3 (files):     Parquet (disk) → ARROW arrays (decoded bytes)       → objects only if needed
```

The difference: a database's buffer pool is **private to one engine**. Arrow is **deliberately shared** across many engines and languages.

## 3.4 How Spark, pandas, and DuckDB handled this *before* Arrow

Before Arrow, every tool decoded Parquet into its **own private** in-memory layout:

```
Parquet file
   ├─ pandas  decodes → NumPy arrays     (pandas' private layout)
   ├─ Spark   decodes → Tungsten rows    (Spark's private layout)
   ├─ DuckDB  decodes → DuckDB vectors    (its private layout)
   └─ Java app decodes → Java objects     (its private layout)
```

Two costs:

1. **Everyone re-implements** Parquet→memory decoding, slightly differently.
2. **Nobody can share decoded data.** Moving data between tools — the classic **Spark (JVM) → Python** hop — meant **serialize → copy → deserialize** across the boundary. On big data that "conversion tax" often dominated runtime.

## 3.5 How they work *with* Arrow

If pandas, Spark, DuckDB, and your app all hold decoded data in the **Arrow** layout, the picture collapses to:

```
Parquet file (disk, encoded)
      │  decode ONCE
      ▼
Arrow arrays in memory  (age = [42,39,51...] raw int32 + null bitmap)
      │   shared with ZERO COPY (everyone reads the same buffers)
      ├──────────► pandas    (pandas 2.x can back DataFrames by Arrow)
      ├──────────► Spark     (Arrow-based PySpark transfer; Pandas UDFs)
      ├──────────► DuckDB    (queries Arrow buffers in place, no copy)
      └──────────► your app
```

- **Decode Parquet → Arrow once**; every Arrow-aware tool uses the same bytes.
- **Spark ↔ Python** can hand over Arrow batches instead of serializing row by row; vectorized operations become possible because the data is already in a contiguous, typed layout.
- **DuckDB** can run SQL directly over Arrow buffers your Python already holds — **zero copy**.
- Moving data across the network uses **Arrow Flight**, skipping row-wise serialization entirely.

Heap objects (the Part 2 CSV-style dicts) are created only at the very end, if you actually need per-row Python/Java objects — not as the working representation for analytics.


