---
title: "Anomaly Detection in AWS Services: What Runs Under the Hood"
date: 2026-11-16
tags: [analytics, anomaly-detection, aws]
publish: true
---

# **Anomaly Detection in AWS Services: What Runs Under the Hood**

*Part 3 of a 3-part series on anomaly detection. Part 1: [Fundamentals](anomaly_detection_fundamentals.md). Part 2: [Anomaly Detection Models](anomaly_detection_models.md).*

## **Introduction**

Several AWS services offer anomaly detection as a built-in feature. They are easy to switch on, but how well they work depends on whether the underlying model's assumptions match your data. This post summarises what **public AWS and OpenSearch documentation** says about the model behind each service, what data it works on, and where it fits.

Everything here is drawn from the public documentation linked in each section and in the references at the end.

---

## **Summary**

Three of the four services publicly document **Random Cut Forest (RCF)** as their algorithm. CloudWatch does not name its algorithm.

| Service | Underlying model (per public docs) | Data it works on | Dimensions / categories | Mode |
| --- | --- | --- | --- | --- |
| [Amazon CloudWatch anomaly detection](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html) | "Statistical and machine learning algorithms"; the specific algorithm is not named | One time-series metric (or metric math expression) per model | Dimensions identify the metric (up to 30). Each dimension combination is its own model, so low cardinality in practice | Near-real-time; alarms |
| [Amazon Quick Sight](https://docs.aws.amazon.com/quick/latest/userguide/concept-of-ml-algorithms.md) (formerly QuickSight) | Built-in Random Cut Forest | Time-series measures in analyses and dashboards | Can be sliced by fields, mainly for BI use | Scheduled |
| [Amazon OpenSearch Service](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ad.html) | Random Cut Forest | Time-series features aggregated from an index | Category fields split data into one model per entity (high-cardinality AD) | Near-real-time + historical analysis |
| [Amazon SageMaker AI](https://docs.aws.amazon.com/sagemaker/latest/dg/randomcutforest.html) | Built-in Random Cut Forest algorithm | Any numeric vectors, any number of dimensions | Encode categories yourself | Batch training; real-time or batch inference |

---

## **Amazon CloudWatch Anomaly Detection**

When you enable anomaly detection on a metric, CloudWatch applies statistical and machine learning algorithms to the metric's past data to build a model of its expected values. According to the documentation, the model accounts for trend and for hourly, daily and weekly patterns. It trains on up to two weeks of data, but you can enable it on a metric with less history.

> **Note:** public AWS documentation does not name the specific algorithm CloudWatch uses. You will see guesses online (for example, exponential smoothing), but treat them as speculation rather than documented fact.

The model produces a band of expected values. The metric math function `ANOMALY_DETECTION_BAND(m1, 2)` returns that band for metric `m1`. The second argument sets the band's width in standard deviations: a higher value means a wider band and fewer alerts. An alarm fires when the metric goes outside the band.

```json
[
  {
    "Id": "m1",
    "MetricStat": {
      "Metric": {
        "Namespace": "AWS/EC2",
        "MetricName": "CPUUtilization",
        "Dimensions": [{ "Name": "InstanceId", "Value": "i-1234567890abcdef0" }]
      },
      "Period": 300,
      "Stat": "Average"
    }
  },
  {
    "Id": "ad1",
    "Expression": "ANOMALY_DETECTION_BAND(m1, 2)",
    "Label": "CPU anomaly band"
  }
]
```

Other points from the documentation:

* You can exclude known unusual periods (deployments, outages) from training so they don't distort the band.
* For many common AWS metrics, CloudWatch keeps the band within logical values (for example, CPU utilisation never below 0%).
* CloudWatch also supports [PromQL-based anomaly bands](https://docs.amazonaws.cn/AmazonCloudWatch/latest/monitoring/anomaly_detection_promql.html) for Prometheus-compatible metrics, built from functions like `avg_over_time` and `stddev_over_time`. One query can cover many label values — effectively a rolling z-score.

**Fit:** univariate operational metrics with regular patterns (CPU, latency, request counts). It is not designed for per-customer or per-transaction detection.

---

## **Amazon Quick Sight**

Quick Sight uses a built-in version of RCF for its ML-powered anomaly detection (and for forecasting). From the [documentation on how RCF detects anomalies](https://docs.aws.amazon.com/quick/latest/userguide/how-does-rcf-detect-anomalies.html):

* RCF builds a forest of trees, each a binary space-partitioning tree over a sample of the data.
* Each data point gets an anomaly score; higher scores mean more anomalous. The score is roughly inversely proportional to the point's depth in the trees.
* The forest's score is the average across trees, scaled with respect to the sample size.
* A point is flagged when its score differs significantly from recent points.
* RCF works on contiguous segments ("shingles") of the series, so the recent window gives context to the newest point.

The documentation also notes the built-in algorithm handles seasonality and trends, excludes outliers, and imputes missing values.

**Fit:** business users exploring anomalies in dashboard KPIs, with contributor analysis to see which dimension values drove an anomaly.

---

## **Amazon OpenSearch Service**

OpenSearch anomaly detection finds anomalies in near real time using RCF. A detector defines:

* **Features:** aggregations over index fields per interval (for example, `max(latency)`). The default limit is 5 features per detector.
* **Category field:** splits data by a dimension like IP address, product ID or country code, so each entity gets its own model. This gives per-entity, contextual detection at high cardinality — something Quick Sight's RCF does not offer in the same way. The OpenSearch project has [published work on scaling this to a million entities](https://opensearch.org/?p=1209).
* **Window size (shingle):** how many intervals the model considers together, which helps catch collective anomalies.

Separately, the OpenSearch [ML Commons plugin](https://docs.opensearch.org/2.14/ml-commons-plugin/algorithms) exposes two RCF variants you can call directly:

* **Batch RCF:** for non-time-series data.
* **Fixed in Time (FIT) RCF:** for time-series data. It trains on historical data and stores the model; the stored model is not refreshed with new data.

The OpenSearch project's [RCF 2.0 post](https://opensearch.org/?p=1550) explains why RCF was chosen: small footprint, accuracy, explainability, and fast streaming updates without rebuilding models.

**Fit:** log and event data already in OpenSearch, especially when you need anomalies per entity (per host, per customer, per IP).

---

## **Amazon SageMaker AI**

[RCF is a built-in, unsupervised SageMaker algorithm](https://docs.aws.amazon.com/sagemaker/latest/dg/randomcutforest.html) designed for input of any dimension. It scales with the number of features, dataset size and number of instances.

From [How RCF Works](https://docs.aws.amazon.com/sagemaker/latest/dg/rcf_how-it-works.html):

* **Training:** reservoir-samples the data, splits the sample across trees, then repeatedly cuts each tree's bounding box on a randomly chosen dimension. Dimensions with more spread are chosen more often.
* **Tuning:** the main hyperparameters are `num_trees` and `num_samples_per_tree`. A labelled `test` channel is optional, but needed if you want to tune against F1.
* **Inputs:** numeric only. One-hot encode or embed categorical dimensions first.

SageMaker also offers DeepAR as a built-in forecasting algorithm, which can be used for forecast-based anomaly detection (flagging actuals outside its predicted quantiles).

**Fit:** custom, multivariate detection when you need control over features, training and deployment.

---

## **A Constraint Shared by the RCF-Based Services**

RCF takes numeric vectors of any length, but not categorical values. That makes it a strong fit for numeric time series. If your data has categorical dimensions (merchant category, device type, country), you have three options:

1. **Encode** them as numbers (one-hot, embeddings).
2. **Split** by them, so each value gets its own model — this is what OpenSearch category fields do.
3. **Choose a different model** that handles them more naturally, such as tree models on encoded features (see Part 2).

---

## **Choosing Between Them**

| If your situation is… | Start with |
| --- | --- |
| One operational metric with daily/weekly patterns | CloudWatch anomaly detection |
| Business KPIs in dashboards, users want to explore | Quick Sight ML-powered anomaly detection |
| Log or event data in OpenSearch, per-entity anomalies | OpenSearch anomaly detection with a category field |
| Custom multivariate data, full control over the pipeline | SageMaker RCF (or your own model, see Part 2) |

---

## **References**

**AWS documentation**

* [Using CloudWatch anomaly detection](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html)
* [CloudWatch anomaly detection using PromQL](https://docs.amazonaws.cn/AmazonCloudWatch/latest/monitoring/anomaly_detection_promql.html)
* [CloudWatch AnomalyDetector API reference](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_AnomalyDetector.html)
* [Operationalizing CloudWatch Anomaly Detection (AWS Cloud Operations Blog)](https://aws.amazon.com/blogs/mt/operationalizing-cloudwatch-anomaly-detection)
* [Amazon Quick Sight: ML algorithm concepts](https://docs.aws.amazon.com/quick/latest/userguide/concept-of-ml-algorithms.md)
* [Amazon Quick Sight: How RCF is applied to detect anomalies](https://docs.aws.amazon.com/quick/latest/userguide/how-does-rcf-detect-anomalies.html)
* [Amazon OpenSearch Service anomaly detection](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ad.html)
* [SageMaker AI Random Cut Forest algorithm](https://docs.aws.amazon.com/sagemaker/latest/dg/randomcutforest.html) and [How RCF Works](https://docs.aws.amazon.com/sagemaker/latest/dg/rcf_how-it-works.html)
* [Use the built-in Amazon SageMaker Random Cut Forest algorithm for anomaly detection (AWS ML Blog)](https://aws.amazon.com/blogs/machine-learning/use-the-built-in-amazon-sagemaker-random-cut-forest-algorithm-for-anomaly-detection/)

**OpenSearch project**

* [OpenSearch anomaly detection documentation](https://docs.opensearch.org/2.11/observing-your-data/ad/index/)
* [ML Commons supported algorithms (Batch RCF, FIT RCF)](https://docs.opensearch.org/2.14/ml-commons-plugin/algorithms)
* [Random Cut Forest 2.0 in OpenSearch](https://opensearch.org/?p=1550)
* [Improving Anomaly Detection: One million entities in one minute](https://opensearch.org/?p=1209)

**Paper**

* [Guha et al., Robust Random Cut Forest Based Anomaly Detection on Streams (ICML 2016)](http://proceedings.mlr.press/v48/guha16.pdf)

---

**Previous:** [Part 2 — Anomaly Detection Models](anomaly_detection_models.md) · **Start of series:** [Part 1 — Fundamentals](anomaly_detection_fundamentals.md)
