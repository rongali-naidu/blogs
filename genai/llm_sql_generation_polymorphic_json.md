# Polymorphic JSON in DynamoDB : Challenges for Glue, Athena, and AI-Based SQL Generation

Modern event-driven systems often use DynamoDB to capture user actions, operational events, or audit logs. DynamoDB’s schema-less design enables rapid iteration and flexibility at write time. However, that same flexibility introduces challenges when this data flows into data lakes, analytics engines, or AI-based data discovery systems.

In this post, we explore a **real-world DynamoDB Streams example**, explain **why polymorphic JSON is stored as STRING**, how Glue interprets it, provide **Athena SQL examples**, and describe **the challenges for LLM-based SQL generation**.

---

## 1. The Event Use Case

Imagine a system where a single producer captures user actions:

* Each event has an `actionName` describing the type of action.
* Each event has metadata in `parameters`, whose structure depends on the action.
* Examples of actions:

  * `LOGIN`: contains IP, device type, expiration timestamp.
  * `PURCHASE`: contains orderId, amount, currency, expiration timestamp.

### The Polymorphic Challenge

The `parameters` field is **heterogeneous**:

```text
parameters =
  if actionName == LOGIN    → { ip, device, expiretimestamp }
  if actionName == PURCHASE → { orderId, amount, currency, expiretimestamp }
```

DynamoDB has no native support for union types or conditional schemas, so a flexible storage approach is needed.

---

## 2. How the TypeScript Producer Handles Polymorphism

In the TypeScript producer, the `parameters` object is **marshalled into DynamoDB AttributeValue JSON**:

```ts
import { marshall } from "@aws-sdk/util-dynamodb";

const parameters = {
  expiretimestamp: 1823576851,
  ip: "1.2.3.4",
};

const marshalled = marshall(parameters);
/* marshalled =
{
  expiretimestamp: { N: "1823576851" },
  ip: { S: "1.2.3.4" }
} 
*/

item.parameters = {
  S: JSON.stringify(marshalled)
};
```

**Key points:**

* `parameters` is stored as a STRING in DynamoDB.
* Its value is a serialized **DynamoDB AttributeValue JSON**, containing `S`, `N`, or nested `M` types.
* This preserves type fidelity and enables replay of events.

This is why, when you look at the DynamoDB console, `parameters` appears like:

```json
"{\"requestByUser\":{\"S\":\"alice\"},\"expiretimestamp\":{\"N\":\"1823576851\"},\"ip\":{\"S\":\"1.2.3.4\"}}"
```

---

## 3. How Data Becomes `dynamodb.NewImage` and Glue Table Schema

### DynamoDB Streams

When data is captured in DynamoDB Streams:

* Each event record contains `NewImage` (after insert/update) or `OldImage` (before delete/update).
* All attributes use **DynamoDB AttributeValue JSON**, e.g., `{ "S": "string" }` or `{ "N": "123" }`.

Example Streams record:

```json
{
  "eventSource": "aws:dynamodb",
  "dynamodb": {
    "NewImage": {
      "actionName": { "S": "LOGIN" },
      "username": { "S": "alice" },
      "parameters": {
        "S": "{\"ip\":{\"S\":\"1.2.3.4\"},\"device\":{\"S\":\"ios\"},\"expiretimestamp\":{\"N\":\"1823576851\"}}"
      },
      "actionTimestamp": { "S": "2024-01-10T10:15:00Z" }
    }
  }
}
```

* `dynamodb.NewImage` is therefore a **nested struct**: top-level attributes as structs, and the polymorphic `parameters` as a string.
* Glue Crawlers use this structure to infer table schema.

---

### Glue Table Schema

Based on DynamoDB Streams, Glue infers:

```sql
dynamodb struct<
    NewImage: struct<
        actionName: struct<S:string>,
        username: struct<S:string>,
        parameters: struct<S:string>,
        actionTimestamp: struct<S:string>
    >
>
eventName string
eventSource string
```

**Observations:**

* `parameters.S` is typed as **STRING** because Glue cannot infer a consistent nested schema.
* Any manual description or nested type is overwritten by the crawler.
* Glue column descriptions are limited to **250 characters**, making it impossible to fully document polymorphic keys.

---

## 4. Athena SQL Examples

### a) Extracting DynamoDB AttributeValue JSON Fields

```sql
SELECT
  dynamodb.NewImage.actionName.S AS actionName,
  dynamodb.NewImage.username.S AS username,
  
  json_extract_scalar(
    trim('"' FROM regexp_replace(dynamodb.NewImage.parameters.S, '\\\\', '')),
    '$.requestByUser.S'
  ) AS requestByUser,

  cast(
    json_extract_scalar(
      trim('"' FROM regexp_replace(dynamodb.NewImage.parameters.S, '\\\\', '')),
      '$.expiretimestamp.N'
    ) AS bigint
  ) AS expire_ts_bigint

FROM glue_dynamodb_table
WHERE dynamodb.NewImage.actionName.S = 'LOGIN'
LIMIT 10;
```

---

### b) Handling Polymorphic Fields: Separate Queries per Action Type

Since `parameters` varies by `actionName`, each action type requires a **distinct query**.

#### LOGIN action:

```sql
SELECT
  dynamodb.NewImage.username.S AS username,
  json_extract_scalar(
    trim('"' FROM regexp_replace(dynamodb.NewImage.parameters.S, '\\\\', '')),
    '$.ip.S'
  ) AS ip,
  json_extract_scalar(
    trim('"' FROM regexp_replace(dynamodb.NewImage.parameters.S, '\\\\', '')),
    '$.device.S'
  ) AS device,
  cast(
    json_extract_scalar(
      trim('"' FROM regexp_replace(dynamodb.NewImage.parameters.S, '\\\\', '')),
      '$.expiretimestamp.N'
    ) AS bigint
  ) AS expire_ts_bigint
FROM glue_dynamodb_table
WHERE dynamodb.NewImage.actionName.S = 'LOGIN'
LIMIT 10;
```

#### PURCHASE action:

```sql
SELECT
  dynamodb.NewImage.username.S AS username,
  json_extract_scalar(
    trim('"' FROM regexp_replace(dynamodb.NewImage.parameters.S, '\\\\', '')),
    '$.orderId.S'
  ) AS orderId,
  cast(
    json_extract_scalar(
      trim('"' FROM regexp_replace(dynamodb.NewImage.parameters.S, '\\\\', '')),
      '$.amount.N'
    ) AS double
  ) AS amount,
  cast(
    json_extract_scalar(
      trim('"' FROM regexp_replace(dynamodb.NewImage.parameters.S, '\\\\', '')),
      '$.expiretimestamp.N'
    ) AS bigint
  ) AS expire_ts_bigint
FROM glue_dynamodb_table
WHERE dynamodb.NewImage.actionName.S = 'PURCHASE'
LIMIT 10;
```

**Key takeaway:** LLMs cannot infer a universal query for `parameters.S` because each action type has a different schema.

---

## 5. Implications for AI-Assisted SQL

Polymorphic payloads in `parameters.S` make AI-based SQL generation challenging.

### 5.1 LLM Challenges

* **Opaque column**: Glue sees `parameters` as STRING; LLMs cannot detect nested keys.
* **Polymorphic structure**: Different `actionName` values imply different keys and types.
* **DynamoDB AttributeValue JSON**: Adds `.S` / `.N` type layer, requiring LLM guidance.
* **Escaping & formatting**: Quotes and backslashes require `regexp_replace` and `trim`.

### 5.2 Implications

* LLMs require **per-action type examples** and **explicit instructions**.
* Column descriptions cannot fully explain the polymorphic payload due to **Glue’s 250-character limit**, so even rich metadata is insufficient.
* Downstream AI-assisted SQL generation must rely on **examples, documentation, or flattened projections**, not just inferred schema.

