# Ontology Terminology Reference

## Core Concepts

### Domain

    Definition: The scope or subject area the ontology describes
    Example: "Human Resources", "Physical Access Control", "E-Commerce"
    Purpose: Bounds what the ontology covers and what is out of scope


### Class

    Definition: A category or type of thing that exists in the domain
    Also Known As: Entity, Concept, Type, Category
    Example: Employee, Department, Order, Product
    SQL Equivalent: Table
    Reference: [OWL 2 Primer — Classes and Instances](https://www.w3.org/TR/owl2-primer/#Classes_and_Instances)


### Instance (Individual)

    Definition: A specific member of a class; a concrete occurrence
    Also Known As: Individual, Entity Instance, Member
    Example: "John Smith" is an instance of Employee; "PERM" is an instance of emp_category
    SQL Equivalent: Row in a table, or a specific allowed value in an enum column
    Reference: [OWL 2 Primer — Classes and Instances](https://www.w3.org/TR/owl2-primer/#Classes_and_Instances)


### Data Property

    Definition: An attribute that relates an instance to a literal value (string, number, date)
    Also Known As: Attribute, Field, Characteristic
    Example: Employee has name (string), age (integer), joining_date (date)
    SQL Equivalent: Column with a primitive data type
    Reference: [OWL 2 Primer — Data Properties](https://www.w3.org/TR/owl2-primer/#Data_Properties)


### Object Property

    Definition: A relationship that connects one instance to another instance (not a literal value)
    Also Known As: Relationship, Association, Link
    Example: Employee "belongs_to" Department; Order "placed_by" Customer
    SQL Equivalent: Foreign Key / JOIN relationship
    Reference: [OWL 2 Primer — Object Properties](https://www.w3.org/TR/owl2-primer/#Object_Properties)


### Annotation

    Definition: Human-readable metadata attached to any ontology element (label, comment, description)
    Also Known As: Label, Comment, Description, Documentation
    Example: Column COMMENT 'Employee category. Contains TEMP, PERM, CONTR'
    SQL Equivalent: Column/table COMMENT or description
    Reference: [OWL 2 Primer — Annotation Properties](https://www.w3.org/TR/owl2-primer/#Annotation_Properties)


### Axiom

    Definition: A statement asserted to be true within the ontology; a logical rule that always holds
    Also Known As: Rule, Assertion, Fact, Truth
    Example: "Every headcount calculation excludes contractors"
    SQL Equivalent: A verified golden query; a business rule enforced in every query
    Reference: [OWL 2 Primer — Axioms](https://www.w3.org/TR/owl2-primer/#Axioms)


### Constraint (Restriction)

    Definition: A condition that limits the values or relationships a property can have
    Also Known As: Restriction, Validation Rule, Invariant
    Example: "age must be between 18 and 70"; "emp_category can only be PERM, TEMP, or CONTR"
    SQL Equivalent: CHECK constraint, WHERE clause, allowed values
    Reference: [OWL 2 Primer — Property Restrictions](https://www.w3.org/TR/owl2-primer/#Property_Restrictions)


### Hierarchy (SubClass Relationship)

    Definition: A parent-child relationship between classes where the child inherits properties of the parent
    Also Known As: Taxonomy, Inheritance, Is-A relationship, Subsumption
    Example: "APAC" is a subclass of "Region"; "Laptop" is a subclass of "Electronics"
    SQL Equivalent: Category hierarchy (dimension levels in analytics)
    Reference: [OWL 2 Primer — Class Hierarchies](https://www.w3.org/TR/owl2-primer/#Class_Hierarchies)


### Synonym (Equivalent Labels)

    Definition: Multiple names for the same concept
    Also Known As: Alias, Alternative Label, Equivalent Term
    Example: "Revenue" = "GMS" = "Gross Merchandise Sales"
    SQL Equivalent: Business term that maps to the same column/expression
    Reference: [SKOS Primer — Labels](https://www.w3.org/TR/skos-primer/#seclabel)


### Derived Concept (Defined Class)

    Definition: A concept defined by a logical expression over other concepts, not stored directly
    Also Known As: Computed Concept, Calculated Field, Virtual Property
    Example: "Active Customer" defined as customer with order in last 90 days
    SQL Equivalent: Metric definition (formula + required filters)
    Reference: [OWL 2 Primer — Defined Classes](https://www.w3.org/TR/owl2-primer/#Defined_Classes)


### Cardinality

    Definition: The number of values a property can have (how many relationships are allowed)
    Types:
        1:1 — One to one (each employee has exactly one badge)
        1:N — One to many (one department has many employees)
        N:1 — Many to one (many employees belong to one department)
        N:M — Many to many (employees can be in multiple projects)
    SQL Equivalent: JOIN cardinality; determines if JOIN causes row fan-out
    Reference: [OWL 2 Primer — Property Cardinality Restrictions](https://www.w3.org/TR/owl2-primer/#Property_Cardinality_Restrictions)


### Domain and Range

    Definition: 
        Domain = which class a property belongs to (the subject)
        Range = what type of value the property points to (the object)
    Example: 
        Property "belongs_to" has Domain: Employee, Range: Department
        Property "age" has Domain: Employee, Range: Integer
    SQL Equivalent:
        Domain = which table owns the column
        Range = target table (FK) or data type (INT, STRING, DATE)
    Reference: [OWL 2 Primer — Domain and Range Restrictions](https://www.w3.org/TR/owl2-primer/#Domain_and_Range_Restrictions)


### Disjoint Classes

    Definition: Classes that cannot share instances (an individual cannot be both)
    Also Known As: Mutually Exclusive Categories
    Example: "Permanent" and "Contractor" are disjoint — an employee cannot be both
    SQL Equivalent: Column value is exclusive (emp_category can be PERM or CONTR, not both)
    Reference: [OWL 2 Primer — Disjoint Classes](https://www.w3.org/TR/owl2-primer/#Disjoint_Classes)


### Inverse Property

    Definition: The reverse direction of a relationship
    Also Known As: Reverse Relationship, Bidirectional Link
    Example: "belongs_to" (Employee → Department) has inverse "has_members" (Department → Employee)
    SQL Equivalent: Same JOIN read from either direction
    Reference: [OWL 2 Primer — Inverse Properties](https://www.w3.org/TR/owl2-primer/#Inverse_Properties)


### Transitive Property

    Definition: If A relates to B and B relates to C, then A relates to C
    Example: "reports_to" — if Alice reports to Bob and Bob reports to Carol, Alice indirectly reports to Carol
    SQL Equivalent: Recursive/hierarchical query (WITH RECURSIVE or CONNECT BY)
    Reference: [OWL 2 Primer — Property Characteristics](https://www.w3.org/TR/owl2-primer/#Property_Characteristics)


## Formal Standards

### W3C Specifications

- [OWL 2 Web Ontology Language Primer](https://www.w3.org/TR/owl2-primer/) — Comprehensive introduction to all OWL 2 concepts
- [RDF 1.1 Primer](https://www.w3.org/TR/rdf11-primer/) — Foundation data model (subject-predicate-object triples)
- [RDFS (RDF Schema)](https://www.w3.org/TR/rdf-schema/) — Basic vocabulary for describing classes and properties
- [SKOS (Simple Knowledge Organization System)](https://www.w3.org/TR/skos-primer/) — Standard for synonyms, labels, broader/narrower relationships
- [SPARQL 1.1 Query Language](https://www.w3.org/TR/sparql11-query/) — Query language for RDF graphs (equivalent of SQL for knowledge graphs)
- [OWL 2 Quick Reference Guide](https://www.w3.org/TR/owl2-quick-reference/) — Concise summary of OWL 2 constructs


### Textbooks & Papers

- [Knowledge Graphs — Hogan et al. (2021)](https://arxiv.org/abs/2003.02320) — Comprehensive survey covering definitions, creation, and applications
- [Ontology Engineering — Keet (2020)](https://people.cs.uct.ac.za/~mkeet/OEbook/) — Formal ontology design patterns and methodology
- [A Semantic Web Primer — Antoniou & van Harmelen](https://mitpress.mit.edu/9780262018289/a-semantic-web-primer/) — Practical introduction to OWL, RDF, SPARQL
- [Knowledge Representation and Reasoning — Brachman & Levesque](https://www.elsevier.com/books/knowledge-representation-and-reasoning/brachman/978-0-12-382227-5) — Theoretical foundations


### Tools

- [Protégé (Stanford)](https://protege.stanford.edu/) — Open-source ontology editor; good for learning OWL structure
- [Schema.org](https://schema.org/) — Lightweight real-world ontology used by Google, Bing, Yahoo
- [WebVOWL](http://vowl.visualdataweb.org/webvowl.html) — Visual notation for OWL ontologies
- [OWL Validator](http://mowl-power.cs.man.ac.uk:8080/validator/) — Validate OWL ontologies online


### Applied / Text-to-SQL Relevant

- [Ontology-Based Data Access (OBDA)](https://www.w3.org/2012/ldp/wiki/Ontology_Based_Data_Access) — W3C pattern for querying relational data through ontology layer
- [R2RML: RDB to RDF Mapping Language](https://www.w3.org/TR/r2rml/) — W3C standard for mapping relational databases to ontologies
- [Virtual Knowledge Graphs (Xiao et al.)](https://link.springer.com/chapter/10.1007/978-3-030-49461-2_30) — Virtualizing relational data as knowledge graphs without data duplication
- [BIRD Benchmark](https://bird-bench.github.io/) — Text-to-SQL benchmark requiring external knowledge (domain terms, value meanings)
- [Spider Benchmark](https://yale-lily.github.io/spider) — Cross-database text-to-SQL evaluation


## Mapping Summary: Ontology → SQL → This Document

| Ontology Term | SQL Concept | Schema Ontology Section |
|---------------|-------------|------------------------|
| Domain | Schema/Database | Domain |
| Class | Table | Classes |
| Instance | Row / Enum value | Instances |
| Data Property | Column (primitive type) | Data Properties |
| Object Property | Foreign Key / JOIN | Object Properties |
| Annotation | COMMENT / Description | Within each section |
| Axiom | Verified query / Business rule | Axioms |
| Constraint | CHECK / WHERE filter | Constraints |
| Hierarchy | Dimension levels | Hierarchies |
| Synonym | Business term alias | Synonyms |
| Derived Concept | Metric formula | Derived Concepts |
| Cardinality | JOIN type (1:N, N:1) | Object Properties |
| Disjoint | Mutually exclusive values | Constraints |
| Anti-Pattern | Invalid assumption | Anti-Patterns |
