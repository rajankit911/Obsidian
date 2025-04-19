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
3. **Fault Tolerance:** System should tolerate machine failures, clock drift, or restarts.
4. **Clock Synchronization Tolerance:** Should tolerate slight clock drifts or fallback safely.

# Popular Approaches

Multiple options can be used to generate unique IDs in distributed systems.
- Database Auto-Increment or Sequence
- Multi-master replication
- UUIDs, ie, Universally unique identifier
- Timestamp
- Timestamp + Server ID
- Ticket server
- Redis/Zookeeper/etcd-based Counters
- Twitter snowflake approach

## Database Auto-Increment or Sequence


> [!question] Why can’t we directly use IDs as 1,2,3,… i.e., the auto-increment feature in SQL to generate the IDs?
>	Sequential IDs work well on local systems but not in distributed systems. Two different systems might assign the same ID to two different requests in distributed system environment. So, the uniqueness property will be violated here.

### ✅ Pros

- Super simple.
- Guaranteed ordering.
- Good for small-scale systems.

### ❌ Cons

Auto-increment does not work in a distributed environment due to following reasons:
- **Concurrency issues:**
    Different hosts may produce the same number, leading to potential collisions.

- **Security breaches:**
    Auto-increment IDs are predictable, which can lead to security breaches, especially in public-facing APIs.

- **Non-uniqueness across tables:**
    Each table has its own sequence, so an identical value may be found as the primary key of different entities.


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

### ❌ Cons

- It does not scale well when a server is added or removed
- Hard to scale with multiple data centres
- IDs do not go up with time across multiple servers

## UUIDv4

A **UUID (v4)** is a **random 128-bit** identifier, typically represented as:

```
a2b1c3d4-e5f6-7890-abcd-ef1234567890
```


### ✅ Pros

- Generating UUID is simple
- No coordination between servers is needed so there will not be any sychronization issues
- Easy to scale as each server is responsible for generating its own IDs
- Universally unique (very high probability)

### ❌ Cons

- Not human readable
- Not sequential
- Absence of a time correlation between the ID and the time of generation
- Bad for DB indexing (esp. v4 – causes index fragmentation).


> [!question] Why can’t we use UUID?
>	We cannot use UUID here because UUID is random. UUID is not numeric. The property of being incremental is not followed, although the values are unique.

## Timestamp

> [!question] Why can’t we use Timestamp values to generate the IDs?
>	The reason is again same — the failure to assign unique ID values in distributed system environment. It may be possible that two requests land on two different systems at the same timestamp. So, if the timestamp parameter is utilized, both requests will be assigned the same ID based on epoch values (the time elapsed between the current timestamp and a pre-decided given timestamp). So, here the uniqueness property gets violated.

## Timestamp + ServerID

> [!question] Why can’t we use Timestamp + ServerID values to generate the IDs?
>	The question here would be how many bits are to be assigned to the Timestamp and ServerID. This situation would be tricky.

## Ticket Server

- Developed by Flickr, It is a dedicated database server, with a single database on it.
- Used to generate distributed unique primary keys
- In that database there are tables like `Tickets32` for 32-bit IDs, and `Tickets64` for 64-bit IDs.

The idea is to use a centralized _auto_increment_ feature in a single database server (Ticket Server).

PROS:
- Numeric IDs
- Easy to implement
- Works well for small to medium-scale applications

CONS:
- Single point of failure

To avoid a SPOF, we can set up multiple ticket servers. However, this will introduce new challenges such as data synchronization.


## Redis/Zookeeper/etcd-based Counters

Use Redis/ZK/etcd to maintain a centralized counter via atomic increment (INCR, CAS, etc).

### ✅ Pros

- Simple.
- Strong consistency.
- Globally unique.

### **❌ Cons:**

- Network latency.
- Doesn’t scale well under high QPS.
- Single point of bottleneck unless sharded.

## Twitter Snowflakes Algorithm

Twitter's unique ID generation system called **Snowflake**. Snowflake ID is a 64-bit unique identifier used in distributed systems.

- IDs are unique and sortable
- IDs include time. (ordered by date)
- IDs fit 64-bit unsigned integers.
- Only numerical values.

A **64-bit Snowflake ID** might look like:

```
010110101000101000110000110001001010000000000000000000000001
```

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

- **Data Center ID (5 bit):** There can be (2⁵) = 32 data centres.

- **Machine ID (5 bit):** There can be (2⁵) = 32 machines per data centre.

- **Sequence number(12-bit):** These bits are kept to ensure the uniqueness property to be maintained when multiple requests land on the same machine on the same data centre at the same timestamp. So, these bits are used for generating sequence numbers for IDs that are generated at the same timestamp. The sequence number is reset to zero at every millisecond. Since we have reserved 12 bits for this, we can have (2¹²) = 4096 sequence numbers which are certainly more than the IDs that are generated every millisecond by every single machine.

Datacenter IDs and machine IDs are chosen at the startup time, generally fixed once the system is up running. Any changes in datacenter IDs and machine IDs require careful review since an accidental change in those values can lead to ID conflicts. Timestamp and sequence numbers are generated when the ID generator is running.

>[!Note]
>**Ensure clock synchronization, consider the impact of low concurrency, and prioritize high availability for the mission-critical system**.

### How Snowflake ID Works

1. Snowflake ID generator is a distributed system that consists of multiple workers, each responsible for generating unique IDs.

2. When a worker requests a new ID, it first retrieves the current timestamp, then combines it with its worker ID and a sequence number.

3. The sequence number ensures that if multiple IDs are generated within the same millisecond by the same worker, each ID will be unique.

4. If the worker generates more than one ID in the same millisecond, the sequence number is incremented to ensure that each ID is unique.

5. Finally if in the same millisecond, if the sequence number also reaches its max value, the generator waits for the next millisecond and then starts generating IDs again.

### The benefits of using Snowflake ID

- **High precision:** The timestamp component of the ID provides high precision and accuracy for tracking events and transactions.
- **Scalability:** The worker ID component allows for scaling the generator across multiple nodes, providing high availability and horizontal scaling.
- **Uniqueness:** The combination of timestamp, worker ID, and sequence number ensures that each ID generated is unique.
- **Performance:** The generator is designed to generate IDs quickly and efficiently, minimizing the overhead for generating IDs.
- 
### Advantages

- It is 64-bit long, it is half the size of UUIDs
- Scalable (it can accommodate 1024 machines)
- Highly available (Each machine can generate 4096 unique IDs each millisecond)
- Some of the UUID versions do not include a timestamp. In this case, Twitter Snowflake has a sortable advantage.
- 
### Disadvantages

- Design requires Zookeeper (disadvantage)
- The generated IDs are not random like UUIDs. Future IDs can predictable.
- The maximum timestamp that can be represented in 41 bits is (~ 69 years). Need a solution after this :)