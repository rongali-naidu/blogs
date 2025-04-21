## 🚀 AWS S3 Table Buckets – More Federated Catalogs (or More Silos for Unifying :))

### 🧱 What's New in Amazon S3

Amazon S3 has introduced a feature called **S3 Table Buckets** for working with Iceberg table format.

> 🔗 [Official Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html)

THis week, i tried it. 

### 🧩 Key Enhancements to supports this

####  S3 :
New Section for S3 Table Buckets. Each S3 Table Bucket provides options to create multiple namespaces and mmultiple tables within each namespace.
Note : Not all S3-Table buckets operations are supported through AWS Console. some are supported through CLI and REST API , like deleting bucket, managing resource policies

#### ✅ Lakeformation Catalog : Separate Data Catalogs

- Added new section for Catalogs. 

- **Default Catalog** – Named `<AWSAccountID>` in Lake Formation.
- **S3 Tables Catalog** – Named `<AWSAccountID>:S3TablesCatalog/<S3TableBucketName>`  in Lake Formation.
  
- Each **S3 Table Bucket** is now treated as a **separate Glue Catalog**, referred to as a *Federated Catalog* in **Lake Formation**.
- You can create multiple **namespaces** (database equivalents) within a single S3 Table Bucket.
- Each table is tied to a namespace, and all of this is **discovered** through Lake Formation, not Glue.
- Each S3-Bucket based table has Location which looks like `s3://<table-id>--table-s3` (this isn’t listed as regular bucket via `aws s3 ls`)

#### ✅ Glue Catalog  : No changes.
- Glue Console doesn’t currently show these federated S3TableCatalog entries. They appear only in **Lake Formation**

### 🔄 Analytical Services Integration

First time, we create S3 Table Bucket, we get an option to chose "Enable Integration". When you enable integration with analytics services, AWS creates a federated catalog (`s3tablescatalog`)  and IAM role `S3TablesRoleForLakeFormation`



### 🧪 Observations and Challenges

- ❌ **S3Table Buckets related Tables aren’t visible in Glue Console**, only in Lake Formation.
- ✅ Tables are queryable using SQL via Athena and other services with the correct catalog context.
- ❌ Not directly importable into QuickSight using the default data source.
  - ✅ Workaround: Use **custom SQL** like:

    ```sql
    SELECT * FROM "S3TablesCatalog/<S3TabeBucketName>.<NamespaceName>.<TableName>"
    ```

- ❓ **S3 Bucket not listable** via CLI (`aws s3 ls` fails with MethodNotAllowed).
  - This is expected – S3Tables are **not standard S3 buckets**. 


### 🔍 Open Questions

Open Questions:
* What is the S3 bucket pointed to ? WHich account has it? how to get access to the S3 bucket for checking the metadata and the actual data (aws s3 ls in the same account didnt list it)?


