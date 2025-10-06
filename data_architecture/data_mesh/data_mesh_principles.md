### Principle 1: Domain Data Ownership

A domain represents an internal division or department within a company that owns a specific part of the business process. These processes are typically automated through multiple micro-services, which generate the corresponding operational data. The domain that owns this operational data is responsible for making it available for analytics and downstream consumption.

### Principle 2: Data as a Product
* Emphasizes data quality. It is a general guiding principle for treating the data and should not tied to any underlying data architecture.
* Providing detailed documentation (metadata) and data lineage
* Ensuring data is available as per SLA and reliable.

### Principle 3: Self-Serve Data Platform
* makes it feasible for domain teams to manage the lifecycle of their data products with autonomy, and utilize the skillets of their generalist developer to do so
* The APIs allow data consumers to discover, learn, access, and use the data products

### Principle 4: Computational Federated Governance

* The governance model that heavily relies on codifying and automated execution of policies at a fine-grained level, for each and every data product.
