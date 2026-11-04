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
| **Ontology format** | OWL in Turtle, stored in Amazon Neptune | terms, SQL attributes and joins as rows in **Postgres + pgvector**; exchanged as **Apache Ossie YAML** (no RDF/OWL) |
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

* **What the ontology contains:** three kinds of definition.
  * **Terms:** business concepts such as *Customer*, each with a
    description and synonyms, mapped to tables and columns.
  * **SQL attributes:** reviewed SQL expressions under a term, such as
    revenue or margin.
  * **Relationships:** how tables join, inferred from keys and from the
    data.
* **Format:** stored in Postgres, with embeddings in pgvector. Models are
  imported and exported as **Apache Ossie YAML**.
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
| **Standards** | OWL, R2RML, SPARQL, SKOS; grounding to FIBO, schema.org | Apache Ossie YAML for import / export |
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

### One example, two representations

Here is the same piece of business knowledge in each tool. It comes from
the retail example in my
[reference repo](https://github.com/rongali-naidu/semantics-ontology-graphs/tree/main/examples/retail):

> *What was revenue from premium customers?*
> (In the example data, the answer is **1678.45**.)

To answer it, the agent needs four facts:

1. what "premium customers" means, and that they are customers;
2. how revenue is computed;
3. how orders join to customers;
4. which column marks a customer as premium.

#### AWS COA: a formal OWL ontology, a mapping and a metric

**The ontology** is a standard OWL file in Turtle. Concepts are *classes*,
and rules are *axioms* that a reasoner understands:

```turtle
retail:Customer a owl:Class ;
    rdfs:label "Customer"@en ;
    skos:definition "A person with an account who can place orders."@en .

retail:PremiumCustomer a owl:Class ;
    rdfs:subClassOf retail:Customer ;          # every premium customer IS a customer
    rdfs:label "Premium customer"@en .

retail:placedBy a owl:ObjectProperty , owl:FunctionalProperty ;
    rdfs:domain retail:Order ;
    rdfs:range  retail:Customer ;
    owl:inverseOf retail:placedOrder .

retail:Order rdfs:subClassOf [                 # every order has exactly one customer
    a owl:Restriction ;
    owl:onProperty retail:placedBy ;
    owl:qualifiedCardinality "1"^^xsd:nonNegativeInteger ;
    owl:onClass retail:Customer ] .

retail:subCategoryOf a owl:TransitiveProperty . # Laptops ⊂ Computers ⊂ Electronics
```

**The R2RML mapping** connects the ontology to the tables. Here, a boolean
column becomes a class:

```turtle
map:PremiumCustomer a rr:TriplesMap ;
    rr:logicalTable [ rr:sqlQuery "SELECT * FROM customers WHERE is_premium = 'true'" ] ;
    rr:subjectMap [ rr:template "https://example.org/retail/data/{customer_id}" ;
                    rr:class retail:PremiumCustomer ] .
```

**The governed metric** is registered in COA's metric service:

```json
{ "name": "revenue",
  "description": "Sum of quantity * unit price actually paid (not list price).",
  "expression": { "dialects": [ { "dialect": "POSTGRESQL",
      "expression": "SELECT SUM(quantity * unit_price) AS revenue FROM retail.order_lines" } ] },
  "sourceTable": "order_lines",
  "ontologyConcepts": [ "https://example.org/retail/ontology#OrderLine",
                        "https://example.org/retail/ontology#unitPrice",
                        "https://example.org/retail/ontology#quantity" ] }
```

**How COA answers:**

1. It finds the `revenue` metric.
2. The question adds a qualifier ("premium customers"), so it moves to the
   ontology tier.
3. A SPARQL query over `retail:PremiumCustomer` and `retail:placedBy` is
   rewritten into SQL (Ontop) through the mapping.
4. The SQL runs on the live tables.

#### NVIDIA Auto Ontology: terms, SQL attributes and joins

The same knowledge as Auto Ontology records. *These are illustrative.* The
field names come from its source code (`dal/terms.py`,
`dal/sql_attributes.py`); the values are mine, because the docs publish no
sample payloads:

```yaml
term:
  name: Customer
  description: A person with an account who can place orders.
  synonyms: [client, buyer, shopper]
  name_certified: true
  represents: customers                 # table → term link

term:
  name: Premium customer
  description: A customer with a paid membership.
  synonyms: [premium member, member]
  columns: [customers.is_premium]       # column → term link

sql_attribute:                          # under the term "Order line"
  name: revenue
  expression: SUM(order_lines.quantity * order_lines.unit_price)
  certified: true

relationship:                           # inferred from keys and data
  orders.customer_id → customers.customer_id
  order_lines.order_id → orders.order_id
```

**How Auto Ontology answers:**

1. "premium customers" matches the term *Premium customer* by synonym.
2. The term leads to the column `customers.is_premium`.
3. The certified `revenue` SQL attribute supplies the formula.
4. The join relationships supply the path.
5. The agent writes the SQL, and it's validated against the certified
   definitions.

#### What each representation can express

| Knowledge | AWS COA (OWL) | NVIDIA Auto Ontology |
|---|---|---|
| synonyms and descriptions | ✅ `rdfs:label`, `skos:altLabel`, `skos:definition` | ✅ `synonyms`, `description` |
| business term → table / column | ✅ R2RML mapping | ✅ term ↔ table and column links |
| governed metric formula | ✅ metric service | ✅ certified SQL attribute |
| how tables join | ✅ object properties + mapping | ✅ join relationships |
| *a premium customer **is a** customer* | ✅ `rdfs:subClassOf`; a reasoner infers it | ❌ two separate terms, "related" only because they share the `customers` table |
| *Laptops are in Electronics* (via Computers) | ✅ `owl:TransitiveProperty`; inferred | ❌ no hierarchy; you'd encode it in SQL |
| *every order has exactly one customer* | ✅ cardinality axiom, machine-checkable | ❌ at most a sentence in a description |
| exchange format | OWL / Turtle, queried with SPARQL | Apache Ossie YAML, queried through MCP / API |

The top four rows are what text-to-SQL needs most, and **both tools cover
them**. The bottom rows are where an ontology in the formal sense adds
*reasoning*: knowledge that's inferred and checked rather than written down
in every query.

### So why does NVIDIA call it an ontology?

Because "ontology" has two meanings, and both are in use.

* **The broad meaning.** In Tom Gruber's classic definition, an ontology is
  "an explicit specification of a conceptualization": a shared, written-down
  model of the concepts in a domain and how they relate. Nothing in that
  definition requires OWL. Business terms with definitions, synonyms,
  mappings and relationships meet it.
* **The formal meaning.** In the Semantic Web tradition, an ontology is an
  OWL model with classes, hierarchies and logical axioms that a reasoner can
  use to infer new facts and detect contradictions.

Deborah McGuinness's *ontology spectrum* puts these on one line:

> glossary → thesaurus → taxonomy → classes and properties → formal axioms

* **Auto Ontology** sits at the lighter end: a glossary with synonyms (a
  thesaurus), plus mappings, governed SQL and a join graph. In the terms of
  my repo, it's closer to a **semantic model plus a context graph** than to
  a formal ontology.
* **COA** sits at the formal end.

NVIDIA isn't alone in using the broad meaning. **Palantir Foundry
Ontology** and **Microsoft Fabric IQ** also call their business-concept
layers "ontologies" without using OWL.

**The practical takeaway:** when a product says "ontology", ask four
questions:

1. Does it model **hierarchies** (is-a)?
2. Can it express **rules** (cardinality, disjointness)?
3. Does it **reason** (infer and check)?
4. What's the **exchange format** (OWL, Ossie, proprietary)?

The answers tell you which end of the spectrum you're buying.

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
no SPARQL and no formal reasoner, so it's an ontology in the broad sense
rather than the formal one (see above).

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
