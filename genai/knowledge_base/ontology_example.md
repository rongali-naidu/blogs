# Schema Ontology: hr_db

## Domain

    Name: Human Resources
    Description: Employee lifecycle, organizational structure, and workforce analytics
    Owner: hr-data-team
    Source Systems: HRIS (Workday), Payroll System, Facilities Management
    Business Process: Hiring → Onboarding → Department Assignment → Active Employment → Offboarding
    Primary Users: HR Business Partners, People Analytics, Finance (headcount planning)
    Refresh Frequency: Daily (T+1)
    Data Classification: Confidential (contains PII)


## Classes

### Employee

    Definition: Individual employed by or contracted to the organization
    Table: employee_dtls
    Grain: One row per employee (current state)
    Record Count: ~50,000
    Partition: region_id
    Primary Key: id
    SCD Type: Type 1 (overwrite)

### Department

    Definition: Functional organizational unit within the company
    Table: department_dtls
    Grain: One row per department
    Record Count: ~25
    Primary Key: id

### Location

    Definition: Physical office, warehouse, or campus facility
    Table: location_dtls
    Grain: One row per location
    Record Count: ~100
    Primary Key: id


## Data Properties

### Employee Data Properties

    id: Unique employee identifier (INT, PK, not null)
    name: Full name of the employee (STRING, PII)
    age: Current age in years (INT, range: 18-70)
    emp_category: Employment classification (STRING, enumerated)
    dept_id: Department assignment (INT, FK → department_dtls.id)
    location_id: Office assignment (INT, FK → location_dtls.id)
    joining_date: Date employee started (DATE, format: YYYY-MM-DD, range: 2015-01-01 to current)
    region_id: Geographic region (STRING, partition key, enumerated)

### Department Data Properties

    id: Unique department identifier (INT, PK)
    name: Department name (STRING)
    location_id: Headquarters location (INT, FK → location_dtls.id)

### Location Data Properties

    id: Unique location identifier (INT, PK)
    name: Location/facility name (STRING)
    address: Physical address (STRING)


## Object Properties (Relationships)

    Employee belongs_to Department
        Join: employee_dtls.dept_id = department_dtls.id
        Cardinality: N:1
        Description: Each employee works in one department

    Employee located_at Location
        Join: employee_dtls.location_id = location_dtls.id
        Cardinality: N:1
        Join Type: LEFT (some remote workers have NULL location)
        Description: Each employee is assigned to one office

    Department headquartered_at Location
        Join: department_dtls.location_id = location_dtls.id
        Cardinality: N:1
        Description: Each department has a headquarters location

    Note: Employee location ≠ Department location. They are different concepts.


## Instances (Enumerated Values)

### emp_category Instances

    PERM: Permanent employee - full-time ongoing contract
    TEMP: Temporary employee - fixed-term contract
    CONTR: Contractor - external third-party resource

### region_id Instances (Partition Values)

    AMER: Americas - US, Canada, Brazil, Mexico
    EMEA: Europe, Middle East, and Africa - UK, Germany, France, UAE
    APAC: Asia Pacific - India, Japan, Australia, Singapore

    Performance Note: Always filter on region_id for partition pruning.


## Derived Concepts

### Headcount

    Definition: Total number of active employees (excludes contractors)
    Synonyms: employee count, staff count, HC, FTE count
    SQL: COUNT(DISTINCT employee_dtls.id)
    Required Filter: emp_category IN ('PERM', 'TEMP')
    Exclusion: Contractors (CONTR) are NEVER included
    Classes Involved: Employee
    Grain: Can be grouped by region_id, dept_id, joining_date

### Average Tenure

    Definition: Mean years of employment for permanent staff
    Synonyms: average experience, years employed, avg tenure
    SQL: AVG(DATE_DIFF('year', employee_dtls.joining_date, CURRENT_DATE))
    Required Filter: emp_category = 'PERM'
    Classes Involved: Employee
    Note: Only applicable to permanent employees

### Department Size

    Definition: Number of employees per department
    Synonyms: dept headcount, team size
    SQL: COUNT(DISTINCT employee_dtls.id)
    Required Join: employee_dtls.dept_id = department_dtls.id
    Classes Involved: Employee, Department
    Grain: Grouped by department_dtls.name


## Synonyms

### Class Synonyms

    Employee: staff, worker, team member, associate
    Department: dept, team, org unit, division
    Location: office, site, facility, campus

### Instance Synonyms (Term → SQL Mapping)

    Permanent employee / FTE / full-time → emp_category = 'PERM'
    Temporary / temp / fixed-term → emp_category = 'TEMP'
    Contractor / vendor / external → emp_category = 'CONTR'
    North America / NA / Americas → region_id = 'AMER'
    Europe / EU / EMEA → region_id = 'EMEA'
    Asia / APAC / Asia Pacific → region_id = 'APAC'
    New hire / recent joiner → joining_date >= CURRENT_DATE - INTERVAL '90 days'
    Veteran / long-tenured → joining_date <= CURRENT_DATE - INTERVAL '5 years'

### Acronyms

    HC: Headcount
    FTE: Full-Time Equivalent
    HRIS: Human Resource Information System
    YoY: Year over Year
    MoM: Month over Month
    HQ: Headquarters


## Hierarchies

### Geography Hierarchy

```
Region (region_id)
├── AMER
│   ├── US
│   ├── CA
│   ├── BR
│   └── MX
├── EMEA
│   ├── UK
│   ├── DE
│   ├── FR
│   └── AE
└── APAC
    ├── IN
    ├── JP
    ├── AU
    └── SG
```

### Organization Hierarchy

```
Company
└── Department (department_dtls)
    └── Employee (employee_dtls)
```


## Constraints

    Headcount excludes contractors: emp_category IN ('PERM', 'TEMP')
    Partition pruning required: Always include region_id in WHERE clause
    Date type enforcement: Use DATE 'YYYY-MM-DD' or CAST('YYYY-MM-DD' AS DATE)
    Nullable locations: LEFT JOIN for employee → location (remote workers)
    Location semantics: Employee location ≠ Department location


## Anti-Patterns

    ✗ emp_category = 'permanent'       → ✓ emp_category = 'PERM'
    ✗ region_id = 'North America'      → ✓ region_id = 'AMER'
    ✗ JOIN location for dept location   → ✓ Join department_dtls → location_dtls
    ✗ COUNT(*) for headcount           → ✓ COUNT(DISTINCT id) with category filter
    ✗ Missing partition filter          → ✓ Always include region_id
    ✗ joining_date > '2024-01-01'      → ✓ joining_date > DATE '2024-01-01'


## Axioms (Verified Queries)

### Headcount by region
    Question: How many permanent employees are in each region?
    SQL: SELECT region_id, COUNT(DISTINCT id) AS headcount
         FROM employee_dtls
         WHERE emp_category = 'PERM'
         GROUP BY region_id

### Cross-class query with Object Property traversal
    Question: List employees in Engineering who joined after 2024
    SQL: SELECT e.name, e.joining_date, d.name AS department
         FROM employee_dtls e
         JOIN department_dtls d ON e.dept_id = d.id
         WHERE d.name = 'Engineering'
           AND e.joining_date > DATE '2024-01-01'

### Derived Concept with aggregation
    Question: Which department has the most employees?
    SQL: SELECT d.name AS department, COUNT(DISTINCT e.id) AS headcount
         FROM employee_dtls e
         JOIN department_dtls d ON e.dept_id = d.id
         WHERE e.emp_category IN ('PERM', 'TEMP')
         GROUP BY d.name
         ORDER BY headcount DESC
         LIMIT 1

### Derived Concept with constraint
    Question: What is the average tenure of permanent employees in Americas?
    SQL: SELECT AVG(DATE_DIFF('year', joining_date, CURRENT_DATE)) AS avg_tenure_years
         FROM employee_dtls
         WHERE emp_category = 'PERM'
           AND region_id = 'AMER'
