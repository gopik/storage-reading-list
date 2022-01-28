# Logs management

Let's say we maintain a service which generates useful logs to help us understand how it functions. Going through the logs, we can find out what happened when something went wrong.
But managing this logs is not easy in a production system with 100s of services with many replicas that are handling 1000s of requests every minute. The amount of logs generated is huge and will fill up disk space quickly.
To avoid the disks getting filled up, log rotation is set which makes it easy to keep creating logs in new files and discard old logs regularly before disk runs out of space. If the logs need to be deleted quickly to keep up, we will lose information if we needed those logs to debug something. So a standard approach is to ship logs out to a cheaper storage system instead of deleting them. If needed, the logs can be fetched from this system.

# Search
Once we have the logs available, it is now possible to find the information needed. But this process is extremely slow. Without indexed logs, a large amount of time is spent in fetching the logs, grepping through them (potentially multiple times), writing scripts for further analysis etc. What we really need is a powerful analytics system which can answer structured queries from potentially unstructured data.

# Logs storage

Ideally we would index these either in a database or a search engine. But logs search use case is different from either of these solutions. Database is
used for transactional operations where we make updates and make point queries. Writes in these dbs are expensive since each write will update multiple indices which will be used to retrieve data. Other dbs are optimized for writes where data is ingested in bulk and indexed in batches which makes writes cheaper.

For example full text search engines are optimized for data that is written once but queried frequently. Analytics databases are optimized for full scans, they don't need full indexing. But they are stored in columnar format which makes them extremely space efficient. Both these need that data be ingested in batches for maximum efficiency.

One way logs usage is different from analytics db or a search engine is, read to write ratio is extremely small. Logs are written once, but used rarely. Probably while debugging and even in that case a tiny fraction of data is really ever looked at. But in case of analytics db, although fewer clients use them, there are large queries which routinely scan the whole data. Similarly for search engines typically have a large number of clients performing searches which read a large number of documents. In both cases read to write ratio is much higher than in debug logs scenario.

# Costs
Given the skewed read/write ratio, we need to ensure that cost of the writes are justified by the amount of reads done. Hence this solution should be much cheaper compared to analytics db and full text indices. Ideally we shouldn't incur any cost in "serving" for writing the data since there's no update involved. The "serving" cost should only be incurred when doing the queries. If there are no queries done, we shouldn't need a service. The workflow would involve creating index directly as the log writes are done.

Unfortunately this is not possible, indexing when done as a batch operation can be done only after enough data has been collected. If done as an interactive operation, requires a service that mutates the index based on new writes.

# Understanding log document structure
We'll consider each log statement as a new document being created. These documents are then appended to some store. The key property of this document is the log message. But there are many auxiliary fields that are very useful, like the timestamp, current service, file name, line number, certain environment variables etc. Typically log store is an ever increasing document collection. So these additional fields help narrow down the documents where we should search. We'll assume that there are 2 required fields of each document, timestamp and a "message" field that contains arbitrary text. There can be any number of other fields which will contain primitive data (like a string, number etc).

# Indexing
Log store access has bias towards recent documents. Hence we can limit the storage to some recent past. Even with this log storage keeps increasing since the scale of the service might increase, new services come in etc. To control this, this store is typically sharded by various parameters and each shard is indexed independently. Say, if we allow a lag of X minutes, data is appended to a temporary location and every X minutes (or if enough log data is generated sooner), this data is converted to an index and written to log store. This keeps the index size limited although there may be many of them.

We'll partition the indices by timestamp. No index spanning data for more than 1 hour. But for a given hour, there may be many indices.

Apart from this, there are other attributes we should use to partition the documents. For example we don't want different team's search results being polluted with each other's document. There might be separation due to security domains etc. All these are independent collections. But within each collection the data is partitioned by timestamp.

# Querying
The querying subsystem needs to be pointed to a specific collection based on certain attributes. Once it's pointed, the query system will look at correct indices which are partitioned by timestamp. If we need data across different collections, they can be handled as individual queries to each collection and then the results can be merged by the client. For the rest of the post, we'll assume that document belonging to a specific collection and we will only consider that a single collection is present. Other collections can be considered as copies of the same system.

Ideally the indexing system should require a constant memory for searches. This means we can't do any accumulate type operations that can't be done in constant memory. For example, we can't sort a large collection as part of search. Also, this memory requirement shouldn't grow over time. This means memory requirement can't be proportional to the data size. Memory requirement should be proportional to the concurrency required and not to the dataset size being searched. This implies, we will need to traverse on disk structures while searching which would be slow. But if the structures are optimized, the on disk access should be proportional to the result size.

Hence the resource requirements for querying are -

1. RAM (and cpu) - proportional to query concurrency
2. Storage - proportional to data size
3. Query time - proportional to the result size

Note: Query time constraint would limit what kind of queries are possible in such system. Also, it's not possible to achieve query time proportional to the result size for arbitrary data size. The idea is, the growth in query time should increase slowly as compared to data increase. For example, 10% increase in query time when data size doubles is still useful when compared to 100% increase. Like in a B-Tree based index, query time is proportional to the height of the tree. Doubling the size increases the height by at most 1 level.

# Index formats
We need index formats which are disk based to keep RAM requirements constant. The index should be hierarchical so that within few IOs, we find the results. We will explore 2 index formats for this purpose - Lucene and Parquet.

## Lucene
Lucene is a full text index format which is an advanced data structure that has been heavily optimized over time. It stores the data in a format that is both compressed and fast to search. The record (or the log structure) is split into individual fields and for each field, every value is mapped to a list of documents it's contained in. It supports numeric fields search like find documents where this field is less than certain value, support text fields where each word in the text field is mapped to a list of documents. This is all that we need for selective queries, where we want to find all log lines that occurred during certain time range and contain certain terms. It supports aggregate queries as well, like top 10 over specific fields or count or sum of certain fields etc. It also supports faceting results by all values of certain field, like count the number of log lines containing specific field by hour. Lucene querying is very powerful and its index format is designed to support these queries.


Lucene index is optimized for querying. It separates the core index and raw documents, hence for resolving the documents for a given query, we don't need to load lot of data. The index allows efficient retrieval for all kinds of queries where parquet would need full scans for certain queries. 

The key difference is, although parquet stores data in columns, it's retrieval is still like a row. This implies that all columns are iterated together to output a single row. To achieve this, the smallest unit of retrieval is a subset of columns from a row group. 

But in Lucene, all columns (or fields) are stored independently. They can be retrieved independent of each other. To generate a row from these columns, Lucene needs an id for the row (called docid). Once all columns have been retrieved based on whatever query needs, the rows are union or intersection of the docids. Then a separate step loads the relevant docs from a different store.

The on disk serialization of the index is [optimized](https://github.com/quickwit-inc/tantivy/blob/main/ARCHITECTURE.md#general-information-about-these-data-structures) such that reading doesn't need to be deserialized in another format for search. This implies low memory usage (only the memory required for file buffer cache)


## Parquet
Parquet takes a different approach where it is not explicitly designed to power any specific type of system. It's a generic storage format to store values by column. It's strength lies in support for complicated document schemas and still storing them extremely efficiently. Columnar storage is very efficient for storage (can compress much better compared to row oriented storage) and is very efficient in full scans of the data when only specific columns need to be retrieved. But it's tricky to store nested object structures in columnar format. Parquet solves this problem where it's able to store objects have fields that are lists of other objects. It organizes the data into row groups, and each row group, columns are stored together. For each such group, there's metadata that describes the data blob like column statistics, size etc. It allows storing custom metadata for these groups. This enables other systems to optimize their data access. For example, if data is sorted by columns, the scanning code can skip entire row groups and only read the specific row groups that has data range that intersect with the range of interest. Similarly, if a table with large number of columns doesn't impact queries if only a subset is being retrieved. Apart from this, parquet is not optimized for any specific retrieval pattern.

Parquet has a compact representation for the data statistics which is independent of the data size. It stores per row group statistics for each column. For structured data, like strings, numbers, timestamps etc, this allows for easy filtering if the columns are sorted properly. For example, consider set of columns like timestamp, a category of operation, a keyword, if columns are sorted in this order, queries that specify filters in all these 3 columns will be able to quickly narrow down a row group to be read. For example, if only a keyword column is specified, it will probably require a scan of all row groups.

This means, it's very easy to support a system that manages a large number of indices since almost no data needs to be loaded to allow query over an index. But the indices are effective for only a narrow set of queries, the column sort order must be chosen carefully that optimizes a large fraction of queries. Others will need full scan which is still better than row oriented storage because of efficient storage and retrieval.

Also, column statistics don't help with full text search. For example, we can't match rows where a multi line field contains a specific word. This is limited to primitive types only.

## Index comparison
### Size
[SSD, 4 core, 32G]

* **Raw data size** - 6M log lines, 4.5 GB (123 MB gzipped)
* **Parquet size** - 52 MB, Took 1 minute (naive single threaded)
* **Lucene (tantivy index size)** - 1.5 GB, Took 3 minutes (naive impl on single host)

Here's a brief comparison between parquet and lucene index creation. Parquet is fast compared to lucene since it doesn't have to manage complex data structures, their serialization etc. Also, it's very small (even smaller than gzipped input!) due to more efficient compression when data is stored in columnar fashion. Lucene has still very good indexing speed considering the amount of work it has to do. Of 1.5GB, the actual raw documents storage takes 1 GB. 500 MB is for index structures that allow quick lookup for arbitrary queries. Raw documents are stored compressed, but they are designed for random lookup of specific document, hence the compression is done for smaller blocks which is inefficient.

Both parquet and lucene store data in columnar format. One key difference between lucene and parquet formats are, how they merge the columns of a row together. In case of lucene, each row is identified by a unique id (docid). Each column has row ids sorted by the column value. Hence, search first collects docids for each filtered column and then finds the intersection. In case of parquet, there's no unique rowid. All columns although stored independently, are stored in the row order. Hence it's important to identify the key columns that we want to be sorted  for the given row order. DocValues in lucene and fastvalues in tantivy allow columnar storage of data (independent of the index).

## Index organization

Since logs are continuously generated append only data, the indices are best organized by sharding them by time such that immutable indices are generated in batch. This also means that we don't need to maintain any service for index generation, some batch job runs regularly and creates the indices. Given the batch nature of index generation, this introduces a lag in the data being generated and this data being available in the search results. This batch job can produce the indices in an object store (like aws s3 or gcp cloud storage) which reduces storage related maintenance of the index pipeline.

This introduces a challenge for querying service. The querying service is completely stateless and has to figure out which indices are available and how to select the indices based on the query. For example, if the indices are partitioned by hour of the day, and a query searches for a range of time which overlaps with multiple hours, the querying system will first resolve the correct indices and use the remaining part of the query against appropriate indices and then merge the results.

# Query system design
For query processing, lucene is a clear winner since the index is designed to support powerful queries and it comes with built in query parser. For parquet, we need an additional query processing layer that would convert queries into optimized access into parquet data. Parquet in turn has very efficient storage for raw documents. Hence it would make sense to use lucene for indexing and use parquet as a raw document store. In either case we need a way for query node to efficiently access the index data.

The querying system doesn't have access to the indices in it's local file system. To avoid copying over the large files for each query, it needs to load relevant sections from the object store. This will be much slower compared to local file system reads and much slower than loading the entire data in memory. For this design, it's not feasible to load the indices in memory since the number of indices will be large and keep growing over time. This should be solved by building multi layer cache, the slowest layer is the object store. The data that's fetched from object store must be cached in local disk for some time and the actively used part should be in memory. Given this is an immutable data, the cache management is much easier, compared to say in the system where indices could be updated.

<img src="https://raw.githubusercontent.com/gopik/storage-reading-list/main/QueryNode.drawio.svg"/>

This is how indices are managed in the [historical nodes in druid](https://druid.apache.org/docs/0.13.0-incubating/design/historical.html). These nodes manage queries from old data that have been shipped to object stores.

## Scaling
Scaling this is little complex. We want to utilize the local disk caching, so we would like to route queries hitting same indices to the same node. But at the same time would like to avoid hot spotting and if an index is hit too many times, we want multiple nodes to serve queries for this index.

In the simplest case, we can assume that all indices are equally useful. In this case, if we have say N nodes, an index is mapped to some 3 nodes from this set. Queries for this index is sent randomly to one of the these 3 nodes. We can also, assume that older data is less useful than new data, hence recent index is mapped to multiple nodes but historical index is mapped to only one of these. A good design would distribute the load dynamically based on the load on the node while utilizing the local cache available at a given node as much as possible.

The index to node mapping is done via [consistent hashing](https://en.wikipedia.org/wiki/Consistent_hashing) techniques. This reduces the index ownership changes when there are node changes (addition/removal of nodes). For more details - https://medium.com/omarelgabrys-blog/consistent-hashing-beyond-the-basics-525304a12ba

## Local filesystem as a cache
The query system requests specific byte regions from the cache layer. These needs to be retrieved from the object store over network. These objects will be cached in local file system in an LRU cache. This is not effective for very large index files since cache will only use a small part of the file. Here we'll need to segment the files and download only the part that's accessed. For example, a 500 MB file can be segmented into 500 1MB chunks and caching can be implemented at the chunk level based on the byte regions requested. If we convert each byte range request to a set of fixed size requests (say requests that align at 1 MB boundary and 1 MB size), http cache can be used as local file system cache. This can be implemented in one of the ways described below -

<img src="https://raw.githubusercontent.com/gopik/storage-reading-list/main/Caching.svg"/>

In one case, we configure the http client with appropriate caching parameters. With this, the client will first check if http request can be served locally (response exists and cache control headers are valid). Then client either decides to check the server with a head request if the data has changed, if not serves the data locally. The advantage of this approach is, this system respects cache control headers and data is not cached beyond it's validity. It's also possible to cache an incomplete file since the http client will never cached data if it's become stale on the server.

In the next approach, we implement caching outside of http layer. Here the data is fetched from http when there's a cache miss, then the block is cached locally (either using simple filesystem or an embedded database like rocksdb or bboltdb). Between the 2 approaches, the http cache makes more sense given we can use http cache semantics for invalidation. For immutable data that's valid forever, there won't be any difference in the 2 approaches.

# Conclusion
We explored log search as a sub problem of generic text search and considered how this impacts different design choices. We also learnt about different indexing formats (parquet and lucene) and their tradeoffs. Lucene is well suited for querying since it allows accessing documents using any of the indexed field. But with large collection of indices, the query system needs to be designed carefully.

# References
* [Lucene index file format](https://lucene.apache.org/core/9_0_0/core/org/apache/lucene/codecs/lucene90/package-summary.html#package.description)
* [Parque integration with other formats](https://dzone.com/articles/understanding-how-parquet)
* [Full text indexing library - rust](https://github.com/quickwit-inc/tantivy)
* [Understanding parquet nested fields](https://blog.twitter.com/engineering/en_us/a/2013/dremel-made-simple-with-parquet)
* [Full text indexing library - golang](https://github.com/blevesearch/bleve)