# Unlocking Athena Query History: The Why and the How of Persisting Execution Metadata

## Purpose

The Athena query history captures valuable information through embedded identifiers in SQL queries.
These identifiers are automatically injected by client applications like Quicksight, Internal developed Reporting tools that sends queries to Athena.

Key benefits of persisting this query history:

* Traceability: Identifies which specific report triggered each Athena query
* Cost  Management :  per-query execution costs and opportunities for optimization
* User Context: Shows which user ran particular reports
* Debugging Aid: Helps troubleshoot missing data issues by revealing:
    * Exact SQL queries executed
    * Actual filters applied
    * Specific report parameters used

This historical data is essential for effective troubleshooting, as without it, tracking the precise query execution details and filter conditions becomes significantly challenging.

## Quicksight Query Identifiers

Following details are relevant for Quicksight datasets that uses Athena engine.

```sql
/* QuickSight c95e8461-2083-45cd-990e-f36ae6eeb694
{"partner":"QuickSight",
"entityId":"e1c6c85f-34e1-445b-be12-c073b2d8d581",
"sheetId":"e1c6c85f-34e1-445b-be12-c073b2d8d581_31740da0-0220-4589-9038-582bf58452fb",
"visualId":"e1c6c85f-34e1-445b-be12-c073b2d8d581_d436bef0-1f4f-4b69-936d-53ed25036398"
}
```

    * QuickSight currently embeds the above shown UUIDs in Athena queries.
        * Quicksight session identifier (i.e UUID right after the "QuickSight" keyword)
        * entityId (aka  is dashboard-id)
        * sheetId
        * visualID
        * More details on these UUIDs .
    * For SPICE Data refresh related queries, Quicksight sends just session identifier. It will be helpful if quicksight includes dataset identifier in the SPICE refresh related queries. If dashboards or visuals uses datasets which uses SPICE, Quicksight doesnt send any other Athena queries except SPICE dataset refresh.
    * Steps for getting the dashboard related ids from the Quicksight UI
        * For getting these UUIDs, Open a Dashboard → select a sheet --> select a visual → Menu Options → Embed Visual → Expands IDS for Developers
        *


## Solution Overview

image.png

* This solution uses Athena APIs to extract Athena Query Execution History, with the goal of capturing query metadata for operational insights, cost attribution, and potential data quality triage.
* The code runs as an AWS Lambda function, scheduled daily using an Amazon EventBridge Rule.
* The Lambda accepts the following runtime parameters:
    * S3 Bucket to store extracted results
    * Output Format (e.g., JSON or CSV)
    * Full vs. Incremental Refresh mode
    * Use of Batch API toggle to improve performance
* Output is stored in S3 with partition on snapshot_day, and crawled by a Glue Crawler to register a table in the Glue Data Catalog, enabling further analysis via Athena.
* This data can be combined with CloudTrail logs to enrich it with user identity and service context (e.g., queries triggered by QuickSight).



## Implementation Notes and Challenges

1. Athena APIs do not support filtering by timestamp, which makes incremental extraction challenging.
2. Athena UI shows query history only for recent runs, but Athena stores query metadata for up to 45 days.
3. Performing a full refresh daily results in duplicates and can cause Lambda timeouts, especially in accounts with multiple workgroups and heavy Athena usage.
4. The list_query_executions API appears to return query IDs sorted by submission time, though this is not officially documented. This solution relies on that behavior to implement incremental fetching.
5. If the workgroup is not specified, the list_query_executions API defaults to the primary workgroup.
6. batch_get_query_execution API is supposed to improve runtime efficiency by processing up to 50 query IDs per call instead of fetching details one-by-one but i observed the `get_query_execution  worked better and need to check this observation further

## Lambda code
```python



import boto3
import json
import csv
import io
import os
from datetime import datetime, timedelta, timezone

athena = boto3.client('athena')
s3 = boto3.client('s3')





S3_BUCKET = os.getenv('S3_BUCKET', 'default-bucket')
S3_PREFIX = os.getenv('S3_PREFIX', 'default-prefix')
OUTPUT_FORMAT = os.getenv('OUTPUT_FORMAT', 'json').lower()

# Environment variables are strings, so convert FULL_REFRESH to boolean
FULL_REFRESH = os.getenv('FULL_REFRESH', 'false').lower() == 'true'
USE_BATCH_API = os.getenv('USE_BATCH_API', 'true').lower() == 'true'

# Get previous day start and end in UTC
today_utc = datetime.utcnow().replace(hour=0, minute=0, second=0, microsecond=0, tzinfo=timezone.utc)
start_time = today_utc - timedelta(days=1)
end_time = today_utc

def default_serializer(obj):
    """
    Converts Python datetime objects to ISO format strings for JSON serialization.
   
    Notes:
    - Boto3 SDK converts datetime strings in API responses to Python datetime objects.
    - These objects must be serialized to string format (e.g., ISO 8601) when dumping to JSON.
    """
    if isinstance(obj, datetime):
        return obj.isoformat()
    raise TypeError(f"Type {type(obj)} not serializable")

def lambda_handler(event, context):
    query_ids = []
    workgroups = athena.list_work_groups()['WorkGroups']
    # workgroups = [{'Name':'QuickSightEnterpriseAthenaWG3'}]
    records = []

    for wg in workgroups:
        wg_name = wg['Name']
        next_token = None    

        while True:
            params = {
                'MaxResults': 50,
                'WorkGroup': wg_name
            }
            if next_token:
                params['NextToken'] = next_token

            response = athena.list_query_executions(**params)
            query_ids_batch = response.get('QueryExecutionIds', [])
            next_token = response.get('NextToken')          

            if USE_BATCH_API:
                # Use batch_get_query_execution in batches of 50                
                # Split into batches of 50 (defensive, though MaxResults=50)
                stop_processing = False
                for i in range(0, len(query_ids_batch), 50):
                    if stop_processing:
                        break            
                    batch = query_ids_batch[i:i+50]
                    try:
                        batch_resp = athena.batch_get_query_execution(QueryExecutionIds=batch)
                        # q = athena.get_query_execution(QueryExecutionId=qid)['QueryExecution']
                        for q in batch_resp.get('QueryExecutions', []):
                            submission_time = q['Status']['SubmissionDateTime']

                            if submission_time >= end_time:
                                continue  # Too new
                            elif  not FULL_REFRESH and submission_time < start_time:
                                next_token = None  # for breaking while loop
                                stop_processing = True       # break outer i-loop                  
                                break   # break inner q-loop

                            records.append(q)
                    except Exception as e:
                        print(f"Batch fetch failed for workgroup '{wg_name}': {str(e)}")
                       
            else:
                # Use get_query_execution calls
                for qid in query_ids_batch:
                    try:
                        q = athena.get_query_execution(QueryExecutionId=qid)['QueryExecution']
                        submission_time = q['Status']['SubmissionDateTime']

                        if submission_time >= end_time:
                            continue  # Too new
                        elif   not FULL_REFRESH and ssubmission_time < start_time:
                            next_token = None   # for breaking while loop
                            break  # for breaking qid-loop

                        records.append(q)

                    except Exception as e:
                        print(f"Failed to fetch query {qid}: {str(e)}")                

            if not next_token:
                break  

    # Prepare S3 key and file content
    snapshot_partition = f"snapshot_day={start_time.strftime('%Y-%m-%d')}"
    timestamp = datetime.utcnow().strftime('%Y-%m-%dT%H-%M-%S')
    s3_key = f"{S3_PREFIX}/{snapshot_partition}/query_history_{timestamp}.{OUTPUT_FORMAT}"

    if OUTPUT_FORMAT == 'json':
        # dumps() converts multi line JSON to single line JSON
        # serializers handles Python Datetime objects
        # Serializes records into a single JSON array
        # We iterate over the JSON array to make it newline separat records for querying throu Athena    
        lines = [json.dumps(r, default=default_serializer) for r in records]
        file_content = "\n".join(lines)
        content_type = 'application/json'
    elif OUTPUT_FORMAT == 'csv':
        output = io.StringIO()
        writer = csv.DictWriter(output, fieldnames=records[0].keys())
        writer.writeheader()
        writer.writerows(records)
        file_content = output.getvalue()
        content_type = 'text/csv'
    else:
        raise ValueError("Unsupported OUTPUT_FORMAT")

    # Upload to S3
    s3.put_object(
        Bucket=S3_BUCKET,
        Key=s3_key,
        Body=file_content.encode('utf-8'),
        ContentType=content_type
    )

    return {
        'status': 'success',
        's3_uri': f's3://{S3_BUCKET}/{s3_key}',
        'records_written': len(records)
    }
```


## Athena Lambda Role Permissions

```json
{
    "Version": "2012-10-17",
    "Statement": [
         {
            "Sid": "AthenaQueryHistoryAccess",
            "Effect": "Allow",
            "Action": [
                "athena:ListQueryExecutions",
                "athena:GetQueryExecution",
                "athena:ListWorkGroups"
            ],
            "Resource": "*"
        },
        {
            "Sid": "S3WriteAccess",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "<S3_BUCKET_ARN>"
        }
    ]
}
```
## Lambda schedule

Event Bridge Rule Cron expression

Once a day at 1 AM.

```sql

0 1 * * ? *

```



## Glue Crawler

Create Glue Crawler pointing to S3 bucket . It will create Glue table

## Athena SQL

```sql

select
queryexecutionid,
workgroup,
queryexecutioncontext.catalog,
queryexecutioncontext.database,
status.state,
status.submissiondatetime,
status.completiondatetime,
statistics.engineexecutiontimeinmillis,
statistics.datascannedinbytes,
statistics.totalexecutiontimeinmillis,
round(statistics.datascannedinbytes*5.0/(cast(1024 as BIGINT)*1024*1024*1024),6 ) as quyercost,
query
from athena_query_history
where
snapshot_day='2025-07-11'
and workgroup not in ('primary')
limit 10
```

## Other Alternatives to get the Athena Query Details

These alternatives doesnt provide all the details provided by Athena API like query statistics.

### Querying Athena-Query details through Cloudtrail events

Using Cloudtrail table setup as per these steps .

```sql
SELECT
  from_iso8601_timestamp(eventTime) AS event_time,
  userIdentity.type AS user_type,
  userIdentity.arn AS user_arn,
  userIdentity.principalid AS principalid,
  json_extract_scalar(responseelements, '$.queryExecutionId') AS queryExecutionId,
  json_extract_scalar(requestParameters, '$.queryExecutionContext.database') AS database,
  json_extract_scalar(requestParameters, '$.queryString') AS query_text,
  json_extract_scalar(requestParameters, '$.queryExecutionId') AS query_id
FROM cloudtrail_log
WHERE eventSource = 'athena.amazonaws.com'
  AND eventName = 'StartQueryExecution'
ORDER BY event_time DESC
```



### Cloudtrail Lookup Events


Since list_query_executionsAthena API lacks time based filters, one other we could consider is CloudTrail Lookup Events API .

Sample Code

Note :  Similar to Athena APIs, this API might also return 50 events in one iteration,we might need to add pagination logic:

```python

import boto3
import json
from datetime import datetime, timedelta

def main():
    # Create CloudTrail client
    cloudtrail = boto3.client('cloudtrail', region_name='us-west-2')
   
    # Set time range
    start_time = datetime(2025, 7, 13)
    end_time = datetime(2025, 7, 15)
   
    try:
        response = cloudtrail.lookup_events(
            LookupAttributes=[
                {
                    'AttributeKey': 'EventName',
                    'AttributeValue': 'StartQueryExecution'
                }
            ],
            StartTime=start_time,
            EndTime=end_time
        )
       
        # Extract queryExecutionIds from the events
        query_ids = []
        for event in response['Events']:
            cloud_trail_event = json.loads(event['CloudTrailEvent'])
            if 'responseElements' in cloud_trail_event:
                query_ids.append({
                    'queryExecutionId': cloud_trail_event['responseElements'].get('queryExecutionId')
                })
               
       
        # Print results
        for query_id in query_ids:
            print(json.dumps(query_id))
           
        return query_ids

    except Exception as e:
        print(f"Error occurred: {str(e)}")
        return None
       
 main()
```

### cloud trail  lookup CLI example

```sh
aws cloudtrail lookup - events\
    --lookup - attributes AttributeKey = EventName, AttributeValue = StartQueryExecution\
    --start - time 2025 - 07 - 13\
    --end - time 2025 - 07 - 15\
    --region us - west - 2\
    --output json | jq - r '.Events[].CloudTrailEvent | fromjson.responseElements | {queryExecutionId}'
```
