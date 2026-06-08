---
tags:
  - redis
  - database-internals
  - system-design
  - architecture
date_created: 2026-06-08
last_modified: 2026-06-08
---

# Redis Topology Architecture — Production Deployments

This note covers the primary topology architectures used to deploy Redis in production, from a single-node setup to distributed, highly available clusters.

---

## 1️⃣ Standalone Architecture

The simplest deployment model where a single Redis instance runs on a single host.

### Diagram
```
┌───────────┐      Read/Write      ┌───────────────┐
│  Client   │ ───────────────────> │ Redis Primary │
└───────────┘                      └───────────────┘
```

### Key Characteristics
- **Instance Count**: Exactly 1.
- **Operations**: Handles all read and write commands.
- **High Availability**: None. If the node dies, the service is down.
- **Scaling**: Vertical scaling only (adding more RAM/CPU).

> [!CAUTION]
> **Production Suitability**: Highly discouraged for production workloads unless the data is purely transient, easily reproducible, and downtime does not impact the system.

### Pros & Cons
| Pros | Cons |
| :--- | :--- |
| • Zero complexity to set up and maintain. | • Single point of failure (SPOF). |
| • Maximum consistency (no replication lag). | • Limited by single-server memory/CPU capacity. |
| • Low cost (requires only one server). | • No automatic recovery or failover. |

---

## 2️⃣ Master-Slave (Replication) Architecture

A primary-secondary replication setup where one Master node accepts writes and replicates data to one or more Read-Only Slave (Replica) nodes.

### Diagram
```
                          ┌───────────────┐
                          │  Client Write │
                          └───────┬───────┘
                                  │
                                  ▼
                          ┌───────────────┐
                          │ Redis Master  │
                          └───────┬───────┘
                                  │ (Asynchronous Replication)
                 ┌────────────────┴────────────────┐
                 ▼                                 ▼
         ┌───────────────┐                 ┌───────────────┐
         │  Redis Slave  │                 │  Redis Slave  │
         └───────┬───────┘                 └───────┬───────┘
                 ▲                                 ▲
                 └────────────────┬────────────────┘
                          ┌───────┴───────┐
                          │  Client Read  │
                          └───────────────┘
```

### Key Characteristics
- **Data Flow**: One-way replication from Master to Slaves.
- **Write Path**: Handled exclusively by the Master node.
- **Read Path**: Can be distributed across Slaves to scale read throughput.
- **Replication**: Typically asynchronous, which means there is a chance of slight replication lag.
- **Failover**: **Manual**. If the Master node crashes, a Slave must be manually promoted to Master, and clients must be reconfigured to point to the new Master.

> [!WARNING]
> While this topology improves read throughput and data redundancy, it is **not** highly available because failover requires manual intervention.

### How Read Request Distribution Works
Since Redis itself **does not** automatically route client traffic or load balance read requests from the Master node to the Slaves, distribution must be handled externally. There are two main approaches:

#### A. Client-Side Routing (Preferred & Common)
Smart Redis clients (e.g., Lettuce or Redisson in Java, `redis-py` in Python) are configured with the addresses of both the Master and all Slave nodes. The client library handles the routing logic internally:
- **Read Preferences**: You configure the client's read policy:
  - `MASTER`: Reads only from the Master node (default).
  - `REPLICA` / `SLAVE`: Reads only from Slave nodes.
  - `REPLICA_PREFERRED`: Attempts to read from Slave nodes, falling back to Master if all Slaves are unavailable.
  - `NEAREST`: Reads from the node with the lowest network latency.
- **Load Balancing**: The client library distributes read requests across the healthy Slaves, typically using a round-robin algorithm.

#### B. Proxy-Side / Load Balancer Routing
An external TCP load balancer (e.g., HAProxy, NGINX TCP load balancing, or Envoy) is placed in front of the Redis nodes:
- **Setup**: Two separate connection pools or virtual IP (VIP) addresses are created:
  - **Write VIP**: Points only to the Master node.
  - **Read VIP**: Points to all Slave nodes. The load balancer distributes traffic among them using policies like round-robin or least connections.
- **Health Checks**: The load balancer runs periodic checks (e.g., executing the `PING` or `ROLE` command) to remove unhealthy Slaves from the pool.

> [!IMPORTANT]
> **Replication Lag & Consistency**: Because replication is asynchronous, a write to the Master might take a few milliseconds (or longer under high load) to propagate to the Slaves. Client reads routed to Slaves are **eventually consistent** and may temporarily return stale data.

### Pros & Cons
| Pros | Cons |
| :--- | :--- |
| • Read scalability by adding more slaves. | • Master is still a write bottleneck and SPOF. |
| • Data redundancy (backup replicas). | • Failover is manual and leads to write downtime. |
| • Slaves can perform persistence (RDB/AOF) to offload Master. | • Asynchronous replication can lead to data loss during failover. |

---

## 3️⃣ Redis Sentinel

A high-availability solution built on top of Master-Slave replication. A separate set of Sentinel processes monitors the Redis instances and orchestrates automatic failover.

### Diagram
```
                     ┌──────────────────┐
                     │ Sentinel Cluster │  (Monitors & Promotes)
                     └────────┬─────────┘
                              │
                              ▼
                      ┌───────────────┐
                      │ Redis Master  │
                      └───────┬───────┘
                              │ (Replication)
               ┌──────────────┴──────────────┐
               ▼                             ▼
       ┌───────────────┐             ┌───────────────┐
       │  Redis Slave  │             │  Redis Slave  │
       └───────────────┘             └───────────────┘
```

### Key Characteristics
- **Monitoring**: Sentinels constantly check if the Master and Slave instances are functioning correctly.
- **Notification**: Sentinels can notify clients or system administrators via an API when something is wrong.
- **Automatic Failover**: If the Master goes down, Sentinels vote (using Raft-like quorum) to promote one of the Slaves to Master.
- **Configuration Provider**: Clients connect to the Sentinels first to discover the address of the current active Redis Master.

> [!NOTE]
> Sentinels do not shard data; they only provide High Availability (HA) for a single logical Master-Slave group. To prevent split-brain scenarios, a minimum of 3 Sentinel instances is required.

### Pros & Cons
| Pros | Cons |
| :--- | :--- |
| • Fully automatic failover and recovery. | • Master node remains a write bottleneck (no sharding). |
| • High availability without manual intervention. | • Higher complexity (requires managing Sentinel processes). |
| • Built-in health monitoring. | • Clients must support the Sentinel protocol. |

---

## 4️⃣ Twemproxy (Proxy-based Sharding)

A fast, lightweight proxy developed by Twitter (`nutcracker`) that acts as a gateway, sharding data across multiple standalone Redis nodes or Master-Slave pairs.

### Diagram
```
                         ┌──────────────┐
                         │   Clients    │
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │  Twemproxy   │ (Sharding Gateway)
                         └──────┬───────┘
                                │
          ┌─────────────────────┼─────────────────────┐ (Consistent Hashing)
          ▼                     ▼                     ▼
 ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
 │ Redis Shard A   │   │ Redis Shard B   │   │ Redis Shard C   │
 └─────────────────┘   └─────────────────┘   └─────────────────┘
```

### Key Characteristics
- **Sharding**: Twemproxy shards key-value pairs across back-end Redis instances using consistent hashing algorithms (e.g., ketama).
- **Client Simplicity**: Clients connect directly to Twemproxy using the standard Redis protocol, unaware that they are talking to a distributed backend.
- **Connection Pooling**: Reduces connection overhead on backend Redis nodes by maintaining a pool of persistent connections.
- **Failover**: Twemproxy itself does not handle backend Redis replication or failover. It is usually paired with Redis Sentinel or custom scripts to handle node failure.

> [!INFO]
> Twemproxy is a static sharding solution. Adding or removing Redis shards in Twemproxy requires a configuration reload and causes partial cache invalidation (data redistribution).

### Pros & Cons
| Pros | Cons |
| :--- | :--- |
| • Offloads sharding complexity from the client code. | • Introduces an extra network hop (proxy latency). |
| • Reduces connection count on Redis backend. | • Dynamic rescalability (resharding) is complex. |
| • Supports multiple hashing algorithms. | • Does not support multi-key operations (e.g., MGET, MSET) across different shards. |

---

## 5️⃣ Redis Cluster

The native, decentralized distributed implementation of Redis that automatically shards data and provides high availability without external components.

### Diagram
```
                       ┌──────────────┐
                       │   Clients    │
                       └──────┬───────┘
                              │
         ┌────────────────────┼────────────────────┐ (Smart Routing)
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Master Node A   │  │ Master Node B   │  │ Master Node C   │
│ (Slots: 0-5460) │  │(Slots:5461-10922│  │(Slots:10923-1638)
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │ (Repl)             │ (Repl)             │ (Repl)
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Replica A      │  │  Replica B      │  │  Replica C      │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### Key Characteristics
- **Data Sharding**: The keyspace is divided into **16,384 hash slots**. Slots are distributed across Master nodes.
  $$\text{slot} = \text{CRC16}(\text{key}) \pmod{16384}$$
- **Decentralized (Shared-Nothing)**: Nodes communicate via a gossip protocol to manage cluster state and detect failures. There is no central controller or proxy.
- **High Availability**: Each Master node is paired with one or more Replicas. If a Master fails, the remaining Masters elect a Replica to be promoted.
- **Smart Clients**: Clients query any node in the cluster. If the node does not own the hash slot for that key, it returns a `MOVED` or `ASK` redirection response pointing the client to the correct node. The client then caches the slot-to-node map.

> [!TIP]
> Use **Hash Tags** (e.g., `{user123}:profile` and `{user123}:settings`) to force specific keys into the same hash slot. This enables multi-key operations (like transactions or MGET) within a Redis Cluster.

### Pros & Cons
| Pros | Cons |
| :--- | :--- |
| • Scales both write and read capacity horizontally. | • Client libraries must be cluster-aware (handle redirections). |
| • Native automatic failover and gossip-based recovery. | • Multi-key operations are restricted (unless hash tags are used). |
| • Online resharding (add/remove nodes dynamically without downtime). | • Increased operational complexity (requires managing multiple ports). |

---

## Summary Comparison

| Metric | Standalone | Master-Slave | Redis Sentinel | Twemproxy | Redis Cluster |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **High Availability** | ❌ No | ❌ No | ✅ Yes (Auto) | ❌ No (Relies on backends) | ✅ Yes (Auto) |
| **Write Scaling** | ❌ No | ❌ No | ❌ No | ✅ Yes (Sharding) | ✅ Yes (Sharding) |
| **Read Scaling** | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **Data Partitioning**| ❌ No | ❌ No | ❌ No | ✅ Yes (Consistent Hash) | ✅ Yes (16384 Hash Slots) |
| **Protocol Overhead**| None | Replication Lag | Heartbeats | Latency hop | Redirection on changes |
| **Decentralized** | N/A | N/A | ❌ No (Sentinel Pool) | ❌ No (Proxy Node) | ✅ Yes |

---
_End of Redis Topology notes._
