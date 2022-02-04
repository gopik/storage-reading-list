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

At a high level this system appeas as below:

<img src=>