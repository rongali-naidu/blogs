# Why One Iteration of a Nested Loop Join Can Be Slower Than a Direct Index Seek

Sometimes a seemingly simple SQL query with a join runs slower than expected, even when fetching just a single row, and with perfect indexing.

Let’s explore **why one iteration of a Nested Loop Join can be slower than a direct index seek** for a **single-row lookup**. We'll use an example where the **Employee table has a unique PayrollID referencing PayrollDetails**.

---

## The Scenario: Employee → PayrollDetails via PayrollID

* **Employee** table: stores basic employee info plus a **unique PayrollID** column.
* **PayrollDetails** table: stores payroll info keyed by PayrollID.

This is a **1:1 relationship** via **Employee.PayrollID = PayrollDetails.PayrollID**.

### Business Use Case:

> "Given an EmployeeNumber, fetch the payroll details linked by PayrollID."

---

## Example 1: Join-Based Lookup (Nested Loop Join)

```sql
SELECT p.Salary, p.BankAccountNumber
FROM Employee e
JOIN PayrollDetails p ON e.PayrollID = p.PayrollID
WHERE e.EmployeeNumber = 'EMP12345';
```

* Filters Employee by EmployeeNumber.
* Joins PayrollDetails on the unique PayrollID.
* SQL Server likely uses a **Nested Loop Join** with an index seek on PayrollDetails.

---

## Example 2: Variable-Based Direct Seek

```sql
DECLARE @PayrollID INT;

SELECT @PayrollID = PayrollID
FROM Employee
WHERE EmployeeNumber = 'EMP12345';

SELECT Salary, BankAccountNumber
FROM PayrollDetails
WHERE PayrollID = @PayrollID;
```

* Retrieves PayrollID into a variable.
* Performs a direct seek on PayrollDetails using the variable.

---

## Why Example 2 Often Runs Faster

* Both queries use indexes effectively.
* Both retrieve the same single payroll record.
* Yet, Example 2 often executes faster.

---

## The Real Reasons

### 1. Join Operator Overhead

* The join operator requires setting up iteration control, buffering, and row flow.
* Even for one row, this overhead exists.

### 2. Direct Seek Plan Simplicity

* The variable-based query compiles with a known constant (the variable’s value).
* SQL Server creates a streamlined plan focusing only on a direct seek.

### 3. Parameterized Seek vs Direct Seek

> When SQL Server compiles the query plan, it treats the seek on the inner table differently:

* **In the join query (Example 1), the seek on PayrollDetails is parameterized** — the actual PayrollID value isn’t known at compile time but passed dynamically for each outer row.

* This causes SQL Server to create a **more general plan** that works for any parameter value, but with slightly more overhead to handle the variability.

* **In the variable-based query (Example 2), the seek uses a constant value** (the variable’s value is fixed at runtime before execution).

* This lets SQL Server generate a **direct (constant) seek plan**, optimized specifically for that single value, with minimal overhead.

### 4. Flow Control Mechanics

* Join operator manages coordination between outer and inner inputs.
* The direct seek simply fetches the row.

---

## Execution Plan Difference

| Feature        | Join-Based (Example 1)               | Variable-Based (Example 2)    |
| -------------- | ------------------------------------ | ----------------------------- |
| Outer Input    | Seek on Employee                     | Seek on Employee              |
| Inner Input    | Parameterized Seek on PayrollDetails | Direct Seek on PayrollDetails |
| Operator       | Nested Loop Join                     | No Join (simple select)       |
| Predicate Type | Parameterized Seek                   | Constant Seek                 |
| Overhead       | Join iteration setup                 | Direct seek, no loop control  |

The join operator and parameterized seek add control flow overhead not present in the variable-based query.

