# Querying S3-Parquet with DuckDB

## The scenario

I launched a **new AWS account** and tried to run **S3 Select** on a Parquet file in S3 to do a quick `SELECT` but failed with error `An error occurred (MethodNotAllowed) when calling the SelectObjectContent operation:`

I needed another way to run simple `SELECT`s over Parquet in S3 . Agree, i could run Athena after cataloguing and `parquet-tools` is one option, but I wanted to give it a try through **DuckDB** 

---

## 1. Install DuckDB (Windows)

Docs: https://duckdb.org/install/?platform=windows&environment=cli

```powershell
winget install DuckDB.cli --source winget

duckdb --version
# v1.5.5 (Variegata) d8cdaa33fd

2. Install + configure the AWS CLI

Docs: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

Configure credentials for the account you want to query:

```
aws configure
```

3. Point DuckDB to AWS Credentials
```
CREATE OR REPLACE SECRET facman (
    TYPE s3,
    PROVIDER credential_chain,
    REGION 'us-east-1'          -- set to the bucket's real region
);
```


3. Query the single file

```
SELECT *
FROM read_parquet('s3://../../20260922-180105012.parquet')
LIMIT 20;
```

```
DESCRIBE SELECT * FROM read_parquet('s3://../../18/20260922-180105012.parquet');
```
```
SELECT path_in_schema, compression, encodings
FROM parquet_metadata('s3://../../20260922-180105012.parquet');

```

4. Cleaning the credentials 


```
FROM duckdb_secrets();
```

```
DROP SECRET aws_credentials_duckdb;        -- removes aws_credentials_duckdb
```


	
