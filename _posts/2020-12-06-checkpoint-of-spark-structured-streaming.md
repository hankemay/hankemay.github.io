---
layout: post
title: "Checkpoint of Spark Structured Streaming"
subtitle: "Checkpoint Limitation and Scenarios"
date: 2020-12-06
author: "Hanke"
header-style: "text"
tags: [Spark, Spark Structured Streaming]
---

### Usages in Query
This checkpoint location perserves all the essential information that uniquely identifies a query. Hence, *each query must have a different checkpoint location*, and multiple queries should never have the same location. See the [Structured Streaming Programming Guide](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html) for details.

```scala
noAggDF
.writeStream
.format("parquet")
.option("checkpointLocation", "path/to/checkpoint/dir")
.option("path", "path/to/destination/dir")
.start()
```


### Recovery Semantics after Changes in a Streaming Query
**Types of changes**
Refer to the [page](https://docs.databricks.com/spark/latest/structured-streaming/production.html#types-of-change)  
+ **Changes in the number or type (i.e. different source) of input sources**: This is not allowed.
+ **Changes in the parameters of input sources**: Whether this is allowed and whether the semantics of the change are well-defined depends on the source and the query. Here are a few examples.  
    + <font color="green"><b>Addition/deletion/modification of rate limits is allowed</b></font>: `spark.readStream.format("kafka").option("subscribe", "article")` to `spark.readStream.format("kafka").option("subscribe", "article").option("maxOffsetsPerTrigger", ...)`
    + <font color="red"><b>Changes to subscribed articles/files is generally not allowed as the results are unpredictable</b></font>: `spark.readStream.format("kafka").option("subscribe", "article")` to `spark.readStream.format("kafka").option("subscribe", "newarticle")`
+ **Changes in the type of output sink**: Changes between a few specific combinations of sinks are allowed. This needs to be verified on a case-by-case basis. Here are a few examples.
    + File sink to Kafka sink is allowed. Kafka will see only the new data.
    + Kafka sink to file sink is not allowed.
    + Kafka sink changed to foreach, or vice versa is allowed.
+ **Changes in the parameters of output sink**: Whether this is allowed and whether the semantics of the change are well-defined depends on the sink and the query. Here are a few examples.
    + Changes to output directory of a file sink is not allowed: `sdf.writeStream.format("parquet").option("path", "/somePath")` to `sdf.writeStream.format("parquet").option("path", "/anotherPath")`
    + <font color="green"><b>Changes to output article is allowed</b></font>: `sdf.writeStream.format("kafka").option("article", "somearticle")` to `sdf.writeStream.format("kafka").option("path", "anotherarticle")`
    + Changes to the user-defined foreach sink (that is, the ForeachWriter code) is allowed, but the semantics of the change depends on the code.
+ **Changes in projection / filter / map-like operations**: Some cases are allowed. For example:
    + <font color="green"><b>Addition / deletion of filters is allowed</b></font>: `sdf.selectExpr("a")` to `sdf.where(...).selectExpr("a").filter(...)`.
    + Changes in projections with same output schema is allowed: `sdf.selectExpr("stringColumn AS json").writeStream` to `sdf.select(to_json(...).as("json")).writeStream`.
    + Changes in projections with different output schema are conditionally allowed: `sdf.selectExpr("a").writeStream` to `sdf.selectExpr("b").writeStream` is allowed only if the output sink allows the schema change from "a" to "b".
+ **Changes in stateful operations** - Some operations in streaming queries need to maintain state data in order to continuously update the result. Structured Streaming automatically checkpoints the state data to fault-tolerant storage (for example, DBFS, AWS S3, Azure Blob storage) and restores it after restart. However, this assumes that the schema of the state data remains same across restarts. This means that any changes (that is, additions, deletions, or schema modifications) to the stateful operations of a streaming query are not allowed between restarts. Here is the list of stateful operations whose schema should not be changed between restarts in order to ensure state recovery:
    + <font color="red"><b>Streaming aggregation</b></font>: For example, `sdf.groupBy("a").agg(...)`. **Any change in number or type of grouping keys or aggregates is not allowed**.
    + **Streaming deduplication**: For example, `sdf.dropDuplicates("a")`. Any change in number or type of grouping keys or aggregates is not allowed.
    + **Stream-stream join**: For example, `sdf1.join(sdf2, ...)` (i.e. both inputs are generated with sparkSession.readStream). Changes in the schema or equi-joining columns are not allowed. Changes in join type (outer or inner) not allowed. Other changes in the join condition are ill-defined.
    + **Arbitrary stateful operation**: For example, `sdf.groupByKey(...).mapGroupsWithState(...)` or `sdf.groupByKey(...).flatMapGroupsWithState(...)`. Any change to the schema of the user-defined state and the type of timeout is not allowed. Any change within the user-defined state-mapping function are allowed, but the semantic effect of the change depends on the user-defined logic. If you really want to support state schema changes, then you can explicitly encode/decode your complex state data structures into bytes using an encoding/decoding scheme that supports schema migration. For example, if you save your state as Avro-encoded bytes, then you are free to change the Avro-state-schema between query restarts as the binary state will always be restored successfully.

**Notes**
From [Page](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#additional-information)
+ Several configurations are not modifiable after the query has run. <font color="red"><b>To change them, discard the checkpoint and start a new query</b></font>. These configurations include:
    + <font color="red"><b>spark.sql.shuffle.partitions</b></font>
        + This is due to the physical partitioning of state: state is partitioned via applying hash function to key, hence the number of partitions for state should be unchanged.
        + If you want to run fewer tasks for stateful operations, coalesce would help with avoiding unnecessary repartitioning.
            + After coalesce, the number of (reduced) tasks will be kept unless another shuffle happens.
    + **spark.sql.streaming.stateStore.providerClass**: To read the previous state of the query properly, the class of state store provider should be unchanged.
    + **spark.sql.streaming.multipleWatermarkPolicy**: Modification of this would lead inconsistent watermark value when query contains multiple watermarks, hence the policy should be unchanged.

### Local Env Checkpoint Exmaple
On Spark3  
```scala
val df = spark.readStream.schema(L0.Transaction).option("maxFilesPerTrigger", 1).parquet(jobList(0))
```

```bash
 stream3 ●  ll
total 8
drwxr-xr-x  6 hmxiao  FREEWHEELMEDIA\Domain Users   192B Dec  7 12:20 commits
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users    45B Dec  7 12:20 metadata
drwxr-xr-x  6 hmxiao  FREEWHEELMEDIA\Domain Users   192B Dec  7 12:20 offsets
drwxr-xr-x  3 hmxiao  FREEWHEELMEDIA\Domain Users    96B Dec  7 12:20 sources
drwxr-xr-x  3 hmxiao  FREEWHEELMEDIA\Domain Users    96B Dec  7 12:20 state
```

#### Commits Folder
Content
```bash
 stream3 ●  ll
total 40
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users    41B Dec  7 12:34 0
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users    41B Dec  7 12:34 1
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users    41B Dec  7 12:34 2
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users    41B Dec  7 12:34 3
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users    41B Dec  7 12:34 4
 stream3 ●  cat 0
v1
{"nextBatchWatermarkMs":1607315657155}%
 stream3 ●  cat 1
v1
{"nextBatchWatermarkMs":1607315657155}
 stream3 ●  cat 2
v1
{"nextBatchWatermarkMs":1607315670060}
```

#### Offsets Folder
```bash
 stream3 ●  ll
total 40
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users   470B Dec  7 12:34 0
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users   482B Dec  7 12:34 1
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users   482B Dec  7 12:34 2
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users   482B Dec  7 12:34 3
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users   482B Dec  7 12:34 4
stream3 ●  cat 0
v1
{"batchWatermarkMs":0,"batchTimestampMs":1607315657155,"conf":{"spark.sql.streaming.stateStore.providerClass":"org.apache.spark.sql.execution.streaming.state.HDFSBackedStateStoreProvider","spark.sql.streaming.join.stateFormatVersion":"2","spark.sql.streaming.flatMapGroupsWithState.stateFormatVersion":"2","spark.sql.streaming.multipleWatermarkPolicy":"min","spark.sql.streaming.aggregation.stateFormatVersion":"2","spark.sql.shuffle.partitions":"8"}}
{"logOffset":0}%
stream3 ●  cat 1
v1
{"batchWatermarkMs":1607315657155,"batchTimestampMs":1607315665204,"conf":{"spark.sql.streaming.stateStore.providerClass":"org.apache.spark.sql.execution.streaming.state.HDFSBackedStateStoreProvider","spark.sql.streaming.join.stateFormatVersion":"2","spark.sql.streaming.flatMapGroupsWithState.stateFormatVersion":"2","spark.sql.streaming.multipleWatermarkPolicy":"min","spark.sql.streaming.aggregation.stateFormatVersion":"2","spark.sql.shuffle.partitions":"8"}}
{"logOffset":1}%
stream3 ●  cat 2
v1
{"batchWatermarkMs":1607315657155,"batchTimestampMs":1607315670060,"conf":{"spark.sql.streaming.stateStore.providerClass":"org.apache.spark.sql.execution.streaming.state.HDFSBackedStateStoreProvider","spark.sql.streaming.join.stateFormatVersion":"2","spark.sql.streaming.flatMapGroupsWithState.stateFormatVersion":"2","spark.sql.streaming.multipleWatermarkPolicy":"min","spark.sql.streaming.aggregation.stateFormatVersion":"2","spark.sql.shuffle.partitions":"8"}}
{"logOffset":2}%
```
.....

#### Sources Folders
```bash
sources/0   stream3 ●  ll
total 32
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users   143B Dec  7 12:34 0
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users   143B Dec  7 12:34 1
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users   143B Dec  7 12:34 2
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users   143B Dec  7 12:34 3
stream3 ●  cat 0
v1
{"path":"file:///Users/hmxiao/Documents/Freewheel/Data/Stream/2019010315--20190110-080000.gz.parquet","timestamp":1566471227000,"batchId":0}%
stream3 ●  cat 1
v1
{"path":"file:///Users/hmxiao/Documents/Freewheel/Data/Stream/2019083120--20190902-030000.gz.parquet","timestamp":1567479235000,"batchId":1}%
stream3 ●  cat 2
v1
{"path":"file:///Users/hmxiao/Documents/Freewheel/Data/Stream/2019071603--20190806-080000.gz.parquet","timestamp":1595325060000,"batchId":2}%
```

#### State Folders
```bash
state/0/0   stream3 ●  ll
total 40
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users    46B Dec  7 12:34 1.delta
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users    46B Dec  7 12:34 2.delta
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users   625B Dec  7 12:34 3.delta
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users   548B Dec  7 12:34 4.delta
-rw-r--r--  1 hmxiao  FREEWHEELMEDIA\Domain Users    46B Dec  7 12:34 5.delta
```

### How to Use the Checkpoint
[Checkpoint on S3](https://cm.engineering/using-hdfs-to-store-spark-streaming-application-checkpoints-2146c1960d30)

[Checkpoint on Mounted EFS](https://blog.yuvalitzchakov.com/improving-spark-streaming-checkpoint-performance-with-aws-efs/)

[Customized Solution](https://www.qubole.com/blog/structured-streaming-with-direct-write-checkpointing/)

#### Our Solution
`HDFS + Backup on S3 + Sync latest Checkpoint when Re-Provision a Cluster`
Refer to the solution in [here](https://cm.engineering/using-hdfs-to-store-spark-streaming-application-checkpoints-2146c1960d30)

```shell
#1) Put the checkpoint in HDFS
noAggDF
.writeStream
.format("parquet")
.option("chgeckpointLocation", "hdfs://checkpoint_dir")

#2) Copy checkpoints from HDFS to local disk
hdfs dfs -copyToLocal hdfs:///[checkpoint_dir] /tmp/[checkpoint-dir]

#3) Copy from local disk to S3 (to avoid using the cluster's computing resources)
aws s3 cp --recursive /tmp/[checkpoint-dir] s3://[destination-path]

#4) When to switch the cluster, sync the latest checkpoint to the new cluster
```

### Reference
+ Change which can't be recovered from the checkpoint [Guide][1]
+ [Production][2] on Databricks  

[1]: https://docs.databricks.com/spark/latest/structured-streaming/production.html#recover-after-changes-in-a-streaming-query
[2]: https://docs.databricks.com/spark/latest/structured-streaming/production.html

