Most databases are built using a two-level memory hierarchy:
- Slower persistent storage (disk)
- Faster main memory (RAM).

To reduce the number of accesses to persistent storage, pages are cached in memory. When the page is requested again by the storage layer, its cached copy is returned.

> The ***page cache*** (also known as the ***buffer cache*** or ***buffer pool***) is a critical component in database systems, responsible for managing how data pages are cached in memory to optimize performance and minimize disk I/O.

All databases have their own ***buffer pools***. A default ***buffer pool*** is created at the time of creation of new database.

# Primary Functions

## Keeps Cached Page Contents in Memory

- The buffer manager maintains a pool of main memory divided into frames, each capable of storing a database page (block) fetched from disk.
- When a page is needed for processing (for example, to satisfy a query), the buffer manager checks if it’s already in memory. If so, it can be accessed directly from the cache, which is much faster than reading from disk.
- This mechanism allows frequently accessed data to remain in memory, reducing the need for slow disk operations and significantly improving database performance.

## Buffers Modifications to On-Disk Pages

- When a page is modified (e.g., data is updated), the changes are first made to the in-memory cached version, not directly to the disk.
- These modified pages are marked as ***dirty***. A **dirty flag** set on the page indicates that its contents are out of sync with the disk and have to be flushed for durability.
- The actual write to disk is deferred and can be performed in batches, allowing multiple changes to be written together. This reduces the number of disk I/O and improves efficiency.
- By buffering writes, the page cache also enables features like ***[[Write Ahead Logging (WAL)|write-ahead logging]]*** and ***[[Write Ahead Logging (WAL)#Checkpoint Process|checkpointing]]***, which are essential for durability and crash recovery.

## Paging in Requested Pages When Not Present and Space is Available

- If a requested page is not in memory and there is available space in the buffer pool, the page cache loads (***pages in***) the required page from disk into a free frame in memory.
- The cached version is then returned to the requester (such as a query processor or transaction engine).
- This process ensures that only the necessary pages are loaded into memory, optimizing the use of available RAM.

## Returning Cached Version if Already Present

- If the requested page is already present in the cache, the buffer manager simply returns the in-memory version, avoiding a disk read entirely.
- This is called a ***cache hit*** and is the ideal scenario for performance, as memory access is orders of magnitude faster than disk access.
- The ***buffer cache hit ratio*** is a key metric, measuring how often requested pages are found in the cache. Higher ratios mean better performance.

## Evicting and Flushing Pages When Space is Insufficient

- If the buffer pool is full and a new page needs to be loaded, the cache must evict (remove) an existing page to make space.
- If the page to be evicted has been modified (is dirty), its contents must be written (flushed) back to disk before it is removed from the cache to ensure no data is lost.
- This process maintains consistency between the in-memory cache and the on-disk database.

> [!note]
> Since triggering a flush on every eviction might be bad for performance, some databases use a ==separate background process== that cycles through the dirty pages that are likely to be evicted, updating their disk versions. For example, PostgreSQL has a ***background flush writer*** that does just that.

