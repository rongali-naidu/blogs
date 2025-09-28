# Data Transformation : Part 1: Marshalling & Unmarshalling

## What is Data Transformation?

Here, I am explaining **data transformation from a data engineering perspective** and also relating it to other domains that deal with data processing.

When we talk about **data transformation in data engineering**, we usually mean all the steps where data changes its shape, type, or format as it moves through the data pipeline.

But transformation plays a relevant role **beyond pipelines** as well — for example:

* as **marshalling/unmarshalling (serialization)** in software systems design,
* as **schema mapping** when syncing data between multiple systems (RDBMS ↔ NoSQL, APIs ↔ databases, etc.),
* or as **tokenization of data** as part of feature engineering in data science applications.

So, **Data Transformation** is a broad umbrella and can include:

* **Data type conversion** (e.g., string `"123"` → integer `123`)
* **Data cleaning** (handling nulls, removing duplicates, fixing inconsistencies)
* **Business logic transformations** (e.g., creating derived attributes or metrics like `price * qty` → `order_amount`)
* **Data aggregation** (e.g., computing *gross margin sales per product per day*)
* **Converting between formats** (CSV → Parquet, JSON → Avro, text → JSON)
* **Restructuring data** (flattening nested JSON, pivoting, denormalization)
* **Feature engineering** (e.g., deriving new attributes or metrics for Machine Learning / Data Science use cases)
* **Data standardization / normalization** (e.g., converting all timestamps to UTC, standardizing currencies to USD, normalizing text case `"NY"` → `"New York"`)
* **Data anonymization / masking / tokenization** (e.g., transforming sensitive PII fields for compliance with GDPR or HIPAA)
* **Data compression / encoding** (e.g., base64 encoding binary blobs, dictionary encoding categorical values for efficiency)

Different terms are often used for these steps — data cleaning, data aggregation, serialization, encoding, marshalling, casting, enrichment, feature engineering, etc. At the end of the day, they are all forms of *data transformation*, serving one goal: **making data usable in the next system of the pipeline**.

---

## Marshalling & Unmarshalling


* **Marshalling**: Converting an in-memory data structure or object into a format suitable for storage or transmission.
* **Unmarshalling**: Reconstructing the original data structure or object from that stored or transmitted format.
* Essentially, it's **serialization / deserialization**, but “marshalling” is often used in contexts where data is being sent **between systems**.

Here, I am focusing specifically on **Marshalling & Unmarshalling in AWS SDK for JavaScript**. We use this when working with **DynamoDB in Node.js/JavaScript applications**.
Why is this needed? Because there are **two different representations of the same data**:

### 1. **Your app world (plain JavaScript objects)**

You work with normal JSON-like structures:

```js
const user = {
  userId: "u123",
  age: 42,
  active: true,
  tags: ["dev", "aws"],
  address: { city: "Seattle", zip: "98101" }
};
```

### 2. **DynamoDB world (typed JSON schema)**

DynamoDB doesn’t store plain JSON. It requires every attribute to include a **type annotation**:

```json
{
  "userId": { "S": "u123" },
  "age": { "N": "42" },
  "active": { "BOOL": true },
  "tags": { "L": [ { "S": "dev" }, { "S": "aws" } ] },
  "address": {
    "M": {
      "city": { "S": "Seattle" },
      "zip": { "S": "98101" }
    }
  }
}
```

Here:

* `"S"` = string
* `"N"` = number (stored as string)
* `"BOOL"` = boolean
* `"L"` = list (array)
* `"M"` = map (nested object)

---

## Marshalling & Unmarshalling in Practice

The AWS SDK for JavaScript provides helpers in the package `@aws-sdk/util-dynamodb`:

* **`marshall(jsObject)`** → Converts JS objects into DynamoDB’s JSON format.
* **`unmarshall(ddbJson)`** → Converts DynamoDB JSON back into JS objects.

### Example

```js
import { marshall, unmarshall } from "@aws-sdk/util-dynamodb";

const item = {
  userId: "u123",
  age: 42,
  active: true,
  tags: ["dev", "aws"],
  address: { city: "Seattle", zip: "98101" }
};

// JS → DynamoDB JSON
const marshalled = marshall(item);
console.log("Marshalled:", JSON.stringify(marshalled, null, 2));

// DynamoDB JSON → JS
const unmarshalled = unmarshall(marshalled);
console.log("Unmarshalled:", unmarshalled);
```

---

## Do You Always Need Marshalling?

It depends on which client you use:

* **`DynamoDBClient` (low-level client)** → You must explicitly call `marshall`/`unmarshall`.
* **`DynamoDBDocumentClient` (high-level client)** → It handles marshalling automatically, so you can work directly with plain JS objects.

```js
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, PutCommand } from "@aws-sdk/lib-dynamodb";

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

// No manual marshalling needed
await docClient.send(new PutCommand({
  TableName: "Users",
  Item: { userId: "u123", age: 42 }
}));
```

---

## Takeaway

* **Marshalling/unmarshalling** is a specific, technical case of data transformation.
* Unlike business transformations (e.g., creating metrics, cleaning nulls), this is about **format conversion** between **native app types** and **DynamoDB’s typed JSON schema**.
* It reminds us that in data pipelines, transformation happens at multiple levels — from low-level structural conversions to high-level business rules.
