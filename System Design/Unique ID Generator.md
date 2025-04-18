# Functional Requirements

- The ID should be unique and numerical
- IDs fit into 64-bit
- The value of the ID must be incremental
- Unique Id with some fixed size and being sortable (Tweets Id, Instagram image id’s etc.)
- The system should be capable of generating **10,000 IDs per second.**


> [!question]
> Why can’t we directly use IDs as 1,2,3,… i.e., the auto-increment feature in SQL to generate the IDs?
	Sequential IDs work well on local systems but not in distributed systems. Two different systems might assign the same ID to two different requests in distributed system environment. So, the uniqueness property will be violated here.

Auto-increment does not work in a distributed environment due to following reasons:
- **Concurrency issues:**
    Different hosts may produce the same number, leading to potential collisions.

- **Security breaches:**
    Auto-increment IDs are predictable, which can lead to security breaches, especially in public-facing APIs.

- **Non-uniqueness across tables:**
    Each table has its own sequence, so an identical value may be found as the primary key of different entities.


Multiple options can be used to generate unique IDs in distributed systems.
- Multi-master replication
- UUID, ie, Universally unique identifier
- Timestamp
- Timestamp + Server ID
- Ticket server
- Twitter snowflake approach

#### Multi-master replication

This approach uses _auto-increment_ feature of database. Instead of increasing the next ID by 1, we increase it by _k_, where _k_ is the number of database servers in use. This solves some kind of scalability issues because IDs can scale with number of database servers.


![](https://miro.medium.com/v2/resize:fit:1400/1*v13_tVHwxF96MSurP7qG4A.png)


CONS:
- Hard to scale with multiple data centres
- IDs do not go up with time across multiple servers
- It does not scale well when a server is added or removed


> [!question]
> Why can’t we use multi-master replication method?
	Let us assume there are three different machines in a distributed computing environment. Let us number the machines as M1, M2, and M3. Now suppose the machine M1 is set to assign the IDs, which are a multiple of 3n, machine M2 is set to assign IDs that are a multiple of 3n+1, and machine M3 is set to assign IDs that are a multiple of 3n+2.
	If we assign IDs using this technique, the uniqueness problem will be solved. But it might violate the 2nd property i.e. assign IDs (values) in incremental nature in a distributed system.


![image](https://assets.leetcode.com/users/images/5c66da37-20ad-4851-bd3e-b54d33b1709b_1674530701.2510424.jpeg)



#### UUID

UUID is a 128-bit number used to identify information in computer systems.

PROS:
- Generating UUID is simple
- No coordination between servers is needed so there will not be any sychronization issues.
- Easy to scale as each server is responsible for generating its own IDs

CONS:
- Absence of a time correlation between the ID and the time of generation


> [!question]
> Why can’t we use `UUID`?
	We cannot use UUID here because UUID is random. UUID is not numeric. The property of being incremental is not followed, although the values are unique.



#### Timestamp


> [!question]
> Why can’t we use `Timestamp` values to generate the IDs?
	The reason is again same — the failure to assign unique ID values in distributed system environment. It may be possible that two requests land on two different systems at the same timestamp. So, if the timestamp parameter is utilized, both requests will be assigned the same ID based on epoch values (the time elapsed between the current timestamp and a pre-decided given timestamp). So, here the 1st property for uniqueness gets violated.


#### Timestamp + ServerID


> [!question]
> Why can’t we use `Timestamp + ServerID` values to generate the IDs?
	The question here would be how many bits are to be assigned to the Timestamp and ServerID. This situation would be tricky.



#### Ticket Server
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


#### Twitter Snowflakes Algorithm

- Twitter's unique ID generation system called **snowflake**


Snowflake ID is a 64-bit unique identifier that consists of three parts: timestamp, worker ID, and sequence number. The timestamp is a 41-bit integer that represents the number of milliseconds since a certain epoch time.

The worker ID is a 10-bit integer that identifies the worker generating the ID, and the sequence number is a 12-bit integer that ensures uniqueness in case multiple IDs are generated within the same millisecond by the same worker.

In this algorithm, we have to design a 64-bit solution. The structure of the 64 bits looks as follows :
- provides unique IDs that are sortable by time and offers scalability


![image](https://assets.leetcode.com/users/images/20f7e113-c9f0-4410-a5b5-726a3e976cf9_1674530681.1110442.jpeg)

Refer below for each of the section details one by one -

- **Sign Bit:**
	The sign bit is never used. Its value is always zero. This bit is just taken as a backup that will be used at some time in the future.


- **Timestamp Bits:**
	This time is the epoch time. The standard Epoch for time is January 1st, 1970. It takes **41 bits** of a long to represent the number of milliseconds since the Epoch. However, Twitter changed the Epoch benchmark time from 1st Jan 1970 to 4th November 2010.


- **Data center Bits:**
	5 bits are reserved for this, which means that there can be (2⁵) = 32 data centres.

 - **Machine ID Bits:**
	5 bits are reserved for this, which again means that there can be (2⁵) = 32 machines per data centre.

- **Sequence No Bits:**
	These bits are kept to ensure the uniqueness property to be maintained when multiple requests land on the same machine on the same data centre at the same timestamp. So, these bits are used for generating sequence numbers for IDs that are generated at the same timestamp. The sequence number is reset to zero at every millisecond. Since we have reserved 12 bits for this, we can have (2¹²) = 4096 sequence numbers which are certainly more than the IDs that are generated every millisecond by every single machine.


Datacenter IDs and machine IDs are chosen at the startup time, generally fixed once the system is up running. Any changes in datacenter IDs and machine IDs require careful review since an accidental change in those values can lead to ID conflicts. Timestamp and sequence numbers are generated when the ID generator is running.



**Ensure clock synchronization, consider the impact of low concurrency, and prioritize high availability for the mission-critical system**.


As you can see, a Spaceflake is composed of 4 main parts:

0            1                                                                   42          47                  52.               64
+-------+---------------------------------------+--------+----------+-----------+
| sign bit | timestamp (milliseconds since epoch)  | node ID | worker ID | sequence  |
+-------+---------------------------------------+--------+----------+-----------+

- **Timestamp**: The timestamp is the milliseconds elapsed since the base epoch, _it is not necessarily the Unix epoch_, when the ID was generated.
- **Node ID**: The node ID represents the ID of the node that generated the Spaceflake. It is a number between 0 and 31.
- **Worker ID**: The worker ID represents the ID of the worker that generated the Spaceflake. It is a number between 0 and 31.
- **Sequence**: The sequence is the number of Spaceflakes generated by the worker on the node. It is a number between 0 and 4095.



## How Snowflake ID Works

Snowflake ID generator is a distributed system that consists of multiple workers, each responsible for generating unique IDs.

When a worker requests a new ID, it first retrieves the current timestamp, then combines it with its worker ID and a sequence number.

The sequence number ensures that if multiple IDs are generated within the same millisecond by the same worker, each ID will be unique.

If the worker generates more than one ID in the same millisecond, the sequence number is incremented to ensure that each ID is unique.

Finally if in the same millisecond, if the sequence number also reaches its max value, the generator waits for the next millisecond and then starts generating IDs again.


## Use Cases and Benefits

Snowflake ID is widely used in distributed systems for generating unique IDs for various use cases, including:

- Distributed databases
- Message queues
- Microservices
- Big data systems
- Social networks


## The benefits of using Snowflake ID

- **High precision:** The timestamp component of the ID provides high precision and accuracy for tracking events and transactions.
- **Scalability:** The worker ID component allows for scaling the generator across multiple nodes, providing high availability and horizontal scaling.
- **Uniqueness:** The combination of timestamp, worker ID, and sequence number ensures that each ID generated is unique.
- **Performance:** The generator is designed to generate IDs quickly and efficiently, minimizing the overhead for generating IDs.



## What is Twitter’s snowflake approach?

It is a solution to generate unique IDs in distributed systems. Twitter uses this approach in Tweets, DM’s, Lists and etc.

- IDs are unique and sortable
- IDs include time. (ordered by date)
- IDs fit 64-bit unsigned integers.
- Only numerical values.

**Sign bit (1 bit):** Reserved bit (It is always 0). This can be reserver for future requests. It can be potentially used to make the overall number positive.

**Timestamp(41 bit):** Epoch timestamp in a millisecond (Snowflake’s default epoch is equal to Nov 04, 2010, 01:42:54 UTC)

**Machine ID(10-bit):** accommodates 1024 machines

**Sequence number(12-bit):** It is a local counter per each machine and increments by 1. The number reset to 0 in every millisecond. Theoretically, a machine can support a max of 4096 (2¹²) new IDs per second.

##   Advantages & Disadvantages of the Twitter Snowflake Approach

- It is 64-bit long, it is half the size of UUIDs
- Scalable (it can accommodate 1024 machines)
- Highly available (Each machine can generate 4096 unique IDs each millisecond)
- Some of the UUID versions do not include a timestamp. In this case, Twitter Snowflake has a sortable advantage.
- Design requires Zookeeper (disadvantage)
- The generated IDs are not random like UUIDs. Future IDs can predictable.
- The maximum timestamp that can be represented in 41 bits is (~ 69 years). Need a solution after this :)




# **Efficient ID Generation for Large-Scale URL Shorteners**

  

A URL shortening service at scale must generate **unique, non-colliding, and distributed** short IDs efficiently. Here, we explore various strategies for **ID generation**, their **trade-offs**, and how to implement them in a **highly scalable** system.

---

**1. Key Requirements for ID Generation**

For a large-scale URL shortener, an ID generation system must be:


✅ **Globally Unique** – No two URLs should get the same short ID.
✅ **Scalable** – Should handle billions of requests per day.
✅ **Efficient** – ID generation must be fast (~millisecond latency).
✅ **Ordered (Optional)** – If analytics require time-based ordering.
✅ **Fixed-Length (Optional)** – If a predictable Base62 format is required.

---

**2. Approaches for ID Generation**

**Approach 1: Auto-Incrementing Database ID**


**How it Works:**

• Each new URL is stored in a relational database (MySQL, PostgreSQL) with an **auto-incrementing primary key** (BIGINT).

• This **numeric ID** is then **Base62-encoded** into a short URL.

**Example:**

```
INSERT INTO urls (long_url) VALUES ('https://example.com');
SELECT LAST_INSERT_ID();  -- Returns 12345
Base62(12345) → "dnh"
```

**Pros:**

✅ **Simple to implement** with relational databases.
✅ **Efficient lookups** using the numeric primary key.

**Cons:**

❌ **Single point of failure** (DB bottleneck at high scale).
❌ **Not distributed** – Requires master-slave replication or sharding.


🚀 **Solution?** Use **sharded auto-increment IDs** or **UUID-based approaches** below.

---

**Approach 2: Distributed ID Generators (Snowflake IDs)**


**How it Works:**

• Instead of a centralized database, each server generates **unique IDs locally** using a structured format like **Twitter Snowflake IDs**.

• A 64-bit **Snowflake ID** consists of:
• **Timestamp (41 bits)** – Ensures chronological order.
• **Machine ID (10 bits)** – Uniqueness across servers.
• **Sequence Number (12 bits)** – Handles multiple requests per millisecond.
• **Reserved Bits (1 bit)** – Unused or reserved for future use.

  

**Example:**

A **64-bit Snowflake ID** might look like:

```
010110101000101000110000110001001010000000000000000000000001
```

When converted to Base62, it results in a **fixed-length short ID**.

**Pros:**

✅ **Globally unique** and **distributed** (no central DB bottleneck).
✅ **Time-ordered** IDs enable efficient indexing.
✅ **Scales to millions of requests per second.**

**Cons:**

❌ **Requires coordination** across multiple servers (Zookeeper, etcd).
❌ **Not cryptographically secure** (can be predicted if exposed).

🚀 **Solution?** Use **UUIDs** if you need randomness.

---

**Approach 3: UUIDs (Universally Unique Identifiers)**

**How it Works:**

• A **UUID (v4)** is a **random 128-bit** identifier, typically represented as:

```
a2b1c3d4-e5f6-7890-abcd-ef1234567890
```

• Convert the **UUID to Base62** and use a substring (e.g., first 7 characters).
  

**Example:**

UUID:

```
550e8400-e29b-41d4-a716-446655440000
```

Base62:

```
"XyZabc9"
```

**Pros:**

✅ **Globally unique** (probability of collision is astronomically low).
✅ **No central coordination needed** (can be generated anywhere).
✅ **Fast & scalable** for distributed systems.

  

**Cons:**

❌ **Longer than necessary** (UUIDs are 128 bits, but Base62 only needs ~42 bits for 7 characters).
❌ **Not sequential** (randomness can impact indexing performance).

🚀 **Solution?** Use **shortened ULIDs (Universally Unique Lexicographically Sortable Identifiers)**

---

**Approach 4: ULID (Universally Unique Lexicographically Sortable Identifier)**

**How it Works:**

• A **ULID** is a 128-bit identifier designed for **sortable unique IDs**.
• Unlike UUIDs, **ULIDs include a timestamp** component for ordering.
• Encoded in **Base32** by default, but can be converted to **Base62**.


**Example ULID:**

```
01H7XVR6J6GACVX43P7G6ND9WQ
```

(Base62 shortened version: "F6NpX8mT")

**Pros:**

✅ **Globally unique**, like UUIDs.
✅ **Lexicographically sortable** (better for time-based indexing).
✅ **Efficient Base62 conversion** (shorter than UUIDs).

**Cons:**

❌ **More complex implementation than simple auto-incrementing IDs**.


🚀 **Solution?** Use **ULIDs for scalable, time-ordered, and unique ID generation**.

---

**3. Which Approach to Use?**

|**Approach**|**Scalability**|**Uniqueness**|**Orderable**|**Best For**|
|---|---|---|---|---|
|**Auto-Increment ID + Base62**|❌ (DB Bottleneck)|✅ Yes|✅ Yes|Small-scale deployments|
|**Snowflake ID + Base62**|✅ High|✅ Yes|✅ Yes|Large-scale systems needing ordering|
|**UUID + Base62**|✅ High|✅ Yes|❌ No|Randomized unique IDs (security-focused)|
|**ULID + Base62**|✅ High|✅ Yes|✅ Yes|Time-ordered, scalable unique IDs|

  
---

**4. Conclusion: The Best Approach for Large-Scale URL Shortening**

✅ **For a small-scale system** → **Auto-Increment IDs with Base62**
✅ **For a high-scale distributed system** → **Snowflake IDs or ULIDs**
✅ **For cryptographic uniqueness** → **UUID v4 + Base62**


🚀 **Recommended Approach:**

**Use Snowflake IDs or ULIDs for a globally unique, time-ordered, scalable URL shortener.**



