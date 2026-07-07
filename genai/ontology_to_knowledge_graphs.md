# From Raw Data to Knowledge Graphs : Schema, Metadata, Ontology, RDF, and Reasoning

# Introduction

These are my notes while trying to understand a fundamental question:**What exactly is an ontology?** 
Is it simply another way of representing a data model — similar to a schema with tables and relationships? Or does it represent something more?
And when we talk about a **Knowledge Graph**, is it just storing the same data in a different format, or is there a deeper shift in how we model information?
When people first enter the world of **Knowledge Graphs**, **Ontology**, **RDF**, and **Reasoning**, these concepts can feel confusing because they are closely related.

However, each one solves a different problem:

* **Schema** defines the structure of data.
* **Metadata** explains the meaning and context of data.
* **Ontology** defines concepts, relationships, and rules about a domain.
* **RDF** provides a standard way to represent information as connected facts.
* **Knowledge Graph** stores real-world entities and relationships using that model.
* **Reasoning** allows systems to derive new knowledge from existing facts.

A simple way to understand the journey is:

```
Raw Data
    ↓
Schema
    ↓
Metadata
    ↓
Ontology
    ↓
RDF Representation
    ↓
Knowledge Graph
    ↓
Reasoning
    ↓
New Knowledge
```

The goal of this article is to understand how we move from simply storing data to creating a representation of knowledge that machines can understand and reason about.


---

# Step 1: Raw Data (The Values)

Imagine a building security system stores this record:

```
101, John, Doe, 2026-05-12, True
```

At this stage, the computer only sees a sequence of values.

It does not know:

* Is `101` an employee ID or a building number?
* Is `2026-05-12` a joining date or a certification date?
* Does `True` mean the employee is active or the security badge is active?

The values exist, but their meaning is unknown.

Raw data answers:

> What values do we have?

It does not answer:

> What do these values represent?

---

# Step 2: Schema (The Structure)

Raw data contains values, but values alone are not useful without structure.

Now that we understand the need to organize raw information, let's see how a schema gives that data a defined shape.

A **schema** defines the structure of data.

It describes:

* Field names
* Data types
* Required fields
* Validation rules

Example:

| Column            | Data Type |
| ----------------- | --------- |
| EmployeeID        | Integer   |
| FirstName         | String    |
| LastName          | String    |
| CertificationDate | Date      |
| IsActive          | Boolean   |

Now the raw data becomes a structured record:

| EmployeeID | FirstName | LastName | CertificationDate | IsActive |
| ---------- | --------- | -------- | ----------------- | -------- |
| 101        | John      | Doe      | 2026-05-12        | True     |

The schema ensures:

* EmployeeID contains numeric values.
* CertificationDate contains valid dates.
* IsActive contains True or False.

The schema answers:

> How is the data organized?

However, it does not answer:

> What does this data mean from a business perspective?

---

# Step 3: Metadata (The Meaning and Context)

A schema tells us how data is organized, but it does not tell us what the data means in the real world.

Now that we understand how structure is created, let's explore how metadata adds business meaning and context to that structure.

Metadata is **data about data**.

A simple way to think about it:

> Schema describes the structure of a table. Metadata describes the story behind the table.

Imagine a data engineer discovers a table:

```
EmployeeCertification
```

The schema tells us:

| Column            | Type    |
| ----------------- | ------- |
| EmployeeID        | Integer |
| CertificationDate | Date    |
| IsActive          | Boolean |

But many questions remain:

* What certification does this date represent?
* Who owns this data?
* Which system created it?
* How frequently is it refreshed?
* What does "active" actually mean?

Metadata answers these questions.

## Table-Level Metadata

| Metadata             | Description                                                                                         |
| -------------------- | --------------------------------------------------------------------------------------------------- |
| Table Name           | EmployeeCertification                                                                               |
| Business Description | Stores employee hazardous materials certification information used by the building security system. |
| Data Owner           | Human Resources                                                                                     |
| Source System        | Building Security Application                                                                       |
| Refresh Frequency    | Daily                                                                                               |

## Column-Level Metadata

| Column            | Business Meaning                                                       |
| ----------------- | ---------------------------------------------------------------------- |
| EmployeeID        | Unique identifier assigned to each employee.                           |
| CertificationDate | Date when OSHA Hazardous Materials Handling Certification was granted. |
| IsActive          | Indicates whether the employee's security badge is enabled.            |

The schema says:

> CertificationDate is a Date.

Metadata says:

> CertificationDate represents when hazardous materials certification was granted.

Metadata connects technical structures with business understanding.

---

# Step 4: Ontology (The Semantic Model)

Metadata helps people understand individual datasets, but organizations usually have hundreds of datasets spread across many systems.

Different systems may describe the same thing differently:

* Employee
* Worker
* Staff Member
* Personnel

To create a shared understanding across systems, we need something more powerful.

This is where an **ontology** comes in.

An ontology is a formal model of a domain.

It defines:

* Types of things that exist
* Relationships between them
* Properties they have
* Rules about how they behave

Think of ontology as the **blueprint of the real world**.

---

## Classes (Types of Things)

Classes define categories.

Examples:

```
Person
Employee
Certification
Building
Department
EmergencyResponder
```

A class describes a type, not an individual record.

For example:

```
Employee
```

defines what an employee is.

It does not represent John yet.

---

## Relationships

Relationships describe how different types of things connect.

Examples:

```
Employee → worksIn → Department

Employee → hasCertification → Certification

Employee → reportsTo → Manager
```

Unlike a foreign key in a database, relationships are meaningful business concepts.

---

## Properties (Attributes)

Properties describe characteristics.

Examples:

```
Employee:
    employeeID
    firstName
    lastName

Certification:
    issueDate
    expirationDate
```

---

## Hierarchies

Ontologies can organize types using inheritance.

Example:

```
Person
   |
Employee
   |
EmergencyResponder
```

An EmergencyResponder is an Employee.

An Employee is a Person.

Therefore, EmergencyResponder inherits properties from both.

---

## Rules and Constraints

Ontologies can define logical meaning.

Example:

```
EmergencyResponder =
Employee
AND
Active Badge
AND
Valid Certification
```

This rule applies to every employee, not just one person.

---

# Step 5: RDF (Writing Graph Data)

An ontology gives us the concepts, relationships, and rules that describe our domain.

However, machines need a standard way to represent and exchange this knowledge.

This is where **RDF (Resource Description Framework)** becomes important.

RDF is a standard format for representing information as graph statements.

Instead of storing information as rows:

| EmployeeID | Name     | Certification      |
| ---------- | -------- | ------------------ |
| 101        | John Doe | OSHA Certification |

RDF represents information as triples:

```
Subject → Predicate → Object
```

Examples:

```
John Doe → hasCertification → OSHA Certification

John Doe → worksIn → Building A

John Doe → employeeID → 101
```

These statements naturally create a graph.

```
             hasCertification
John Doe --------------------> OSHA Certification

    |
    |
 worksIn

    ↓

Building A
```

RDF is the standard language used to write and exchange graph information.

---

# Step 6: Knowledge Graph (The Connected Data)

RDF provides the language for expressing connected facts, but those facts need to be stored and organized into a usable system.

A **knowledge graph** brings these facts together into a connected representation of real-world information.

The ontology defines the model.

The knowledge graph contains the actual data.

## Relational Database

Information is stored as rows:

```
Employee Table

101 | John Doe | OSHA Certification | Active
```

## Knowledge Graph

The same information is represented as connected objects:

```
          John Doe
             |
    --------------------
    |                  |
hasCertification     worksIn
    |                  |
    ↓                  ↓
OSHA Certification  Building A
```

A knowledge graph contains:

### Nodes

Things in the real world:

* People
* Organizations
* Products
* Locations
* Events

### Relationships

Connections between things:

* worksIn
* owns
* dependsOn
* hasCertification

The difference is:

Traditional database:

> What information belongs together in a record?

Knowledge graph:

> How are things connected?

---

# Step 7: Reasoning (Discovering New Knowledge)

A knowledge graph can store millions of facts and relationships.

However, the real power comes from applying the meaning defined by the ontology.

A reasoning engine can analyze these connections and discover new knowledge.

Existing facts:

```
John Doe is an Employee.

John Doe has an active badge.

John Doe has valid certification.
```

Ontology rule:

```
Employee
+
Active Badge
+
Valid Certification

=

EmergencyResponder
```

The reasoner concludes:

```
John Doe is an EmergencyResponder.
```

This information was not explicitly stored.

It was inferred from existing knowledge.

---

