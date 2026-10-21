
#  User Behavior and Experience Analytics Using CloudWatch RUM and Athena

Before exploring CloudWatch RUM analytics, it’s important to understand the foundational concept of clickstream data.

## Clickstream Data Analysis

Clickstream data refers to the **sequence of user interactions recorded as events while a user navigates through a website or application**. These interactions are captured as events within a session and form a chronological record of user activity.These events are typically reconstructed into sessions to analyze user journeys as ordered sequences rather than isolated actions.

A typical clickstream includes:

* Page views and navigation paths
* Clicks on UI elements
* Form interactions (inputs, submissions)
* Session start and session end events
* Referrer and landing page information

From this event stream, we can derive meaningful product and business insights such as:

* **User journeys** (how users navigate through the application)
* **Funnels and conversions** (where users drop off or complete actions)
* **Engagement metrics** (time spent, repeat visits, interaction depth)
* **Behavioral segmentation** (grouping users based on usage patterns)


## RUM Data Analysis

Real User Monitoring (RUM) focuses on understanding **real-world application experience from the end-user perspective**. RUM data is essentially a clickstream enriched with performance and error telemetry


### Key aspects:

* Measures **client-side performance metrics**

  * Page load times (e.g., LCP, TTI)
  * Interaction latency (e.g., INP / FID)
  * Layout stability (CLS)

* Captures **runtime and reliability signals**

  * JavaScript errors
  * Failed network requests
  * Resource loading issues

* Provides **end-user experience visibility**

  * How fast pages feel in real conditions
  * Whether interactions are smooth or delayed
  * Where performance degradation impacts users

* Enables **application health analysis**

  * Correlation between performance and user drop-offs
  * Identification of slow or failing user journeys
  * Optimization of frontend performance and reliability


## Relationship Between RUM and Clickstream

While often treated separately, RUM and clickstream analytics operate on a **shared underlying event stream**, but differ in emphasis:

* Clickstream focuses on **behavioral sequence**
* RUM focuses on **behavior + performance context**


> Clickstream tells you *what users did*
> RUM tells you *what users did, and how the application performed while they did it*


## What is CloudWatch RUM?

[Amazon CloudWatch RUM (Real User Monitoring)](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-RUM.html) helps you collect and analyze user interactions from web applications in real time.

It captures:

* Page loads
* Errors
* Performance metrics (e.g., Largest Contentful Paint)
* User sessions and navigation behavior


---

##  Where RUM Fits in CloudWatch

```
CloudWatch
 └── Application Signals (APM)
      └── RUM
           └── App Monitor
                └── Data Storage → CloudWatch Log Group

CloudWatch Logs
 └── Log Groups
      ├── Metric Filters → Metrics → Alarms
      └── Subscription Filters
           ├── Kinesis Data Streams
           ├── Firehose
           ├── OpenSearch
           └── Lambda
```

---

## Step 1: Enable CloudWatch RUM

### 1. Create an App Monitor

* Go to CloudWatch → RUM
* Click **Create App Monitor**
* Provide:

  * App name
  * Domain
  * Sampling rate
  * Telemetry types (performance, errors, HTTP)
  * Data Storage → Log Group

### 2. Add RUM JavaScript Snippet

Insert the generated script into your frontend app:

```html
<script>
  (function(n,i,v,r,s,c,x,z){
    x=window.AwsRumClient={q:[],n:n,i:i,v:v,r:r,c:c};
    window[n]=function(c,p){x.q.push({c:c,p:p});};
    z=document.createElement('script');
    z.async=true;
    z.src=s;
    document.head.appendChild(z);
  })('cwr','APP_MONITOR_ID','1.0.0','us-east-1','https://client.rum.us-east-1.amazonaws.com/1.0.0/cwr.js',{
    sessionSampleRate: 1,
    guestRoleArn: "ROLE_ARN",
    identityPoolId: "IDENTITY_POOL_ID",
    endpoint: "https://dataplane.rum.us-east-1.amazonaws.com",
    telemetries: ["performance","errors","http"]
  });
</script>
```
## RUM Dashboard

With above config, your application sends the data to Cloudwatch RUM and you can monitor the metrics in [RUM Dashboard](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-RUM-view-data.html)
Here are the details of the Data Events captured by RUM : [CloudWatch-RUM-datacollected](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-RUM-datacollected.html)
Below steps are to get the same data for Analytics


## Build the Data Pipeline

To enable advanced analytics, export logs into a data lake.


```
CloudWatch Logs
   ↓ (Subscription Filter)
Kinesis Data Streams
   ↓
Firehose
   ↓ (optional Lambda transformation)
S3 (cross-account supported)
   ↓
Glue Crawler
   ↓
Glue Table
   ↓
Athena / SQL Queries
```



## Sample RUM Event

Example of a Largest Contentful Paint event:

```json
{
  "event_type": "com.amazon.rum.largest_contentful_paint_event",
  "timestamp": 1735689600000,
  "event_details": {
    "value": 2450,
    "element": "img.hero-banner"
  },
  "metadata": {
    "browser": "Chrome",
    "device": "desktop",
    "country": "US"
  }
}
```



## Sample Analytics Queries (Athena SQL)

### Page Views per Month

```sql
WITH page_views AS (
    SELECT
        DATE_TRUNC('month', FROM_UNIXTIME(timestamp/1000)) as month,
        COUNT(*) as page_views
    FROM "rum_metrics"
    WHERE event_type = 'com.amazon.rum.page_view_event'
        AND FROM_UNIXTIME(timestamp/1000) >= DATE('2025-01-01')
    GROUP BY DATE_TRUNC('month', FROM_UNIXTIME(timestamp/1000))
)
SELECT * FROM page_views
ORDER BY month;
```

---

###  Apdex Score Calculation

[Apdex categorizes](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-RUM-view-data.html#CloudWatch-RUM-apdex) user experience:

* **Satisfied**: ≤ 2s
* **Tolerating**: 2–8s
* **Frustrated**: > 8s

```sql
WITH apdex_data AS (
    SELECT
        DATE_TRUNC('month', FROM_UNIXTIME(timestamp/1000)) as month,
        (
            COUNT(CASE 
                WHEN CAST(JSON_EXTRACT_SCALAR(event_details, '$.value') AS DOUBLE) <= 2000 
                THEN 1 END)
            +
            COUNT(CASE 
                WHEN CAST(JSON_EXTRACT_SCALAR(event_details, '$.value') AS DOUBLE) > 2000
                 AND CAST(JSON_EXTRACT_SCALAR(event_details, '$.value') AS DOUBLE) <= 8000 
                THEN 1 END) * 0.5
        ) / COUNT(*) as apdex_score
    FROM "rum_metrics"
    WHERE event_type = 'com.amazon.rum.largest_contentful_paint_event'
        AND FROM_UNIXTIME(timestamp/1000) >= DATE('2025-01-01')
        AND JSON_EXTRACT_SCALAR(event_details, '$.value') IS NOT NULL
        AND CAST(JSON_EXTRACT_SCALAR(event_details, '$.value') AS DOUBLE) > 0
    GROUP BY DATE_TRUNC('month', FROM_UNIXTIME(timestamp/1000))
)
SELECT * FROM apdex_data
ORDER BY month;
```



