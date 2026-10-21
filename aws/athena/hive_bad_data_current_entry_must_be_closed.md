# Debugging "HIVE\_BAD\_DATA: Error Parsing a column in the table: Current entry must be closed before a null can be written" in Athena

When querying deeply nested JSON data in AWS Athena, encountering the following error can be frustrating and challenging to resolve:

```
HIVE_BAD_DATA: Error Parsing a column in the table: Current entry must be closed before a null can be written
```


### Why I'm Writing This

While working through this error, I found that there are various versions of the HIVE_BAD_DATA error, but the available documentation did not cover this specific case in detail. It took considerable time to identify the root cause and pinpoint the bad records.This blog outlines a step-by-step approach to diagnose and resolve this error when querying an S3-backed AWS Glue table with deeply nested JSON data. If you have alternative solutions, feel free to share them in the comments.

## Background

Consider the following scenario:

- You have an S3 bucket where JSON data is stored.
- AWS Glue Crawler is used to infer the schema and create a table (`json_data_table`).
- The table structure includes:
  - `unique_key_value`: Unique identifier for each record.
  - `attribute1`: A standard attribute.
  - `attribute2`: A `struct` field that contains six levels of nested fields.

When running a simple query like:

```sql
SELECT * FROM json_data_table;
```

you encounter the `HIVE_BAD_DATA` error, despite attempts to resolve it using various options.

## Preliminary Checks Before Isolating the Record

1. **Reviewed AWS Documentation:**

   - Checked the AWS Troubleshooting Documentation on [HIVE\_BAD\_DATA: Error parsing field value](https://docs.aws.amazon.com/athena/latest/ug/troubleshooting-athena.html#troubleshooting-athena-hive_bad_data-error-parsing-field-value).

2. **Tried ****`ignore.malformed.json=true`****:**

   - Added `ignore.malformed.json=true` both under SerDe properties and Table properties. However, the error persisted.
   - As per the [OpenX JSON SerDe documentation](https://docs.aws.amazon.com/athena/latest/ug/openx-json-serde.html), this option should allow skipping malformed JSON, but it did not resolve the issue.

3. **Understanding the Error:**

   - The SerDe parses each JSON record from the S3 file and checks for schema compliance against the Glue table definition. If a record doesn't match the schema, it's considered malformed. depending on the malformed settings, it will substitute NULL or thrown an error.
   - The "closed" reference in the error points to the placement of `}` when parsing the JSON structure. The error suggests that the JSON structure is improperly closed before SerDe can decide wether to substitue NULL or not depending on malformed settings
   - Despite enabling `ignore.malformed.json`, the SerDe did not substitute `NULL` for the malformed records. I raised an issue to track this behavior: [Hive-JSON-Serde Issue #242](https://github.com/rcongiu/Hive-JSON-Serde/issues/242).

## Step 1: Isolating the Problematic Column

To identify the problematic column:

1. **Run a Query Selecting Only the Unique Key:**

```sql
SELECT unique_key_value FROM json_data_table;
```

If this query works without error, the issue lies within other columns.

2. **Iterate Over Columns in Batches:**
   - Select columns in small groups to identify the faulty one.

Example:

```sql
SELECT unique_key_value, attribute1 FROM json_data_table;
```

If successful, expand the selection to include other fields (e.g., `attribute2`) until you trigger the error. This helps pinpoint the exact column causing the problem.

In my case, `attribute2` (a `struct` type) was the problematic column. Further column-level isolation within `attribute2` did not work due to its deep nesting.

## Step 2: Isolating the Problematic Row

Once you identify the problematic column, focus on finding the specific row causing the error.

1. **Retrieve a List of Unique Keys:**

```sql
SELECT unique_key_value FROM json_data_table;
```

2. **Query the Table in Batches Using These Unique Keys:**

Example (with a batch size of 100):

```sql
SELECT * FROM json_data_table WHERE unique_key_value IN ('key1', 'key2', ..., 'key100');
```

Repeat this process. Once you identify a batch containing the problematic record, narrow down with in the batch to locate the exact record causing the error. In my case, I found the faulty record by the third batch manually.

### Alternative Approach:

Use a Python script to automate the search for bad records. For each unique key, execute a query and capture records causing errors.

Example using `boto3`:

```python
import boto3

client = boto3.client('athena')

def check_record(key):
    query = f"SELECT * FROM json_data_table WHERE unique_key_value = '{key}'"
    response = client.start_query_execution(
        QueryString=query,
        QueryExecutionContext={'Database': 'your_database'},
        ResultConfiguration={'OutputLocation': 's3://your-output-bucket/'}
    )
    return response

for key in unique_keys:
    try:
        check_record(key)
    except Exception as e:
        print(f"Error with key {key}: {e}")
```

## Step 3: Analyzing the Bad Record

Once you identify the unique key of the faulty record:

1. **Download the Original JSON File from S3:**

   - Search for the record by its unique key value.

2. **Compare the Fields of the Bad Record Against the Glue Table Schema:**

In my case, the root cause was that a nested field (at level 5 within `attribute2`) had a **string** value, while the Glue schema expected a **struct**.

### Example of the Mismatched Data

**Expected (from Glue schema):**

```json
{
  "nested_field": {
    "sub_field": {
      "value": "abc"
    }
  }
}
```

**Actual (in JSON record causing error):**

```json
{
  "nested_field": "unexpected_string"
}
```

## Step 4: Fixing the Issue

### Option 1: Clean the Bad Data

- Identify and correct malformed JSON records in your S3 data source.

### Option 2: Partially Flatten Nested Fields

- Only change the specific nested field causing issues to a `STRING` type, preserving most of the original schema.

Update the Glue table schema manually and configure the crawler to prevent schema overwrites.

### Option 2: Modify the Schema to Use `STRING`

- Change the problematic `struct` field to a `STRING` type. This allows malformed JSON to be read as-is but sacrifices nested querying capabilities.

To prevent Glue from overwriting this schema change, update the table manually and disable schema updates in the crawler.









