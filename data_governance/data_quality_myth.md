
## Why I Think “Data as a Product” Principle in Data Mesh Doesn’t Fix the Data Quality Problem

Couple of years back, We’ve implemented a Data Lakehouse based on AWS’s Modern Data Architecture — combining scalable data lakes with purpose-built analytics systems to support reporting, machine learning, and real-time use cases.

Today, I was going through the book: [Data Mesh: Delivering Data-Driven Value](https://www.amazon.com/Data-Mesh-Delivering-Data-Driven-Value/dp/1492092398). It’s a good read, and I’ve completed the first two chapters so far. While reading, I kept reflecting on how it differs from AWS’s Modern Data Architecture.
One of the foundational principles it emphasizes is “Data as a Product.” This principle suggests that by treating data as a product and assigning ownership to domain teams, organizations can finally address long-standing data quality problems.

It’s a great intention — that data quality should be tracked where the data is generated rather than only where it is consumed — but I believe merely shifting ownership is not enough to fix the data quality problem. The issue runs deeper than ownership; it’s fundamentally about incentives, priorities, and organizational design


### The Real Conflict: Different Goals Between Software Product Teams and Data Consumers

Let’s take an example from an e-commerce platform.
The **Software Product team** managing product listings ensures that users can browse, search, and buy products without issues. Their focus and KPIs revolve around system uptime, performance, and user experience.

Now imagine their product catalog table stores category values inconsistently —
some rows say *“Electronics”*, some say *“Elec”*, and others *“ELC”*.

From the **Software Product team’s perspective**, nothing is broken:

* Users can still shop successfully.
* Orders get fulfilled.
* No alerts are triggered.

So there’s **no immediate incentive** for them to fix these inconsistencies.
But when this data flows downstream for analytics — say, to calculate *sales by category* — the inconsistencies lead to incorrect aggregates and misinformed business decisions.

That’s the root of the conflict:
What’s *“good enough”* for operations can still be **bad data** for analytics, reporting, or ML.


### What Might Work Better

The solution isn’t just architectural — it’s **cultural**. **data quality metrics need to be treated with equal importance as the application’s operational health metrics**.This ensures that domain teams proactively maintain reliable, consistent data rather than relying on downstream users to detect issues.


