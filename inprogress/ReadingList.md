# Papers List
## Distributed Hash Table

[Chord: A Scalable Peer-to-peer Lookup Service for Internet Applications](https://pdos.csail.mit.edu/papers/chord:sigcomm01/chord_sigcomm.pdf)

Solves problem of mapping a key to a node. Can be used has key-value lookup service if the node mapped by a key is made responsible for storing the value for the key locally.

If N nodes in the system, each node needs O(logN) for routing and can reach the correct node for the key in O(logN) hops. Chord provides following functionality -


```java
interface Chord {
    IPAddress lookup(String key);
}
interface KeySpaceChangeEvent {
    // whenever node leaves or joins the system
}
```

Responsibilities of application using chord:  authentication, caching, replication, and user-friendly naming of data

### Consistent Hashing
An m-bit ring a set of points identified by m-bit numbers (0 to 2^(m - 1)). Identifiers are mapped on to the points by taking SHA-1 of the id with modulo (2^m). Nodes are mapped onto the circle by hashing ip address of the node and keys are mapped onto the circle by hashing the key. A key is mapped to a node that hashes to same point as key or follows it clockwise in the circle.

### Routing
Each node n maintains a table of size m. Each entry in the table has a reference to the node that is responsible for key `(n + 2^i)` for `0 <= i < m`. Each consecutive pair in this table span `2^i` range of the circle. To route a key, a node finds the smaller pair whose range contains key. Then the node consults the smaller id in this pair for the correct node. Since each successive consultation would map this key to smaller and smaller ranges, eventually it'll find the correct node. This is guaranteed to be done in m steps, but if nodes are distributed uniformly in the circle, it'll finish in `log(n)` steps.

### Node join
Invariants to maintain for corretness -
* Each node knows it's successor
* Each key k is managed by node successor(k)

For performance, the finger table should also be correct


## Atomicity
A detailed introduction to what atomicity means. Covers A and I of ACID.

https://ocw.mit.edu/resources/res-6-004-principles-of-computer-system-design-an-introduction-spring-2009/online-textbook/atomicity_open_5_0.pdf

## DB Schedulers for concurrency control
Book chapters from Concurrency Control and Recovery in Database Systems - https://www.microsoft.com/en-us/research/people/philbe/book/

All chapters on schedulers are for serializable schedules - they achieve Serializable isolation in ACID.
To ensure durability, additional modifications are needed. The schedules need to be recoverable (if crashed in the middle of a transaction, we should be able to remove the affect of unfinished transactions). The undo shouldn't impact committed transactions, avoid cascaded aborts etc.

### Two phase locking
This is a pessimistic approach for concurrency control. Very simple to understand and widely used in practice.

### Non locking scheduler
Doesn't need explicit locking of data items. Does still require serializable access to the data items. This is 
managed by a scheduler that uses timestamps for ordering the access.

### Multi version concurrency control
The data is versioned. Each write creates a new version. Readers never block writers in MVCC.
The scheduler resolves read and write requests to appropriate versions
based on timestamps or locking. MVTO (Multiversion timestamp ordering) uses timestamps to resolve which version to use.
MV2PL typically keeps 2 versions of the data and uses 2PL to synchronize data access. 

## Silo
[Speedy transactions in multicore databases](https://dl.acm.org/doi/10.1145/2517349.2522713)

Implementation of transactions on main memory databases using optimistic concurrency control. Achieves durability using logging.

Introduces interesting concepts like -
* Optimistic concurrency control
* Garbage collection using epochs (RCU)

## Sinfonia
[Sinfonia - A new paradigm for building scalable distributed systems](https://dl.acm.org/doi/10.1145/1629087.1629088)

Introduces mini transactions as an abstractions for the application layer. Similar to Silo's optimistic concurrency control.

Then uses this abstraction to build distributed apps like cluster file system and group communication service.

## Percolator
[Large-scale Incremental Processing Using Distributed Transactions and Notifications](https://research.google/pubs/pub36726)

Uses Bigtable as multi version key value store that provides row level transactions to implement multi table/multi row transactions
with snapshot isolation. There are no other services like lock manager, scheduler, recovery manager etc. Everything is built into
a client library. TiDB is based on this.

## Bw Trees
[The Bw-Tree: A B-tree for New Hardware Platforms](https://www.microsoft.com/en-us/research/publication/the-bw-tree-a-b-tree-for-new-hardware/)

Implementation of B-Trees without using locks/latches. Splits multi step operations which would otherwise need a critical section into
smaller operations that can be implemented using CAS (compare and swap). Also introduces the persistence of the B-Tree in log structured
manner.

## RAFT
[In Search of an Understandable Consensus Algorithm (Extended Version)](https://raft.github.io/raft.pdf)

A consensus protocol with reference implementation. Covers all aspects of a practical replicated state machine (like latencies, replacing nodes in the system, checkpoints etc).

## ZooKeeper
[ZooKeeper: Wait-free coordination for Internet-scale systems](https://www.usenix.org/legacy/event/atc10/tech/full_papers/Hunt.pdf)

ZooKeeper is an implementation of a replicated state machine that uses a protocol similar to RAFT (called ZAB - zookeeper atomic broadcast).
The interesting idea from this paper is the zookeeper API (and corresponding client implementation). The API is designed to extract out the
replicated state machine aspects of many applications outside so that developers of the applications can focus on the core. For example, an app providing replicated/sharded/consistent store can use zookeeper to store the replica locations, replication status, leader for replica etc. Similarly in hot stand by application, the application can depend on distributed locking functionality provided by zookeeper. 

Another cluster application can use zookeeper to distribute work among worker nodes. The basic idea is, anywhere we need an agreement among a set of nodes about some metadata, that can be kept in zookeeper. Remaining things can be built on top of it.

## Spanner
[Spanner: Google's Globally-Distributed Database](https://research.google/pubs/pub39966/)

Spanner solves many hard problems in one product -
1. Strict serializability - many single node databases don't have serializability as default isolation - Strict serializability is a stronger version of serializability which also has linearizability.
2. Distributed transactions - Most distributed KV stores provide single row atomicity, fewer provide single row transactions.
3. Consistent replication - This is usually only done for metadata stores (like etcd, zookeeper etc)

Spanner uses
1. TrueTime (GPS and atomic clocks in the network) to provide external consistency (strict serializability).
2. Paxos to replicate shards.
3. 2 phase commit to provide cross shard transactions.
4. Paxos for the coordinator to avoid 2PC blocking due to coordinator crashes.
5. Multi Version concurrency control with an integrated scheduler so that read transactions are never blocked by update transactions.

Apart from this spanner also provides interesting features like atomic schema updates, auto balancing of shards across nodes etc.

## Design of a Timeseries database
[Writing a Time Series Database from Scratch](https://fabxc.org/tsdb/)
This post explains a redesign of prometheus persistence. The key idea is to split the data along time dimension called chunks. In a time series store, only the most recent chunk is mutable and all older chunks are immutable. Each chunk is persisted as it's own "db" partition with it's own index. The disk based data structures used to store the chunks and the corresponding indices is amazing.

[Index design](https://github.com/prometheus-junkyard/tsdb/blob/master/docs/format/index.md)

For the last chunk, WAL is used to provide durability to the in memory data structures of the last mutable chunk.

## Lightning Memory Mapped Database
[MDB: A Memory-Mapped Database and Backend for OpenLDAP](http://www.lmdb.tech/media/20111010LDAPCon%20MDB.pdf)

A B-tree based embedded database that doesn't need any memory for caching. It avoids multiple caches in traditional databases (shared buffers, page cache, fs cache etc) and only uses the mmap based files where the caching is managed by OS. This was only possible when the processor address space got bigger than the size of the databases. This provides serializable transactions (ACID compatible) that doesn't need data level locks. It maintains 2 versions of the db, has a single writer which executes update transactions and read only transactions don't block updates and updates don't block read only transactions. Extremely fast read performance.

## Recovery for atomicity

[Centralized Recovery](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/05/chapter6.pdf)

If atomicity is analogous to exactly once, recovery is the analogous of uniqueness. If we implement exactly once by applying at least once and uniqueness criteria, we will allow more than once but ensure that and duplicates are dropped. Recovery is the equivalent of dropping the duplicates to ensure atomicity of the transaction. A transaction containing multiple steps has exactly one commit point in time (splitting all or none). Before commit point, transaction is considered to be not present at all (no effects should be visible), after commit point, all changes are visible. Recovery is the component that ensures this.

## Apache Druid - Scalable time series analytics system

[Druid: A Real-time Analytical Data Store](http://static.druid.io/docs/druid.pdf)

Druid continuously ingests event stream and is able to provide real time analytics. The interesting ideas related to storage
1. The database only allows appends, it's chunked into time based segments. All segments except the last one are immutable.
2. The segment (a collection of all events typically 4-5 million in a given time window) is a unit of query/storage.
3. Each segment is a column based store and has its index built in. Hence each segment can be queried independently without looking at another data source.
4. Query caching is done by storing results for (query, segment[immutable]). This query caching at segment level can be used by multiple queries that different in time range but remaining query is same.
5. The data model is, there's a timestamp column, then there are set of dimension columns and there are metric columns. The timestamp column defines the rows that belong to same segment, dimension columns allow filtering and group by and metric columns are aggregated for given dimensions [either during rollup or during queries]
6. This allows optimised indexing for dimension columns (like bitmap indexing that allows faster filtering).
7. Data ingestion uses Kafka as a WAL. Latest data is handled in a real time segment that is mutable and continuously updated from Kafka. Only when the segment is persisted does druid update the offset in kafka. With this, historical immutable segments are made durable by storing multiple copies and real time durability is provided by kafka.
8. Similarity with prometheus storage design
   * Divided into time based segments of fixed time size.
   * Each segment has it's own index
   * Only most recent segment is mutable, there's a WAL for durability of recent events which are otherwise only kept in memory.

## Memcached - Scaling Memcache at Facebook

[Scaling Memcache at Facebook](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf)

How facebook scales the database systems by putting memcache systems in front of them. This reduces read load on the database servers
but introduces new problems as the clients scale. This paper discusses interesting data access patterns that emerge with caching, the
problems they cause and how was it addressed.

1. Incast Congestion - With 100s of memcache servers with cached data distributed between them, all clients end up connecting to all the servers. A naive implementation would cause congestion and network errors. This is mitigated by smart clients that batch requests to reduce
the number of individual connections from one client to a memcache server. There's a tradeoff between how long a client should wait to
get a big enough batch size and the latency introduced by this scheme. Also this client reduces the latency by handling compression, serialization, error handling etc

2. Stale values - With multiple concurrent reads and writes to value, it's possible to write a stale value in the cache (since requests can be handled out of order due to different reasons). To prevent this, a key is leased before making an update. When the key is deleted meanwhile, the lease token is invalidated and the write would fail preventing stale values in the cache.

3. Thundering herd - If a key was deleted because of an update on the database, multiple cache clients will find it missing in the cache and
attempt to read from the database and update the cache. This lasts for the duration of the time a read db and update cache takes. This can also be avoided by leases. If there's a lease on the key, only the client with the lease will attempt to read from db and update cache, others will wait.

4. Stale sets - While a lease is taken on the key, the cache also maintains previous previously deleted values for some time. Some applications are ok with slightly out of date and can avoid going back to the db in such cases.

5. Cache pools - This introduces optimizing the cache for data types with different access patterns. 
   * There can be data which is used frequently and cache miss is inexpensive. It's ok for these to be removed from cache. - A small pool is allocated for this type of data.
   * There can be data which is used infrequently but cache miss is expensive. If this is put with items with data type above, they'll always be flushed from cache and cause expensive reads. - A large pool is allocated for this type of data.
   * There can be data which are accessed a lot compared to others. These will take up the entire server capacity for reads. To handle this, the data must be replicated. - Replicated pools are created for data
     ** that are accessed in a big batch (increases network throughput for the node)
     ** entire dataset fits in one or 2 memcache servers (otherwise splitting the dataset across many nodes reduces this issue). Splitting the smaller dataset would still require the same request rate if the keys typically accessed in a batch get split.
     ** request rate is much higher than a single server can manage (hotspots).
   * The data can be high churn or low churn. High churn data can evict low churn data even when high churn data isn't being accessed as much. - These are put in different pools to avoid this interference. Also, high churn pool can be sized appropriately based on the cost of fetching the data on cache miss.

6. Gutters - These are a small set of idle servers to be used as secondary caches if there are server failures while accessing the cache. This is different from rehashing the keys based on available servers. For example, if there's a single key access causing servers to fail under load, rehashing the key will cause failure in other servers (cascading failures). In case of gutters, these are idle servers that take up load of few failed servers and reduce the load on the backend db.

## Approximate Query Processing
[Synopses for Massive Data: Samples, Histograms, Wavelets, Sketches](https://dsf.berkeley.edu/cs286/papers/synopses-fntdb2012.pdf)

These methods proceed by computing a lossy, compact synopsis of the data, and then executing the query of interest against the synopsis rather than the entire dataset. This is also suitable for streaming analysis where while streaming the synopses are built and the analysis are done on the synopses.

Synopsis - A lossy, compressed representation of the dataset.

Family of synopses

### Random Samples
Either tuples can be sampled or for persistent data, blocks can be sampled.

### Histogram
Data is bucketed and the summary statistics are used to approximately reconstruct data in the buckets.
The summary is either equi width where the domain is split into equal intervals and counts in each bucket is calcuated or equi depth (quantiles) where the domain is divided at intervals where the counts are equal.

v-optimal histograms: Find histograms using dynamic programming, that minimize error subject to upper bound on histogram size. For dimensions more than 1, the cost of histogram increases exponentially (with the number of histograms).

Sample query: "compute total sales between July and September for adults from age 25 through 40" that can be answered approximately based on histograms.

### Wavelet
Transform the data in wavelet domain by computing coefficients of wavelet basis functions. The coeffs are then thresholded to reduce the size of dataset that needs to be stored.

Similar to histograms, where histograms store statistics from original data, wavelets store statistics from transformed domain (frequency domain for example). Widely discussed wavelet transform - HWT (Haar wavelet transform)

Mostly used for range sums (?). Select/Project/Join on relational summaries can also be performed in this do

### Sketches
Linear sketches for example consider the data to be a numeric vector which is then multiplied by a fixed matrix. These are highly parallelizable. They can accomodate stream of transactions in which records are both added and deleted.




# TBR
* [LLAMA: A Cache/Storage Subsystem for Modern Hardware](http://www.vldb.org/pvldb/vol6/p877-levandoski.pdf)
* [Making Snapshot Isolation Serializable](http://www.cse.iitb.ac.in/infolab/Data/Courses/CS632/Papers/p492-fekete.pdf)
* [Constant Time Recovery in Azure SQL Database](https://15721.courses.cs.cmu.edu/spring2020/papers/10-recovery/p2143-antonopoulos.pdf)
* [ARIES protocol](https://web.stanford.edu/class/cs345d-01/rl/aries.pdf)
* [Minuet: A Scalable Distributed Multiversion B-Tree](http://vldb.org/pvldb/vol5/p884_benjaminsowell_vldb2012.pdf)


