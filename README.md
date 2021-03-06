# ETL Light

A light and rather effective ETL job based on Spark. Targeted for Kafka to destination storage (HDFS, should be extended to S3) extract-transform-load use cases.

A single job run goes through the following steps:

* Reads the last state file (Json format) containing consumed Kafka topics/partitions offsets of the last successful job run.
* On the first run (no available state yet) or whenever a new topic or partition is added, a configuration property is used to determine the starting offset (smallest / largest) to be consumed.
* Processing is done per topic/partition (max of one Spark executor per Kafka partition).
* Each partition processing transform available events read from Kafka into an Avro GenericRecord and saves them into a Parquet format file partitioned by type (using configuration to map event types into base folder names) and time (using a timestamp field of the read events to map into folder time partition).
* Only after successful processing of all partitions - a new state file is saved in target storage (e.g. HDFS/S3) to be used as a starting point by the next job run.


Features the following:

* **Scalable:** by leveraging Spark framework, scalability is achieved by default. Using Kafka RDD source enables maximum efficiency having as many executors as the number of topics partitions (can be configured for less). 

* **Easy upgrades:** as this ETL runs as a Spark job either on demand or more typically using some external job-scheduler (e.g. oozie), upgrading it means stopping current job-scheduling, deploying the new artifacts (jar, config) and restarting job-scheduling (no management of long running processes lifecycle running in multiple machines).

* **Kafka consumed topics/partitions tracking and transparancy:** all consumed Kafka topics/partitions offsets within a single executed job are being committed (saved) into a single file (in Json format) as the last step of the job execution, apart from using this state as a starting point for the next job to run it also provides an easy way to view the history of recent job executions (number of recent files to maintain is configurable), their time and topics/partitions/offsets.

* **Exactly once delivery:** this relates to the process of moving events between Kafka to target storage, since each job run starts from the last successful saved state and critical failures exits the job without committing a new state - any job running after that will try to re-consume all events since last successful run until success. duplicates are avoided by creating a unique file names containing the job id, any re-run of a job will use the same job-id and will override existing files.

* **Error handling:** each consumed Kafka event that failed the processing phase (e.g. because of parsing error) will be written into a corresponding output error file including the processing error message so it can be later inspected and resolved.

* **Replay offsets:** As each job run starts from the last saved state file, it is possible to run jobs starting from an older state just by deleting all proceeding state files, this will cause the next job run to consume all events starting from that state offsets.

* **A single job run at a time:** uses a distributed lock (through zookeeper) to promise a single Spark job runs concurrently avoiding write contention and potential inconsistency/corruption.
 
## Configuration

See configuration example under: core/src/main/resources/application.conf

**spark.conf:** properties used to directly configure Spark settings, passed to SparkConf upon construction.

**kafka.topics:** a list of kafka topics to be consumed by this job run.

**kafka.conf:** properties used to directly configure Kafka settings, passed to KafkaUtils.createRDD(...). 

**etl.lock:** can be used to set a distributed lock in order to prevent the same job from running concurrently avoiding possible contention/corruption of data. 

**etl.state:** sets the destination folder that holds the state files and the number of last state files to keep.

**etl.errors-folder:** location of error folder to hold failed processed events (e.g. parse errors).

**etl.pipeline:** defines a pair of transformer and writer used together to create a processing pipeline for processing a single Kafka topic partition. 

**etl.pipeline.transformer.config:** transformer configurations.

**etl.pipeline.writer.config:** writer configurations. 


## Running a single job using spark-submit

Copy assembly jar and application.conf file into \<deploy folder\>. 
 
Run in **yarn client** mode (running driver locally):

    $ cd <deploy folder>
    $ spark-submit --driver-java-options "-DSPARK_YARN_MODE=true" --master yarn-client <assembly jar> <application.conf>
    
Run in **yarn-cluster** mode (running driver in yarn application master):
    
    $ cd <deploy folder>
    $ spark-submit --driver-java-options "-DSPARK_YARN_MODE=true" --master yarn-cluster <assembly jar> <application.conf>

## Running periodically using 'oozie' job-scheduler

[TBD]