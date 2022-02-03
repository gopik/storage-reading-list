# Introduction
https://www.influxdata.com/blog/apache-arrow-parquet-flight-and-their-ecosystem-are-a-game-changer-for-olap/
# Parquet
# Arrow
# DeltaLake
[Delta Lake: High-Performance ACID Table Storage over
Cloud Object Stores](https://cs.stanford.edu/people/matei/papers/2020/vldb_delta_lake.pdf)

Builds an access pattern over the cloud data store to improve the big data workflows in different ways.
The key change is, there's a metadata about the objects maintained which provides a consistent view to the object store.

The metadata updates are serialized either via the atomic facilities of the object store provider or some external synchronization.
Updates to the datastore is visible only via metadata updates. This makes the object store view consistent. There can be any number
of change that can be made to the object store, but they are not visible to readers until the metadata is updated. Metadata is updated
by atomically making the set of updates visible at the same time.

Although the paper claims this protocol achieve a serializable isolation, it appears to achieve snapshot serialization if a read/update transaction is considered. It's a weaker version of snapshot isolation since only the commit record is serialized and conflicts within commit records doesn't abort the transaction (which can be a big computation).

# SparkSQL optimizer

# DataFusion
## NYC Taxi data set benchmark using datafusion
* Dataset download - from https://github.com/cswinter/LocustDB/ https://www.dropbox.com/sh/4xm5vf1stnf7a0h/AADRRVLsqqzUNWEPzcKnGN_Pa?dl=0
* Copy zipped data to amazon ec2
* Unzip data as csv into s3 bucket - https://medium.com/tensult/aws-how-to-mount-s3-bucket-using-iam-role-on-ec2-linux-instance-ad2afd4513ef
* fuse-devel libcurl4-dev libxml2-dev libssl-dev_


# Components
* Client
  * Browser that connects to sql frontend
* SQL frontend
  * http api to run sql on tables provided by catalog
* Ballista scheduler and workers
* Catalog
  * Map the data into tables to be queried - Is it manually created, like pointing a table to blob path prefix?
* Data
  * Raw data pointed to by the catalog - Data is produced by existing systems.

# Handling updates to Parquet files
Consider parquet "table" partitioned into multiple files. Let's call this source. Let's say we have changes to certain rows of parquet table as another parquet table (or sstable). Objective is to create a new parquet table that causes smallest number of reads and writes overall.

# Trivial implementation

1. Set sourceRowGroup = 0 and updateRowGroup = 0, isUpdateRowGroupEmpty = `True`
2. Read the source row group `sourceRowGroup`, `sourceRowGroup++`.
3. if isUpdateRowGroupEpty: Read the update row group `updateRowGroup`, `updateRowGroup++`, `isUpdateRowGroupEmpty = False`.
4. If there's an intersection between rows of source and update row groups, update sourceRowGroup from the intersection. Write sourceRowGroup
5. Remove intersection from updateRowGroup. if updateRowGroup is empty, `isUpdateRowGroupEmpty = True`.
6. Goto Step 2.

In this approach, we read the source table and update table and write source table. So the cost is R(Source) + R(Update) + W(Source). One optimization that can be made is to skip writing a row group 

# Skip unmodified RowGroups
In this implementation, the parquet file is set as externalized row groups. The parquet file is a small metadata file that contains references to row groups stored in other files. Each update to parquet is done by only writing row groups that have been modified and then updating the parquet file to contain the reference to new row group file.

So the cost of this approach is R(Source) + R(Update) + W(UpdatedRowGroup) + W(Metadata)

# Don't read all row groups
In this implementation, the parquet file is a metadata file with all row groups externalized. Apart fromt this, we also store row group statistics in the metadata file. First update file is read and we identify all the row groups that need to be updated. Then we read the row groups that are relevant only rewrite those.

The cost of this approach is

2*R(Update) + R(UpdatedRowGroups) + W(UpdatedRowGroups) + W(Metadata)