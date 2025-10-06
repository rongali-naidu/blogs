
[How to Move Beyond a Monolithic Data Lake to a Distributed Data Mesh](https://martinfowler.com/articles/data-monolith-to-mesh.html)

## Call out 1 : 
Though the principles discussed in data-mesh architecture provides good concepts/guiding pricniples, the statement “It is centralized, monolithic and domain agnostic aka data lake.” in the data-mesh reference does not necessarily hold true for [modern AWS-based Data Lakes](https://docs.aws.amazon.com/wellarchitected/latest/analytics-lens/modern-data-architecture.html).
Since no specific evidence is provided to support the claim, it is reasonable to assume that the “centralized, monolithic, domain-agnostic data lake” critique refers to first-generation on-premises data lakes (HDFS-based or Hadoop clusters) — which indeed had monolithic characteristics .        

## Call out 2 :  
Big Enterprises doesn't have one-centralized team. Each org (departement/domain) will have their dedicated Software Application, Data Engineering, and Data Science teams that publish their data to  their respective modern data lakes.

