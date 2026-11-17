---
title: "Anomaly Detection Fundamentals: Outliers, Anomalies, Drift and Context"
date: 2026-11-16
tags: [analytics, anomaly-detection, machine-learning]
publish: true
---

# **Anomaly Detection Fundamentals: Outliers, Anomalies, Drift and Context**

*Part 1 of a 3-part series on anomaly detection. Part 2: [Anomaly Detection Models](anomaly_detection_models.md). Part 3: [Anomaly Detection in AWS Services](anomaly_detection_in_aws_services.md).*

## **Introduction**

Many cloud services and third-party tools now offer anomaly detection (AD) out of the box. They're easy to switch on, but each one is only as effective as the model running underneath it — and how well that model's assumptions match *your* data.

No single anomaly detection model fits every dataset. Before choosing a model or a managed service, it helps to be precise about the vocabulary: what counts as an anomaly, what kind of anomaly you need to catch, and how "normal" itself changes over time.

This post covers those fundamentals. Part 2 surveys the model families, and Part 3 looks at which models AWS services use, based on public AWS documentation.

Throughout the series, I'll use one running example: **a bank customer's card transactions.**

---

## **Terminology**

| Term | Meaning | Example |
| --- | --- | --- |
| Variable | A measurable, quantitative attribute | Transaction amount |
| Dimension | A categorical or contextual attribute that groups data or gives it context | Country, device, merchant category |
| Feature | Any input to an ML model. It can be a variable, a dimension, or something derived from them | Amount, hour of day, transactions in the last 30 min |
| Univariate analysis | One variable or feature analysed at a time | Transaction amount over time for one customer |
| Multivariate analysis | Several variables and/or contextual features analysed together, so their joint patterns count. They can be numeric, categorical or both | Transaction amount together with customer location |
| Cardinality | The number of distinct values in a dimension | Country is low (about 200); customer ID is high (millions) |

---

## **The Running Example: A Normal Baseline**

Before spotting anomalies, you need to know what normal looks like. For our customer:

* Typical transaction amount: $50–$200
* Usual location: mainly New York, USA
* Usual shopping hours: 9 AM–9 PM local time
* Usual devices: a personal laptop and a smartphone
* Usual pace: purchases spread through the day, rarely more than 2–3 an hour

---

## **Outlier vs Anomaly**

An **outlier** is anything that deviates from normal — in value, behaviour, timing, location and so on. Many outliers are harmless.

An outlier becomes an **anomaly** when it points to a deeper issue, such as fraud, a system fault or an unexpected event. Anomalies usually need investigation or action. Whether an outlier matters depends on its context and its likely impact.

**Example:** the customer makes a single $5,000 transaction. It is an outlier against their usual range, but it may be genuine — international flight tickets, for instance. If it also comes from a country they never shop in, at 3 AM, through a known scam merchant, it becomes an anomaly that suggests fraud.

---

## **What Is Anomaly Detection?**

Anomaly detection identifies deviations from the normal patterns of a system (events) or of a person (actions): transaction values, activity timing, location and other metrics. The system learns what normal looks like, then flags data points that stand out.

Normal behaviour can be modelled as:

* A **statistical distribution**
* A **temporal pattern** (a time series with trend and seasonality)
* **Clusters or neighbourhoods** of similar points
* A learned **prediction or reconstruction** of the data

Any of these can be **univariate** (one variable) or **multivariate** (several dimensions at once).

---

## **Outlier Types**

The type of outlier you need to catch largely decides the model family.

| Type | Also called | What makes it unusual | Running example | Model families that fit |
| --- | --- | --- | --- | --- |
| Global | Point outlier | The value alone is extreme against the whole dataset, whatever the context | A single $5,000 transaction where the customer base rarely goes over $500 | Rules, z-score/IQR, Isolation Forest, RCF |
| Contextual | Conditional outlier | Normal globally, unusual within its context (time, place, device, segment) | $300 at 3 AM from a country never visited, on a new device | Per-entity or per-segment models, time-series models, multivariate ML |
| Collective | Sequential outlier | Each point is normal; the group or sequence together is not | Five purchases of about $180 within 30 minutes at different online stores | Sequence models (LSTM), windowed features (counts and velocity), shingled RCF |

**Contextual outliers.** Context usually comes from extra dimensions: category, region, or — in our example — device and time. When time defines the context, treat the data as a time series.

**Collective outliers.** In the example, each purchase is under the $200 ceiling. The burst is what is anomalous, and it may point to card theft or automated purchasing. A model that scores each point on its own will miss it, so add windowed features (count, sum and distinct merchants in the last N minutes) or use a sequence-aware model.

---

## **Anomaly vs Data Drift**

Normal is not fixed. User behaviour, business growth, seasonality and inflation shift it over time. This shift is **data drift**.

You may not be trying to detect drift, but it affects anomaly detection. If a customer's average daily spend rises gradually from $200 to $500, a model trained on old behaviour starts flagging legitimate transactions. The result is more false positives and lost trust in alerts.

Two common forms:

* **Covariate (feature) drift:** the distribution of inputs changes — for example, more mobile transactions.
* **Concept drift:** the relationship between inputs and what counts as normal changes — for example, a new fraud pattern.

Mitigations include retraining on a rolling window, models that update online (such as Random Cut Forest), and monitoring the anomaly rate itself for sudden change. Part 2 covers drift detection methods in more detail.

---

## **Detection vs Prediction**

| | Anomaly detection | Anomaly prediction (prevention) |
| --- | --- | --- |
| When | After the event | Before or during the event |
| Stance | Reactive | Proactive |
| Data | Historical or near-real-time batches | Real-time stream, low latency |
| Action | Raise an alert, investigate | Block, hold or challenge the action |
| Banking example | Flag fraud after it happens and contact the customer | Stop the transaction, or ask for approval by one-time passcode or phone call |

---

## **Anomaly Detection vs Forecasting**

Anomaly detection answers *"what happened that doesn't usually happen?"* Forecasting answers *"if things continue as expected, what happens next?"*

The two are closely linked: many time-series detectors forecast an expected value and flag actual values that fall outside a band around it.

---

## **Glossary**

| Term | Definition |
| --- | --- |
| Anomaly | An observation that differs from most others in the same sample and signals something worth acting on. Also called an exception or deviation |
| Anomaly score | A model's numeric measure of how unusual a point is; a threshold turns it into a flag |
| Contamination rate | The expected share of anomalies in the data, used by some libraries to set the threshold |
| Data point | One unit of data, usually a row. One row can hold several data points when a measure is split across dimensions |
| Forecast | A prediction of future values from current and past behaviour |
| Reconstruction error | The difference between an input and an autoencoder's reconstruction of it |
| Seasonality | Patterns that repeat on a fixed cycle (hourly, daily, weekly, yearly) |
| Shingle | A sliding window of the last w values, used as one vector so a model sees recent context |
| Stationary series | A time series whose mean and variance do not change over time |
| Time series | Data ordered by a timestamp, where the order matters |

---

**Next:** [Part 2 — Anomaly Detection Models: From Rules to LLMs](anomaly_detection_models.md)
