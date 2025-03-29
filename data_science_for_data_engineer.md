### From Data Engineering to Data Science: My Journey of Understanding Models and Data Drift

### Note on Data Engineering:
Data Engineering is about building robust data architectures for organizations and comes under the broader umbrell of building Data Warehouse/Data Lakes. 
It involves acquiring data from various systems (RDBMS, NoSQL, Streaming) in various formats (Text, CSV, JSON, Parquet), performing cleaning and transformation, and loading it into data lakes or databases using layered ETL/ELT patterns. This is achieved with the help of orchestration tools (like Apache Airflow) or ETL tools (like Informatica) and data processing engines (Big Data engines like Spark or RDBMS).
Data modeling follows denormalization-based techniques like star and snowflake schemas.
While ETL tools, databases, data formats, data processing engines, storage systems, data scale, and business domains vary, the core concepts remain consistent and form the foundations of data engineering.
When I started with Data Warehousing projects, my journey began as a DW Engineer (a combination of ETL and BI enginner) with Ralph Kimball’s books and blogs guiding me on DW concepts (data modeling and ETL Design), Informatica as the ETL tool, Oracle as the DW database, and Cognos as the reporting tool. It’s been 20 years, and while ETL tools/Orchestration tools, databases, data formats, data processing engines, storage systems, data scale, and business domains vary, the core concepts remain consistent and form the foundations of data engineering ... so the essence keep learning and adapting ... some unlearning bound to happen.

#### Introduction
Working as a data engineer for several years alongside data scientists and applied scientists has been an eye-opening journey. I started off focusing purely on data pipelines and ETL processes, but over time, I found myself drawn to understanding how the models I was helping deploy actually worked. One concept that particularly caught my attention was **data drift**—the way models can slowly break down over time as data changes. In this blog, I’ll share my learnings and break down the core ideas behind data science models and data drift.

#### What Does a Model Really Mean?
In simple terms, building a data science model is like trying to find an equation that can predict something using multiple variables. Imagine you’re trying to predict house prices. You have variables like the size of the house, location, number of bedrooms, and maybe even the age of the building.

Mathematically, a model can be represented as an equation like this:

**house_price = x * house_size + y * number_of_bedrooms + z * age_of_building + ...**

Here, `x`, `y`, and `z` are coefficients that we don’t know initially. Finding these coefficients based on past historical data is the essence of model development. In other words, a model is a kind of mathematical equivalent between the **target value** (also called the output) and **input values** (also called features). The challenge is, you don’t know from the start which variables are the most important. That’s where **feature selection** comes into play.

Feature selection is like carefully picking ingredients for a recipe—you need just the right combination to make it work. Data scientists perform **Exploratory Data Analysis (EDA)** to understand the relationships between variables and the target outcome. They calculate metrics like the **correlation coefficient** to see how strongly a variable relates to the target. For example, if larger house size generally leads to a higher price, you might see a positive correlation.

But it’s not just about finding correlations. Data scientists also look at **relative weights**, which help understand the contribution of each variable to the prediction. Sometimes, variables that seem correlated might not actually add much predictive power. That’s why **cleaning and transformation** are crucial—handling missing values, fixing outliers, and encoding categorical variables (like converting house types into numerical codes) are all part of preparing data for modeling.

#### Training and Deployment
Once the data is cleaned and the features are selected, the model is trained. This means using historical data to “learn” the patterns and relationships between input variables and the target outcome. After training, the model is validated and tested to make sure it performs well on unseen data. When it’s finally deployed, it’s expected to make accurate predictions on new data.

But here’s the thing—model deployment isn’t the end of the journey. Unlike a static formula, data science models are dynamic—they’re only as good as the data they were trained on. And that’s where data drift comes in.

#### Data Drift: The Real Challenge
Imagine you built a model last year to predict house prices based on features like interest rates, square footage, and neighborhood quality. The model worked great initially. However, over the past year, the economic situation changed, and interest rates soared. Suddenly, your model that once predicted house prices accurately is giving wildly inaccurate results. What happened?

This is **data drift** in action. The relationships between features and the target variable changed, altering the model’s effectiveness. Interest rates might now play a bigger role than they did previously, while neighborhood quality might not correlate as strongly with prices anymore. When data drift occurs, the relative importance of features can shift, and sometimes, new features need to be added while outdated ones are dropped.

#### Dealing with Data Drift
To catch and manage data drift, data scientists monitor model performance regularly. They compare current predictions to actual outcomes and track metrics like accuracy and error rates. They also monitor **feature importance** to see if the most influential variables have changed. If data drift is detected, it may be time to **retrain the model** with the latest data or even redesign it to include new features.

#### Conclusion
Working closely with data scientists has given me a fresh perspective on how fragile models can be without proper monitoring and maintenance. It’s not just about deploying a model and moving on—it’s about continuous observation and adaptation. As data engineers, understanding these concepts makes us better collaborators and helps us build more resilient data pipelines that can support evolving models.

I hope sharing my learnings will help fellow data engineers appreciate the complexity behind model maintenance and data drift. Feel free to share your own experiences and insights in the comments!

