>[!question]
>	Design a distributed unique ID generator like Twitter Snowflake.

# Functional Requirements

1. **Generate Unique IDs:** Every request should return a globally unique ID.
2. **Low Latency ID Generation:** Must generate and return the ID in less than 1ms.
3. **High Throughput Support:** Should support generating up to 10,000 IDs/sec across the system.
4. **ID Ordering (Optional):** IDs should be roughly sortable by creation time.
5. **Multi-instance Support:** IDs can be generated from multiple machines or services without collision.
6. **No Duplicate IDs, even on Restart:** The system should handle restarts or crashes gracefully without issuing duplicate IDs.
7. **APIs to Request IDs:** Could be an HTTP or RPC endpoint that returns an ID on request.

# Non Functional Requirements

1. **Scalability:** System should scale horizontally to support 10x more load with minimal change.
2. **Availability:** Should have > 99.99% uptime. ID generation should never be blocked system-wide.
3. **Consistency:** Every ID must be unique even in distributed and concurrent environments.
4. **Fault Tolerance:** System should tolerate machine failures, clock drift, or restarts.
5. **Clock Synchronization Tolerance:** Should tolerate slight clock drifts or fallback safely.

# Popular Approaches

Multiple options can be used to generate unique IDs in distributed systems.
- Database Auto-Increment or Sequence
- UUIDs, ie, Universally unique identifier
- Multi-master replication
- Twitter snowflake approach

## Database Auto-Increment or Sequence


> [!question] Why can’t we directly use IDs as 1,2,3,… i.e., the auto-increment feature in SQL to generate the IDs?
>	Sequential IDs work well on local systems but not in distributed systems. Two different systems might assign the same ID to two different requests in distributed system environment. So, the uniqueness property will be violated here.

### Potential Issue

Auto-increment does not work in a distributed environment due to following reasons:
- **Concurrency issues:**
    Different hosts may produce the same number, leading to potential collisions.

- **Security breaches:**
    Auto-increment IDs are predictable, which can lead to security breaches, especially in public-facing APIs.

- **Non-uniqueness across tables:**
    Each table has its own sequence, so an identical value may be found as the primary key of different entities.


| ✅ Pros                       | ❌ Cons                                          |
| ---------------------------- | ----------------------------------------------- |
| Super simple                 | Centralized bottleneck                          |
| Guaranteed ordering          | Poor performance under high concurrency         |
| Good for small-scale systems | Not ideal for microservices or multiple regions |

## UUIDv4

A **UUID (v4)** is a **random 128-bit** identifier, typically represented as:

```
a2b1c3d4-e5f6-7890-abcd-ef1234567890
```

| ✅ Pros                                                                                   | ❌ Cons                                                                  |
| ---------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| Generating UUID is simple                                                                | Not human readable                                                      |
| No coordination between servers is needed so there will not be any sychronization issues | Not sequential                                                          |
| Easy to scale as each server is responsible for generating its own IDs                   | Absence of a time correlation between the ID and the time of generation |
| Universally unique (very high probability)                                               | Bad for DB indexing (esp. v4 – causes index fragmentation)              |

> [!question] Why can’t we use UUID?
>	We cannot use UUID here because UUID is random. UUID is not numeric. The property of being incremental is not followed, although the values are unique.


## Multi-master replication

**Multi-Master Replication** approach for **ID generation** is a technique where **multiple nodes** (masters) are capable of **generating IDs independently**, without a single point of failure.

This approach uses _auto-increment_ feature of database. Instead of increasing the next ID by 1, we increase it by _k_, where _k_ is the number of database servers in use. This solves some kind of scalability issues because IDs can scale with number of database servers.

![[Excalidraw/System Design/images/multi_master_id_generation.svg]]


> [!question] Why can’t we use multi-master replication method?
> 	Let us assume there are three different machines in a distributed computing environment. Let us number the machines as M1, M2, and M3. Now suppose the machine M1 is set to assign the IDs, which are a multiple of 3n, machine M2 is set to assign IDs that are a multiple of 3n+1, and machine M3 is set to assign IDs that are a multiple of 3n+2.
> 	
> 	If we assign IDs using this technique, the uniqueness problem will be solved. But it might violate the incremental property i.e. assign IDs (values) must be in incremental nature in a distributed system.
> 	
> 	For Example:
> 	1st Request at T1 from M2 gets the value 100
> 	2nd Request at T2 from M1 gets the value 99

| ✅ Pros                                             | ❌ Cons                                                   |
| -------------------------------------------------- | -------------------------------------------------------- |
| Multiple nodes generating IDs in parallel          | Without proper partitioning, IDs can overlap             |
| No central bottleneck. Local ID generation is fast | It does not scale well when a server is added or removed |
| If one master fails, others can still issue IDs    | IDs do not go up with time across multiple servers       |
| Can run masters in multiple regions/data centers   | Requires machine ID uniqueness                           |

## Twitter Snowflakes Algorithm

Twitter's unique ID generation system called **Snowflake**. Snowflake ID is a 64-bit unique identifier used in distributed systems.

- IDs are unique and sortable
- IDs include time. (ordered by date)
- IDs fit 64-bit unsigned integers.
- Only numerical values.

Snowflake ID is widely used in distributed systems for generating unique IDs for various use cases, including:

- Distributed databases
- Message queues
- Microservices
- Big data systems
- Social networks

As you can see, a Snowflake is composed of 5 main parts:

0            1                                                                   42                         47                  52                 64
+-------+---------------------------------------+----------------+------------+-----------+
| Sign bit | Timestamp (milliseconds since epoch)  | Data Center ID | Machine ID  | Sequence   |
+-------+---------------------------------------+----------------+------------+-----------+

- **Sign bit (1 bit):** Reserved bit (It is always 0). The sign bit is never used. This bit is just taken as a backup that will be used at some time in the future. It can be potentially used to make the overall number positive.

- **Timestamp (41 bit):** Epoch timestamp in a millisecond (Snowflake’s default epoch is equal to Nov 04, 2010, 01:42:54 UTC)

- **Node ID (5 bit):** There can be (2⁵) = 32 nodes.

- **Worker ID (5 bit):** There can be (2⁵) = 32 workers per node.

- **Sequence number(12-bit):** These bits are kept to ensure the uniqueness property to be maintained when multiple requests land on the same machine on the same data centre at the same timestamp. So, these bits are used for generating sequence numbers for IDs that are generated at the same timestamp. The sequence number is reset to zero at every millisecond. Since we have reserved 12 bits for this, we can have (2¹²) = 4096 sequence numbers which are certainly more than the IDs that are generated every millisecond by every single machine.

Node IDs and Worker IDs are chosen at the startup time, generally fixed once the system is up running. Any changes in Node IDs and Worker IDs require careful review since an accidental change in those values can lead to ID conflicts. Timestamp and sequence numbers are generated when the ID generator is running.

> [!question] Why can’t we use Timestamp values to generate the IDs?
>	The reason is again same — the failure to assign unique ID values in distributed system environment. It may be possible that two requests land on two different systems at the same timestamp. So, if the timestamp parameter is utilized, both requests will be assigned the same ID based on epoch values (the time elapsed between the current timestamp and a pre-decided given timestamp). So, here the uniqueness property gets violated.

> [!question] Why can’t we use Timestamp + ServerID values to generate the IDs?
>	The question here would be how many bits are to be assigned to the Timestamp and ServerID. This situation would be tricky.

A **64-bit Snowflake ID** might look like:

```
0-10110101 00010100 01100001 10001001 01000000 0-00000-00000-000000010000
```


![[twitter_snowflake.svg]]

### How Snowflake ID Works

1. Snowflake ID generator is a distributed system that consists of multiple workers, each responsible for generating unique IDs.

2. When a worker requests a new ID, it first retrieves the current timestamp, then combines it with its worker ID and a sequence number.

3. The sequence number ensures that if multiple IDs are generated within the same millisecond by the same worker, each ID will be unique.

4. If the worker generates more than one ID in the same millisecond, the sequence number is incremented to ensure that each ID is unique.

5. Finally if in the same millisecond, if the sequence number also reaches its max value, the generator waits for the next millisecond and then starts generating IDs again.

![[snowflake_working.svg]]

### Coordination Service (Optional)

- Needed to assign unique **machine IDs** to each generator instance.
- Could use:
    - **Zookeeper** or **etcd**.
    - Or even a static config in small deployments.

### The benefits of using Snowflake ID

- **High precision:** The timestamp component of the ID provides high precision and accuracy for tracking events and transactions.
- **Scalability:** The worker ID component allows for scaling the generator across multiple nodes, providing high availability and horizontal scaling.
- **Uniqueness:** The combination of timestamp, worker ID, and sequence number ensures that each ID generated is unique.
- **Performance:** The generator is designed to generate IDs quickly and efficiently, minimizing the overhead for generating IDs.

### Advantages

- It is 64-bit long, it is half the size of UUIDs
- Scalable (it can accommodate 1024 machines)
- Highly available (Each machine can generate 4096 unique IDs each millisecond)
- Some of the UUID versions do not include a timestamp. In this case, Twitter Snowflake has a sortable advantage.

### Disadvantages

- Design requires Zookeeper (disadvantage)
- The generated IDs are not random like UUIDs. Future IDs can predictable.
- The maximum timestamp that can be represented in 41 bits is (~ 69 years). Need a solution after this :)


| ✅ Pros                                     | ❌ Cons                                             |
| ------------------------------------------ | -------------------------------------------------- |
| K-sortable (roughly time-ordered)          | Relies on system clock (needs clock sync handling) |
| High throughput (~1M IDs/sec per node)     | Limited to a fixed number of nodes/sequence bits   |
| Can be generated locally, very low latency |                                                    |

# Other Approaches

## Redis/Zookeeper/etcd-based Counters

Use Redis/ZK/etcd to maintain a centralized counter via atomic increment (INCR, CAS, etc).

| ✅ Pros             | ❌ Cons                                    |
| ------------------ | ----------------------------------------- |
| Simple             | Network latency                           |
| Strong consistency | Doesn’t scale well under high QPS         |
| Globally unique    | Single point of bottleneck unless sharded |

## Batch ID Preallocation (Segmented ID Generator)

**Example:** Facebook’s HiLo or Twitter’s Leaf

- Central service allocates a range of IDs (e.g. 1000–1999) to each service/node.
- Nodes generate within that range.

| ✅ Pros                                                       | ❌ Cons                             |
| ------------------------------------------------------------ | ---------------------------------- |
| Reduces DB load                                              | Requires occasional coordination   |
| Supports distributed ID generation with central coordination | Risk of ID waste (unused segments) |
|                                                              | Still not totally decentralized    |

## Hash-Based IDs (e.g., Hash of Request + Salt)

- Generate a hash of request metadata (timestamp + node ID + randomness + salt).
- Often SHA-256 or MurmurHash.

| ✅ Pros                            | ❌ Cons                                     |
| --------------------------------- | ------------------------------------------ |
| Deterministic                     | Need to ensure hash collisions are handled |
| Compact                           | No ordering                                |
| Works well when inputs are unique |                                            |

## KSUID / ULID / Sonyflake / NanoID

| **Format**    | **Ordered?** | **Size** | **Description**                      |
| ------------- | ------------ | -------- | ------------------------------------ |
| **KSUID**     | Yes          | 27 chars | K-sortable, timestamp + random       |
| **ULID**      | Yes          | 26 chars | Base32-encoded time + randomness     |
| **Sonyflake** | Yes          | 64 bits  | Like Snowflake but works well on AWS |
| **NanoID**    | No           | Custom   | URL-safe, customizable size          |

| ✅ Pros                   | ❌ Cons                                   |
| ------------------------ | ---------------------------------------- |
| Human-friendly (some)    | May not be standardized                  |
| Some offer K-sortability | Some require node coordination or config |
| Shorter than UUIDs       |                                          |

## Using Kafka Offset or Log Sequence Numbers

- Each record in Kafka or log has a unique **offset or sequence number**.
- You can piggyback on those as unique IDs.

| ✅ Pros                                    | ❌ Cons                                         |
| ----------------------------------------- | ---------------------------------------------- |
| Already unique and ordered in a partition | Not suitable for general-purpose ID generation |
| Great for event sourcing or logs          | Only works within Kafka/log domain             |

## Custom Token Services (e.g., ID Generation Microservice)

- Build a centralized microservice that hands out unique IDs via API.

| ✅ Pros                                               | ❌ Cons                                              |
| ---------------------------------------------------- | --------------------------------------------------- |
| Central logic                                        | Needs to be highly available and scalable           |
| Can include metadata in the ID (e.g., sharding info) | Can become a bottleneck or SPOF without replication |

## Ticket Server

- Developed by Flickr, It is a dedicated database server, with a single database on it.
- Used to generate distributed unique primary keys
- In that database there are tables like `Tickets32` for 32-bit IDs, and `Tickets64` for 64-bit IDs.

The idea is to use a centralized _auto_increment_ feature in a single database server (Ticket Server).

| ✅ Pros                                            | ❌ Cons                  |
| ------------------------------------------------- | ----------------------- |
| Numeric IDs                                       | Single point of failure |
| Easy to implement                                 |                         |
| Works well for small to medium-scale applications |                         |


To avoid a SPOF, we can set up multiple ticket servers. However, this will introduce new challenges such as data synchronization.


---
# **TL;DR – What to Use When?**

| **Scenario**                        | **Recommended Approach** |
| ----------------------------------- | ------------------------ |
| Low-scale, single instance          | DB auto-increment        |
| Fully distributed, time-ordered IDs | Snowflake / ULID / KSUID |
| Stateless, collision-safe IDs       | UUIDv4 or UUIDv7         |
| High-scale with coordination        | Segment-based allocation |
| Event/log ID generation             | Kafka offset             |


Absolutely, let’s dive into **Clock Synchronization Tolerance** – it’s a **crucial challenge** in distributed systems, especially in systems like distributed ID generators that rely on **timestamps**.

---

## **⏱️ What is Clock Synchronization Tolerance?**

  

**Clock synchronization tolerance** refers to the system’s ability to **handle discrepancies** between the system clocks of different nodes/machines **without malfunctioning** or producing incorrect results.

  

> In a distributed system, each machine has its own local clock. These clocks are not guaranteed to be perfectly in sync, even when using NTP (Network Time Protocol).

---

## **🧨 The Problem in Distributed ID Generation**

  

If your ID format includes a **timestamp** (e.g., Snowflake-style), this causes several risks:

  

### **1.** 

### **Clock Drift**

- Each machine’s clock can drift slightly over time, especially if NTP sync is delayed or network latency varies.
    
- For example: Machine A thinks it’s 12:00:01, while Machine B thinks it’s 12:00:00.500.
    

  

### **2.** 

### **Clock Skew**

- Worse than drift – this is a **fixed offset** in time across machines.
    
- Some machines can consistently run ahead or behind due to hardware or config issues.
    

  

### **3.** 

### **Clock Going Backwards (🫠)**

- Happens if:
    
    - NTP adjusts a fast-running clock _backward_.
        
    - Manual clock changes.
        
    - Virtual machines are paused/resumed.
        
    
- If your node had issued ID T=123456, and clock moves to T=123455, you risk:
    
    - **ID Collisions** if you’re generating the same timestamp again.
        
    - **Sequence logic breaking** (e.g., if IDs must be monotonically increasing).
        
    

---

## **⚠️ Impact on System**

|**Problem**|**Impact**|
|---|---|
|Clock goes backward|May create **duplicate IDs** or throw exceptions|
|Clocks are out of sync|**Time-based ordering** is broken|
|Large skew|Some nodes may issue **“future” timestamps**|
|Inconsistent behavior|Debugging & tracing becomes hard|

  

---

## **🛡️ Mitigation Strategies**

  

### **✅ 1.** 

### **Monotonic Clock Check**

- Track the **last timestamp** used.
    
- If current time < last timestamp → **wait until last timestamp + 1ms**.
    
- Simple, safe, but might delay generation briefly.
    

  

### **✅ 2.** 

### **Use Logical Clocks or Hybrid Clocks**

- Hybrid Logical Clocks (HLC) combine **physical time** and **logical counter**.
    
- Used in Spanner & CockroachDB.
    
- Helps retain order while tolerating minor clock issues.
    

  

### **✅ 3.** 

### **NTP Monitoring**

- Continuously monitor NTP sync status.
    
- Mark a node “unhealthy” if clock drifts beyond a threshold (e.g. 250ms).
    

  

### **✅ 4.** 

### **Don’t Rely on System Time Alone**

- Consider storing the **max ID generated** and prevent any lower timestamp reuse.
    
- If clock moves backward, you can either:
    
    - Wait it out.
        
    - Or use a **sequence number** to disambiguate.
        
    

  

### **✅ 5.** 

### **Clock Sync Daemons**

- Run services like chronyd or ntpd with high-precision, frequent updates.
    

---

## **🧪 Example Scenario**

  

Imagine this:

- Node A generates an ID at timestamp T=1700000000000.
    
- NTP adjusts the clock back to T=1699999999000.
    
- Now node A tries to generate another ID with this lower timestamp → It **collides** with an earlier ID unless you handle it.
    

---

## **💡 Best Practice in Snowflake-like Systems**

  

> “If clock moves backwards, **pause ID generation until the clock catches up**.”

  

This avoids ID duplication but can **briefly affect availability** of the service.

---

Let me know if you want a diagram to visualize this, or we can go through how to implement this in code (e.g., Java, Go).
