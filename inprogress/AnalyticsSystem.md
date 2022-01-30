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
