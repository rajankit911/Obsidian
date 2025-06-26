A ***write-ahead log*** (WAL for short, also known as a commit log) is an append-only auxiliary disk-resident structure used for crash and transaction recovery.

> A log where all changes are recorded before they are applied to the actual database files. This ensures that, if a crash happens, the database can replay the log to recover.

- When a change is made, it’s first written to the WAL, then applied to the in-memory [[Page Cache| page cache]].
- The WAL keeps a record of all changes for recovery purposes.
- Once the corresponding in-memory page is **flushed** to disk, the WAL record for that change is no longer needed and can be safely discarded.
- This guarantees that, in the event of system crash, the system can recover to a consistent state using the WAL.
- By using WAL and page cache, databases can batch writes and optimize disk usage, but must coordinate them carefully to maintain durability.

> [!note]
> - WAL is an immutable, append-only data structure, so all writes to the log are sequential.
> - Readers can safely access its contents up to the latest write threshold while the writer continues appending data to the log tail.

# Log Semantics

- The WAL consists of log records which has a unique, monotonically increasing ***log sequence number*** (LSN).
- Usually, the LSN is represented by an internal counter or a timestamp.
- Since log records do not necessarily occupy an entire disk block, their contents are cached in the log buffer and are flushed on disk in a force operation. Forces happen as the log buffers fill up, and can be requested by the transaction manager or a page cache. All log records have to be flushed on disk in LSN order.

# Checkpoint Process

> **Checkpoint process** is a background task in most databases that periodically writes all dirty (modified) pages from memory to disk. It is essential for durability.

- It synchronizes the WAL and page cache, ensuring that no data is lost and that the database can always recover to a consistent, committed state—even after a crash.
- Dirty pages must be flushed and their corresponding WAL records kept until the flush is complete, guaranteeing durability.


The checkpoint process plays a critical role in coordinating the Write-Ahead Log (WAL) and page cache during crash recovery through three key mechanisms:

## 1. **Synchronization Point Creation**

Checkpoints create recovery starting points by:

- Flushing all dirty pages (modified in-memory data) existing at checkpoint start time to disk [2](https://www.postgresql.org/docs/9.2/wal-configuration.html)[3](https://postgrespro.com/blog/pgsql/5967965)
    
- Writing a checkpoint record to WAL containing:
    
    - Log Sequence Number (LSN) marking the recovery start position
        
    - List of active transactions and dirty pages [1](https://github.com/Akshat-Jain/database-internals-notes/blob/main/Part%201:%20Storage%20Engines/Chapter%205%20-%20Transaction%20Processing%20and%20Recovery.md)[4](https://www.slideshare.net/slideshow/unit-4-crash-and-recoverypdf/259456411)
        

This ensures **WAL and disk state alignment** - any WAL records before the checkpoint can be safely discarded as their changes are already persisted [2](https://www.postgresql.org/docs/9.2/wal-configuration.html)[5](https://www.sqlite.org/walformat.html).

## 2. **Crash Recovery Workflow**

During recovery, the system:

1. **Locates the latest checkpoint** via `pg_control` file or WAL metadata [2](https://www.postgresql.org/docs/9.2/wal-configuration.html)[5](https://www.sqlite.org/walformat.html)
    
2. **Analyzes WAL from checkpoint onward** to identify:
    
    - Transactions needing redo (committed but not flushed)
        
    - Transactions needing undo (uncommitted at crash time) [1](https://github.com/Akshat-Jain/database-internals-notes/blob/main/Part%201:%20Storage%20Engines/Chapter%205%20-%20Transaction%20Processing%20and%20Recovery.md)[4](https://www.slideshare.net/slideshow/unit-4-crash-and-recoverypdf/259456411)
        
3. **Reapplies WAL records** in chronological order to rebuild database state:
    
    text
    
    `Recovery Process: [Checkpoint LSN] → [Redo WAL Records] → [Undo Uncommitted Transactions]`
    

## 3. **Page Cache-WAL Coordination**

The checkpoint enforces strict ordering:

- **No WAL truncation** until associated dirty pages are flushed [1](https://github.com/Akshat-Jain/database-internals-notes/blob/main/Part%201:%20Storage%20Engines/Chapter%205%20-%20Transaction%20Processing%20and%20Recovery.md)[3](https://postgrespro.com/blog/pgsql/5967965)
    
- **No page eviction** from cache until its WAL records are persisted [3](https://postgrespro.com/blog/pgsql/5967965)[5](https://www.sqlite.org/walformat.html)
    
- **Atomic checkpoint completion** - only marks success after both:
    
    - All checkpoint-start dirty pages are flushed
        
    - `end_checkpoint` WAL record is written [3](https://postgrespro.com/blog/pgsql/5967965)[4](https://www.slideshare.net/slideshow/unit-4-crash-and-recoverypdf/259456411)
        

This coordination ensures **recovery integrity** - the WAL always contains sufficient information to reconstruct database state up to the crash instant, while avoiding redundant disk I/O during normal operation [1](https://github.com/Akshat-Jain/database-internals-notes/blob/main/Part%201:%20Storage%20Engines/Chapter%205%20-%20Transaction%20Processing%20and%20Recovery.md)[2](https://www.postgresql.org/docs/9.2/wal-configuration.html)[5](https://www.sqlite.org/walformat.html).

## Performance Impact

Modern implementations use **fuzzy checkpoints** to minimize disruption:

- Allow new transactions during checkpointing
    
- Track dirty pages using bitmaps rather than global locks
    
- Spread I/O load over checkpoint interval via throttling [2](https://www.postgresql.org/docs/9.2/wal-configuration.html)[3](https://postgrespro.com/blog/pgsql/5967965)
    

For example, PostgreSQL's checkpointer:

sql

`checkpoint_timeout = 5min  -- Default interval checkpoint_completion_target = 0.9 -- Complete 90% of I/O by next checkpoint`

This balances recovery time guarantees with runtime performance [2](https://www.postgresql.org/docs/9.2/wal-configuration.html)[3](https://postgrespro.com/blog/pgsql/5967965).

Add to follow-up

Check sources

Share

Export

Rewrite

---


---

What mechanisms ensure that only committed log records are discarded from the WAL

Mechanisms that ensure only committed log records are discarded from the Write-Ahead Log (WAL) include:

- **Commit Log Records:** A transaction is only considered committed once a special commit record is written to the WAL and flushed to stable storage. This ensures that only transactions with a recorded commit can be considered for log record removal[2](https://web.stanford.edu/class/cs345d-01/rl/aries.pdf)[3](https://www.sqlite.org/wal.html)[5](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-log-architecture-and-management-guide?view=sql-server-ver17).
    
- **Checkpoint Process:** During a checkpoint, the database flushes all dirty pages (pages modified in memory) to disk. Only after the data represented by WAL records has been safely persisted to disk can the corresponding log records be discarded. This guarantees that the effects of committed transactions are durable before their logs are removed[1](https://en.wikipedia.org/wiki/Write-ahead_logging)[3](https://www.sqlite.org/wal.html)[5](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-log-architecture-and-management-guide?view=sql-server-ver17).
    
- **Log Sequence Numbers (LSN):** Each page in the database stores the LSN of the latest log record that modified it. The checkpoint process tracks which pages have been flushed and their LSNs; only WAL records with LSNs less than or equal to the oldest flushed page’s LSN can be safely discarded[2](https://web.stanford.edu/class/cs345d-01/rl/aries.pdf).
    
- **Transaction Status Tracking:** The WAL contains information about the status of each transaction. Only log records for transactions that are marked as committed and whose changes have been flushed to disk are eligible for removal from the WAL[2](https://web.stanford.edu/class/cs345d-01/rl/aries.pdf)[5](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-log-architecture-and-management-guide?view=sql-server-ver17).
    

These mechanisms collectively guarantee that uncommitted or not-yet-durable changes remain in the WAL, preserving the ability to recover the database to a consistent state after a crash

---
In what ways does the checkpoint process maintain consistency between logs and cached pages

The checkpoint process maintains consistency between logs (WAL) and cached pages through several coordinated mechanisms:

- **Write-Ahead Logging Enforcement:** Before any dirty (modified) page in the cache is written (flushed) to disk, the associated log records describing those changes are first written to the WAL and flushed to disk. This ensures that, in the event of a crash, all changes made to the data pages are also present in the log, allowing for proper recovery[5](https://www.sqlskills.com/blogs/paul/how-do-checkpoints-work-and-what-gets-logged/)[6](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-log-architecture-and-management-guide?view=sql-server-ver17).
    
- **Flushing Dirty Pages:** During a checkpoint, the process scans the buffer cache for dirty pages and writes them to disk. This guarantees that the state of the data files on disk is synchronized with the log up to a certain point, reducing the amount of log that must be replayed during recovery[2](https://www.sqlshack.com/sql-server-checkpoint-lazy-writer-eager-writer-and-dirty-pages-in-sql-server/)[6](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-log-architecture-and-management-guide?view=sql-server-ver17).
    
- **Recording Checkpoint Information:** The checkpoint process writes a special record to the WAL marking the start (and end) of the checkpoint. It also records the log sequence number (LSN) of the checkpoint in the database metadata (such as the boot page), which serves as a recovery marker[2](https://www.sqlshack.com/sql-server-checkpoint-lazy-writer-eager-writer-and-dirty-pages-in-sql-server/)[5](https://www.sqlskills.com/blogs/paul/how-do-checkpoints-work-and-what-gets-logged/)[6](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-log-architecture-and-management-guide?view=sql-server-ver17).
    
- **Determining Log Pruning:** Only after a checkpoint confirms that all dirty pages up to a certain LSN have been flushed to disk can the WAL records before that LSN be safely discarded. This ensures that no required log records for unflushed changes are lost[1](https://community.neo4j.com/t/checkpointing-and-log-pruning-interactions/351)[3](https://docs.oracle.com/en/database/other-databases/timesten/22.1/operations/checkpoint-operations.html)[6](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-log-architecture-and-management-guide?view=sql-server-ver17).
    
- **Minimizing Recovery Time:** By periodically flushing dirty pages and marking the checkpoint LSN, the checkpoint process reduces the amount of WAL that needs to be processed during crash recovery, ensuring that the database can be restored to a consistent state quickly[1](https://community.neo4j.com/t/checkpointing-and-log-pruning-interactions/351)[8](https://www.adyen.com/knowledge-hub/checkpoint-tuning-for-rto-improvements).
    

In summary, the checkpoint process ensures that for every change made to cached pages, the corresponding log records are safely persisted before the data pages are written to disk, and that the WAL and data files are kept in sync to guarantee consistency and durability[5](https://www.sqlskills.com/blogs/paul/how-do-checkpoints-work-and-what-gets-logged/)[6](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-log-architecture-and-management-guide?view=sql-server-ver17).


----

In what way does checkpointing determine which transaction logs can be pruned

Checkpointing determines which transaction logs can be pruned by marking a specific point in the transaction log where all previous transactions have been safely persisted to storage. Here’s how this works:

- **Checkpoint Entry in the Log:** When a checkpoint occurs, a special entry is written to the transaction log marking the last transaction included in the checkpoint. This entry indicates that all changes up to this point have been flushed from the page cache (memory) to the persistent store files[1](https://neo4j.com/developer/kb/checkpointing-and-log-pruning-interactions/)[2](https://neo4j.com/docs/operations-manual/current/database-internals/checkpointing/).
    
- **Safe Log Pruning:** After a checkpoint, any transaction logs that only contain transactions up to and including this checkpoint entry are no longer needed for crash recovery, since their changes are already durable on disk. These logs can be safely pruned (deleted)[1](https://neo4j.com/developer/kb/checkpointing-and-log-pruning-interactions/)[2](https://neo4j.com/docs/operations-manual/current/database-internals/checkpointing/).
    
- **Triggering Pruning:** Pruning is always triggered by checkpointing. If no checkpoint has occurred, pruning cannot happen, as there’s no guarantee that the transactions in the logs have been persisted to disk[1](https://neo4j.com/developer/kb/checkpointing-and-log-pruning-interactions/)[2](https://neo4j.com/docs/operations-manual/current/database-internals/checkpointing/).
    
- **Log Rotation Requirement:** Typically, a log file must have been rotated (closed and a new one started) and a checkpoint must have occurred in a more recent log file before the old log can be pruned[2](https://neo4j.com/docs/operations-manual/current/database-internals/checkpointing/)[5](https://github.com/neo4j/neo4j/issues/12807).
    

In summary, checkpointing defines the boundary for safe log removal: only logs containing transactions up to the last checkpoint can be pruned, as their data is guaranteed to be on disk and not just in memory[1](https://neo4j.com/developer/kb/checkpointing-and-log-pruning-interactions/)[2](https://neo4j.com/docs/operations-manual/current/database-internals/checkpointing/).


----

How do different checkpoint policies affect the relationship between logs and pages

Different checkpoint policies directly shape the relationship between logs (WAL) and pages (cached and persisted data) by determining when and how often dirty pages are flushed to disk and which transaction logs can be pruned. Here’s how various policies affect this relationship:

---

## **1. Periodic Checkpointing**

- **Description:** The system triggers a checkpoint at regular time intervals (e.g., every 10 or 15 minutes) if there are pending changes to flush.
    
- **Effect:**
    
    - Ensures that dirty pages are flushed to disk on a predictable schedule.
        
    - Transaction logs (WAL) can be pruned up to the last checkpoint, as all changes before this point are guaranteed to be persisted.
        
    - If checkpoints are infrequent, more logs accumulate and more cached changes remain unflushed, increasing recovery time after a crash[2](https://neo4j.com/developer/kb/checkpointing-and-log-pruning-interactions/)[5](https://neo4j.com/docs/operations-manual/current/database-internals/checkpointing/).
        

---

## **2. Volume-Based (Volumetric) Checkpointing**

- **Description:** A checkpoint is triggered when the transaction logs reach a specified size or volume (e.g., every 256 MiB of logs).
    
- **Effect:**
    
    - Tightly couples log growth with checkpointing: as soon as enough log data accumulates, a checkpoint ensures those changes are flushed.
        
    - Prevents excessive log file buildup and keeps the gap between logs and persisted pages small.
        
    - Log pruning is directly tied to log volume, so the system maintains a closer alignment between logs and disk state[2](https://neo4j.com/developer/kb/checkpointing-and-log-pruning-interactions/)[5](https://neo4j.com/docs/operations-manual/current/database-internals/checkpointing/).
        

---

## **3. Transaction-Based Checkpointing**

- **Description:** A checkpoint occurs after a certain number of transactions have been processed.
    
- **Effect:**
    
    - Flushes dirty pages based on transaction activity, not time or log size.
        
    - Keeps the number of transactions needing replay after a crash bounded.
        
    - Log pruning is aligned with transaction count, ensuring logs and pages are synchronized after each checkpoint[5](https://neo4j.com/docs/operations-manual/current/database-internals/checkpointing/).
        

---

## **4. Continuous or Frequent Checkpointing**

- **Description:** The system checkpoints very frequently or even continuously.
    
- **Effect:**
    
    - Minimizes the window between log entries and persisted pages, reducing recovery time and log retention needs.
        
    - Increases I/O overhead but ensures logs and data are always closely aligned[5](https://neo4j.com/docs/operations-manual/current/database-internals/checkpointing/).
        

---

## **Summary Table**

|Checkpoint Policy|Trigger Basis|Log-Page Relationship Impact|
|---|---|---|
|Periodic|Time|Predictable, but may allow log buildup if infrequent|
|Volume-Based|Log Size|Keeps logs and pages tightly aligned, prevents log sprawl|
|Transaction-Based|Transaction Count|Bounded replay window, aligns logs/pages per transaction burst|
|Continuous/Frequent|Ongoing/Very Often|Minimal lag, fastest recovery, highest I/O cost|

---

## **Key Takeaway**

- **Checkpoint frequency and trigger policy determine how closely the transaction logs (WAL) and persisted pages are aligned.**
    
- **More frequent or volume-sensitive checkpoints keep logs and pages in sync, reduce recovery time, and limit log growth, while infrequent checkpoints increase the gap and recovery burden.**
    
- **Pruning is always dependent on checkpointing: only logs whose changes are safely persisted by a checkpoint can be pruned, regardless of the policy used[2](https://neo4j.com/developer/kb/checkpointing-and-log-pruning-interactions/)[5](https://neo4j.com/docs/operations-manual/current/database-internals/checkpointing/).**