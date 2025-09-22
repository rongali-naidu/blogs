# Anomaly Detection: Concepts, Models

## Introduction

Anomaly Detection (AD) is one of the most widely applied techniques in data analytics, fraud detection, system monitoring, and predictive maintenance. While AWS and third-party platforms provide powerful services for anomaly detection, the **effectiveness of these solutions depends on the models they use under the hood**. Understanding these models helps teams align the right tool with their data, business context, and operational requirements.

This article reviews key anomaly detection concepts, summarizes commonly used AD models, and highlights the models embedded within AWS services. It also surfaces the complexity and ambiguity in choosing the “right” model — there is no universal answer, only careful evaluation and iteration.

---

## Univariate vs. Multivariate

* **Variable**: A measurable quantitative value (e.g., transaction amount).

* **Dimension**: A categorical/contextual attribute that provides grouping (e.g., customer location).

* **Feature**: Any input used in ML models (can include variables and dimensions).

* **Univariate AD**: Focuses on one metric at a time.

  * *Example*: Monitoring transaction amount over time for a user.

* **Multivariate AD**: Considers multiple metrics/dimensions simultaneously.

  * *Example*: Evaluating transaction amount *and* customer location together.

---

## Outlier vs. Anomaly

* **Outlier**: A data point that deviates from the norm. Outliers may be harmless or expected.
* **Anomaly**: An outlier with contextual importance — often signaling fraud, system failures, or unusual activity that requires attention.

**Example**

* A \$5,000 purchase by a user who typically spends \$50–\$200 is an **outlier**.
* If that purchase also happens at 3 AM, from an unfamiliar country, and on a new device, it becomes an **anomaly**.

---

## Anomaly vs. Data Drift

* **Anomalies** = sudden deviations from expected behavior.
* **Data Drift** = gradual changes in “normal” behavior over time.

**Example**: A customer’s spending rising from \$200 → \$500/day due to lifestyle changes. Old models may incorrectly flag this as anomalous if drift isn’t accounted for.

---

## Types of Outliers

1. **Global Outlier** (point anomaly): Deviates from the entire dataset.

   * *Example*: \$5,000 transaction when most transactions are <\$500.

2. **Contextual Outlier**: Unusual within a specific context (time, location, device, etc.).

   * *Example*: \$300 spent at 3 AM from an unfamiliar country on a new device.

3. **Sequential/Collective Outlier**: A group of points looks normal individually but abnormal together.

   * *Example*: Five \$180 purchases within 30 minutes when the user usually makes only 2–3 purchases per hour.

---

## Detection vs. Prediction

* **Detection**: Reactive. Flags anomalies after they occur.

  * *Example*: Alerting on fraud post-transaction.

* **Prediction**: Proactive. Anticipates or intercepts anomalies in real time.

  * *Example*: Blocking or requiring OTP approval before a suspicious transaction is processed.

---

## Anomaly Detection Models

### Rule- and Heuristic-Based

* **Rule-based**: Hard thresholds from business rules.

  * *Example*: Flag all transactions > \$10,000.
* **Heuristic-based**: Approximate rules derived from domain knowledge.

  * *Example*: Flag if spending > 3× user’s average.

### Statistical Models

* **Distribution-based**: Model data distribution (mean, std dev, Z-score, IQR).
* **Time-series**: Capture trends & seasonality.

  * **Holt-Winters**.
  * **ARIMA** (auto-regressive + moving average).
  * **Prophet** (seasonality + trend forecasting).

### Machine Learning Models

* **Distance-based**: KNN (outliers = far from neighbors).
* **Density-based**: LOF (outliers = sparse regions).
* **Clustering-based**: K-Means (outliers = far from centroids).
* **Hybrid**: DBSCAN (density + distance + clustering).

### Tree/Ensemble-Based

* **Isolation Forest (iForest)**: Faster isolation = more anomalous.
* **Random Cut Forest (RCF)**

### Regression-Based

* **Supervised Regression**: Deviations between predicted vs. actual = anomalies.
* **TVAD**: Models expected metric values using contextual features.

### Deep Learning Time-Series Models

* **LSTM & LSTM Autoencoders**: Capture sequence patterns, use reconstruction error.
* **DeepAR**: LSTM-based forecasting, trained across multiple series.

### Other

* **One-Class SVM**: Learns boundary of “normal” → flags anything outside.
* **AutoML Approaches**: AutoGluon (tabular AD).
* **LLM-based**: AnoLLM (tabular AD), Chronos (time-series AD).



## Choosing the Right Model

There’s no one-size-fits-all. Selection depends on:

* **Nature of data**: distribution, seasonality, sparsity.
* **Univariate vs. Multivariate**: e.g., Holt-Winters (univariate) vs. RCF/iForest (multivariate).
* **Type of anomalies**: point vs. contextual vs. collective.
* **Detection mode**: batch vs. real-time streaming.
* **Contextual dimensions**: entity IDs, geolocation, device type.
* **Label availability**: supervised (if labeled anomalies exist) vs. unsupervised.
* **Explainability**: business may prefer interpretable models (trees) vs. deep learning.
* **Data drift**: models must be retrained/updated over time.
* **Privacy considerations**: ensure compliance when using fine-grained user data.

---

## Conclusion

Anomaly detection is not a “plug and play” problem. It’s an iterative process of:

1. Understanding your data.
2. Experimenting with multiple approaches.
3. Monitoring drift and retraining models.
4. Aligning model choice with business and compliance requirements.

AWS services like CloudWatch, QuickSight, and OpenSearch provide managed options, but deeper knowledge of the **models powering these services** ensures you make better architectural and operational decisions.

