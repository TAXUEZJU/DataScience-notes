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
    - Example 2-1. Python line count
    ``` Python
    >>> lines = sc.textFile("README.md")    # Create an RDD called lines
    >>> lines.count()    # Count the number of items in this RDD
    103
    >>> lines.first()    # First item in this RDD, i.e. first line of README.md
    u'# Apache Spark'
    ```
    - Example 2-2. Scala line count
    ``` Scala
    scala> val lines = sc.textFile("README.md")
    lines: org.apache.spark.rdd.RDD[String] = README.md MapPartitionsRDD[1] at textFile at <console>:24

    scala> lines.count()
    res0: Long = 103

    scala> lines.first()
    res1: String = # Apache Spark
    ```
#### Introduction to Core Spark Concepts
- Spark application consists of a ***driver program*** that launches various parallel operations on a cluster.
    - The driver program contains applications's **main** function and defines distributed datasets on the parallel operations on a cluster.
    - In the preceding examples, the driver program is the Spark shell itself.
- Driver programs access Spark through a **SparkContext** object, which represents a connection to a computating cluster.
- In the shell, a **SparkContext** is automatically created as the variable called **sc**.
- Example 2-3. Examining the sc variable
    ``` Python
    >>> sc
    <SparkContext master=local[*] appName=PySparkShell>
    ```
    Once you have a SparkContext, you can use it to build RDDs. In `Examples 2-1 and 2-2`, we called `sc.textFile()` to create an RDD representing the lines of text in a file. We can then run various operations on these lines, such as `count()`.
- To run these operations, driver programs typically manage a number of nodes called **executors**. For example, if we were running the count() operation on a cluster, different machines might count lines in different ranges of the file.
- Figure 2-3. Components for distributed execution in Spark
![Figure2-3](./images/Figure2-3.png)
    - Pass functions to its operators to run them on the cluster
    - Functions based operations like filter also parallelize across the cluster
    - Spark automatically takes function and ships it to executor nodes
    - Write code in a single driver program and automatically run it on multiple nodes
- Extand README example bt filtering the lines in the file that contain a word, such as Python, as shown in `Example 2-4` and `Example 2-5`:
    - Example 2-4. Python filtering example
    ``` Python
    >>> lines = sc.textFile("README.md")
    >>> pythonLines = lines.filter(lambda line: "Python" in line)
    >>> pythonLines.first()
    u'high-level APIs in Scala, Java, Python, and R, and an optimized engine that'
    ```
    - Example 2-5. Scala filtering example
    ``` Scala
    scala> val lines = sc.textFile("README.md")
    lines: org.apache.spark.rdd.RDD[String] = README.md MapPartitionsRDD[1] at textFile at <console>:24

    scala> val pythonLines = lines.filter(line => line.contains("Python"))
    pythonLines: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[2] at filter at <console>:26

    scala> pythonLines.first()
    res0: String = high-level APIs in Scala, Java, Python, and R, and an optimized engine that
    ```

#### Standalone Applications
This part is about how to use Spark in standalone programs.Apart from running interactively, Spark can be linked into standalone application
