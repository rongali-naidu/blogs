## Why Data Engineers need to learn Vector Database in Generative AI

Landscape of Generative AI continues to evolve. To make use of these Generative AI models, prompt techniques plays important role. One of the prompting technique is RAG (Retrieval-Augmented Generation). RAG combines the strengths of retrieval systems and generative models to produce highly accurate and context-aware outputs. In RAG, the model doesn't rely solely on pre-trained knowledge; instead, it retrieves relevant external documents or data points to enhance its response generation. This approach is crucial for many applications, such as answering complex queries, generating code snippets, or summarizing large document. 

In this blog, we'll explore how vector databases work in the context of Generative AI and RAG, specifically focusing on Amazon OpenSearch Service, and why data engineers need to upskill in building pipelines for such databases.

## How RAG Works

When a user submits a query to generative AI models, it is first converted into a vector representation (embedding) . This vector captures the semantic meaning of the query.

* Vector Search in OpenSearch: The query vector is then passed to a vector database like OpenSearch. OpenSearch performs a k-NN (k-Nearest Neighbors) search to find documents or data points with similar embeddings, meaning they are semantically close to the query.
* Document Retrieval: Once the most relevant documents are retrieved based on their vector similarity, they are sent back to the generative model.
* Augmented Response Generation: The generative AI model  uses the retrieved documents to produce a more contextually accurate and informative response, augmenting its output with real-time, relevant data.

This is where vector databases play a pivotal role—they enable efficient, low-latency searches across large datasets of vectorized information, ensuring the generative model has the most relevant context to produce high-quality responses.

Refer this AWS Blog for more details on RAG : https://aws.amazon.com/what-is/retrieval-augmented-generation/

## Why RAG and Vector Databases Matter for Data Engineers

Data engineers are key to building the pipelines that feed RAG models. They are responsible for:

* Preparing the Data: Data needs to be cleaned, vectorized, and indexed in a vector database like OpenSearch.
* Ensuring Real-Time Capabilities: Vector databases must support fast retrieval for real-time applications, such as chatbots or recommendation systems.
* Scaling the Pipeline: As data grows, data engineers ensure the pipeline can scale to manage millions of vectors without sacrificing performance.
* By learning how to build these pipelines, data engineers enable AI models to access real-time, contextually accurate data, which leads to smarter and more relevant responses from AI systems

## How do we ingest data into OpenSearch Vecctor DB?

Multiples ways for ingesting data into Vector DB

### Ingesting Data with Apache Spark connector 

```
from pyspark.sql import SparkSession

# Step 2: Initialize Spark session with OpenSearch-Hadoop configurations
spark = SparkSession.builder \
    .appName("OpenSearch Ingestion") \
    .config("spark.jars.packages", "org.opensearch:opensearch-spark-30_2.12:2.5.0") \
    .config("spark.opensearch.nodes", "your-opensearch-domain") \
    .config("spark.opensearch.port", "9200") \
    .config("spark.opensearch.nodes.wan.only", "true") \
    .getOrCreate()

# Step 3: Sample data to write into OpenSearch
data = [
    {"id": 1, "name": "Alice", "age": 25},
    {"id": 2, "name": "Bob", "age": 30},
    {"id": 3, "name": "Charlie", "age": 35}
]

# Convert the data into a Spark DataFrame
df = spark.createDataFrame(data)

# Step 4: Write the DataFrame to OpenSearch index
df.write \
    .format("org.opensearch.spark.sql") \
    .option("opensearch.resource", "people-index/_doc") \
    .mode("Overwrite") \
    .save()

# Step 5: Verifying if data is written successfully
print("Data successfully ingested into OpenSearch")
```

### Ingesting Data with Python Module

```
import json
import boto3
from opensearchpy import OpenSearch

def lambda_handler(event, context):
    # Initialize OpenSearch client
    client = OpenSearch(
        hosts=[{'host': 'your-opensearch-domain', 'port': 443}],
        http_auth=('username', 'password'),
        use_ssl=True
    )
    
    # Example data from S3 event (assuming JSON file)
    data = {
        "id": "001",
        "title": "Data from Lambda",
        "description": "This data was ingested via AWS Lambda."
    }

    # Index the data into OpenSearch
    response = client.index(index="your-index", body=data)
    print(response)

    return {
        'statusCode': 200,
        'body': json.dumps('Data ingested successfully')
    }
```

### Ingesting Data with Glue ETL

Check this AWS Blog : https://aws.amazon.com/blogs/big-data/accelerate-analytics-on-amazon-opensearch-service-with-aws-glue-through-its-native-connector/

### Ingesting Data with Amazon Kinesis Firehose
Amazon Kinesis Firehose allows you to load streaming data directly into OpenSearch. It can continuously capture and transform data in real-time before loading it into OpenSearch

