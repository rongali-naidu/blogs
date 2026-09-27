# Data Engineering Interview Prep — Resource List

## Coding (Python / DSA patterns)
- **[LeetCode](https://leetcode.com/)** — the standard for coding rounds; filter by "Top Interview Questions" and practice sliding window, two-pointer, hash map patterns specifically (most DE coding rounds pull from these categories, not hard graph/DP problems)
- **[DataDriven.io](https://datadriven.io/)** — DE-flavored Python + SQL problems with "common trap" explanations per question
- **[datadriven-io/data-engineering-interview-questions (GitHub)](https://github.com/datadriven-io/data-engineering-interview-questions)** — 1400+ DE interview questions (SQL, Python, schema design, pipeline architecture), each with a runnable browser sandbox and the common trap called out
- **[datadriven-io/data-engineering-interview-handbook (GitHub)](https://github.com/datadriven-io/data-engineering-interview-handbook)** — free full handbook covering SQL, Python, schema design, pipeline architecture, system design, and behavioral rounds, with structured study plans
- **[datadriven-io/awesome-data-engineering-interviews (GitHub)](https://github.com/datadriven-io/awesome-data-engineering-interviews)** — "The DataDriven 75," a curated shortlist of 75 real DE interview questions if the 1400+ set feels like too much to triage yourself

## SQL
- **[StrataScratch](https://www.stratascratch.com/)** — real interview questions sourced from actual companies (Amazon, Meta, etc.), strong SQL + Python mix, filterable by company
- **[DataLemur](https://datalemur.com/)** — SQL-focused, good "explain the query pattern" style breakdowns, free tier is solid
- **[Mode SQL Tutorial](https://mode.com/sql-tutorial/)** — free, good for brushing up window functions (RANK, LAG/LEAD, PARTITION BY) which come up constantly in DE SQL rounds

- # Blogs with Frequently-Asked SQL Interview Question Lists

## Data-Engineering-specific (best fit for your role)
- **[929 SQL Interview Questions for Data Engineers | DataDriven](https://datadriven.io/sql-interview-questions)** — large, DE-scoped question bank, same site as the Python list already on your radar
- **[SQL Interview Questions for Data Engineers: 30 Real Questions with Solutions | PipeCode](https://pipecode.ai/blogs/sql-interview-questions-for-data-engineers)** — smaller curated set, worked solutions rather than just Q&A
- **[80 SQL Interview Questions for Data Engineers | DataVidhya](https://datavidhya.com/blog/sql-data-engineering-interview-questions/)** — framed as "real asks," DE-specific rather than generic SQL trivia

## Window Functions specifically (this is the recurring weak spot in most DE interviews)
- **[12 SQL Window Functions Interview Questions | DataLemur](https://datalemur.com/blog/sql-window-functions-interview-questions)** — short, sharply focused, good if you only have 20 minutes
- **[SQL Window Functions Interview Questions | StrataScratch](https://www.stratascratch.com/blog/sql-window-functions-interview-questions)** — same platform as their practice problems, good follow-through from reading into doing
- **[Top 10 SQL Window Functions Interview Questions | LearnSQL.com](https://learnsql.com/blog/sql-window-functions-interview-questions/)** — clean explanations, good if RANK/LAG/LEAD/PARTITION BY isn't fully reflexive yet

## Broader / general SQL question banks
- **[Top 99 SQL Interview Questions and Answers | DataCamp](https://www.datacamp.com/blog/top-sql-interview-questions-and-answers-for-beginners-and-intermediate-practitioners)** — long, comprehensive, good as a single-pass review rather than deep practice
- **[Top 27 Advanced SQL Interview Questions | LearnSQL.com](https://learnsql.com/blog/advanced-sql-interview-questions/)** — skips the basics, goes straight to the harder end (subqueries, CTEs, advanced joins) — useful given your seniority level
- **[60+ Most Important SQL Interview Questions | InterviewBit](https://www.interviewbit.com/sql-interview-questions/)** — broad general reference, decent as a checklist to skim for gaps

## How I'd actually use this list given your timeline
1. Skim **DataLemur's window functions post** first — highest-frequency topic in DE SQL rounds and the fastest read
2. Do a pass through **DataDriven's 929-question bank**, filtering mentally for anything Iceberg/CDC/dedup-adjacent, since that maps directly to what you've already been asked
3. Only reach for the **LearnSQL advanced list** if you have spare time — it's the "sharpen further" tier, not the "cover the basics" tier

## Spark / Big Data Systems
- **[Apache Spark official docs — RDD & DataFrame programming guide](https://spark.apache.org/docs/latest/rdd-programming-guide.html)** — primary source, good for verifying exact API behavior (e.g., confirming `writeTo` vs `.write` details)
- **[Databricks Blog](https://www.databricks.com/blog)** — frequent deep-dives on skew handling, Adaptive Query Execution, Delta Lake internals — written by the people who built Spark
- **[Iceberg official docs](https://iceberg.apache.org/docs/latest/)** — good for firming up manifest files / snapshot / copy-on-write vs merge-on-read terminology precisely, straight from source

## AWS-Specific (given your ProServe target)
- **[AWS Big Data Blog](https://aws.amazon.com/blogs/big-data/)** — you've already published here; keep skimming recent posts on Glue, Lake Formation, and Iceberg-on-Glue patterns, since interviewers at AWS often reference recent blog content
- **[AWS re:Post](https://repost.aws/)** — good for realistic troubleshooting scenarios (matches your CDC/DMS/Glue experience) framed as Q&A
- **[AWS Well-Architected Framework — Data Analytics Lens](https://docs.aws.amazon.com/wellarchitected/latest/analytics-lens/analytics-lens.html)** — useful for system-design-style questions where they ask you to justify architecture tradeoffs

## System Design for Data Engineering
- **["Designing Data-Intensive Applications" by Martin Kleppmann](https://dataintensive.net/)** — the standard reference book; strong on CDC, partitioning, consistency tradeoffs — all things you already speak to well, this sharpens the vocabulary
- **[Exponent](https://www.tryexponent.com/questions?role=data-engineer)** — has a data-engineering-specific interview question bank with some video walkthroughs

## Behavioral / Consulting-Specific (since this is a ProServe role)
- **[Amazon's Leadership Principles page](https://www.amazon.jobs/en/principles)** — worth re-reading before any Amazon-adjacent interview (including AWS) since STAR answers are often implicitly graded against these
- Practice explicitly framing 1–2 of your existing stories (data mesh influence, ownership advocacy) around "adapting fast in an unfamiliar domain" — that was the one gap flagged in your last interview's review, and it's the single most-tested consulting-specific competency

## Quick prioritization given your timeline
1. **StrataScratch or DataLemur** for SQL drilling — highest ROI for DE-specific rounds
2. **[datadriven-io GitHub repo](https://github.com/datadriven-io/data-engineering-interview-questions)** or **LeetCode** — 5–10 sliding window / hash map problems to lock in the pattern we just covered
3. **Iceberg docs** — 15 minutes skimming terminology precision (manifest lists vs manifest files, exact snapshot mechanics)
4. One behavioral story reworked to address "unfamiliar domain adaptability" specifically
