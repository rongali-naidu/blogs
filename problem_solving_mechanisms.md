# **From Observation to Resolution: A Universal Workflow for Solving Problems**

In every domain—software, healthcare, or business—problems arise. Some are small and temporary, others are complex and persistent. How do experts systematically tackle them? The answer lies in a universal workflow:

**Observability → Investigation → Diagnosis → Resolution**

This blog breaks down each stage, shows how it applies across domains, and provides practical techniques to handle issues efficiently.

---

## **1. Observability / Monitoring / Data Collection**

Observability is the ability to **see what’s happening inside a system**. Without it, you can’t detect problems early or understand their impact.

**Examples by domain:**

| Domain          | Observability / Data Collection Examples                               |
| --------------- | ---------------------------------------------------------------------- |
| Software        | Logs, metrics, traces, dashboards (Grafana, Prometheus, Splunk) [6][7] |
| Healthcare      | Vital signs, lab tests, imaging (MRI, X-ray, blood tests)              |
| Business / Data | KPIs, sales dashboards, customer analytics, financial reports          |

**Key point:** Observability provides the raw data that feeds **investigation**. It’s your early-warning system.

---

## **2. Investigation**

Investigation is the process of **collecting and analyzing information to understand a problem**. It turns raw data into actionable clues.

**Techniques by domain:**

| Domain          | Investigation Methods                                      |
| --------------- | ---------------------------------------------------------- |
| Software        | Debugging, log analysis, profiling                         |
| Healthcare      | Clinical examinations, targeted tests, patient history     |
| Business / Data | Exploratory data analysis (EDA), audits, querying datasets |

**A useful technique:** The **5-Whys** [2][3]. By repeatedly asking “Why?” you drill down from symptoms to root cause.

*Example in software:*

* Problem: Server crashed
* Why? → Memory overflow
* Why? → Cache not cleared
* Why? → Cleanup job failed
* Why? → Misconfigured cron job
* Why? → Deployment script overwrote configuration ✅ Root cause identified

---

## **3. Diagnosis**

Diagnosis is **identifying the exact cause** of the problem. It differs from investigation: investigation gathers clues, diagnosis pinpoints the underlying cause.

**Methods by domain:**

| Domain          | Diagnosis Methods                                                                         |
| --------------- | ----------------------------------------------------------------------------------------- |
| Software        | Root Cause Analysis (RCA) [1][5], Fishbone/Ishikawa diagrams [3], Fault Tree Analysis [4] |
| Healthcare      | Medical diagnosis (disease/condition identification)                                      |
| Business / Data | Advanced analytics, predictive modeling, correlation analysis                             |

**Root Cause Analysis (RCA)** is a formal methodology that may combine multiple techniques to identify complex issues.

---

## **4. Resolution**

Resolution is **implementing a solution to fix or manage the problem**. Without it, investigation and diagnosis don’t provide value.

**Examples by domain:**

| Domain          | Resolution Examples                                  |
| --------------- | ---------------------------------------------------- |
| Software        | Bug fix, patch, configuration update, deployment     |
| Healthcare      | Medication, therapy, surgery, lifestyle intervention |
| Business / Data | Process improvement, strategic decision, automation  |

Resolution often feeds back into observability, creating a **feedback loop** for continuous improvement.

---

## **5. Integrated Workflow Diagram**

Here’s a simplified view of the universal workflow:

```
Observability / Monitoring → Investigation → Diagnosis → Resolution → Feedback
```

| Stage         | Purpose                  | Example (Software / Medical / Business)            |
| ------------- | ------------------------ | -------------------------------------------------- |
| Observability | Detect anomalies         | Logs / Vital signs / KPI dashboards                |
| Investigation | Explore and gather clues | Debugging / Clinical tests / Data analysis         |
| Diagnosis     | Identify root cause      | RCA / Disease identification / Analytics modeling  |
| Resolution    | Fix or mitigate          | Bug fix / Treatment / Process improvement          |
| Feedback      | Improve future detection | Enhanced logging / Follow-up care / KPI monitoring |

---

## **6. Key Takeaways**

* This workflow is **domain-agnostic**: the principles apply to software systems, human health, and business operations alike.
* **Observability is critical**: you can’t fix what you can’t see.
* **Investigation and diagnosis are distinct**: one is about clues, the other about the cause.
* **Resolution closes the loop**, and feedback strengthens future detection and prevention.
* Techniques like **5-Whys** [2][3] and **RCA** [1][5] help make investigation and diagnosis structured and effective.

---

## **7. Optional Tools & Techniques by Domain**

| Domain          | Tools / Techniques                                               |
| --------------- | ---------------------------------------------------------------- |
| Software        | Grafana, Prometheus, Splunk, logging frameworks, debugging tools |
| Healthcare      | MRI, X-ray, blood tests, patient history, diagnostic software    |
| Business / Data | Tableau, Power BI, SQL, Python/R for analytics, dashboards       |

---

## **References / Further Reading**

1. **Root Cause Analysis (RCA)**

   * [Splunk: What Is Root Cause Analysis? The Complete RCA Guide](https://www.splunk.com/en_us/blog/learn/root-cause-analysis.html)
   * [ITIL Foundation: Problem Management & RCA](https://www.axelos.com/best-practice-solutions/itil)

2. **5-Whys**

   * [Atlassian Team Playbook: 5 Whys](https://www.atlassian.com/team-playbook/plays/5-whys)
   * [Kainexus Blog: 5 Whys for Continuous Improvement](https://blog.kainexus.com/continuous-improvement/5-whys)

3. **Fishbone / Ishikawa Diagram**

   * [ASQ: Cause-and-Effect (Fishbone) Diagram](https://asq.org/quality-resources/fishbone)
   * [MindTools: Using a Fishbone Diagram](https://www.mindtools.com/pages/article/newTMC_03.htm)

4. **Fault Tree Analysis (FTA)**

   * [ReliabilityWeb: Introduction to Fault Tree Analysis](https://reliabilityweb.com/articles/entry/introduction-to-fault-tree-analysis)

5. **RCA / Investigation Techniques in SRE**

   * [Zenduty Blog: Mastering Root Cause Analysis: A Guide for SREs](https://zenduty.com/blog/root-cause-analysis-guide-sre/)

6. **Observability / Monitoring**

   * [Splunk: Observability Engineering Guide](https://www.splunk.com/en_us/blog/learn/observability-engineering.html)
   * [Grafana Labs: Introduction to Observability](https://grafana.com/learn/observability/)

7. **Incident Management / ITIL**

   * [InvGate Blog: Incident Management Lifecycle](https://blog.invgate.com/incident-management-lifecycle)
   * [ITIL Problem Management Guide](https://www.axelos.com/best-practice-solutions/itil)

