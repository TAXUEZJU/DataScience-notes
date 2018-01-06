### Chapter2. Downloading Spark and Getting Started
#### Downloading Spark
- Visit https://spark.apache.org/downloads.html, select the package type matching Hadoop version on your machine, and download package.
- Unpack the package with `tar` or any other tools.
    ```
    tar -xf spark-2.2.0-bin-hadoop2.7.tgz
    ```
- We will run Spark in *local mode* first; that is, nondistributed mode, which uses only a single machine. Beyond local mode, Spark can also be run on Mesos, YARN, or the Standalone Scheduler included in the Spark distribution.
#### Introduction to Spark's Python and Scala Shells
- To open the Python version of the Spark shell, which we also refer to as the PySpark Shell, go into your Spark directory and type:
`bin/pyspark`
To open the Scala version of the shell, type:
`bin/spark-shell`
- You can control the verbosity of the logging by creating a file in the *conf* directory called `log4j.properties`. To make the logging less verbose, make a copy of `conf/log4j.properties.template` called `conf/log4j.properties` and find the following line:
`log4j.rootCategory=INFO, console`
Then lower the log level so that we show only the WARN messages, and above by changing it to the following:
`log4j.roorCategory=WARN, console`
- Before we say more about RDDs, let’s create one in the shell from a local text file and do some very simple ad hoc analysis by following two examples
    - Example 2-1 Python line count
    ``` Python
    >>> lines = sc.textFile("README.md")    # Create an RDD called lines
    >>> lines.count()    # Count the number of items in this RDD
    103
    >>> lines.first()    # First item in this RDD, i.e. first line of README.md
    u'# Apache Spark'
    ```
    - Example 2-2 Scala line count
    ``` Scala
    scala> val lines = sc.textFile("README.md")
    lines: org.apache.spark.rdd.RDD[String] = README.md MapPartitionsRDD[1] at textFile at <console>:24

    scala> lines.count()
    res0: Long = 103

    scala> lines.first()
    res1: String = # Apache Spark
    ```
#### Introduction to Core Spark Concepts
