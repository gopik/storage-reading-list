# Building a realtime analytics backend

A typical data platform has following components -

1. Databases (different services use different databases on multiple servers)
2. ETL (Export the data to datawarehouse/datalake)
3. Datawarehouse/Datalake query frontend

The advantage of this setup is, analytic and transactional workloads are totally separated and they can scale independently. Specifically with data lake based systems, another advantage is, there's further decoupling between storage and compute backends. In this design, the writes are performed to the underlying object store directly without going through any DWH system. Queries are handled by stateless compute nodes that directly read data from object store.

One issue with this approach is consistency of query results while the underlying data is being written by data pipelines. This is typically handled by immutable datasets. Each output of the pipeline is a new dataset which is only available after the pipeline completes. This means queries remain consistent to a specific snapshot of the dataset.

## Update/Query Latency
The latency of this analytics system is maximum time between an update and it's result being available in the query. This is defined by the frequency of export pipeline and the time it takes. Let's say we export complete data every hour and it takes 15 minutes to export, the latency would be 1.25 hours.

This export is expensive because it needs to write the entire dataset to get consistent snapshots. Another reason this needs to write the entire dataset is, typically the analytic friendly formats (parquet, orc etc) can't be modified in place. They need to be rewritten to include the updates.

## Delta tables and Streaming ETL
To reduce the latency, we need to fix 2 things
1. Do not rewrite full dataset on export. Ensure the actual writes are minimized.
2. Reduce the delay between exports, instead of running every hour, run it every 5 minutes.

Note, we can only run export every 5 minutes only when the actual export takes much smaller than 5 minutes. The export time is a minimum latency of the system.

### Avoiding full export
To ensure we don't rewrite full dataset on export, we must reorganize the data such that individual files in the dataset aren't too big. Then only the files that contain the changes need to be re written. Then we manage the list of active files via some metadata which should be proportional the file count. Once the files are updated, metadata is updated to replace the old files with updated files.

This is what deltalake provides. The interface to raw object store is now replaced with an interface which now provides a consistesnt collection of objects stored in the underlying object store. It's kept consistent by providing a transactional API to update collection of files. Hence the collection remains consistent even though underlying object store is being modified concurrently by multiple processes.

Another advantage of this collection is, it avoids file listing on object store which is a slow process and delays downstream ops.

But we need more than this to optimize on the writes performend. We need to know which files to write.

### Reduced writes
First the file paths are partitioned by few columns. This means, certain column values retrict the paths on the object store where the rows with those column values are stored. For example, if the table is partitioned by customer id, then all rows containing a specific customer id (say "ABCD") will be found under the path `/customerId=ABCD/*`. There can be more columns used for partitioning.

Second, deltatable file collection stores some column level statistics for every file. Like what are the max and min value for each column contained in the file, count of nulls etc.

Third, instead of exporting the data from scratch, we collect all updates since the last export. Using the above, we identify the list of files (which are not too big, say 10s of MBs compressed). Then for each affected file, a new version is rewritten based on the updates (inserts/modifys/deletes). Once all files across partitions are rewritten, we update the delta table with a transaction that replaces the modified file paths in the collection. Once this transaction commits, the new data is now available to queries started after commit.

## System
At a high level this system appeas as below:

<img src="https://raw.githubusercontent.com/gopik/storage-reading-list/main/DeltaPartitionUpdate.svg">

### DB
This is the transactional component, this can be any number of DBs though. This is what services interact with for transactional load. Like reading rows by primary keys or indices, updating rows etc. All updates to DBs are captured via specific CDC (change data capture) and normalized to a common format (for example using Debezium) and fed to streaming pipeline. Note, the the targets for these updates can be many different analytic tables. Analytic and transactional schemas need not be identical. It's possible that analytic schema is join between different tables (or some other denormalized version). But for this discussion, we'll assume the schemas are identical. To handle more complex case, we'll need another pipeline which consumes the DB changes and output analytic changes.

### Streaming Pipeline
The objective of this component is to minimize reads and writes to the object store. The pipeline is built of multiple worker nodes that manage the updates in parallel. The scaling of the pipeline is acheived by adding more worker nodes. There are 2 key requirements to make this pipeline efficient -

1. The part of the data handled by nodes is mutually exclusive and collectively exhaustive. Mutually exclusive ensurs there are no conflicts.

2. All updates to a given primary key must be handled by the same node. This is required to aggregate updates to the primary key and generate a single update to object store. This is ensured by partitioning the data using primary key components (ideally static partitioning)

The nodes accumulate the recent updates to the db and organize it by object store partitions. Once every few minutes, all the changes since last update are applied to the object store and pipeline is reset. How to do this is explained by using Flink as an example below.

### Partition update service
This service provides an interface for transactional updates to the delta table. The pipeline nodes individually contact and update files independent from each other. Once all updates are complete, a transaction is created to update the deltatable collection that makes all updates visible to the query nodes.

### Object Store
The object store is where all files are persisted and managed using a delta lake. The delta lake manages a catalog of delta tables. These tables are updated by the partition update service. Additionally, the updates can be persisted as independent tables to delta lake which can be combined with primary tables to provide very low latency updates to the real time analytics. To utilize the, the queries must be modified to union the update table with primary table and select the latest version of updated rows and query on top of this.

### Query Nodes
These are pure compute nodes that provide SQL interface, compute a distributed execution plan and scan the underlying delta tables. Example can be sparksql/presto/datafusion ballista etc.

# Streaming update using Flink

Here's how the pipeline would be organized -

<img src="https://raw.githubusercontent.com/gopik/storage-reading-list/main/FlinkDeltaUpdate.drawio.svg">

Initially the CDC is stream is partitioned according to the table partition columns. Then they are sent to the respective nodes. The nodes have list of all the files and their statistics stored locally. For example, a node managing partition 1, 2 and 3 has list of all files that are part of this partition and column statistics of the files. Note this list is from a specific version of the delta table.

As the nodes receive changes from the stream, the node tag the changes to specific files based on the column statistics. There's also inserts outside of these files that are stored separately as simple appends. This is stored as a hashmap where the key is the file and value is the list of row updates.

Once a checkpoint barrier arrives, for each file that has changes, an API request is made to partition update service to update the file with relevant changes. Partition update service creates a new file and merges the changes from old file and updates and writes to new file. It returns the name of the new file to the node.

Once all modified files are updated as above, the file pairs (old and new) are sent to the sink.

The sink node receives updates from all the parittions. Once it receives a checkpoint barrier from all the partitions, it commits the file changes as a transaction to the table by making an API call to partition update service.

The latency of updates depend on the checkpoint frequency of the pipeline which can be a table specific configuration.

In addition to the transactional updates to the delta table, the nodes can keep writing all changes since last checkpoint to a node specific update table. Note that data from different nodes will be in independent tables and won't be consistent. For a low latency inconsistent queries, the query nodes can rewrite the query to include fresh data from update tables.

# Extensions for real time
To be able to get real time results in the analytic queries, this system can be extended (at the risk of increased coupling) by exposing the streaming pipelines state as realtime tables. The query node can merge the results from delta tables and stream realtime tables if required.

For example, we can implement a presto connector that exposes cached updates in the pipeline as tables which can be then unioned with tables stored in object store to make the latest updates available. Note, this will have at most READ COMMITTED consistency, it won’t have dirty reads but the updates at different partitions won’t be consistent.

# Conclusion

We looked at a typical analytics system that refrehes data using an ETL pipeline and looked into an approach that can be used to reduce the latency of the analytics data. The batch pipelines that export the entire dataset are typically done using nigtly jobs that introduce a latency of 24 hours which can be brought down to the order of minutes using streaming ETLs and can be further brought down to seconds by exposing streaming pipeline state to the query nodes.

# References
1. [Debezium](https://github.com/debezium/debezium)
2. [Deltalake and DeltaTables](https://github.com/delta-io/delta/blob/master/PROTOCOL.md)
3. [Flink](https://flink.apache.org/)
