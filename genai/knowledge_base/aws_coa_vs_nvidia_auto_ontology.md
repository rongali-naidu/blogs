---
title: "AWS COA vs NVIDIA Auto Ontology: Two Ways to Give Data Agents an Ontology"
date: 2026-11-04
tags: [genai, knowledge-base]
publish: true
---

## AWS COA vs NVIDIA Auto Ontology: Two Ways to Give Data Agents an Ontology

In 2026, two big vendors released open-source projects with the same goal:
give AI agents the *business meaning* of your data so they write correct SQL.

* **AWS Context Ontology Accelerator (COA)**, announced in July 2026.
* **NVIDIA Auto Ontology**, released more recently.

Both build an ontology from infrastructure you already have and serve it to
agents over MCP. But they make very different design choices. This post
compares them side by side.

> **Scope:** this comparison is based on each project's public
> documentation as of November 2026 (COA `v0.3.8`; Auto Ontology `main`,
> which has no tagged release yet). I haven't run either in production.
> More detail, plus a retail example expressed in every format, is in my
> reference repo:
> [semantics-ontology-graphs](https://github.com/rongali-naidu/semantics-ontology-graphs).

### The problem they both solve

An LLM already knows SQL. What it doesn't know is *your* business:

* which of five "revenue" columns is the governed one;
* how revenue is derived (gross sales minus discounts, with the right
  filters);
* which joins are safe, and at what grain to aggregate;
* that "client", "buyer" and "customer" mean the same thing.

An **ontology** captures that meaning: concepts, relationships, synonyms and
rules. Writing one by hand has always been the expensive part. Both projects
use LLMs to draft it from what already exists, then keep humans in the loop
to approve it.

### TL;DR

| | AWS COA | NVIDIA Auto Ontology |
|---|---|---|
| **Philosophy** | standards-first: a formal **W3C OWL** ontology | pragmatic: a **context layer for text-to-SQL** |
| **Ontology format** | OWL in Turtle, stored in Amazon Neptune | records in **Postgres + pgvector** (no RDF/OWL) |
| **Reasoning** | OWL reasoners (HermiT, ELK) | validates SQL against certified definitions and lineage |
| **Runs on** | AWS only (Neptune, OpenSearch Serverless, Bedrock, AgentCore) | anywhere with Postgres + pgvector and an LLM endpoint |
| **Built-in evaluation** | not part of the project | separate toolkit, **ontology-sql-eval** (BIRD and other benchmarks) |
| **Licence** | Apache 2.0 | Apache 2.0 |

### AWS COA in one paragraph

COA is a semantic context layer you deploy into your AWS account. Its
workflow is **Scan → Model → Serve**:

* **Scan and Model:** Bedrock (Claude) reads table schemas or documents and
  *induces* an OWL ontology, plus **R2RML** mappings from tables to the
  ontology. You can also upload an existing ontology (Turtle, RDF/XML,
  OWL/XML or JSON-LD). It then *grounds* the ontology against standards
  such as FIBO and schema.org.
* **Approval and reasoning:** the result arrives as a **proposal** that
  domain experts approve, and HermiT/ELK reason over it.
* **Serve:** agents get answers in **tiers**:
  1. **governed metrics** first;
  2. then the **ontology**, through a *virtual knowledge graph*: Ontop
     rewrites SPARQL into SQL against the live tables, so no data is copied;
  3. then **documents**.

  MCP tools: `describe_schema`, `translate_sparql`, `graph_traversal` and
  `query`.
* **Governance:** namespaces with roles, and Cedar policies.

Details: [AWS COA in the repo](https://github.com/rongali-naidu/semantics-ontology-graphs/blob/main/ontologies/implementations/providers/aws/README.md).

### NVIDIA Auto Ontology in one paragraph

Auto Ontology is a Python framework, with a web UI, organised as five
steps:

| Step | What it does |
|---|---|
| **Discover** | generates an ontology from catalog metadata, and connects existing ontologies and semantic models |
| **Ground** | gives agents the meaning of metrics, entities and relationships, and reconciles conflicting definitions |
| **Validate** | checks the SQL an agent generates against certified definitions and lineage, and returns provenance with the answer |
| **Expose** | serves the context through MCP, a CLI, a streaming API and agent skills |
| **Learn** | feeds certified corrections and usage back into the ontology |

* **Sources:** Databricks, Snowflake, Postgres and DuckDB.
* **Evaluation:** a companion repo, **ontology-sql-eval**, compares a
  schema-only LLM with the ontology-grounded agent on BIRD, BEAVER,
  FDABench and other benchmarks. Scoring is deterministic execution match,
  with no LLM judge.

Details: [NVIDIA Auto Ontology in the repo](https://github.com/rongali-naidu/semantics-ontology-graphs/blob/main/ontologies/implementations/providers/nvidia/README.md).

### Side by side

| Dimension | AWS COA | NVIDIA Auto Ontology |
|---|---|---|
| **How the ontology is built** | LLM *induction* from schemas or documents, or upload your own OWL | *discovery* from catalog metadata, or connect existing ontologies and semantic models |
| **Link to the data** | explicit **R2RML mappings** (table/column → class/property) | catalog metadata linked to ontology concepts |
| **Reuse of standards** | grounding to FIBO, schema.org (`rdfs:subClassOf`, `skos:*Match`) | not a focus |
| **Human review** | induced ontologies are **proposals**; experts approve them | **certified** definitions; corrections feed back (Learn) |
| **Reasoning** | formal OWL reasoning (HermiT, ELK) | SQL validated against certified definitions and lineage |
| **How agents get answers** | tiers: metric → ontology (SPARQL → SQL through Ontop) → documents | ontology-grounded text-to-SQL, with provenance |
| **Agent interface** | MCP tools, plus a metrics API | MCP server, CLI, streaming API (`/api/chat/completions`), agent skills |
| **Storage** | Neptune (ontology), OpenSearch Serverless (embeddings) | Postgres + pgvector |
| **Infrastructure** | about 16 CDK stacks; COA's docs put the idle cost at about **$930/month** | Postgres + pgvector, an LLM endpoint, Python 3.12+ |
| **Portability** | AWS only | cloud-neutral |
| **Measuring accuracy** | not built in | **ontology-sql-eval**, on public benchmarks |
| **Contributions** | read-only mirror: issues welcome, no PRs | not accepting external contributions |
| **Maturity (Nov 2026)** | tagged releases (`v0.3.x`); AWS says the managed capability will move into **AWS Context** | no tagged release yet |

### Same goal, different bets

**COA bets on standards.** The ontology is real OWL. It can be exchanged
with other tools, reasoned over formally, aligned to industry ontologies and
queried with SPARQL. R2RML mappings make the link from tables to concepts
explicit and testable. You get interoperability and rigour, and you pay for
them in infrastructure, AWS lock-in and a steeper learning curve (OWL,
SPARQL, R2RML).

**Auto Ontology bets on the text-to-SQL loop.** The "ontology" is a working
context layer in Postgres, built to make agent SQL correct and to *prove* it
on benchmarks. It's lighter to run and cloud-neutral, and the Learn step
improves it from real usage. The trade-off: there's no OWL file to exchange,
no SPARQL and no formal reasoner, so it isn't an ontology in the
Semantic-Web sense.

In the terms I use in the repo, COA is a **formal ontology plus a virtual
knowledge graph**, while Auto Ontology is closer to a **context graph** of
metadata, definitions and mappings. Both keep the transactions in the
warehouse, which is the right call at scale
([why](https://github.com/rongali-naidu/semantics-ontology-graphs/blob/main/foundations/04-data-agents.md#42-what-goes-in-the-graph-and-what-stays-in-the-warehouse)).

### What they agree on

The overlap matters more than the differences:

1. **LLM proposes, human approves, machine checks.** Neither lets the
   generated ontology go to agents unreviewed.
2. **Governed definitions come first.** COA answers from registered metrics
   before anything else; Auto Ontology validates against certified
   definitions.
3. **Start from existing infrastructure.** Neither expects you to model your
   business from a blank page.
4. **MCP is the delivery channel.** The ontology is useful only once agents
   can call it as a tool.
5. **The data stays where it is.** Neither copies terabytes of transactions
   into a graph.

### Which one would I pick?

* **COA**:
  * you're on AWS;
  * you work in a regulated domain (finance, healthcare) where formal
    semantics, alignment to standards like FIBO, and auditability matter;
  * you want an ontology that other tools can read.
* **Auto Ontology**:
  * your main goal is **accurate text-to-SQL** over Databricks, Snowflake or
    Postgres;
  * you want to stay cloud-neutral;
  * you want to **measure** the gain before committing (ontology-sql-eval
    makes that easy to try).
* **Either way:** keep a semantic model for the numbers. An ontology tells
  the agent what "revenue" *means*; a semantic layer (dbt MetricFlow, Apache
  Ossie, Snowflake semantic views…) makes sure it's computed the same way
  every time. The repo explains
  [why you need both](https://github.com/rongali-naidu/semantics-ontology-graphs/blob/main/foundations/04-data-agents.md#43-isnt-that-redundant-a-semantic-model-and-a-graph-of-semantic-model-information).

### Learn more

I keep notes, diagrams and a runnable retail example of ontologies,
knowledge graphs and semantic models in one place. Every format (OWL, RDF,
property graph, SQL, Ossie, dbt) is checked in CI to give the same answers:

👉 **[github.com/rongali-naidu/semantics-ontology-graphs](https://github.com/rongali-naidu/semantics-ontology-graphs)**

Useful starting points:

* [What a data agent needs: ontology, semantic model, knowledge graph](https://github.com/rongali-naidu/semantics-ontology-graphs/blob/main/foundations/04-data-agents.md)
* [Mapping tables to an ontology: R2RML, OBDA, LLM-assisted](https://github.com/rongali-naidu/semantics-ontology-graphs/blob/main/ontologies/concepts/07-mapping-ontology-to-tables.md)
* [All ontology products compared](https://github.com/rongali-naidu/semantics-ontology-graphs/blob/main/ontologies/implementations/README.md)

Sources:
[aws/context-ontology-accelerator](https://github.com/aws/context-ontology-accelerator) ·
[COA docs](https://aws.github.io/context-ontology-accelerator/) ·
[NVIDIA/auto-ontology](https://github.com/NVIDIA/auto-ontology) ·
[NVIDIA/ontology-sql-eval](https://github.com/NVIDIA/ontology-sql-eval)
