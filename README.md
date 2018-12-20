# Spark profiling tools

This repository summarizes some profiling tools for Spark jobs.

## Environment

### YARN

Some profiling tools requires YARN. For test purpose, it is easier to setup YARN cluster locally with Docker. The Dockerfile here is based on the [docker-yarn-cluster](https://github.com/lresende/docker-yarn-cluster) project. Besides YARN, the docker-compose.yml adds InfluxDB, MySQL and Dr. Elephant services. InfluxDB is required by statsd-jvm-profiler, and MySQL is used by Dr. Elephant.

### Build YARN docker image

    docker build  -t yarn-cluster .
    
In order to work with Dr. Elephant, this docker image adds MapReduce jobhistory in namenode container, which is not invoked in original container in [docker-yarn-cluster](https://github.com/lresende/docker-yarn-cluster).

#### Hadoop configuration

The config files for Hadoop are in [etc/hadoop](https://github.com/viirya/spark-profiling-tools/tree/master/etc/hadoop).

Dr. Elephant needs to enable `dfs.webhdfs.enabled` in `hdfs-site.xml` and specify `mapreduce.jobhistory.webapp.address` in `mapred-site.xml`, `yarn.resourcemanager.hostname` in `yarn-site.xml`.

Babar needs to enable `yarn.log-aggregation-enable` in `yarn-site.xml`.

The Docker containers use these Hadoop configuration files by Docker volumn.


### Prepare Spark

The Dockerfile doesn't include Spark distribution. After launching the docker containers, prepare Spark environment inside the container `namenode`:

    # Run docker containers to setup YARN cluster
    docker-compose -f docker-compose.yml up -d
    # Gets Spark distribution
    docker exec -it namenode /bin/bash   
    curl https://archive.apache.org/dist/spark/spark-2.3.0/spark-2.3.0-bin-hadoop2.7.tgz > spark-2.3.0-bin-hadoop2.7.tgz
    tar xfvz spark-2.3.0-bin-hadoop2.7.tgz

## Dr. Elephant

### Features

* Based on YARN
* Profiling MapReduce and Spark jobs
* Not compatible to Spark 2.X...
    * https://github.com/linkedin/dr-elephant/issues/327
* Stores the analyzed results in MySQL
* Runs as a standalone application
    * Reads job history data from YARN web API/Spark Job history server API...

### Docker

There is a [Dockerfile](https://github.com/viirya/dr-elephant-docker) that is ready to use for test purpose.

Note that when building the docker image, currently the test phase in `compile.sh` in Dr. Elephant is failed. A work around solution to this issue it to skip test phase by changing `compile.sh` and add modifed `compile.sh` in the Docker image.

Note that in order to profile Spark jobs, the environment variable `SPARK_HOME` is required to set in the Dockerfile.

### Usage

After building the image of Dr. Elephant, submit a Spark job and profile it inside the docker container `namenode`. Don't forget to enable eventlog for Spark:

    # Remember to create `/spark-history` on HDFS before submitting this job
    YARN_CONF_DIR=/usr/local/hadoop/etc/hadoop ./bin/spark-submit --class org.apache.spark.examples.SparkPi --master yarn --deploy-mode cluster --executor-cores 1 --executor-memory 2g --driver-memory 1g --conf spark.eventLog.enabled=true --conf spark.eventLog.dir=hdfs://namenode:8020/spark-history examples/jars/spark-examples*.jar 10

Then open browser to `http://localhost:9001` to see profiling result. Note Dr. Elephant doesn't support Spark 2.x for now.

## Babar

### Features

* Based on YARN
* Profiling YARN jobs
    * MapReduce, Spark, Hive...
* Java-agent based
* Metrics
    * CPU/Memory, GC, I/O, stacktraces
* Collects logs by using YARN log aggregation
* Works without additional databases or visualization tools
    * Generates HTML page which integrates all profiling metrics

### Usage

It is relatively easier to use Babar, compared with other tools. After building, the jar files of babar-agent and babar-processor are available to profile Spark jobs and produce HTML reports.

After babar jar files are built, submit Spark job and profile it inside YARN docker container `namenode`:

    # Run and profile Spark job 
    YARN_CONF_DIR=/usr/local/hadoop/etc/hadoop ./bin/spark-submit --class org.apache.spark.examples.SparkPi --master yarn --deploy-mode cluster --executor-cores 1 --executor-memory 2g --driver-memory 1g --files /path_to_babar/babar-agent/target/babar-agent-0.2.0-SNAPSHOT.jar --conf spark.executor.extraJavaOptions="-javaagent:./babar-agent-0.2.0-SNAPSHOT.jar=StackTraceProfiler,JVMProfiler[reservedMB=2560],ProcFSProfiler" examples/jars/spark-examples*.jar 10
    # Collects logs
    /usr/local/hadoop/bin/yarn logs --applicationId=application_1545097810792_0001 > SparkApp.log
    # Produces HTML report
    java -jar /path_to_babar/babar-processor/target/babar-processor-0.2.0-SNAPSHOT.jar SparkApp.log
  
## statsd-jvm-profiler

### Features

* Profiling Hadoop jobs
* Java-agent based
* Metrics
    * CPU/Memory, GC, stacktraces
* Aggregates logs by using InfluxDB or StatsD
* Provides dashboard for InfluxDB backend
    * Node.js app
    * Not updated recently...
* Strange design of metric prefix
    * User/Job/Flow/Stage/Phase/JVM
    
### Usage

statsd-jvm-profiler supports two backends: InfluxDB, StatsD. Here we use InfluxDB. The `docker-compose.xml` will run a InfluxDB container for it.

Because this is also java-agent based, the usage is quite similar to babar:

    # Run and profile Spark job 
    YARN_CONF_DIR=/usr/local/hadoop/etc/hadoop ./bin/spark-submit --class org.apache.spark.examples.SparkPi --master yarn --deploy-mode cluster --executor-cores 1 --executor-memory 2g --driver-memory 1g --files /path_to_statsd-jvm-profiler/target/statsd-jvm-profiler-2.1.1-SNAPSHOT-jar-with-dependencies.jar --conf spark.executor.extraJavaOptions="-javaagent:./statsd-jvm-profiler-2.1.1-SNAPSHOT-jar-with-dependencies.jar=server=172.17.0.6,port=8086,reporter=InfluxDBReporter,profilers=MemoryProfiler:CPUTracingProfiler:CPULoadProfiler,username=admin,password=admin,database=sparkdb,prefix=statsd-jvm-profiler.spark.spark.spark.spark.spark.oracle,tagMapping=prefix.username.job.flow.stage.phase.jvm" examples/jars/spark-examples*.jar 10

The parameters for statsd-jvm-profiler is a bit complicated. If using InfluxDB as backend, `tagMapping` is required to set when submitting Spark job. These tags are added with recorded metrics in InfluxDB when profiling. Because the dashboard app provided by statsd-jvm-profiler for InfluxDB uses these tags to query metrics data. You need to specify the tags and prefixes even you don't use them.

### InfluxDB dashboard

statsd-jvm-profiler provides a Node.js app as dashboard for InfluxDB backend. But it is quite out of date. The main issue is that the `influx` npm module it uses is too old and not compatible with latest InfluxDB which is used by the InfluxDB Docker image. I've submitted a [PR](https://github.com/etsy/statsd-jvm-profiler/pull/52) to fix it. But this project seems inactive for a while, I'm not sure if it will be reviewd and accepted. If it is not, you can still use the changed code in the forked branch.

## jvm-profiler

### Features

* Profiling JVM processes
* Java-agent based
* Metrics
    * CPU/Memory, GC, I/O, stacktraces
* Collects logs by using Kafka
* Provides a script to generate flamegraph for stacktraces visualization
    * Lacks built-in support for visualizing other metrics
