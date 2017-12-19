### Introduction to Data Analysis with Spark
#### What Is Apache Spark
- A cluster computing platform: *fast* and *general-purpose*
    - **Speed**
        - Extend the popular MapReduce model to efficiently support types of computations, including interactive queries and stream processing;
        - Run programs up to 100x faster than Hadoop MapReduce in memory, or 10x faster on disk
    - **Generality**
        - Cover a wide range of workloads
            batch applications, interactive algorithms, interactive queries, streaming
        - Easy and inexpensive to combine different processing types
        - Reduce the management burden of maintaining separate tools
- Highly accessible, simple APIs in Python, Java, Scala, and SQL, and rich built-in libraries
- Integrate closely with other Big Data tools
- Run in Hadoop clusters and access any Hadoop data source
#### A Unified Stack
- Spark's core is a "computational engine"
    - Schedule, distribute and monitor applications consisting of many computational tasks across many worker machines, or a *computing cluster*
- The core engine of Spark is both fast and general-purpose
    - Power multiple higher-level components specialized for various workloads, such as SQL or machine learning
- Components are designed to interoperate closely
- Benefits of tight integration
    - All libraries and higher-level components benefit from improvements at the lower layers
        - When core engine adds an optimization, SQL and machine learning libraries automatically speed up as well
    - The cost associated with running the stack are minimized
        - Instead of 5-10 independent software systems, only one is needed
        - These costs include deployment, maintenance, testing, support, and others
    - Build applications that seamlessly combine different processing models
        - Use machine learning to classify data in real time from streaming sources
        - Query the resulting data, also in real time, via SQL
- The Spark stack
![](./images/Sparkstack.png)
- **Spark Core**
    - Basic functionality of Spark
        - task scheduling, memory management, fault recovery, interacting with storage systems, and more
    - Home to the API that defines resilient distributed datasets(RDDs), which are Spark's main programming abstraction
        - RDDs: a collection of items distributed across many compute nodes that can be manipulated in parallel
    - Provide many APIs for building and manipulating these collections
- **Spark SQL**
    - Work with structured data
        - Query data via SQL as well as Hive
        - Support many sources of data, including Hive tables, Parquet, and JSON
    - Intermix SQL queries with the programmatic data manipulations supported by RDDs in Python, Java, and Scala
    - This tight integration makes Spark SQL unlike any other open source data warehouse tool
- **Spark Streaming**
