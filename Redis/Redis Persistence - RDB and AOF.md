---
tags:
  - redis
  - database-internals
  - architecture
  - performance
date_created: 2026-06-09
last_modified: 2026-06-09
---

# Redis Persistence: RDB vs AOF & Offloading the Master

Redis is an in-memory database. To prevent data loss when the process terminates, Redis provides two native mechanisms to persist data to disk: **RDB** (snapshots) and **AOF** (write logs).

---

## 1️⃣ RDB (Redis Database Backup)
An RDB persistence model takes point-in-time snapshots of your dataset and saves them to a compact binary file (usually `dump.rdb`).

### How It Works
1. Redis calls the `fork()` system call to create a child process.
2. The parent process continues serving client requests.
3. The child process writes the memory contents to a temporary RDB file.
4. Once done, the temporary file atomically replaces the old RDB file.

### Pros & Cons
| Pros | Cons |
| :--- | :--- |
| • **Compact**: A single compact file representing data at a specific moment. | • **Data Loss**: If Redis crashes, you lose all writes since the last snapshot. |
| • **Fast Recovery**: Restoring from RDB is significantly faster than AOF. | • **Fork Overhead**: `fork()` can freeze the main thread for milliseconds on large datasets. |
| • **Max Performance**: No disk IO is performed by the main server process. | • **CPU Intensive**: Compressing and writing data to disk utilizes significant CPU. |

---

## 2️⃣ AOF (Append Only File)
An AOF persistence model logs every write command received by the server to a file (usually `appendonly.aof`).

### How It Works
1. Every write command is written to an in-memory buffer.
2. Redis flushes the buffer to disk using a sync policy (configured by `appendfsync`):
   - `always`: Syncs after every write (very safe, extremely slow).
   - `everysec`: Syncs once per second (industry standard balance).
   - `no`: Lets the OS handle flushing (fast, unsafe).
3. As the file grows, Redis rewrites the AOF in the background (`AOF Rewrite`) by creating a new file representing the shortest sequence of commands needed to rebuild the current state.

### Pros & Cons
| Pros | Cons |
| :--- | :--- |
| • **Maximum Durability**: Minimal data loss (default is max 1 second). | • **File Size**: AOF files are much larger than RDB snapshots. |
| • **Corrupt-Proof**: Append-only log avoids corruption. Includes `redis-check-aof` tool. | • **Slower Restarts**: Restoring requires replaying every logged command. |
| • **Human-Readable**: Easy to parse or edit if needed. | • **Disk IO Overhead**: Constantly flushing to disk consumes write throughput (IOPS). |

---

## 3️⃣ Offloading the Master Using Slave Persistence

In production replication topologies, persistence overhead can impact client response times. A common pattern is to disable persistence on the Master and enable it only on the Slave nodes.

### Why Persistence Hurts the Master

1. **CPU & Latency Spikes during `fork()`**:
   When generating an RDB snapshot or running an AOF Rewrite, Redis calls `fork()`. This duplicates the page tables of the parent process. 
   If your dataset is large (e.g., 50GB), the `fork()` system call can block the single-threaded Redis event loop for **hundreds of milliseconds**. During this freeze, clients experience sudden latency spikes.
2. **Disk IO Bottlenecks**:
   Continuous AOF writing and RDB snapshotting consume physical disk write bandwidth (IOPS). If disk IO saturates, the operating system blocks write calls, directly slowing down the main Redis write path.

---

### How the Offloading Pattern Works

```
                     ┌──────────────────┐
                     │   Write Client   │
                     └────────┬─────────┘
                              │ (Sub-millisecond write responses)
                              ▼
                     ┌──────────────────┐
                     │   Redis Master   │  <--- Persistence: DISABLED (No RDB, No AOF)
                     └────────┬─────────┘       (No fork() overhead, no disk IO)
                              │
                              │ (Asynchronous replication stream)
                              ▼
                     ┌──────────────────┐
                     │   Redis Slave    │  <--- Persistence: ENABLED (AOF + RDB)
                     └────────┬─────────┘       (Disk IO & forks happen here)
                              │
                              ▼
                         ┌─────────┐
                         │ Disk IO │
                         └─────────┘
```

1. **On the Master Node**:
   - Disable RDB snapshots (remove `save` directives).
   - Disable AOF (`appendonly no`).
   - *Result*: The Master process runs entirely in RAM, performing zero disk writes. It maintains maximum throughput and consistent sub-millisecond latencies.
2. **On the Slave Node**:
   - Enable RDB and/or AOF.
   - *Result*: The Slave handles all disk writes. Any latency spikes during `fork()` or disk bottlenecks only affect the Slave, keeping client-facing operations on the Master completely unaffected.

---

### ⚠️ CRITICAL DANGER: The Auto-Restart Trap

While offloading the master is highly effective, it introduces a severe risk if not managed correctly.

> [!CAUTION]
> **The Auto-Restart Data Wipe Trap**
> If the Master node crashes, has persistence disabled, and is configured with an **auto-restart manager** (like `systemd` with `Restart=always` or Docker Compose `restart: always`), it will restart with an **empty database**.
> 
> Once the empty Master starts, the Slave node will reconnect to it. Redis replication rules dictate that the Slave must sync with the Master's current state. The Slave will immediately wipe its own persisted data to match the Master's empty memory!
> 
> **How to Prevent This:**
> 1. **Use Sentinel/Cluster**: Ensure automatic failover is configured. If the Master dies, Sentinel will promote the Slave to Master *before* the dead Master can restart and wipe the data.
> 2. **Disable Auto-Restart**: Do not allow a crashed Master node to restart automatically unless it is restored from a backup (e.g., copying the Slave's RDB/AOF file back to the Master) or promoted via failover.

---
_End of Persistence and Replication notes._
