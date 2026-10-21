## Understanding Amazon Bedrock Data Automation (BDA)

[Amazon Bedrock Data Automation (BDA)](https://github.com/aws-samples/sample-document-processing-with-amazon-bedrock-data-automation) enables the asynchronous processing of documents using AI-powered blueprints. This workflow is conceptually similar to running an asynchronous query in Athena:

1. **Submit the job**
2. **Poll/check its status**
3. **Fetch and parse results once complete**

---

### 1. Submitting a BDA Job

The main API is `InvokeDataAutomationAsync`, which processes documents asynchronously and stores outputs in S3.

```python
import boto3

bda_client = boto3.client("bedrock-data-automation-runtime", region_name="us-east-1")

response = bda_client.invoke_data_automation_async(
    inputConfiguration={
        "s3Uri": "s3://my-bucket/documents/sample-document.pdf"
    },
    outputConfiguration={
        "s3Uri": "s3://my-bucket/bda-output/job123/"
    },
    blueprints=[{
        "blueprintArn": "arn:aws:bedrock:us-east-1:aws:blueprint/document-extraction-generic",
        "stage": "LIVE"
    }],
    dataAutomationProfileArn="arn:aws:bedrock:us-east-1:123456789012:data-automation-profile/us.data-automation-v1"
)

invocation_arn = response.get("invocationArn")
print(f"Job submitted: {invocation_arn}")
```

**Argument explanations:**

| Argument                   | Description                                                                        |
| -------------------------- | ---------------------------------------------------------------------------------- |
| `inputConfiguration`       | S3 location of the input document to process                                       |
| `outputConfiguration`      | S3 location where processed output will be stored                                  |
| `blueprints`               | List of blueprint ARNs to define extraction logic (stage can be `LIVE` or `DRAFT`) |
| `dataAutomationProfileArn` | ARN of the automation profile defining runtime settings for the BDA job            |

---

### 2. Checking Job Status

After submission, you can monitor the job with `GetDataAutomationStatus`:

```python
status_response = bda_client.get_data_automation_status(invocationArn=invocation_arn)
print(f"Job status: {status_response['status']}")
```

---

### 3. Parsing the Output

Once the job completes, results are available in the S3 output location. You can parse JSON files or metadata to extract structured data.

---

### Analogy with Athena SQL

* **Submit job:** Similar to executing an Athena query
* **Check status:** Poll for query completion
* **Parse results:** Fetch query results from S3

---

### References

1. [AWS Bedrock Data Automation API Reference](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_Operations_Data_Automation_for_Amazon_Bedrock.html)
2. [InvokeDataAutomationAsync](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_data-automation-runtime_InvokeDataAutomationAsync.html)
3. [GetDataAutomationStatus](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_data-automation-runtime_GetDataAutomationStatus.html)
4. [Amazon Athena Documentation](https://docs.aws.amazon.com/athena/latest/ug/what-is.html)

