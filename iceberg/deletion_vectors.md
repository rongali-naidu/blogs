# Iceberg v2 vs v3: "Deletion Vectors" (aka "Deletion Bitmap")

Today i had to know a bit more about Iceberg v3 especially around Deletion vectors . Here is my quick nites.
Apache Iceberg tables carry a **format version** — `1`, `2`, or `3` — that decides which on-disk features are allowed. v3 was finalized in 2025 and its headline change is how row-level deletes are stored. This post covers two things:

1. The **main difference between v2 and v3** 
2. What **"vector"** actually means here — and why this thing is just the **bitmap** you already know.

---

## 1. The main difference: many delete files (v2) → one bitmap per data file (v3)

Iceberg never edits data files in place — the Parquet files sitting in object storage are **immutable**. So "deleting a row" can't mean physically removing it from the file. Instead, the deletion is **recorded elsewhere**, and the query engine skips the row at read time. The real (later) removal happens during **compaction**, when the file is rewritten.

The question is *how* you record "this row is deleted" That's exactly what changed between v2 and v3.

### v2 — merge-on-read via separate *delete files*

*Note: This is not relevant for CoW .

In v2, each delete is written into a **separate small file** that says "in data file X, the row at position N is deleted." Do a lot of updates/deletes, and these little files pile up. At read time, the engine has to load **all** of them and merge them against the data — the classic **small-file problem**.

### v3 — one **deletion bitmap** per data file

In v3, each data file gets **one compact bitmap** marking which of its rows are dead. No scattered files — the "which rows are deleted" info travels as a single per-file structure, and new deletes update that bitmap instead of dropping yet another file.

```
   v2:  data file X  +  delete-file-1 (X,3)              <- many small
                        delete-file-2 (X,6)                 delete files
                        delete-file-7 (X,9) ...

   v3:  data file X  +  ONE bitmap for X:  00010010       <- one per data file
                                    ^        ^
                              rows 3 and 6 are deleted
```

Two details worth keeping precise:

- **Only files that actually have deletes get a bitmap.** A clean file has none — zero overhead.
- **At most one active bitmap per data file.** New deletes supersede the previous bitmap (a new snapshot points to the new version) rather than accumulating more files — so v2's small-file sprawl doesn't come back.

### Why it's better

| | v2 (position delete files) | v3 (deletion bitmap) |
|---|---|---|
| Storage of deletes | Many small files | One bitmap per data file |
| Read path | Load + merge many delete files | Load one bitmap, skip flagged rows |
| Small-file sprawl | Yes | No |
| Footprint | Grows with every delete | Compact (Roaring bitmap) |

At read time the engine just loads file X's bitmap, and for each row checks the bit: `1` -> skip, `0` -> keep. Simple and fast.

> v3 also adds more beyond deletes — the `variant` (semi-structured/JSON) type, `geometry`/`geography` types, nanosecond timestamps, default column values, and row lineage (`_row_id`) for CDC. But the deletion-bitmap change is the one people mean first when they say "v3."

**Adoption caveat:** v2 is supported essentially everywhere. v3 support is still rolling out unevenly across engines (Spark, Trino, Athena, Redshift, Snowflake's Iceberg support, etc.), so only write v3 tables once **every** engine that reads them understands v3.

---

## 2. "Vector" here just means bitmap — not coordinates

The name **"deletion vector"** trips people up, because in math/physics a **vector** means a point or direction in n-dimensional space — an n-tuple of coordinates (x, y, z, …). This has **nothing** to do with that.

The precise relationship:

- A **bitmap is a one-dimensional vector whose values are restricted to {0, 1}.**
- So a bitmap is a subset of vector — a bitmap is the degenerate, 1-D, binary case of a vector.

Calling it a "vector" is therefore *technically* valid (it is a 1-D array) but needlessly abstract: the honest name is **"deletion bitmap."** It's exactly the bitmap you're already familiar with — a flat row of bits, one per row of data:

```
 row position:  0   1   2   3   4   5   6   7
 bitmap:        0   0   0   1   0   0   1   0
                            ^           ^
                       row 3 deleted   row 6 deleted
```



## Spec references

- **Apache Iceberg Table Spec (current, covers v1 / v2 / v3):** https://iceberg.apache.org/spec/
  - Row-level deletes / delete files (v2): see the "Row-level Deletes" and "Delete Formats" sections of the spec.
  - Deletion vectors, row lineage, and new types (v3): see the "Deletion Vectors" and "Format version 3" sections of the same spec page.
- **Iceberg releases / format-version overview:** https://iceberg.apache.org/releases/
- **Puffin spec (the file format that stores deletion vectors as blobs):** https://iceberg.apache.org/puffin-spec/
- **Roaring bitmap (the compression used by deletion vectors):** https://roaringbitmap.org/

