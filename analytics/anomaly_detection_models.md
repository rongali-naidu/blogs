---
title: "Anomaly Detection Models: From Rules to LLMs, and How to Choose"
date: 2026-11-16
tags: [analytics, anomaly-detection, machine-learning]
publish: true
---

# **Anomaly Detection Models: From Rules to LLMs, and How to Choose**

*Part 2 of a 3-part series on anomaly detection. Part 1: [Fundamentals](anomaly_detection_fundamentals.md). Part 3: [Anomaly Detection in AWS Services](anomaly_detection_in_aws_services.md).*

## **Introduction**

There is a wide range of anomaly detection models, and no single one fits every dataset. This post walks through the main model families, from the simplest and most explainable to the most flexible and opaque. It then offers a framework for choosing between them and goes deeper on a few widely used techniques.

Most production systems combine several families — for example, rules for known-bad patterns plus an unsupervised model for unknown ones.

The running example from Part 1 continues: **a bank customer who usually spends $50–$200 per transaction, mainly in New York, between 9 AM and 9 PM, rarely more than 2–3 times an hour.**

---

## **Model Families at a Glance**

| Family | Representative models | Learns normal from data? | Multivariate | Streaming | Explainability |
| --- | --- | --- | --- | --- | --- |
| Deterministic rules | Threshold and business rules | No | Yes, if you write it | Yes | Very high |
| Heuristics | Multiples of a user's average | Partly | Limited | Yes | High |
| Statistical distributions | Z-score, MAD, IQR, Gaussian, Poisson | Yes (parameters) | Mostly univariate | Yes | High |
| Statistical time series | Holt-Winters, ARIMA, Prophet, STL | Yes | Univariate | Yes, with care | High |
| Distance-based | k-NN | Yes | Yes | Limited | Medium |
| Density-based | LOF | Yes | Yes | Limited | Medium |
| Clustering-based | K-Means | Yes | Yes | Limited | Medium |
| Hybrid density clustering | DBSCAN, HDBSCAN | Yes | Yes | Limited | Medium |
| Tree ensembles | Isolation Forest, Random Cut Forest | Yes | Yes (numeric) | RCF yes, iForest batch | Medium |
| Supervised regression | Random Forest, gradient boosting on a target | Yes (labelled target) | Yes | Yes (scoring) | Medium–high |
| Deep learning | LSTM, LSTM autoencoder, DeepAR | Yes | Yes | Yes | Low |
| Boundary-based | One-Class SVM | Yes | Yes | No | Low |
| LLM-based | AnoLLM | Yes | Yes (incl. text) | No | Low–medium |

---

## **1. Deterministic Rules**

Normal is defined explicitly by rules that are binary: a condition is met or it isn't. Anything that breaks a rule is an anomaly.

* **How to build them:** start with data profiling, then align with the business process and business rules. This overlaps heavily with data quality checks.
* **Strengths:** easy to validate, audit and explain.
* **Weaknesses:** rigid; rules go stale as data patterns change, and they catch only what someone thought of.
* **Examples:** flag any transaction over $10,000. Flag two transactions on the same account from distant locations within 5 minutes ("impossible travel").

## **2. Heuristics**

Heuristics are approximate rules drawn from domain expertise or observed patterns. They are quicker to build than precise rules and are often accurate enough.

* **Example:** flag a transaction more than 3× the customer's average.
* **Watch out:** a heuristic that uses a per-user statistic is already a simple personalised model, so it needs enough history for each user.

## **3. Statistical Distribution Models**

Many real datasets follow a known distribution: normal (Gaussian), log-normal, exponential, Poisson and others.

1. Find the distribution that best fits the data (histograms, Q–Q plots, goodness-of-fit tests).
2. Estimate its parameters, such as mean and standard deviation.
3. Compute how likely each new point is under that distribution.
4. Flag points whose likelihood falls below a threshold, static or dynamic.

Because the threshold comes from the data, it adapts better than a fixed rule.

Common scores:

**Z-score** for roughly normal data. Flag when |z| > 3.

$$z = \frac{x - \mu}{\sigma}$$

**Modified z-score (MAD)**, robust to the outliers themselves. Flag when |M| > 3.5.

$$M = \frac{0.6745\,(x - \tilde{x})}{\mathrm{MAD}}, \quad \mathrm{MAD} = \mathrm{median}\left(|x_i - \tilde{x}|\right)$$

**IQR (Tukey fences)**, no distribution assumed:

$$x < Q_1 - 1.5\,\mathrm{IQR} \quad \text{or} \quad x > Q_3 + 1.5\,\mathrm{IQR}, \qquad \mathrm{IQR} = Q_3 - Q_1$$

**Poisson tail probability** for counts per interval (logins per hour, errors per minute) — see the deep dive below.

## **4. Statistical Time-Series Models**

Use these when data has trend, seasonality or other time structure. Each forecasts an expected value and flags actual values that fall outside a confidence band around it.

* **Holt-Winters (triple exponential smoothing):** models level, trend and seasonality with three smoothing parameters. Light and fast; one seasonal period per model.
* **ARIMA / SARIMA:** combines autoregression, differencing (for trend) and moving averages; SARIMA adds seasonality. Needs a stationary series after differencing.
* **Prophet:** an open-source forecasting library from Meta. It fits trend, several seasonalities and holiday effects; anomalies are actuals outside its uncertainty interval. Copes with missing data and changes in trend.
* **STL decomposition:** splits a series into trend, seasonal and remainder parts; you then run a statistical test on the remainder.

## **5. Machine Learning Models**

Normal behaviour is learned from historical data, and points that deviate from it are flagged. Most of these are unsupervised.

### **5a. Distance-Based**

No distribution is assumed. A point is an outlier if it is far from its nearest neighbours.

* **k-Nearest Neighbours (k-NN):** the score is the distance to the k-th nearest neighbour, or the mean distance to the k nearest. Points far from dense regions of normal data score high.
* **Cost:** pairwise distances grow quickly with data size; scale your features first.

### **5b. Density-Based**

Compares how densely packed a point's neighbourhood is with its neighbours' neighbourhoods. Density here means roughly the number of points within a given distance.

* **Local Outlier Factor (LOF):** compares a point's local density with that of its neighbours. A score near 1 is normal; well above 1 means the point sits in a sparser region than its neighbours. It shines on data whose clusters have different densities. See the deep dive below.

### **5c. Clustering-Based**

Group the data into clusters, then analyse each point's distance to its cluster centre statistically.

* **K-Means:** flag points whose distance to their centroid exceeds a threshold — for example, the cluster's mean distance + 2 standard deviations.
* **Limits:** you must choose k; clusters are assumed roughly spherical; outliers pull centroids during training.

> **Key observation:** distance-based, density-based and clustering-based methods all use a distance measure. They differ in what they do with it: raw distance to neighbours, relative local density, or distance to a cluster centre.

### **5d. Hybrid: Density + Distance + Clustering**

* **DBSCAN (Density-Based Spatial Clustering of Applications with Noise):** groups closely packed points into clusters without being told how many. Points in low-density regions belong to no cluster and are labelled noise — those are your anomalies. Handles clusters of irregular shape. Needs two parameters: the neighbourhood radius (`eps`) and the minimum points per cluster.
* **HDBSCAN:** a hierarchical version that copes with clusters of different densities and needs less tuning.

### **5e. Tree and Ensemble Models**

* **Isolation Forest (iForest):** isolates points with random decision trees. Each split picks a random feature and a random split value along that one axis. Anomalies are rare and different, so they are isolated in fewer splits. The anomaly score comes from the average path length across the forest: shorter means more anomalous. Handles multivariate input, scales well and learns complex relationships.
* **Random Cut Forest (RCF):** also isolates points with random cuts, but chooses the cut dimension in proportion to each dimension's range, so wide dimensions are cut more often. It is built for streams: trees are updated incrementally as points arrive and age out. Points that are isolated quickly, or whose insertion would change the tree most, score higher. Sometimes called Robust Random Cut Forest (RRCF). RCF is the algorithm AWS documents for several of its managed anomaly detection features (see Part 3).

**Tooling tip:** [PyOD](https://github.com/yzhao062/pyod) offers 40+ unsupervised detectors (Isolation Forest, LOF, k-NN, autoencoders and more) behind one API, so you can compare several quickly. Its companion [PyThresh](https://github.com/KulikDM/pythresh) turns raw anomaly scores into labels with data-driven thresholds instead of a hand-picked contamination rate.

### **5f. Supervised Regression (Predict, Then Compare)**

Train a model to predict a target metric from contextual features. Large gaps between actual and predicted values (residuals) are anomalies.

* **Random Forest or gradient-boosted regression:** for example, predict the expected transaction amount from merchant category, time and location, then flag large residuals.
* **Contextual target-variable models:** the general pattern of modelling a metric's expected value from its context (segment, time, entity) and flagging deviations. It catches contextual anomalies well and explains them through feature importance.
* **Fully supervised classifiers** become an option when you have labelled anomalies (for example, confirmed fraud). They need class-imbalance handling and can miss new kinds of anomaly.

### **5g. Deep Learning for Sequences and Time Series**

* **LSTM networks:** recurrent neural networks that learn dependencies across a sequence. They catch suspicious behaviour that no single event shows. For example, an LSTM trained on login session sequences (login from a known device, browser and location → browse → checkout → log out) learns normal flows and can spot credential-stuffing rings: coordinated, often automated, login attempts using leaked usernames and passwords that rotate IPs, devices and user agents.
* **LSTM autoencoders:** learn to reconstruct normal sequences. A high reconstruction error flags an anomaly.
* **DeepAR:** an LSTM-based probabilistic forecasting model trained across many related time series at once. It learns shared patterns, which helps series with little history. Flag actuals outside its predicted quantiles.
* **Transformers** for time series are a newer alternative with the same forecast-or-reconstruct pattern.

## **6. Other Models**

* **One-Class SVM:** learns a boundary around normal data in a high-dimensional feature space. Points outside it are anomalies. Sensitive to kernel and parameter choices; slow on large datasets.
* **AutoGluon:** an open-source AutoML library. Its EDA module includes an [automatic anomaly detection](https://auto.gluon.ai/dev/tutorials/eda/eda-auto-anomaly-detection.html) workflow that is a fast way to get a baseline.
* **AnoLLM:** a [research framework (ICLR 2025)](https://proceedings.iclr.cc/paper_files/paper/2025/hash/165bbd0a0a1b9470ec34d5afec582d2e-Abstract-Conference.html) that uses large language models for unsupervised anomaly detection on tabular data. It serialises each row as text, fine-tunes a pre-trained LLM on it, and scores rows by the LLM's negative log-likelihood. Its edge is on mixed-type tables (numeric, categorical and free-text columns); on purely numeric data it is roughly on par with classic methods, at much higher cost.

---

## **Which Model Should You Use?**

Start with two or three candidates, test them on your own data, and choose from the results. For time series, the usual shortlist is:

* **Statistical:** Poisson (counts), z-score/MAD, Holt-Winters, Prophet
* **ML:** Random Cut Forest, Isolation Forest, Local Outlier Factor
* **Managed services** that run these kinds of models for you (see Part 3)

### **Selection Factors**

| Factor | Question to ask | How it narrows the choice |
| --- | --- | --- |
| Data privacy | Aggregated metrics, or row-level data with PII? | Aggregated device or system metrics carry less risk. Row-level or PII data must follow privacy rules, which may rule out some services, regions or LLM-based methods |
| Nature of the data | Distribution, seasonality, trend, sparsity, noise? Stationary or intermittent? | Seasonal and dense → Holt-Winters, Prophet, RCF. Sparse counts → Poisson or negative binomial. Heavy tails → MAD or IQR instead of z-score |
| Univariate vs multivariate | One metric, or many together? | Holt-Winters and ARIMA are univariate. RCF, Isolation Forest, LOF and autoencoders handle many dimensions |
| Anomaly type | Point, contextual or collective? | Point → statistical, iForest. Contextual → per-entity or conditional models. Collective → sequence models, shingling, windowed features |
| Contextual dimensions and cardinality | Per device, customer, region? How many entities? | Low cardinality → one model per series. High → per-entity models at scale, or a global model with entity features |
| Detection mode | Real-time (online) or batch (offline)? | Streaming → RCF, online statistical models. Batch → iForest, LOF, DBSCAN, deep models |
| Labels | Do confirmed historical anomalies exist? | Yes → supervised or semi-supervised models, plus labels to measure precision and recall. No → unsupervised only |
| Explainability | Do business or compliance need reasons for each flag? | Rules, statistical and tree-based models explain better than deep models or LLMs |
| Scale and cost | Rows, features, entities, and latency budget? | k-NN, LOF and One-Class SVM scale poorly. iForest and RCF scale well. LLMs are costly per row |
| Drift and maintenance | How fast does normal change? | Fast change → models that update online, plus a retraining schedule and monitoring of alert rate and drift |

### **Quick Decision Guide**

| If your situation is… | Start with |
| --- | --- |
| Event counts per interval (logins per hour) | Poisson or negative-binomial tail test |
| One metric with daily/weekly patterns | Holt-Winters, Prophet, or a managed metric anomaly detector |
| Multivariate numeric data, streaming | Random Cut Forest |
| Multivariate tabular data, batch | Isolation Forest, compared with LOF (try both via PyOD) |
| Clusters of different shapes or densities | DBSCAN / HDBSCAN, or LOF |
| Clear target metric with drivers | Regression on context, flag large residuals |
| Event sequences (sessions, clickstreams) | LSTM or LSTM autoencoder |
| Mixed numeric, categorical and text columns | Tree models on encoded features, or AnoLLM in research settings |
| Known bad patterns that must never slip through | Deterministic rules, layered on top of any model |

### **Evaluating Candidates**

* **With labels:** precision, recall, F1, and precision at the top k alerts (what reviewers actually see).
* **Without labels:** inject synthetic anomalies (spikes, level shifts, bursts) into real data and measure how many are caught. Also review a sample of top-scored points with domain experts.
* **Operationally:** alerts per day per analyst, time to detect, and the share of alerts that lead to action.
* **Thresholds:** decide how scores become alerts — a fixed cut-off, a percentile, a contamination rate, or a data-driven method like PyThresh. Revisit them as data drifts.

---

## **Deep Dives**

### **Local Outlier Factor (LOF)**

LOF asks whether a point sits in a sparser region than its neighbours do. Because it compares densities locally, it catches outliers next to a dense cluster that a single global threshold would miss.

1. **k-distance:** for each point, the distance to its k-th nearest neighbour.
2. **Reachability distance** of A from B: the larger of B's k-distance and the actual distance from A to B. This smooths out noise inside dense regions.
3. **Local reachability density (lrd):** the inverse of A's average reachability distance to its k neighbours.
4. **LOF score:** A's neighbours' average lrd divided by A's own lrd.

$$\mathrm{LOF}_k(A) = \frac{1}{|N_k(A)|} \sum_{B \in N_k(A)} \frac{\mathrm{lrd}_k(B)}{\mathrm{lrd}_k(A)}$$

* LOF ≈ 1: as dense as its neighbours (normal). LOF clearly above 1 (often 1.5–2+): sparser than its neighbours (outlier).
* **Tuning:** k (`n_neighbors`) is the main knob; 20 is a common start. Scale features first.
* **Limits:** cost grows roughly with the square of the data size; batch only in its basic form; scores are relative, so thresholds differ between datasets.

### **Random Cut Forest (RCF)**

RCF builds a forest of trees over a sample of the data. It scores a point by how much it would change the tree structure: points that are isolated quickly, or that would reshape the tree a lot, score high.

1. **Sample:** keep a fixed-size random sample per tree, using reservoir sampling so the sample stays current in a stream.
2. **Build:** repeatedly take the bounding box of the points, choose a dimension with probability proportional to its range, and cut at a uniform random value in that range.
3. **Score:** insert the point and measure its displacement (how many points it separates from) or its depth. Average across trees and normalise by sample size.
4. **Stream:** as new points arrive, insert them and drop old ones. The model adapts without retraining.

**Shingling** turns a univariate series into vectors of the last *w* values — for example, a window of 8 intervals. RCF then scores the shape of the recent window, which catches collective anomalies and shifts in pattern as well as single spikes.

Key parameters: number of trees (often 50–100), sample size per tree (often 256), shingle size, and the score threshold.

The original paper is [Guha et al., Robust Random Cut Forest Based Anomaly Detection on Streams (ICML 2016)](http://proceedings.mlr.press/v48/guha16.pdf).

### **Isolation Forest vs Random Cut Forest**

| | Isolation Forest | Random Cut Forest |
| --- | --- | --- |
| Choosing the split dimension | Uniformly at random | In proportion to each dimension's range |
| Split | Axis-aligned, random value | Axis-aligned, random value within the bounding box |
| Score | Average path length to isolate the point | Displacement (or depth) caused by inserting the point |
| Data handling | Batch: retrain to update | Streaming: incremental insert and delete |
| Irrelevant dimensions | Picked as often as useful ones, which can dilute the score | Narrow-range dimensions are cut less often |
| Where you get it | scikit-learn, PyOD | Open-source random-cut-forest-by-aws library, rrcf (Python), several AWS services |

A common misconception is that RCF cuts along hyperplanes at arbitrary angles. Classic RCF cuts are axis-aligned, like Isolation Forest's. The main differences are the range-weighted choice of dimension, the displacement-based score, and streaming updates. *Extended* Isolation Forest is the variant that uses angled cuts.

### **Poisson Distribution for Count Anomalies**

Use a Poisson model when the metric is a count of events in a fixed interval: transactions per hour, failed logins per minute, errors per batch. It fits better than a z-score when counts are low and skewed.

If λ is the expected count for the interval (for example, the customer's average number of transactions in that hour of the week), the probability of seeing k events is:

$$P(X = k) = \frac{\lambda^{k} e^{-\lambda}}{k!}$$

Flag an interval when the tail probability of seeing at least the observed count falls below a threshold α (for example 0.001):

$$P(X \ge k_{obs}) = 1 - \sum_{i=0}^{k_{obs}-1} \frac{\lambda^{i} e^{-\lambda}}{i!} < \alpha$$

**Running example:** the customer averages λ = 0.5 transactions in a given half hour. Five transactions in 30 minutes has a tail probability of about 0.00017, so it is flagged as a collective anomaly.

Caveats:

* Poisson assumes the mean equals the variance. If the variance is much larger (overdispersion, common with bursty traffic), use a negative binomial model.
* Many empty intervals → consider a zero-inflated model.
* Make λ seasonal (per hour of day or day of week), or the daily rhythm itself will look anomalous.

### **Detecting Data Drift**

Drift detection tells you when the "normal" your model learned is out of date, so you know when to retrain or re-baseline.

| Method | What it compares | Typical use |
| --- | --- | --- |
| Population Stability Index (PSI) | Binned distribution of a feature: reference vs current | Tabular features; PSI > 0.2 is a common alert level |
| Kolmogorov–Smirnov test | Two continuous distributions | One numeric feature at a time |
| Chi-square test | Category frequencies | Categorical features |
| Jensen–Shannon / KL divergence | Probability distributions | Any binned feature |
| ADWIN, Page-Hinkley, DDM | A streaming statistic or error rate over adaptive windows | Online drift in streams |
| Discriminative (classifier-based) detection | Train a classifier to tell old data from new. If it clearly beats chance (AUC well above 0.5), the distributions differ | Multivariate drift; the classifier's features show what changed |

A practical loop: monitor input features and the anomaly rate → alert on significant drift → check whether the change is real (a new normal) or a problem → retrain or reset baselines when it is the new normal.

---

## **References**

* [Guha et al., Robust Random Cut Forest Based Anomaly Detection on Streams (ICML 2016)](http://proceedings.mlr.press/v48/guha16.pdf)
* [Tsai et al., AnoLLM: Large Language Models for Tabular Anomaly Detection (ICLR 2025)](https://proceedings.iclr.cc/paper_files/paper/2025/hash/165bbd0a0a1b9470ec34d5afec582d2e-Abstract-Conference.html)
* [PyOD](https://github.com/yzhao062/pyod) · [PyThresh](https://github.com/KulikDM/pythresh) · [AutoGluon anomaly detection](https://auto.gluon.ai/dev/tutorials/eda/eda-auto-anomaly-detection.html)

---

**Previous:** [Part 1 — Fundamentals](anomaly_detection_fundamentals.md) · **Next:** [Part 3 — Anomaly Detection in AWS Services](anomaly_detection_in_aws_services.md)
