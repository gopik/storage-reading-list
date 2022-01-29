
# Dimensions

* Transactional vs non transactional (multi key atomic read modify write)
  * BTree - LMDB, BoltDB, libmdbx
  * LSMTree - Badger DB, RocksDB?
  * LSMTree non transactional - LevelDB
* WAL vs no WAL
  * BTree - The ones based on mmap and single writer thread usually don't have a WAL. This makes them inefficient for a write heavy load. This performance can be improved by batching. Can we use WAL and use it for batching? Bolt doc suggests this.
* Read amplification
* Write amplification
* Space amplification
* Random read throughput
* Random write througput
* Range scan throughput
* Sequential write througput
* LSM or Btree

# LSM compaction strategies
Survey: https://smalldatum.blogspot.com/2018/08/name-that-compaction-algorithm.html

## Leveled compaction (LevelDB)
Ref: 