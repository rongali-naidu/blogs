### **With Gen AI: Should We Refocus Our Efforts from Semantic Layers to High-Quality Metadata at Datalake/Database Catalog**

The latest technology trend of Business Intelligence (BI) is being driven by Generative AI (Gen AI). Considering how Gen AI translates user intent to SQLs, should we reconsider where our efforts should go to get the most out of the latest trend?

### **The Semantic Layer: A Legacy of Translation and Governance**

Having worked extensively with reporting tools like OBIEE, I've experienced firsthand the value of the semantic layer (names vary based on the reporting tools...some places, its is called Catalog repository). This critical component serves as a translator within BI architecture, bridging the gap between complex database structures and the language of business.

In traditional BI systems, the semantic layer consists of three key sub-layers:

1. **Physical Layer:** Maps directly to the database tables and columns.
2. **Logical Layer:** Provides an abstraction where physical names are translated into business-friendly terms.
3. **Presentation Layer:** Organizes and exposes the logical layer's business terms for reporting and analysis.

When a user creates a report, they interact with the presentation layer using familiar business terminology. The reporting engine then translates these logical names and their corresponding logical SQL into physical SQL queries against the database.

For years, this approach has been the backbone of self-service BI, offering:

- **User Accessibility:** Empowering non-technical users to explore data independently.
- **Consistency:** Ensuring uniform definitions across reports and dashboards.
- **Governance:** Controlling data access and enforcing business rules.

However, as Gen AI reshapes the BI landscape, the necessity of the traditional semantic layer is being called into question.

### **Gen AI and the Rise of Metadata-Driven BI**

Gen AI introduces a paradigm shift by understanding user intent in natural language and converting it directly into SQL queries. This is made possible by analyzing database schemas and interpreting the meaning of tables and columns through comprehensive metadata.
For more details on this point, refer my AWS Blog on [Enriching metadata for accurate text-to-SQL generation for Amazon Athena](https://aws.amazon.com/blogs/big-data/enriching-metadata-for-accurate-text-to-sql-generation-for-amazon-athena/)

Rather than relying on a manually curated semantic layer, modern AI-driven tools leverage the richness of metadata to:

- Recognize relationships between tables.
- Interpret column meanings.
- Translate natural language queries into precise SQL.

This approach raises a compelling question: should we continue investing in traditional semantic layers, or is it time to prioritize robust metadata management?

### **The Case for Doubling Down on Metadata**

I prefer to shift our focus toward enhancing metadata quality and accessibility. A well-maintained metadata catalog, enriched with clear data lineage, offers several advantages:

1. **Direct Support for Gen AI:** Accurate and detailed metadata enables AI to interpret and query data effectively.
2. **Broader Applicability:** Comprehensive metadata serves multiple initiatives, including data governance, data quality, and data discovery.
3. **Automation Potential:** Gen AI can assist in automating metadata enrichment and maintenance, reducing manual overhead.

Imagine a future where your metadata catalog is so rich that AI tools seamlessly navigate your data landscape. Users could ask questions in plain language and receive accurate insights without the need for predefined semantic models.

### **Rethinking the Semantic Layer in a Metadata-First World**

This shift need not mean to render the other principles of the semantic layer obsolete especially governance. By focusing on metadata, organizations can dynamically generate the semantic layer from the high-quality metadata.

### **Conclusion: Embracing Metadata as the Future of BI**

The future of BI is dynamic and intuitive, propelled by AI's ability to directly interpret and query data. While the semantic layer has served its purpose, the time has come to prioritize metadata as the foundation for next-generation data exploration and analysis.


