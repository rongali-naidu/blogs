# Schema Ontology: hr_db

## Domain Overview

| Attribute | Value |
|-----------|-------|
| Schema Name | hr_db |
| Description | Human Resources domain — employee lifecycle, organizational structure, and workforce analytics |
| Owner | hr-data-team |
| Source Systems | HRIS (Workday), Payroll System, Badge Access System |
| Business Process | Employee Onboarding → Assignment → Performance → Offboarding |
| Primary Users | HR Business Partners, People Analytics, Finance (headcount planning) |
| Refresh Frequency | Daily (T+1) |
| Data Classification | Confidential (contains PII) |

---

## Business Process Context

### Process Flow
```
Hiring → Onboarding → Department Assignment → Active Employment → Offboarding
                          ↓                         ↓
                    Location Assignment        Performance Reviews
                          ↓                         ↓
                    Badge Provisioning          Compensation Changes
```

### Source Applications

| Application | What It Provides | Tables Fed |
|-------------|-----------------|------------|
| Workday (HRIS) | Employee master data, org structure | employee_dtls, department_dtls |
| Facilities System | Office locations, capacity | location_dtls |
| Badge System | Physical access, attendance | (future: attendance_dtls) |

### Key Business Events Captured

| Event | Captured In | Key Columns |
|-------|-------------|-------------|
| Employee hired | employee_dtls | joining_date, emp_category |
| Department transfer | employee_dtls | dept_id (updated) |
| Region transfer | employee_dtls | region_id (new partition) |
| Employee exits | employee_dtls | status = 'INACTIVE' (future) |

---

## Classes (Tables)

### employee_dtls

| Attribute | Value |
|-----------|-------|
| Label | Employee |
| Description | A person employed by or contracted to the organization |
| Grain | One row per employee (current state) |
| Record Count | ~50,000 |
| Partition Key | region_id |
| Primary Key | id |
| SCD Type | Type 1 (overwrite) |

### department_dtls

| Attribute | Value |
|-----------|-------|
| Label | Department |
| Description | An organizational unit within the company |
| Grain | One row per department |
| Record Count | ~25 |
| Primary Key | id |

### location_dtls

| Attribute | Value |
|-----------|-------|
| Label | Location |
| Description | Physical office or facility |
| Grain | One row per location |
| Record Count | ~100 |
| Primary Key | id |

---

## Object Properties (Relationships / Joins)

| Relationship | From | To | Join Condition | Cardinality | Description |
|-------------|------|-----|----------------|-------------|-------------|
| belongs_to_department | employee_dtls | department_dtls | employee_dtls.dept_id = department_dtls.id | N:1 | Each employee works in one department |
| located_at | employee_dtls | location_dtls | employee_dtls.location_id = location_dtls.id | N:1 | Each employee is assigned to one office |
| department_located_at | department_dtls | location_dtls | department_dtls.location_id = location_dtls.id | N:1 | Each department has a headquarters location |

### Join Notes & Caveats

- For "department location": join department_dtls → location_dtls (NOT employee_dtls → location_dtls)
- employee → location is the employee's assigned office, which may differ from their department's HQ
- Some employees may have NULL location_id (remote workers) — use LEFT JOIN

---

## Data Properties (Key Columns with Business Meaning)

### emp_category (Employment Type)

| Value | Label | Definition |
|-------|-------|------------|
| PERM | Permanent | Full-time ongoing employment |
| TEMP | Temporary | Fixed-term contract employee |
| CONTR | Contractor | External third-party contractor |

### region_id (Geographic Region) — Partition Column

| Value | Label | Countries Included |
|-------|-------|--------------------|
| AMER | Americas | US, Canada, Brazil, Mexico |
| EMEA | Europe, Middle East & Africa | UK, Germany, France, UAE, South Africa |
| APAC | Asia Pacific | India, Japan, Australia, Singapore |

**Performance Note:** Always filter on region_id for partition pruning.

---

## Derived Concepts (Metrics)

### headcount

| Attribute | Value |
|-----------|-------|
| Display Name | Headcount |
| Synonyms | employee count, staff count, HC, FTE count |
| SQL | `COUNT(DISTINCT employee_dtls.id)` |
| Required Filters | `emp_category IN ('PERM', 'TEMP')` |
| Excluded | Contractors (CONTR) are NEVER included |
| Tables Required | employee_dtls |
| Grain | Can be grouped by: region_id, dept_id, joining_date |

### avg_tenure_years

| Attribute | Value |
|-----------|-------|
| Display Name | Average Tenure (Years) |
| Synonyms | average experience, years employed, avg tenure |
| SQL | `AVG(DATE_DIFF('year', employee_dtls.joining_date, CURRENT_DATE))` |
| Required Filters | `emp_category = 'PERM'` |
| Tables Required | employee_dtls |
| Notes | Only for permanent employees |

### department_size

| Attribute | Value |
|-----------|-------|
| Display Name | Department Size |
| Synonyms | dept headcount, team size |
| SQL | `COUNT(DISTINCT employee_dtls.id)` |
| Required Joins | `employee_dtls.dept_id = department_dtls.id` |
| Tables Required | employee_dtls, department_dtls |
| Grain | Grouped by department_dtls.name |

---

## Business Glossary (Domain-Specific Terms)

### Terms → SQL Mappings

| Business Term | Synonyms | Maps To |
|---------------|----------|---------|
| Permanent employee | FTE, full-time, perm | `emp_category = 'PERM'` |
| Contractor | vendor, external | `emp_category = 'CONTR'` |
| North America | NA, Americas, US region | `region_id = 'AMER'` |
| Europe | EU, EMEA region | `region_id = 'EMEA'` |
| Asia | APAC, Asia Pacific | `region_id = 'APAC'` |
| New hire | recent joiner, new employee | `joining_date >= CURRENT_DATE - INTERVAL '90 days'` |
| Veteran employee | long-tenured, senior employee | `joining_date <= CURRENT_DATE - INTERVAL '5 years'` |

### Acronyms

| Acronym | Full Form |
|---------|-----------|
| HC | Headcount |
| FTE | Full-Time Equivalent |
| HRIS | Human Resource Information System |
| YoY | Year over Year |
| MoM | Month over Month |
| HQ | Headquarters |

---

## Concept Hierarchies

### Geography

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

### Organization

```
Company
└── Department (department_dtls)
    └── Employee (employee_dtls)
```

---

## Constraints & Business Rules

| Rule | Applies To | SQL Expression | Reason |
|------|-----------|----------------|--------|
| Exclude contractors from headcount | headcount metric | `emp_category IN ('PERM', 'TEMP')` | Business definition |
| Partition pruning | employee_dtls queries | Always include `region_id = <value>` | Performance |
| Date format | joining_date filters | Use `DATE 'YYYY-MM-DD'` or `CAST('YYYY-MM-DD' AS DATE)` | Athena syntax |
| NULL locations | employee → location join | Use LEFT JOIN | Remote workers have no location |

---

## Anti-Patterns (Common Mistakes)

| Mistake | Why It's Wrong | Correct Approach |
|---------|---------------|-----------------|
| `emp_category = 'permanent'` | Values are abbreviated uppercase | `emp_category = 'PERM'` |
| `region_id = 'North America'` | Values are abbreviated codes | `region_id = 'AMER'` |
| `JOIN location_dtls` for department location | That gives employee's office, not dept HQ | Join `department_dtls → location_dtls` |
| Counting all emp_category for headcount | Includes contractors | Filter `IN ('PERM', 'TEMP')` |
| Missing partition filter | Full table scan, slow query | Always include `region_id` filter |
| `joining_date > '2024-01-01'` (string comparison) | Comparing string to date | `joining_date > DATE '2024-01-01'` |

---

## Sample Queries (Verified Golden Queries)

### Q1: Headcount by region
**Question:** How many permanent employees are in each region?
```sql
SELECT region_id, COUNT(DISTINCT id) AS headcount
FROM employee_dtls
WHERE emp_category = 'PERM'
GROUP BY region_id
```

### Q2: Employees by department with join
**Question:** List employees in the Engineering department who joined after 2024
```sql
SELECT e.name, e.joining_date, d.name AS department
FROM employee_dtls e
JOIN department_dtls d ON e.dept_id = d.id
WHERE d.name = 'Engineering'
  AND e.joining_date > DATE '2024-01-01'
```

### Q3: Department size comparison
**Question:** Which department has the most employees?
```sql
SELECT d.name AS department, COUNT(DISTINCT e.id) AS headcount
FROM employee_dtls e
JOIN department_dtls d ON e.dept_id = d.id
WHERE e.emp_category IN ('PERM', 'TEMP')
GROUP BY d.name
ORDER BY headcount DESC
LIMIT 1
```

