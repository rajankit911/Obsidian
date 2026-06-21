---
tags:
  - redis
  - caching
  - system-design
  - performance
date_created: 2026-06-21
last_modified: 2026-06-21
---

> [!todo] Todo list:
>- [ ] Add Excalidraw diagrams for XFetch probabilities.

# 🐎 The Cache Stampede (Thundering Herd) Problem

## **🧨 Problem Statement**

> [!info] 
> A **Cache Stampede** (also known as the **Thundering Herd** problem) occurs in highly concurrent distributed systems when a highly requested cache key (often called a "Hot Key") expires or is evicted. 

> [!question] What causes a Cache Stampede?
> When the key expires, a massive number of concurrent requests—all expecting to hit the cache—simultaneously experience a "cache miss". In a naive implementation, each of these requests will independently attempt to fetch the missing data from the underlying data source to regenerate the cache. 

**The Result:** The database experiences an instantaneous, massive spike in load. This can lead to database connection pool exhaustion, elevated latency, query timeouts, and cascading system failures. 

![[cache-stampede-hot-key.svg]]

---

## **🛡️ Mitigation Strategies**

To prevent a cache stampede without killing system performance, several strategies can be employed:

### **✅ 1. Mutex Locking (The Barrier Approach)**

The most direct way to stop a stampede is to ensure only one thread can rebuild the cache at a time. When a cache miss occurs, the worker must acquire a lock before querying the database.

```java
public byte[] getWithLock(String key, Duration ttl) throws InterruptedException {
    byte[] value = cache.get(key);
    
    if (value != null) {
        return value;
    }

    // Try to acquire a lock for 5 seconds
    String lockKey = key + ":lock";
    boolean acquired = cache.setNX(lockKey, "1", Duration.ofSeconds(5));

    if (acquired) {
        try {
            // Current thread won the race. Fetch and update.
            value = fetchFromDatabase(key);
            cache.set(key, value, ttl);
            return value;
        } finally {
            cache.del(lockKey);
        }
    }

    // Lost the race. Wait and retry reading from cache.
    Thread.sleep(100);
    return getWithLock(key, ttl);
}
```

> [!CAUTION]
> Distributed locks introduce their own failure modes. If the worker holding the lock crashes or times out, you can stall your entire request pipeline. You must set strict, short timeouts on these locks to prevent cascading failures.

### **✅ 2. Distributed Lock with Stale Fallback (Stale-While-Revalidate)**

When a cache miss occurs, the request must first acquire a distributed lock.
*   **The First Request:** Successfully acquires the lock, queries the database, and repopulates the cache.
*   **Subsequent Requests:** Fail to acquire the lock and must serve stale data.


For truly expensive operations — think **multi-table aggregations** or **external API** calls you’re caching to avoid rate limits — you want only one process across your entire fleet to re-compute at a time.

 Redis-based lock handles this, but the pattern that matters is what you return to everyone else while the lock is held. The stale key is the critical detail. Without it, all the threads that lost the lock race are stuck waiting. With it, they serve a slightly stale response — which for most use cases is far preferable to blocking or returning an error.

> [!note] 
> The tradeoff: this adds two Redis round trips per miss (SetNX + DEL) and stale data semantics.

```java
public byte[] getWithLock(Context ctx, String key) throws Exception {
	byte[] val = cache.get(ctx, key);
    if (val != null) {
        return val;
    }

    String lockKey = "lock:" + key;
    String staleKey = "stale:" + key;

    boolean acquired = cache.setNX(ctx, lockKey, "1", Duration.ofSeconds(10));
    
    if (!acquired) {
        // Another process is recomputing — serve stale if available
        byte[] stale = cache.get(ctx, staleKey);
        if (stale != null) {
            return stale;
        }
        // No stale available — wait and retry
        Thread.sleep(50);
        return cache.get(ctx, key);
    }

    try {
        byte[] value = expensiveRecompute(ctx);
        
        Duration ttl = Duration.ofMinutes(5);
        cache.set(ctx, key, value, ttl);
        cache.set(ctx, staleKey, value, ttl.plusMinutes(2)); // stale survives a bit longer
        return value;
    } finally {
        cache.del(ctx, lockKey);
    }
}
```

>[!CAUTION]
> If your system requires strict freshness, this approach needs explicit cache invalidation rather than TTL-based expiry.

### **✅ 3. Request Coalescing (Single-Flight/Promise Deduplication)**

This strategy works on the application node level by deduplicating concurrent requests for the same key. The most elegant fix is to not prevent cache misses — instead, prevent duplicate in-flight work. 

The root cause of a stampede on a single node is caching the *result* of the fetch instead of the fetch itself. By the time the result exists, the window is already open. The fix is to cache the `CompletableFuture` (the Promise) immediately, before any backend call happens.

> [!info] **Mutex Locking vs. Request Coalescing**
> **Mutex Locking** uses a distributed lock (e.g., in Redis) to ensure only ONE process across the entire cluster can fetch the data. 
> **Request Coalescing** is typically process-local. It ensures that within a single application instance (like a JVM), only one thread fetches the data while others wait for that thread's `CompletableFuture` to resolve. 
> *The Catch:* If you are running 10 pods, each pod independently coalesces, but you still fire 10 database queries per expiry event. For most workloads, this is fine—the reduction is enormous. For extremely hot keys with many replicas, you need a distributed solution (like Mutex Locking).

```java
private final ConcurrentHashMap<String, CompletableFuture<User>> inFlightRequests = new ConcurrentHashMap<>();

public CompletableFuture<User> getUser(Context ctx, String userID) {
    // 1. Check the actual cache first
    User cachedUser = cache.get(ctx, userID);
    if (cachedUser != null) {
        return CompletableFuture.completedFuture(cachedUser);
    }

    // 2. Cache the "Promise" (CompletableFuture) synchronously
    return inFlightRequests.computeIfAbsent(userID, id -> {
        // Only one thread executes this block per key
        return CompletableFuture.supplyAsync(() -> {
            try {
                User user = db.getUser(ctx, id);
                cache.set(ctx, id, user, Duration.ofMinutes(5));
                return user;
            } finally {
                // Remove the promise once the work is done
                inFlightRequests.remove(id);
            }
        });
    });
}
```

Now the "Promise" is written synchronously, before any yield happens. Every concurrent caller that comes in after the first one finds the `CompletableFuture` already there and awaits the same result. The fetch fires exactly once (per node) regardless of how many callers are waiting. One fetch. Every caller gets the result. The herd never forms.

This specific technique applies to in-memory caches and process-local deduplication. With Redis or a distributed cache, you’d tackle this differently, typically with a lock (Strategy 1). The underlying problem is identical; only the tool for solving it changes.

### **✅ 4. Probabilistic Early Expiration (XFetch)**

A more elegant approach is to refresh the key _before_ it actually expires, using a probabilistic algorithm. Instead of waiting for the cache TTL to reach zero, requests calculate a probability of triggering a background refresh based on how close the key is to its expiration.

>The **XFetch** algorithm subverts this by probabilistically **re-computing** a key _before_ it expires — with probability inversely proportional to remaining TTL.

As a key approaches expiry, **re-computation probability increases**. By the time TTL hits zero, the value has almost certainly already been refreshed by one lucky early request — quietly, with zero visible latency event.

The standard formula for this is the XFetch algorithm:

$$
-\beta \cdot \delta \cdot \ln(\text{rand}()) > \text{TTL}
$$
Where _delta_ is the delta time it takes to compute the value, and $\beta$ is an aggressive constant greater than 0. For cheap-to-compute, expensive-to-miss keys, this is almost always the right call. Set $\beta$ between 1.0 and 2.0 — higher values push re-computation further from expiry.

> [!Caution]
> This strategy decouples the cache regeneration from the hard expiration time. The tradeoff: you’re doing slightly more re-computation than strictly necessary.

```java
static class XFetchEntry {
    public byte[] value;
    public Instant expiresAt;
    public Duration delta; // how long the last recompute took
}

public byte[] xfetchGet(Context ctx, String key, double beta, Supplier<RecomputeResult> recompute) throws Exception {
    byte[] raw = cache.get(ctx, key);
    if (raw != null) {
        XFetchEntry entry = objectMapper.readValue(raw, XFetchEntry.class);
        
        double ttlRemaining = Duration.between(Instant.now(), entry.expiresAt).toMillis() / 1000.0;
        double gap = -beta * (entry.delta.toMillis() / 1000.0) * Math.log(ThreadLocalRandom.current().nextDouble());
        
        if (gap < ttlRemaining) {
            return entry.value; // still fresh, don't recompute yet
        }
        // Probabilistically decided to recompute early (let execution fall through)
    }

    long start = System.nanoTime();
    RecomputeResult result = recompute.get(); // fetch new value
    Duration delta = Duration.ofNanos(System.nanoTime() - start);

    XFetchEntry newEntry = new XFetchEntry();
    newEntry.value = result.value();
    newEntry.expiresAt = Instant.now().plus(result.ttl());
    newEntry.delta = delta;

    byte[] encoded = objectMapper.writeValueAsBytes(newEntry);
    cache.set(ctx, key, encoded, result.ttl());
    
    return result.value();
}
```

### **✅ 5. Background Refresh with Stale-While-Revalidate**

A simpler operational model: never let a cache key expire during request handling. Instead, serve stale values instantly and trigger a background refresh when the key is approaching expiry.

```java
static class RefreshEntry {
    public byte[] value;
    public Instant expiresAt;
    public Instant refreshAt; // before expiresAt
}

public byte[] getWithBackgroundRefresh(Context ctx, String key, Supplier<byte[]> refresh) throws Exception {
    byte[] raw = cache.get(ctx, key);
    if (raw == null) {
        // Cold miss — synchronous recompute, unavoidable
        return synchronousRefresh(ctx, key, refresh);
    }

    RefreshEntry entry = objectMapper.readValue(raw, RefreshEntry.class);

    if (Instant.now().isAfter(entry.refreshAt)) {
        // Stale enough to warrant refresh — do it in the background
        CompletableFuture.runAsync(() -> {
            try {
                // Execute refresh logic
                synchronousRefresh(ctx, key, refresh);
            } catch (Exception e) {
                // Handle exception
            }
        });
    }

    return entry.value; // always return immediately
}
```

P99 latency for warm requests becomes effectively zero — you never block on re-computation. The background thread races ahead and has the new value ready before the next request wave arrives.

> [!CAUTION]
> The tradeoff is thread lifecycle. You should cap background refresh concurrency using a semaphore, worker pool, or dedicated executor; otherwise, a cold restart can spawn thousands of threads simultaneously and overwhelm the system.

### **✅ 6. TTL Jitter**

A simple but highly effective technique to prevent synchronized expirations. You add a random "jitter" to each key's TTL (e.g., `TTL = 3600s + random(0, 300)s`).

---

## **⚖️ Trade-offs Between Strategies**

| Strategy                    | ✅ Pros                                                            | ❌ Cons                                                                                 |
| :-------------------------- | :---------------------------------------------------------------- | :------------------------------------------------------------------------------------- |
| **Lock-Based Regeneration** | Deterministic, strong consistency. Only one DB query is executed. | Lock contention can add latency. Potential for deadlocks.                              |
| **Request Coalescing**      | Very low performance cost. No complex distributed locks needed.   | Only deduplicates requests per node. Does not prevent cluster-wide stampedes entirely. |
| **Probabilistic (XFetch)**  | Near-zero performance penalty. Lowest latency.                    | Complex to tune the probability distribution. Can serve stale data momentarily.        |
| **Background Refresh**      | Zero latency impact on user requests.                             | Can waste resources recomputing data that is never requested.                          |
| **TTL Jitter**              | Zero performance cost. Trivial to implement.                      | Doesn't solve the problem for a single highly popular "Hot Key".                       |

---

## **📝 TL;DR – What to Use When?**

| **Scenario** | **Recommended Approach** |
| :--- | :--- |
| Critical data requiring strong synchronization | **Mutex Locking** |
| High-traffic services with identical concurrent requests per node | **Request Coalescing** |
| Highly dynamic, massive scale systems | **Probabilistic (XFetch)** |
| Dashboards, highly predictable hot keys | **Background Refresh** |
| General reduction of load spikes for bulk items | **TTL Jitter** |

---

## **⚙️ Production Rules for Handling Hot Keys**

When handling extremely popular "Hot Keys" in a production environment, follow these best practices:

**1. Multi-Tiered Caching (Local Cache + Distributed Cache):**
Implement an in-memory local cache (e.g., Caffeine, Guava) on the application instance itself with a very short TTL, backed by a distributed cache. 

**2. Always Fallback to Stale Data (Stale-While-Revalidate):**
Never let user requests hang or fail just because the cache is expired. 

**3. Circuit Breakers and Bulkheads:**
If the cache completely fails, your application should have circuit breakers to prevent the database from being DDOSed.

**4. Combine Jitter with Coalescing/Locking:**
No single strategy is a silver bullet. Use TTL Jitter to prevent mass simultaneous expirations, combined with Request Coalescing.

**5. Set aggressive lock timeouts:**
If you use distributed locks, never let a lock live longer than twice the maximum expected database query time.

