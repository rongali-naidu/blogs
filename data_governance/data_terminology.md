# How to Talk About Data Without the Jargon
*A one-line cheat sheet — from Metadata to Ontology.*
A single, non-overlapping set of terms for how meaning gets attached to data — and how to decide where anything belongs. *(▲ = umbrella term: names a collection, not a single item.)*
## Core list — one-liner + example
| Term | One-liner | Example |
|---|---|---|
| ▲ **Metadata** | *Data about data* — any descriptive fact attached to data. | See the three buckets below. |
| ▲ **Semantics** | The *meaning* the data carries. | Knowing `US` **means** *United States*, not just a 2-letter string. |
| ▲ **Semantic layer** | The *active translation engine* between BI tools/LLMs and the database. | Turns "revenue by country" into SQL over physical tables in real time. |
| ▲ **Vocabulary** *(Business / Data)* | The whole *set of agreed words* — business Terms and data values. | *Customer*, *Order*, *Country*, `United States`… |
| **Glossary** | The governed *flat list of Terms* + definitions — *what does it mean?* | Approved definitions of *Customer*, *Order*, *Country*. |
| **Taxonomy** | Terms on a *single-axis tree* (`is-a` or `part-of`) — *how is it categorized?* | `Savings Account` is-a `Account`. |
| **Ontology** | Terms + *rich, cross-domain relationships & logic* — *how does it interact?* | `Customer` places `Order`; `Order` ships-to `Country`. |
| **Data dictionary** | *Passive documentation* mapping meanings to columns/fields. | `order.country_code` = "destination country", CHAR(2). |
| **Term** | The canonical *name of a concept*. | `Country` — "a sovereign nation with a recognized code." |
| **Synonym** | Another *word* for the same Term. | "nation" → *Country* |
| **Acronym** | Initials of a Term's own words. | "SKU" → *Stock Keeping Unit* |
| **Reference data** | The *set of allowed values* for an attribute. | `Country` ∈ { United States, Canada, Mexico, … } |
| **Canonical value** | The *one approved form* of a value. | `United States` |
| **Variant** | A *messy alternate* that maps to a canonical value. | "USA", "U.S.", "US" → `United States` |
| **Data domain** | A *subject area* grouping related Terms/data. | *Sales* domain holds `Customer`, `Order`, `Invoice`. |
| **Business rule** | A *constraint* that must always hold. | "Every Order must have a shipping Country." |
| **Business process** | A *workflow* — steps in sequence. | Signup: register → verify → activate → welcome. |
| **Business application** | The *system* that owns/runs the data. | The order-management system owns `Order`. |
## Metadata comes in three buckets
| Bucket | Describes | Example |
|---|---|---|
| **Technical** | The data's *structure*. | `VARCHAR(32)`, index details, table schema. |
| **Business** | The data's *meaning & ownership*. | "This column means Country", steward, business rules. |
| **Operational / Administrative** | The data's *runtime facts*. | Lineage, last-run time, row count, access permissions. |
## Key nuances
**Taxonomy is strictly single-axis.** It handles exactly one relationship type per tree — either `is-a` (inheritance) or `part-of` (composition). The moment you need *multiple* relationship types crossing domains (`places`, `ships-to`, `owns`), you've left taxonomy and entered **ontology**. That single-axis limit is the cleanest line between the two.

**Semantic layer (active) vs. Data dictionary (passive).** Both connect business terms to physical data, but a **data dictionary** is *documentation you read*; a **semantic layer** is an *engine that runs* — it sits live between your BI tools / LLMs and the database, translating business queries into physical SQL on the fly.

**Canonical vs. variants.** The same "one blessed form + its alternates" pattern repeats on two layers — for *words* (Term ← synonyms/acronyms, for humans to understand) and for *values* (canonical value ← variants, for pipelines to normalize). This value-cleansing is where the bulk of pipeline effort actually goes.

## The architecture — how all concepts fit together
```
BUSINESS CONTEXT  (the "Where & Who")
  └── Data Domain (e.g., Sales)
        ├── Business Process (e.g., Signup)
        └── Business Application (e.g., Order System)
KNOWLEDGE & SEMANTICS  (the "Meaning")
  └── Business Vocabulary
        ├── Glossary   — flat definitions
        ├── Taxonomy   — single-axis hierarchy (is-a / part-of)
        └── Ontology   — cross-domain relationship graph
DATA STRUCTURE & GOVERNANCE  (the "Implementation")
  ├── Semantic Layer     — serves business queries (active engine)
  ├── Data Dictionary    — maps terms to physical schemas (passive docs)
  └── Reference Data Management
        ├── Allowed set / Canonical value  — "United States"
        └── Variants / Synonyms / Acronyms — "USA", "US", "SKU"
