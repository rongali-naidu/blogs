# Knowledge Base with Ontology-Based Metadata Enrichment

## 1. Foundation: What We Have (Phase 1)

### Current Lambda Output (per database JSON)

https://github.com/rongali-naidu/ai-data-discovery-agent/blob/main/lambda/glue_kb_builder/handler.py


```json
{
  "table_name": "employee_dtls",
  "fully_qualified_name": "AwsDataCatalog.hr_db.\"employee_dtls\"",
  "table_metadata": { "table_comment": "", "parameters": {"foreign_key_1": "..."} },
  "columns": [{"name": "id", "type": "int", "comment": "Employee id"}],
  "partitions": [{"name": "region_id", "type": "string", "comment": "Region identifier..."}],
  "sample_sqls": [{"query_name": "...", "query_sql": "..."}]
}
```

### What the Blog Demonstrated

https://aws.amazon.com/blogs/big-data/enriching-metadata-for-accurate-text-to-sql-generation-for-amazon-athena/

- Column COMMENTs with distinct values → fixes literal value hallucination
- FK constraints in Glue parameters → enables correct JOINs
- Partition descriptions → enables partition pruning
- SQL generating instructions → guides model behavior

---

## 2. Ontology Concepts Mapped to SQL

### Ontology ↔ SQL ↔ Knowledge Graph Mapping

| Ontology Concept | SQL Equivalent | KG Representation | Purpose for Text-to-SQL |
|-----------------|----------------|-------------------|------------------------|
| **Class** | Table / View | Node (type: table) | Identifies what entities exist |
| **Data Property** | Column + Type | Node attribute | Tells LLM what fields are available |
| **Annotation** | Column COMMENT / Description | Node attribute (description) | Gives business meaning to columns |
| **Object Property** | Foreign Key / JOIN | Edge between table nodes | Tells LLM how to connect tables |
| **Cardinality** | 1:1, 1:N, M:N | Edge attribute | Prevents fan-out errors in JOINs |
| **Instance** | Distinct value / enum | Node attribute (allowed_values) | Fixes literal value hallucination |
| **Concept Hierarchy** | Category tree / inheritance | Parent-child edges | Enables "Electronics includes Laptops" |
| **Derived Concept** | Metric / calculated field | Metric node with formula edge | Defines "Revenue = SUM(x) WHERE y" |
| **Synonym** | Business term alias | Edge (synonym_of) | Maps "revenue" → correct column |
| **Constraint** | WHERE clause / business rule | Edge (requires_filter) | Enforces "always filter cancelled orders" |
| **Domain** | Schema / database | Container node | Scopes context to relevant tables |

### Visual: How Ontology Maps to the Knowledge Graph

```
ONTOLOGY (abstract blueprint):

  ┌─────────────────────────────────────────────────┐
  │ Domain: hr_db                                    │
  │                                                  │
  │   Class: employee_dtls                           │
  │     DataProperty: id (INT, PK)                   │
  │     DataProperty: name (STRING)                  │
  │     DataProperty: emp_category (STRING)          │
  │       Instances: TEMP, PERM, CONTR               │
  │     DataProperty: region_id (STRING, partition)   │
  │       Instances: AMER, EMEA, APAC                │
  │     ObjectProperty: belongs_to → department_dtls │
  │       via: dept_id = department_dtls.id          │
  │       cardinality: N:1                           │
  │                                                  │
  │   Class: department_dtls                         │
  │     DataProperty: id (INT, PK)                   │
  │     DataProperty: name (STRING)                  │
  │                                                  │
  │   DerivedConcept: "headcount"                    │
  │     formula: COUNT(DISTINCT employee_dtls.id)    │
  │     synonym: "employee count", "staff count"     │
  │                                                  │
  │   BusinessTerm: "permanent employee"             │
  │     maps_to: emp_category = 'PERM'              │
  │                                                  │
  └─────────────────────────────────────────────────┘


KNOWLEDGE GRAPH (stored as JSON — no graph DB needed):

  [employee_dtls] ──belongs_to (N:1)──→ [department_dtls]
       │                                      │
       ├─ columns: id, name, emp_category...  ├─ columns: id, name...
       ├─ partition: region_id                 │
       ├─ computes: "headcount" metric         │
       └─ business_rules: [...]               └─ ...
```

---

## 3. Knowledge Graph: Logical Structure (No Graph DB Required)

### Why Not a Graph Database?

For most text-to-SQL use cases (< 500 tables), storing the knowledge graph as **structured JSON in S3** is sufficient:
- Agent performs simple lookups (1-2 hops max)
- No complex multi-hop traversal needed
- Zero infrastructure overhead
- Easy to version and update

### Logical Graph Structure (stored as JSON)

```
Nodes:
├── Schema nodes     (one per Glue database)
├── Table nodes      (one per table/view)
├── Column nodes     (attributes on table nodes)
├── Metric nodes     (derived business calculations)
├── Term nodes       (business glossary entries)
└── Query nodes      (golden/sample SQL queries)

Edges (relationships):
├── schema_contains      → Schema → Table
├── joins_to             → Table → Table (with FK details)
├── computes_metric      → Table → Metric
├── term_maps_to         → Term → Column + filter condition
├── synonym_of           → Term → Term
├── example_query_for    → Query → Table
└── hierarchy_parent     → Term → Term (category trees)
```

---

## 4. Enhanced JSON Format (Phase 2)

### Organization Strategy

```
s3://{BUCKET_NAME}/
├── catalog_metadata/
│   ├── _catalog_manifest.json          ← Index of all schemas
│   ├── _glossary.json                  ← Business terms, acronyms, synonyms
│   ├── _metrics.json                   ← Cross-schema metric definitions
│   ├── _cross_schema_joins.json        ← Joins between tables in different schemas
│   │
│   ├── {schema_name}/
│   │   ├── _schema_summary.json        ← Schema-level overview + intra-schema joins
│   │   ├── {table_name}.json           ← Full table metadata (per-table file)
│   │   └── {table_name}.json
│   │
│   └── {schema_name}/
│       ├── _schema_summary.json
│       └── {table_name}.json
│
├── golden_queries/
│   ├── {schema_name}/
│   │   └── {table_name}_queries.json   ← Verified queries per table
│   └── cross_schema_queries.json       ← Queries spanning multiple schemas
│
└── athena_saved_queries/               ← (existing from Phase 1)
    └── {workgroup}.json
```

### Why This Organization?

| Strategy | Reason |
|----------|--------|
| **Per-table JSON** | Agent retrieves only relevant tables (token-efficient for LLM context) |
| **Schema summary** | Quick lookup to find which tables exist + their joins |
| **Separate glossary** | Reusable across all tables; loaded once per session |
| **Separate metrics** | Metrics often span tables; don't belong to a single table |
| **Cross-schema joins** | Some tables join across databases |
| **Golden queries separate** | Easy to update independently; versioned by domain experts |

---

## 5. File Formats

### 5.1 Catalog Manifest (`_catalog_manifest.json`)

```json
{
  "catalog_name": "AwsDataCatalog",
  "last_updated": "2026-07-30T23:00:00Z",
  "schemas": [
    {
      "schema_name": "hr_db",
      "description": "Human resources data - employees, departments, locations",
      "table_count": 12,
      "owner": "hr-data-team",
      "tables": ["employee_dtls", "department_dtls", "location_dtls", "..."]
    },
    {
      "schema_name": "sales_db",
      "description": "Sales transactions and order data",
      "table_count": 25,
      "owner": "sales-analytics",
      "tables": ["orders", "order_items", "customers", "..."]
    }
  ]
}
```

### 5.2 Business Glossary (`_glossary.json`)

```json
{
  "last_updated": "2026-07-30T23:00:00Z",
  "terms": {
    "headcount": {
      "display_name": "Headcount",
      "definition": "Total number of active employees",
      "synonyms": ["employee count", "staff count", "HC"],
      "sql_definition": "COUNT(DISTINCT employee_dtls.id)",
      "required_filters": ["employee_dtls.emp_category != 'CONTR'"],
      "tables_involved": ["employee_dtls"],
      "category": "HR Metrics"
    },
    "permanent_employee": {
      "display_name": "Permanent Employee",
      "definition": "Full-time employee with ongoing contract",
      "synonyms": ["full-time", "FTE", "perm"],
      "maps_to": {
        "table": "employee_dtls",
        "column": "emp_category",
        "value": "PERM"
      }
    },
    "AMER": {
      "display_name": "Americas",
      "definition": "North and South America region",
      "synonyms": ["Americas", "North America", "NA", "US region"],
      "maps_to": {
        "table": "employee_dtls",
        "column": "region_id",
        "value": "AMER"
      }
    },
    "GMS": {
      "full_name": "Gross Merchandise Sales",
      "definition": "Total value of goods sold, after shipping, before returns",
      "synonyms": ["revenue", "gross sales", "top-line"],
      "not_to_confuse_with": "Net Revenue (subtracts returns)"
    }
  },
  "acronyms": {
    "TEMP": "Temporary employee",
    "PERM": "Permanent employee",
    "CONTR": "Contractor",
    "AMER": "Americas region",
    "EMEA": "Europe, Middle East, and Africa",
    "APAC": "Asia Pacific",
    "FTE": "Full-Time Equivalent",
    "YoY": "Year over Year",
    "MoM": "Month over Month",
    "WoW": "Week over Week"
  },
  "hierarchies": {
    "geography": {
      "levels": ["region_id", "country", "city"],
      "values": {
        "AMER": ["US", "CA", "BR", "MX"],
        "EMEA": ["UK", "DE", "FR", "AE"],
        "APAC": ["IN", "JP", "AU", "SG"]
      }
    }
  }
}
```

### 5.3 Metrics Definition (`_metrics.json`)

```json
{
  "last_updated": "2026-07-30T23:00:00Z",
  "metrics": {
    "headcount": {
      "display_name": "Headcount",
      "synonyms": ["employee count", "staff count", "HC"],
      "sql": "COUNT(DISTINCT employee_dtls.id)",
      "required_filters": [
        "employee_dtls.emp_category IN ('PERM', 'TEMP')"
      ],
      "excluded_filters": [
        "Never include CONTR (contractors) in headcount"
      ],
      "requires_tables": ["employee_dtls"],
      "grain": "Can be grouped by region_id, dept_id, joining_date",
      "notes": "Contractors are excluded by business definition"
    },
    "avg_tenure_years": {
      "display_name": "Average Tenure (Years)",
      "synonyms": ["average experience", "avg years employed"],
      "sql": "AVG(DATE_DIFF('year', employee_dtls.joining_date, CURRENT_DATE))",
      "required_filters": [
        "employee_dtls.emp_category = 'PERM'"
      ],
      "requires_tables": ["employee_dtls"],
      "notes": "Only for permanent employees. Contractors not applicable."
    },
    "department_size": {
      "display_name": "Department Size",
      "synonyms": ["dept headcount", "team size"],
      "sql": "COUNT(DISTINCT employee_dtls.id)",
      "requires_tables": ["employee_dtls", "department_dtls"],
      "requires_joins": ["employee_dtls.dept_id = department_dtls.id"],
      "grain": "Grouped by department_dtls.name"
    }
  }
}
```

### 5.4 Schema Summary (`{schema_name}/_schema_summary.json`)

```json
{
  "schema_name": "hr_db",
  "description": "Human resources data - employees, departments, locations",
  "owner": "hr-data-team",
  "last_updated": "2026-07-30T23:00:00Z",
  "table_count": 3,
  
  "tables_overview": [
    {
      "table_name": "employee_dtls",
      "description": "Employee master data with demographics and categorization",
      "grain": "One row per employee",
      "row_count_approx": 50000,
      "partition_key": "region_id"
    },
    {
      "table_name": "department_dtls",
      "description": "Department reference table",
      "grain": "One row per department",
      "row_count_approx": 25
    },
    {
      "table_name": "location_dtls",
      "description": "Office and facility locations",
      "grain": "One row per location",
      "row_count_approx": 100
    }
  ],

  "joins": [
    {
      "source": "employee_dtls",
      "target": "department_dtls",
      "on": "employee_dtls.dept_id = department_dtls.id",
      "cardinality": "N:1",
      "description": "Each employee belongs to one department"
    },
    {
      "source": "employee_dtls",
      "target": "location_dtls",
      "on": "employee_dtls.location_id = location_dtls.id",
      "cardinality": "N:1",
      "description": "Each employee is assigned to one location"
    },
    {
      "source": "department_dtls",
      "target": "location_dtls",
      "on": "department_dtls.location_id = location_dtls.id",
      "cardinality": "N:1",
      "description": "Each department has a primary location"
    }
  ],

  "anti_patterns": [
    "Never join employee_dtls directly to location_dtls for department location — go through department_dtls",
    "Always filter on region_id partition when querying employee_dtls for performance"
  ]
}
```

### 5.5 Per-Table Metadata (`{schema_name}/{table_name}.json`)

```json
{
  "table_name": "employee_dtls",
  "fully_qualified_name": "AwsDataCatalog.hr_db.\"employee_dtls\"",
  "description": "Employee master data including demographics, categorization, and organizational assignment",
  "grain": "One row per employee",
  "owner": "hr-data-team",
  "refresh_frequency": "daily",
  "last_updated": "2026-07-30T23:00:00Z",

  "table_metadata": {
    "location": "s3://hr-data-lake/employee_dtls/",
    "classification": "EXTERNAL_TABLE",
    "create_time": "2024-01-15T10:00:00Z",
    "update_time": "2026-07-30T06:00:00Z",
    "parameters": {
      "primary_key": "CONSTRAINT pk_1 PRIMARY KEY (id)",
      "foreign_key_1": "CONSTRAINT FK_1 FOREIGN KEY (dept_id) REFERENCES department_dtls(id)"
    },
    "row_count_approx": 50000
  },

  "columns": [
    {
      "name": "id",
      "type": "int",
      "description": "Unique employee identifier",
      "role": "primary_key",
      "nullable": false
    },
    {
      "name": "name",
      "type": "string",
      "description": "Full name of the employee",
      "pii": true
    },
    {
      "name": "age",
      "type": "int",
      "description": "Current age of the employee in years",
      "constraints": {
        "min": 18,
        "max": 70
      }
    },
    {
      "name": "dept_id",
      "type": "int",
      "description": "Department identifier - FK to department_dtls.id",
      "role": "foreign_key",
      "references": {
        "table": "department_dtls",
        "column": "id"
      }
    },
    {
      "name": "emp_category",
      "type": "string",
      "description": "Employment category classification",
      "distinct_values": {
        "type": "static",
        "values": ["TEMP", "PERM", "CONTR"],
        "business_mapping": {
          "TEMP": "Temporary employee - fixed-term contract",
          "PERM": "Permanent employee - ongoing full-time",
          "CONTR": "Contractor - external third-party"
        }
      }
    },
    {
      "name": "location_id",
      "type": "int",
      "description": "Office location identifier - FK to location_dtls.id",
      "role": "foreign_key",
      "references": {
        "table": "location_dtls",
        "column": "id"
      }
    },
    {
      "name": "joining_date",
      "type": "date",
      "description": "Date the employee joined the organization",
      "format": "YYYY-MM-DD",
      "range": "2015-01-01 to current"
    }
  ],

  "partitions": [
    {
      "name": "region_id",
      "type": "string",
      "description": "Geographic region identifier",
      "distinct_values": {
        "type": "static",
        "values": ["AMER", "EMEA", "APAC"],
        "business_mapping": {
          "AMER": "Americas - US, Canada, Brazil, Mexico",
          "EMEA": "Europe, Middle East, and Africa",
          "APAC": "Asia Pacific - India, Japan, Australia, Singapore"
        }
      },
      "performance_note": "Always include region_id filter for partition pruning"
    }
  ],

  "joins": [
    {
      "target": "department_dtls",
      "on": "employee_dtls.dept_id = department_dtls.id",
      "cardinality": "N:1",
      "description": "Each employee belongs to one department",
      "join_type": "INNER"
    },
    {
      "target": "location_dtls",
      "on": "employee_dtls.location_id = location_dtls.id",
      "cardinality": "N:1",
      "description": "Each employee is assigned to one location",
      "join_type": "LEFT",
      "caveat": "Some employees may not have location_id assigned"
    }
  ],

  "business_rules": [
    {
      "rule": "Active employees only",
      "description": "Exclude contractors when counting headcount",
      "sql_filter": "emp_category IN ('PERM', 'TEMP')"
    },
    {
      "rule": "Partition pruning",
      "description": "Always filter on region_id for performance",
      "sql_filter": "region_id = '<value>'"
    }
  ],

  "anti_patterns": [
    "Don't use emp_category = 'permanent' — the value is 'PERM' (uppercase abbreviated)",
    "Don't filter region_id = 'North America' — use 'AMER'",
    "Don't join to location_dtls for department location — use department_dtls.location_id instead"
  ],

  "sample_sqls": [
    {
      "query_name": "Permanent employees in Americas",
      "description": "List all permanent employees in AMER region",
      "query_sql": "SELECT id, name, dept_id FROM employee_dtls WHERE emp_category = 'PERM' AND region_id = 'AMER'",
      "source": "athena_saved_query",
      "verified": true
    },
    {
      "query_name": "Headcount by department",
      "description": "Count employees per department excluding contractors",
      "query_sql": "SELECT d.name AS department, COUNT(DISTINCT e.id) AS headcount FROM employee_dtls e JOIN department_dtls d ON e.dept_id = d.id WHERE e.emp_category IN ('PERM', 'TEMP') GROUP BY d.name",
      "source": "golden_query",
      "verified": true
    }
  ],

  "related_terms": ["headcount", "permanent_employee", "AMER", "EMEA", "APAC"]
}
```

### 5.6 Golden Queries (`golden_queries/{schema_name}/{table_name}_queries.json`)

```json
{
  "table_name": "employee_dtls",
  "schema_name": "hr_db",
  "last_updated": "2026-07-30T23:00:00Z",
  "queries": [
    {
      "id": "gq_001",
      "natural_language": "How many permanent employees are in each region?",
      "sql": "SELECT region_id, COUNT(DISTINCT id) AS headcount FROM employee_dtls WHERE emp_category = 'PERM' GROUP BY region_id",
      "tables_used": ["employee_dtls"],
      "concepts_tested": ["partition_filter", "aggregation", "literal_value"],
      "verified_by": "hr-analytics-team",
      "verified_date": "2026-07-15"
    },
    {
      "id": "gq_002",
      "natural_language": "List employees who joined after 2024 in the Engineering department",
      "sql": "SELECT e.name, e.joining_date, d.name AS department FROM employee_dtls e JOIN department_dtls d ON e.dept_id = d.id WHERE d.name = 'Engineering' AND e.joining_date > DATE '2024-01-01'",
      "tables_used": ["employee_dtls", "department_dtls"],
      "concepts_tested": ["join", "date_filter", "string_filter"],
      "verified_by": "hr-analytics-team",
      "verified_date": "2026-07-15"
    }
  ]
}
```

---

## 6. How the Agent Uses This Structure

### Retrieval Flow

```
User: "How many permanent employees are in North America?"
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 1: Load Glossary (cached per session)                  │
│  Agent loads: _glossary.json                                 │
│  Resolves:                                                   │
│    "permanent" → maps_to: emp_category = 'PERM'             │
│    "North America" → maps_to: region_id = 'AMER'            │
│    "employees" → table: employee_dtls                        │
└─────────────────────────────────────────────────────┬───────┘
                                                      │
                                                      ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 2: Load Schema Summary                                 │
│  Agent loads: hr_db/_schema_summary.json                     │
│  Identifies: employee_dtls is the relevant table             │
│  Notes: partition on region_id                               │
└─────────────────────────────────────────────────────┬───────┘
                                                      │
                                                      ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 3: Load Table Metadata                                 │
│  Agent loads: hr_db/employee_dtls.json                       │
│  Gets: columns, distinct values, joins, anti-patterns        │
└─────────────────────────────────────────────────────┬───────┘
                                                      │
                                                      ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 4: Check Golden Queries (optional)                     │
│  Agent loads: golden_queries/hr_db/employee_dtls_queries.json│
│  Finds: similar question → provides as few-shot example      │
└─────────────────────────────────────────────────────┬───────┘
                                                      │
                                                      ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 5: Build Prompt + Call LLM                             │
│                                                              │
│  System prompt includes:                                     │
│  - Glossary terms (resolved)                                 │
│  - Table DDL with enriched COMMENTs                          │
│  - Join definitions                                          │
│  - Anti-patterns                                             │
│  - Golden query as few-shot example                          │
│  - SQL generating instructions                               │
│                                                              │
│  LLM generates:                                              │
│  SELECT COUNT(DISTINCT id) AS headcount                      │
│  FROM employee_dtls                                          │
│  WHERE emp_category = 'PERM'                                 │
│    AND region_id = 'AMER'                                    │
└─────────────────────────────────────────────────────────────┘
```

### Token Budget Strategy

```
Context Window Budget:
├── Glossary terms (relevant subset)     ~500 tokens
├── Schema summary                       ~300 tokens
├── Table metadata (1-3 tables)          ~800-2400 tokens
├── Golden query example (1-2)           ~200-400 tokens
├── SQL instructions                     ~300 tokens
├── Anti-patterns                        ~200 tokens
└── User question + conversation         ~200 tokens
                                         ─────────────
                                Total:   ~2500-4300 tokens

Per-table files keep this manageable.
Full schema dump would be 50K+ tokens — won't fit.
```

---

## 7. What Changes in the Lambda (Phase 2)

### New Capabilities Needed

| Capability | Source | How to Gather |
|-----------|--------|---------------|
| Column distinct values (static) | Athena query | `SELECT DISTINCT col FROM table LIMIT 20` for low-cardinality columns |
| Business term mapping | Manual / LLM-generated | Curated glossary + LLM can suggest from column names |
| Join detection | Glue FK params + naming conventions | Parse `foreign_key_*` params + match `*_id` columns |
| Metrics | Manual curation | Domain experts define; stored separately |
| Golden queries | Manual + saved Athena queries | Verified subset of saved queries |
| Anti-patterns | Manual curation | Domain experts document common mistakes |
| Table grain | Manual / inferred | "One row per ___" |
| Approximate row counts | Athena query or Glue stats | `SELECT COUNT(*) FROM table` or table parameters |


