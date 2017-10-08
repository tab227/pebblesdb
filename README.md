## Pebblesdb
PebblesDB is a write-optimized key-value store which is built using the novel FLSM (Fragmented Log-Structured Merge Tree) data structure. FLSM is a modification of standard LSM data structure which aims at achieving higher write throughput and lower write amplification without compromising on read throughput. 

PebblesDB is built by modifying [HyperLevelDB] (https://github.com/rescrv/HyperLevelDB) which, in turn, is built on top of [LevelDB] (https://github.com/google/leveldb). PebblesDB is API compatible with HyperLevelDB and LevelDB. Thus, PebblesDB is a *drop-in* replacement for LevelDB and HyperLevelDB. The full paper on PebblesDB can be found [here](http://www.cs.utexas.edu/~vijay/papers/sosp17-pebblesdb.pdf "PebblesDB SOSP'17").

If you are using LevelDB in your deployment, do consider trying out PebblesDB! PebblesDB can also be used to replace RocksDB as long as the RocksDB-specific functionality like column families are not used. 

Please cite the following paper if you use PebblesDB: [PebblesDB: Building Key-Value Stores using Fragmented Log-Structured Merge Trees](http://www.cs.utexas.edu/~vijay/papers/sosp17-pebblesdb.pdf). Pandian Raju, Rohan Kadekodi, Vijay Chidambaram, Ittai Abraham. [SOSP 17](https://www.sigops.org/sosp/sosp17/). [Bibtex](http://www.cs.utexas.edu/~vijay/bibtex/sosp17-pebblesdb.bib)
___

### Installation
`$ cd pebblesdb/`  
`$ autoreconf -i`  
`$ ./configure`  
`$ make`  
`# make install`  
`# ldconfig`  

___

### Running microbenchmark
1. `cd pebblesdb/`  
2. `make db_bench`  
3. `./db_bench --benchmarks=<list-of-benchmarks> --num=<number-of-keys> --value_size=<size-of-value-in-bytes> --reads=<number-of-reads> --db=<database-directory-path>`  
A complete set of parameters can be found in `db/db_bench.cc`  

Sample usage:  
`./db_bench --benchmarks=fillrandom,readrandom --num=1000000 --value_size=1024 --reads=500000 --db=/tmp/pebblesdbtest-1000`

___

### Optimizations in PebblesDB
PebblesDB uses FLSM data structure to logically arrange the sstables on disk. FLSM helps in achieving high write throughput by reducing the write amplification. But in FLSM, each guard can contain multiple overlapping sstables. Hence a read or seek over the database requires examining one guard (multiple sstables) per level, thereby increasing the read/seek latency. PebblesDB employs some optimizations to tackle these challenges as follows:  

#### Read optimization
* PebblesDB makes use of sstable level bloom filter instead of block level bloom filter used in HyperLevelDB or LevelDB. With this optimization, even though a guard can contain multiple sstables, PebblesDB effectively reads only one sstable from disk per level.  
* By default, this optimization is turned on, but this can be disabled by commenting the macro `#define FILE_LEVEL_FILTER` in `db/version_set.h`. Remember to do `make db_bench` after making a change.  

#### Seek optimization
Sstable level bloom filter can't be used to reduce the disk read for `seek` operation since `seek` has to examine all files within a guard even if a file doesn't contain the key. To tackle this challenge, PebblesDB does two optimizations:
1. **Parallel seeks:** PebblesDB employs multiple threads to do `seek()` operation on multiple files within a guard. Note that this optimization might only be helpful when the size of the data set is much larger than the RAM size because otherwise the overhead of thread synchronization conceals the benefits obtained by using multiple threads.  
By default, this optimization is disabled. This can be enabled by uncommenting `#define SEEK_PARALLEL` in `db/version_set.h`.  

2. **Forced compaction:** When the workload is seek-heavy, PebblesDB
can be configured to do a seek-based forced compaction which aims at
reducing the number of files within a guard. This can lead to an
increase in write IO, but this is a trade-off between write IO and
seek throughput.  By default, this optimization is enabled. This can
be disabled by uncommenting `#define DISABLE_SEEK_BASED_COMPACTION` in
`db/version_set.h`.

___

### Tuning PebblesDB
* The amount of overhead PebblesDB has for read/seek workloads as well as the amount of gain it has for write workloads depends on a single parameter: `kMaxFilesPerGuardSentinel`, which determines the maximum number of sstables that can be present within a single guard.
* This parameter can be set in `db/dbformat.h` (default value: 2). Setting this parameter high will favor write throughput while setting it lower will favor read/seek throughputs.
