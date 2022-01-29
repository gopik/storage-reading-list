# OpenTSDB using RocksDB

OpenTSDB is a timeseries database that makes clever use of hbase and can scale to large clusters. The key advantage of OpenTSDB is, it supports data
inserts at any timestamp instead of append only dbs like many other tsdbs.
OpenTSDB needs hbase for persistence which needs a hadoop cluster. To make a self contained TSDB server unit, we can replace the hbase dependency using rocksdb which is an embedded database.

# Log search/analysis using datafusion

Datafusion is a an apache project that enables full sql querying on lot of sources and allows customization of data source. We can build a columnar data source for the log files and can make a cheaper logsearch instead of maintaining an ELK cluster by offloading persistence to blobstore.

# Advanced task scheduler

Build a task scheduler that is more intelligent than a queue. A useful task scheduler ensures atleast one semantics and ensures that at most one worker is executing the task. Enhance such scheduler that shares a worker pool across many queues and ensures fairness and efficiency.

# Build a ACID compliant transactional store on top of AP storage systems (like cassandra/couchbase etc)

Using an existing scalable database, build a separate transactional layer.

# ceph file system on azure

Ceph is a file system that is built on top of OSDs which are primarily an object store. Replace OSDs in ceph with existing cloud blob stores like azure or s3 and use it as a distributed file system in respective cloud.