- Stands for Log Structured Merge Tree
- A data structure that is commonly used in modern distributed and NoSQL databases like Cassandra, RocksDB, LevelDB
- Designed to perform write operations with very high rates

It achieves this optimization by separating the write and read operations into separate components and leveraging a disk-based storage system.

They do this by removing the need to perform dispersed, update-in-place operations.

LSM tree primarily use three concepts to optimize reads and write.

The key characteristics and components of LSM Tree include:
- Write-Optimized Architecture: LSM structures prioritize write operations, enabling quick writes to an in-memory buffer.
- MemTable: It is a data structure used to store recently modified data in memory. The primary purpose of the MemTable is to provide fast write performance by leveraging the speed of main memory. This helps in providing efficient and low-latency write operations.
- SSTable: Sorted String Tables are immutable on-disk data structure that store key-value pairs sorted by their keys, making them an efficient data format for range queries. They are append-only data structures, meaning that once data is written to an SSTable, it cannot be modified. Any updates or deletions result in a new SSTable being created.
- Bloom Filters: Bloom filter is a probabilistic data structure that quickly tests whether an element is a member of a set. It helps determine if a particular key exists in the SSTables, reducing the number of disk reads when searching for data. LSM structures often employ Bloom filters to quickly determine if a key exists in the SSTables, reducing disk reads.


# 1. Memtable

#### System #1

In this system, write calls are directly coming to the database.

#### System #2

In this system, there is an in-memory kind of structure which accepts all incoming write calls and condenses them into a single write call to the database.


![[lsm_tree.jpg]]


Compare the pros & cons of both systems

| Parameters | System #1 | System #2 |
|---|---|---|
|Network Bandwidth|High|Low|
|Resource Consumption|High|Low|
|Write Throughput|Low|High|
|Additional Memory|No|Yes|

This additional memory is nothing but a Memtable.
- In-memory data structure
	- Red Black Tree or Balanced BST
- Stores data in sorted fashion
- Acts as a write-back cache
- After reaching a certain size, gets flushed as a SSTable to DB


When updates arrive they are added to an in-memory buffer, which is usually held as a tree (Red-Black etc) to preserve key-ordering.

This ‘memtable’ is replicated on disk as a write-ahead-log in most implementations, simply for recovery purposes.

When the memtable fills, the sorted data is flushed to a new file on disk.This process repeats as more and more writes come in. Importantly the system is only doing sequential IO as files are not edited.

# 2. Sorted String Table (SSTable)

- Composed of many segments which has keys in sorted order
- Segment files are immutable; they are never updated. New updates go into new files.

> [!question]
> What is simplest and fastest way to write to a database?
	Just Append !!

> [!question]
> What will be the complexity in that case?
	O(N)


> [!question]
> What if the log is sorted?
    O(logN)


# 3. Block Index

A block-index is created at the end of each segment file. This provides a lookup which gets you ‘close’ to your target key. You scan from there as the data is sorted.

- Works better than straight binary search

Even with per-file indexes read operations will get slow as the number of files increases.


# 3. Compaction

- Worst case read time from disk: NLogN where N is the number of SSTables
- As number of SSTables in db increases, the worst case read time also increases
- As we keep flushing Memtable as SSTable to disk, the same key may be present in the multiple SSTables
- So a compactor which usually runs in the background, merges SSTables by removing the redundant & deleted keys and creating a compacted / merged SSTable

When a read operation is requested the system first checks the in memory buffer (memtable). If the key is not found the various files will be inspected one by one, in reverse chronological order, until the key is found. Each file is held sorted so it is navigable. However reads will become slower and slower as the number of files increases, as each one needs to be inspected. This is a problem.

Periodically the system performs a _compaction._ The process is a bit like generational garbage collection.

As more data comes into the system, more immutable and ordered files are created in the disk. As these files are not update, duplicate entries are created to supersede previous records.

Compaction selects these multiple files and merges them together, removing any duplicated updates or deletions.

The process of merging the files is quite efficient because the files are sorted.

This improves the read performance.

Even with compaction reads will still need to visit many files.

# 4. Bloom Filters

Lets say compactor will not run for 30 min, then read latency per key will increase as the number of SSTables will increase. To address this, we fit bloom filters with each SSTable.

- Probabilistic data structure
- Check if the key is present in the SSTable or not with O(1) time complexity
- Improves the read performance for keys to some extent
- False positive match is also possible


Bloom filters are a memory efficient way of working out whether a file contains a key.

# Advantages

- Support key value storage
- High write throughput
- Attractive to store data with high update rates
- Minimized storage overhead
- Ease of recovery and durability

# Disadvantages

- Slow reads
- Compaction process sometime interfere with the performance of ongoing reads and writes




#   
Write Operation

When a write operation occurs in an LSM data structure, the following steps are typically involved:

- Write to MemTable: Initially, the data is written to a MemTable, which resides in memory. The MemTable is a sorted, in-memory data structure that allows for quick writes. It can be implemented using a skip list, a red-black tree, or any other suitable data structure. The write operation appends the new key-value pair to the MemTable.
- Memory Size Threshold: As the MemTable grows, it eventually reaches a size threshold. Once this threshold is crossed, the MemTable is considered full and needs to be flushed to disk as an SSTable.
- SSTable Creation: When the MemTable is flushed, it is persisted to disk as an immutable SSTable. The SSTable is a sequential file that contains the sorted key-value pairs. Each SSTable typically represents a range of keys. This ensures durability and prevents data loss.

# Read Operation

When a read operation occurs in an LSM Tree, the following steps are typically involved

- Search in MemTable: The read operation first searches for the desired key in the MemTable, which resides in memory. If the key is found in the MemTable, the corresponding value is returned, and the operation completes.
- Search in SSTables: If the key is not found in the MemTable, the search continues in the on-disk SSTables. The LSM data structure employs a multi-level organisation of SSTables, often referred to as “levels.”
- Levelling: The levels are organised based on their size or creation time, with the highest level containing the most recent data. The lower levels contain compacted SSTables, where redundant or overlapping key-value pairs are eliminated. This organisation helps improve read performance by reducing the number of SSTables to search through.
- Bloom Filters: Before searching the SSTables, LSM data structures often use Bloom filters. It helps determine if a particular key exists in the SSTables without needing to read each SSTable individually. This reduces the number of disk reads during the search process.
- Search in SSTables: Starting from the highest level, the LSM data structure searches the SSTables in each level, from the most recent to the oldest. It sequentially reads the SSTables and performs key lookups until it either finds the desired key or reaches the end of the SSTables. If the key is found, the corresponding value is returned.

# Merge Process

To optimise the read performance and manage disk space, the LSM tree periodically performs a merge process that compacts and merges SSTables. This process involves the following steps:

- Merge Selection: The merge process selects a set of SSTables to merge based on criteria such as their size, age, or overlapping key ranges.
- Duplicate Elimination: During the merge process, duplicate keys are eliminated. When multiple SSTables contain the same key, only the most recent value is retained.
- Sorting: The merged SSTable resulting from the merge process is sorted by key to ensure efficient read operations.
- Level Assignment: After the merge, the merged SSTable is assigned to an appropriate level based on its size or creation time. This helps maintain the multi-level organisation of the SSTables.

# Compaction

Compaction is the process of compacting and eliminating redundant or obsolete data in the LSM Tree. It occurs during the merge process and helps manage disk space efficiently. Compaction involves the following steps:

- Overlapping Key Ranges: During the merge process, SSTables with overlapping key ranges are identified. These overlapping ranges are resolved to eliminate redundant data.
- Tombstones: In some LSM Tree, tombstones are used to mark keys that have been deleted. During compaction, SSTables containing tombstones can be safely discarded, freeing up disk space.

![](https://miro.medium.com/v2/resize:fit:1280/1*gQUWjSYBKEd584v8RRPDeQ.png)

> By performing periodic merges and compacting SSTables, LSM data structures maintain efficient read performance by reducing the number of SSTables to search through and eliminating redundant data.


