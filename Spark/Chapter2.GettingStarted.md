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
