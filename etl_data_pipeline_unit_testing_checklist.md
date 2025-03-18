# ETL/Data Pipeline Unit Testing Checklist

## **Unit of Code:**
An ETL/Data Pipeline Job (Extract, Transform, and Load)

## **What Needs to be Tested in Unit Testing?**

### **1. Row-Level Testing:**
Focuses on the correct processing of individual data records.

#### **Data Extraction:**

- **Logic Verification:**
    - **Last Updated-Based Logic:** Does the pipeline correctly extract records where `last_updated_timestamp > last_run_timestamp`?
      - **Example:** Create a test scenario where a record has a `last_updated_timestamp` newer than the simulated last run time and another that doesn't. Verify only the newer record is extracted.
    - **Event Date-Based Logic:** Does the pipeline extract records from a log file where the `event_date` falls within a specific date range (e.g., between '2025-03-10' and '2025-03-16')?
      - **Example:** Create log entries with dates inside and outside the specified range. Verify only the entries within the range are extracted.

- **Filter Accuracy:**
    - **Case Sensitivity:** Does the pipeline correctly filter records where `product_category = 'Electronics'` (case-sensitive) and ignore records with `product_category = 'electronics'`?
      - **Example:** Include records with both 'Electronics' and 'electronics' in the source and confirm only the uppercase version is filtered.
    - **Value Lists:** Does the pipeline correctly filter records where `status` is in ('Shipped', 'Delivered')?
      - **Example:** Include records with 'Shipped', 'Delivered', and 'Pending' status and verify only the first two are extracted.
    - **Date Conversion:** Does the pipeline correctly convert a date string like '20250317' from the source to a `DATE` data type in the staging area?
      - **Example:** Provide a record with the date in the specified string format and verify it's converted to a proper date type.

#### **Data Transformations:**

- **Missing Data Handling:**
    - Does the pipeline replace null values in the `customer_name` field with 'Unknown' or skip records with missing `order_amount`?
      - **Example:** Provide records with null values and verify the expected handling.

- **Data Type Conversions:**
    - Does the pipeline convert a string representing a price (e.g., '$10.99') to a `DECIMAL` data type?
      - **Example:** Provide a record with the price in string format and verify it's correctly converted.

- **Derived Fields and Computed Columns:**
    - Does the pipeline correctly calculate `total_price = quantity * unit_price`?
      - **Example:** Provide records with different quantities and unit prices and verify the `total_price` calculation.

- **Data Aggregations:**
    - Are summary tables correctly aggregating data (e.g., summing `order_amount` by `customer_id`)?
      - **Example:** Provide records with multiple orders per customer and verify the aggregated output.

- **Data Truncation:**
    - Are string fields truncated to the expected length (e.g., `VARCHAR(50)`)?
      - **Example:** Insert a record exceeding the field length and verify proper truncation or error handling.

#### **Data Loading:**

- **Insert/Merge Logic:**
    - **Inserts:** Are new records loaded as expected?
    - **Updates:** Are changed fields updated in existing records?
      - **Example:** Provide test cases with new and updated records to validate insertion and merging.

- **Truncate/Partition Handling:**
    - For truncate operations, does the pipeline clear the partition before reloading?
      - **Example:** Verify that older records are removed before new records are loaded.

- **Insert-Only and Merge-Only Modes:**
    - Does the pipeline support specific insert-only and merge-only configurations?

#### **Aggregate Testing:**

- **Row Count Validation:** Ensure the correct number of records are processed.
- **Duplicate Checks:** Verify that duplicate records are not inserted.
- **Metric Summation:** Confirm sum aggregation calculations are correct.

### **2. Performance Testing:**

- **Data Volumes:** Validate the pipeline can process expected data volumes efficiently.
- **Run Time Analysis:** Measure execution times of key components.
- **Parameter Optimization:** Ensure configurable parameters (e.g., batch size, partition strategy) are optimized.

### **3. Integration Testing:**

#### **Scope of Integration Testing:**
- Validate end-to-end flow from source to target.
- Include all intermediate stages:
    - Source data extraction
    - Staging tables
    - Base/reference tables
    - Aggregation tables

#### **What to Validate in Integration Testing?**

- **Row-Wise Comparison:** Perform a `MINUS` operation between the source and target.
- **Referential Integrity:** Ensure all foreign keys match the respective dimensions or lookup tables.

### **4. System Testing:**

#### **Scope of System Testing:**
- Validate the entire pipeline, including downstream consumers:
    - Reporting systems (Dashboards/BI tools)
    - Machine learning applications

#### **What to Validate in System Testing?**

- **Report Accuracy:** Ensure that report outputs match the underlying data.
- **Downstream Data Integrity:** Confirm that all downstream applications consume data correctly.
- **Metric Consistency:** Ensure consistent metric calculations across all data outputs.

### **5. Code and Design Review Considerations:**

These aspects are typically covered during code/design review and are not included in unit testing:

- **Join Logic:** Validate the correctness of join columns and strategies (e.g., inner join vs. left outer join).
- **Grain Consistency:** Ensure aggregation granularity is aligned across datasets.
- **Metric Definitions:** Verify consistent metric definitions across datasets and reporting layers.

