
[How to Move Beyond a Monolithic Data Lake to a Distributed Data Mesh](https://martinfowler.com/articles/data-monolith-to-mesh.html)

## Call out 1 : 
Though the principles discussed in data-mesh architecture provides good concepts/guiding pricniples, the statement “It is centralized, monolithic and domain agnostic aka data lake.” in the data-mesh reference does not necessarily hold true for [modern AWS-based Data Lakes](https://docs.aws.amazon.com/wellarchitected/latest/analytics-lens/modern-data-architecture.html).
Since no specific evidence is provided to support the claim, it is reasonable to assume that the “centralized, monolithic, domain-agnostic data lake” critique refers to first-generation on-premises data lakes (HDFS-based or Hadoop clusters) — which indeed had monolithic characteristics .        

## Call out 2 :  
Big Enterprises doesn't have one-centralized team. Each org (departement/domain) will have their dedicated Software Application, Data Engineering, and Data Science teams that publish their data to  their respective modern data lakes.

## Call Out 3 :
Data Mesh’s “decentralized architecture” is about giving each domain the freedom to publish and manage its own data — but within a framework of standardized APIs for data publishing and access. In practice, it’s neither cost-effective nor sustainable for every domain (or department within a company) to build and maintain its own independent data platform ( not referring to the data pipelines). Instead, organizations need a shared platform that provides common infrastructure while still enabling domain-level autonomy. Such a platform should abstract the underlying storage layer (e.g., Amazon S3) and offer separable namespaces with decentralized ownership and governance controls. In essence, this shared yet federated infrastructure represents a Multi-Tenant Data Lake. 
