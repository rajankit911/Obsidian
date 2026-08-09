## What is a Hot Key?

A **hot key** is a Redis key that receives a disproportionately high number of requests compared to others. For example, if a Redis instance handles 5,000 requests per second and one key accounts for 2,000 of those, it’s a hot key. Hot keys often arise due to sudden surges in traffic, such as flash sales, trending news, or popular social media posts.

## Dangers of Hot Keys

Hot keys can strain Redis by:

- Consuming excessive **CPU and bandwidth**, slowing down other operations.
- Overloading a single node in a Redis cluster, causing **data skew** or **access skew**.
- Exceeding Redis’s processing capacity, leading to **crashes** or cascading failures to backend databases.
- Triggering **timeout issues** or **performance degradation** for clients.

### 🚨 The “Hot Key” Problem

Most caches are designed assuming keys are **uniformly accessed**. But in reality, access patterns are rarely uniform.

Sometimes, a **single key becomes extremely popular**:

- A **celebrity tweet** gets millions of likes.
- A **flash sale product** goes viral.
- A **news alert** spreads globally.
- A **live sports event** updates scores every second.

When this happens, that one key turns into a **hot key**.

A hot key isn’t just “popular.” It’s **dangerous**:

- ⚡ It can receive **millions of requests per second**.
- 🕸 It can **overload a single cache shard**.
- 💥 If it expires, it can trigger a **cache stampede**, slamming the backend.


###  🧩 Why Hot Keys Break Systems

**Load Amplification**

- Caches distribute keys across shards.
- A hot key can overload the shard that owns it, while others sit idle.

**Cache Expiry Storms**

- If the hot key expires, **all requests miss at once**, hammering the DB.

**Global Propagation**

- In a geo-distributed cache, a hot key can spike in multiple regions at once.

**Latency Variability**

- Most keys are fast, but the hot key causes **long-tail latency** spikes.

### 🏗️ Hot Key Protection Strategies

### 4️⃣ Shard Replication of Hot Keys

- Copy hot key to multiple shards.
- Requests load-balanced across them.
- Reduces per-node stress.

### 5️⃣ Tiered Caching

Multi-level caching absorbs spikes:

- **L1 (per app instance)** → ultra-short TTL.
- **L2 (shared cache cluster)** → longer TTL.
- **CDN edge cache** → for global users.

👉 Ensures no single layer is overwhelmed.


### 🔍 Observability for Hot Keys

Hot keys can’t be prevented if you can’t **see them**.

Key metrics:

- **Per-key QPS** — detect sudden spikes.
- **Top N keys by traffic** — hot key visibility.
- **Cache shard imbalance** — one shard overloaded?
- **Hit ratio of hot keys** — are they missing often?

Dashboards should flag:

- “This key = 25% of all traffic.”
- “This shard = 80% load from one key.”

👉 Without per-key observability, hot keys are invisible until it’s too late.



### 📖 Real-World Lessons

- **Twitter**: Celebrity tweets triggered hot key failures. Fix: request coalescing + hot key dashboards.
- **Netflix**: Popular shows pre-warmed in caches before release, avoiding cold-start surges.
- **E-commerce sites**: Black Friday sales → products pre-loaded into regional caches.
- **Gaming companies**: Leaderboards often become hot keys — solved with sharded + replicated caches.

### ⚠️ Common Pitfalls

- ❌ Assuming all keys are equal — skew is inevitable.
- ❌ Using a single TTL for all keys — guarantees synchronized expiry.
- ❌ Ignoring hot keys in monitoring — focusing only on global hit ratio hides the problem.
- ❌ Over-relying on DB fallback — DBs are never ready for sudden floods.

### 🎯 Best Practices

- ✅ Always **detect & track hot keys**.
- ✅ Use **request coalescing** to avoid stampedes.
- ✅ Apply **stale-while-revalidate** for resilience.
- ✅ Add **TTL jitter** to prevent synchronized expiry.
- ✅ Replicate hot keys when needed.
- ✅ Pre-warm caches before known events.
- ✅ Monitor hot keys as **first-class metrics**.

### ✅ Takeaway

Hot keys are the **celebrity effect of the internet** — rare but inevitable.  
A single tweet, sale, or event can suddenly turn one key into the most requested object in your system.

The difference between **a smooth experience** and **a global outage** is preparation:

- Design caches to **handle celebrity traffic**.
- Build **observability dashboards for hot keys**.
- Combine **coalescing, stale-while-revalidate, jitter, and replication**.

## How to Discover Hot Keys in Spring Boot

Below are several methods to identify hot keys in a Spring Boot application, with practical code examples where applicable.

## 1. Using `redis-cli --hotkeys`

Redis 4.0.3 introduced the `--hotkeys` parameter for `redis-cli`, which reports the access frequency of keys. This requires setting the `maxmemory-policy` to a **Least Frequently Used (LFU)** algorithm (`volatile-lfu` or `allkeys-lfu`) in your Redis configuration.

**Configuration**:  
Edit `redis.conf` or set via `CONFIG SET`:

```
# Use volatile-lfu for keys with expiration  
CONFIG SET maxmemory-policy volatile-lfu  
# Or use allkeys-lfu for all keys  
CONFIG SET maxmemory-policy allkeys-lfu
```


**Run the Command**:

```
redis-cli --hotkeys
```

**Note**: The `--hotkeys` command performs a global scan, increasing CPU and memory usage. Use it cautiously in production, and consider running it with `-i 0.1` to reduce load by pausing every 100 SCAN commands

## 2. Using the `MONITOR` Command

The `MONITOR` command provides real-time visibility into all Redis operations, including key access patterns. You can redirect its output to a file for analysis to identify hot keys.

**Example Usage**:
```
redis-cli MONITOR > redis_monitor.log
```


## Solutions to Handle Hot Keys

## 1. Read-Write Separation

Use Redis’s master-slave replication to distribute read requests to slave nodes, reducing load on the master.

**Note**: Ensure your Redis setup includes sentinel or cluster for replication.

## 2. Redis Cluster

Distribute hot keys across multiple nodes in a Redis Cluster to balance load. Redis Cluster automatically shards keys across nodes based on their hash.

## 3. Secondary Cache (Local Cache)

Store hot keys;EA key data in a local cache like Caffeine to reduce Redis load.
**Benefits**:

- Reduces Redis requests for frequently accessed keys.
- Provides fast local access.

## 4. Key Splitting

Split hot keys into multiple keys to distribute load across nodes or shards.

**Example**:  
Instead of a single key `product:123:stock`, use multiple keys like `product:123:stock:shard1`, `product:123:stock:shard2`, etc., and distribute requests randomly.

The first mistake was treating certain data as inherently singular.

Keys like:

- `homepage:data`
- `global:config`
- `top:products`

felt natural.

But they forced all traffic through one slot → one node.

### What we changed

We started _intentionally duplicating or partitioning access_.

Instead of:

homepage:data

We moved to:

homepage:data:1  
homepage:data:2  
homepage:data:3

And distributed reads across them.

### Why this works

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*tvuwIjauscJoBu6yNdJ9vA.png)

- Same data
- Multiple keys
- Load distributed across nodes

### Trade-off

- Slight duplication
- Slight complexity

But massive improvement in load distribution.


## 2. Read Sharding at Application Layer

Redis clustering shards _data_, not _requests for the same data_.

So we added sharding logic at the application level.

### Pattern

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*hBGxNE1OLePcbpWTivx9fg.png)

We used:

- Random selection (simple)
- Consistent hashing (for stability)

### When to use this

- High-read, low-write data
- Config, metadata, reference datasets


   ## 3. Local Caching to Absorb Heat

Not every request needs to reach Redis.

For extremely hot data, we introduced **local in-memory caching**.

### Flow

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*2FPHnY70i0KBYilfsbK51g.png)

### Benefits

- Reduces pressure on Redis
- Cuts network latency
- Shields Redis from burst traffic

### Trade-off

- Staleness window increases
- Requires TTL tuning

### Where it worked best

- Feature flags
- Static configs
- Frequently accessed reference data


## 4. Separating Concerns Across Redis Instances

One of our biggest mistakes was mixing everything:

- Cache
- Locks
- Rate limits
- Sessions

in the same Redis cluster.

### What we changed

We split workloads:

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:678/1*vX8GtU6EZWdZbPTZEsWGvQ.png)

### Why this matters

Hot keys in one domain:

- No longer impact others

Memory pressure:

- Doesn’t evict critical coordination data

### Insight

> _Isolation is more valuable than optimization._


## 5. Designing for Graceful Degradation

We stopped assuming Redis is always available and fast.

Instead, we designed **fallback paths**.

### Example: Rate Limiting

Instead of:

Redis failure → block requests

We moved to:

Redis failure → allow limited traffic (safe fallback)

### Example: Config

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*ScTsn8Aa6dXKN-4PBZ-xMA.png)

### Why this matters

Redis degradation should not:

- Break the system
- Block critical flows


## 6. Handling Failover as a Performance Event

We used to treat failover as binary:

- Up or down

That was wrong.

### What actually happens

After failover:

- Cache is cold
- Replica may lag
- Load spikes immediately

### What we changed

We introduced:

- Gradual traffic ramp-up
- Cache warm-up strategies
- Retry backoff (to avoid storms)

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*Te9sx61jzWZRKxw4av_Heg.png)

### Key idea

> _Failover is not recovery.  
> It’s the start of a new stress phase._

## 7. Monitoring What Actually Matters

We were monitoring:

- CPU
- Memory
- Latency

But missing the real signal.

### What we added

- Per-key access frequency
- Top N hot keys
- Node-level skew

### Why this matters

Hot key problems are invisible unless you look at:

- **distribution**, not just averages

## 8. Reducing Critical Dependency on Redis

This was the hardest shift.

We identified flows where Redis was acting as a **gatekeeper**:

- Auth/session validation
- Payment idempotency
- Locking critical operations

### What we changed

- Added DB-backed fallbacks
- Reduced strict dependency on Redis responses
- Allowed eventual consistency where safe

### Insight

> _Redis should accelerate your system — not control its correctness._

## Closing Thoughts

We didn’t eliminate hot keys.  
We didn’t eliminate failover issues.

We made them **predictable and survivable**.

Because the goal is not:

> _“Make Redis perfect”_

The goal is:

> _“Ensure Redis behavior doesn’t break the system when it’s under stress”_

And that shift — from optimization to resilience — is what actually made the difference.

> _Liked this deep dive story? If Yes Please_ **_👏 Clap(50)_** _|_ **_📤 Share_** _|_ **_🔔 Follow_**

=======

## Practical Considerations

- **Performance Trade-offs**: Methods like `MONITOR` and `--hotkeys` can degrade performance; use them sparingly. Custom monitoring or cloud tools are safer for production.
- **Scalability**: Redis Cluster and read-write separation are ideal for high-traffic applications.
- **Complexity**: Key splitting and secondary caching add code complexity but significantly improve performance.
- **Monitoring**: Regularly monitor key access patterns during high-traffic events (e.g., flash sales) to proactively manage hot keys.