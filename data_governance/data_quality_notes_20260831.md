# Why "It Didn't Crash" Isn't the Same as "It's Correct": Type Safety and Data Quality, Untangled

This is a follow up to https://github.com/rongali-naidu/blogs/blob/main/data_governance/what_exactly_is_data_quality.md . A lot of confusion around Python's type system, validation libraries, and data quality tools comes from mashing three separate questions into one. This post pulls them apart:

1. **When** is a type actually checked — before the program runs, or while it's running?
2. **What scope** is being validated — one record, or a whole batch/dataset?
3. **What should happen** when something's wrong — crash, or handle it gracefully?

## 1. Static typing vs. runtime validation

Python is dynamically typed. A type hint like `port: int` is an annotation for humans and tooling — the interpreter never checks it at runtime. You can pass a string into that field and Python won't complain.

**Mypy** closes part of this gap, but only statically. It reads your source code before execution and flags mismatches between your own functions and objects. Two important limits:

- It's opt-in and advisory. Code still runs even if mypy reports errors — mypy isn't wired into the interpreter the way a compiler is.
- It can't validate *external* data. A JSON payload, a database row, or user input has no type until it lands in your code. Mypy checks the contract between your own typed objects; it has nothing to check before that data arrives.

A common comparison: "Java does this automatically." It doesn't, not for external data. Java's compiler enforces types between your own objects, always, with no opt-out — that part is a real, structural advantage over Python. But data crossing a boundary (an API request, a file) still needs an explicit runtime step in Java too, typically a deserialization library like Jackson. The difference is scope, not runtime magic: Java guarantees the *internal* contract for free; neither language validates *external* data for free.

**Pydantic** is what fills that boundary gap in Python — it validates a value's shape and type at the moment an object is constructed, at runtime, and raises immediately if something doesn't match.

You can approximate Java's guarantee in Python by making mypy mandatory — a required pre-commit hook or CI gate — plus full annotation coverage (mypy's strict mode). At that point you have the same outcome, just assembled from opt-in tools rather than built into the language.

## 2. Per-record validation vs. batch data quality

**Pydantic** operates on one object at a time: does this record have the right fields, and does each field hold the right type? It's a natural fit at the point a single unit of data enters your system — an API request handler, or a loop processing one message at a time off a queue.

**Batch/pipeline data quality tools** (Great Expectations, Deequ/PyDeequ, and similar) operate across a whole dataset or batch. They answer questions no single-record check can: Is this column actually unique across the batch? What percentage of values are null? Has the distribution drifted from yesterday's run? These need to see many records at once.

This is also where Spark's own schema enforcement sits — and it's more limited than either of the above. A Spark schema checks shape and basic type only. It doesn't give you real uniqueness constraints or enforced not-null the way a database schema does; it's a structural description you can apply a fail-fast/permissive/drop-malformed mode to, not a rule engine. PyDeequ (and tools like it) exist specifically to add that missing layer — uniqueness, completeness thresholds, value ranges, drift detection over time — on top of what Spark's schema alone gives you.

**Rule of thumb:** if your "contract" is genuinely just shape and type, Spark's built-in schema handling covers it. Once the contract includes uniqueness, completeness, or statistical health, that's a different category of check, and it's what dedicated data quality tools are for.

## 3. Crashing vs. handling gracefully — and why it matters more in one context than the other

Catching an error doesn't change *whether* something failed — it changes what you're left with afterward. An uncaught exception propagates until the program stops. A caught one lets you log the record, route it somewhere (a dead-letter queue, an S3 bucket), and keep the rest of the system running.

**Why this matters more for application services than data pipelines:** a failed pipeline batch can usually just be logged and rerun — nobody's waiting in real time, so the cost is mostly delay. A crashed application service, by contrast, is breaking a live request or cascading into other systems depending on it right now. That's why runtime validation and graceful handling get so much more attention in service architectures than in batch pipelines.

One more thing worth being explicit about: **letting mismatched data flow through unchecked is not the same as resilience.** Python not crashing on a bad type doesn't mean the bad value gets skipped — it means execution continues *with* the bad value, which can silently corrupt a downstream calculation or a database write, or surface as a confusing failure far from where the bad data actually entered. Not validating doesn't prevent failure; it just relocates and disguises it. That's the actual case for runtime validation — not that crashes are bad in themselves, but that an unhandled bad value produces a worse failure than a caught one.

## Summary table

| | Scope | When it runs | What it catches |
|---|---|---|---|
| Type hints | One codebase | Never (annotations only) | Nothing on its own |
| Mypy | Your own code | Before execution (static) | Mismatches between your typed objects — if enforced as a gate |
| Pydantic | One record | At object construction (runtime) | Bad shape/type in a single record, especially at system boundaries |
| Spark schema | One batch | At DataFrame creation (runtime) | Structural shape/type mismatches, per configured mode |
| PyDeequ / data quality tools | Whole dataset | After batch is loaded (runtime) | Uniqueness, completeness, ranges, drift — statistical/cross-record issues |

The throughline: static checking, per-record validation, and batch-level data quality are three different tools answering three different questions. None of them is a substitute for the others, and "the program kept running" is only a good outcome if what kept running was trustworthy.
