# **Optimizing JSON Ingestion for Data Lakes: From Raw to Query-Ready Format**

## **Introduction**

Handling JSON data efficiently is a critical aspect of modern data lakes. Raw JSON files often contain nested structures, special characters in keys, and inconsistent formatting that can cause issues when querying them with tools like **AWS Athena, Redshift Spectrum, or Apache Spark**. Many teams try to flatten JSON **before** ingesting it into the data lake, which can lead to limitations when new fields emerge.

A more scalable approach is to **ingest both raw and cleaned JSON into the data lake** and then flatten it dynamically based on analytical needs. This ensures that no data is lost due to premature flattening and allows for flexible schema evolution. However, note that raw JSON data might not always be directly linkable to Glue tables due to possible schema inference issues caused by inconsistent structures and special characters.

In this blog, we’ll explore:

- **Common challenges with JSON ingestion**
- **Why pre-flattening JSON before ingestion is not always ideal**
- **A Python function to clean JSON for Athena compatibility**
- **How to efficiently process JSON within the data lake using multiple options**
- **Why storing both raw and cleaned JSON is beneficial**

---

## **Common Challenges with JSON Ingestion**

### **1. Athena-Compatible JSON Format**

AWS Athena requires **JSON records to be newline-delimited**, meaning each JSON object should be on a separate line without being wrapped in an array.

❌ **Incorrect format:**

```json
[
  {"name": "John", "age": 30},
  {"name": "Alice", "age": 25}
]
```

✅ **Correct format:**

```json
{"name": "John", "age": 30}
{"name": "Alice", "age": 25}
```

❌ **Another incorrect format (multiple records on the same line):**

```json
{"name": "John", "age": 30}  {"name": "Alice", "age": 25}
```

This causes Athena to only recognize the first record while ignoring the second.

### **2. Special Characters in JSON Keys**

Some JSON sources may contain special characters (e.g., spaces, dots, colons, or backslashes) in keys, which cause issues when querying in Athena or Redshift. Additionally, AWS Glue Crawlers may misinterpret column names due to these characters, leading to incorrect schema inference.

❌ **Problematic JSON:**

```json
{"user name": "John", "user-info": {"email.id": "john@example.com", "role:type": "admin", "path\\to\\file": "data.csv"}}
```

✅ **Fixed JSON:**

```json
{"user_name": "John", "user_info": {"email_id": "john@example.com", "role_type": "admin", "path_to_file": "data.csv"}}
```

### **3. Flattening Before Ingestion Can Be Restrictive**

Flattening JSON **before ingestion** requires deciding upfront which fields to extract. If new fields appear later, reprocessing the entire dataset is necessary. Instead, we should ingest the full JSON and flatten it **on-demand** using tools like **Spark, Athena, or Redshift Spectrum**.

---

## **Solution: Cleaning JSON for Efficient Ingestion**

Here’s a Python function to clean JSON before ingestion:

```python
import json
import re

def clean_json_keys(obj):
    """Recursively clean JSON keys to replace special characters with underscores."""
    if isinstance(obj, dict):
        new_obj = {}
        for key, value in obj.items():
            clean_key = re.sub(r'[^a-zA-Z0-9_]', '_', key)
            new_obj[clean_key] = clean_json_keys(value)
        return new_obj
    elif isinstance(obj, list):
        return [clean_json_keys(item) for item in obj]
    else:
        return obj

# Example usage
raw_json = '[{"user name": "John", "user-info": {"email.id": "john@example.com", "role:type": "admin", "path\\to\\file": "data.csv"}}]'
parsed_json = json.loads(raw_json)
cleaned_json = [clean_json_keys(item) for item in parsed_json]
print("\n".join(json.dumps(record) for record in cleaned_json))
```

This function:
✅ Replaces **spaces, dots, colons, backslashes, and hyphens** in keys with underscores.
✅ Works **recursively** for deeply nested JSON.
✅ Ensures the JSON is **Athena-compatible** before ingestion.
✅ Converts an **array of JSON objects into newline-delimited JSON records**.

---

## **Why Store Both Raw and Cleaned JSON in the Data Lake?**

- **Raw JSON helps in debugging**: If data processing introduces errors, raw JSON provides a reference point to identify discrepancies.
- **Schema evolution support**: If new fields emerge in the source data, they remain available in raw JSON for future processing.
- **Flexible querying**: Cleaned JSON allows easier querying while preserving the original data for any reprocessing needs.
- **Note:** Raw JSON may not always be linkable to Glue tables due to potential schema inference issues.

---

## **Processing JSON After Ingestion**

Once the cleaned JSON is stored in the data lake, we can flatten it dynamically using multiple methods:

### **1. Using AWS Athena**

```sql
SELECT user_name, user_info.email_id, user_info.role_type FROM json_table;
```

### **2. Using Apache Spark (PySpark)**

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col

spark = SparkSession.builder.appName("JSONProcessing").getOrCreate()
df = spark.read.json("s3://your-bucket/json-data/")
df.select(col("user_name"), col("user_info.email_id"), col("user_info.role_type")).show()
```

### **3. Using Redshift Spectrum**

```sql
SELECT user_name, user_info.email_id, user_info.role_type FROM spectrum.json_table;
```

---

## **Key Takeaways**

✔️ **Ingest JSON in raw and cleaned format** instead of flattening upfront.
✔️ **Use a cleaning function** to fix special characters in JSON keys and convert arrays into newline-delimited records.
✔️ **Query JSON dynamically** using Athena, Spark, or Redshift Spectrum.
✔️ **Ensure schema flexibility** by processing JSON at query time.

This approach provides a scalable and flexible way to handle JSON in data lakes while avoiding schema rigidity. 🚀

**What are your strategies for handling JSON in data lakes? Share your thoughts below!**

