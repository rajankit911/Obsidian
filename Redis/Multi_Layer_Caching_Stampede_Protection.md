---
tags:
  - caching
  - redis
  - system-design
  - performance
  - cache-stampede
date_created: 2026-06-27
last_modified: 2026-06-27
---

# Multi-Layer Caching with Stampede Protection

## **💡 The Cache Hierarchy Explained**

When dealing with massive read volumes, a single cache layer isn't enough. A multi-layer (tiered) cache speeds up responses by keeping data at different distances from the application.

> [!info] The 3 Layers of Caching
> - **L1 Cache (In-Process):** Extremely fast, limited capacity, scoped to a single application instance (e.g., Caffeine, Guava).
> - **L2 Cache (Distributed):** Slower (requires a network hop), much larger capacity, shared across all application instances (e.g., Redis, Memcached).
> - **L3 (Source of Truth):** The slowest and most expensive layer, usually the primary database or an upstream API.

> [!example] Real-Life Example: A Coffee Shop
> - **L1:** Counter display (fastest, but tiny capacity).
> - **L2:** Back fridge (slower to reach, but larger).
> - **L3:** Supplier warehouse (slowest, most expensive to restock).

## **🧨 The Danger: Cache Stampedes**

If a highly requested key expires from all cache layers simultaneously, it can trigger a "thundering herd" where hundreds of requests bypass the cache and hammer the database at the exact same time.

### **🛡️ Stampede Protection Techniques**
1. **Single-flight (Per-key lock):** Only *one* request is allowed to recompute a missing item; all other concurrent requests for the same key must wait.
2. **Stale-while-revalidate:** Serve slightly stale data to users while a background thread fetches fresh data.
3. **Jittered expirations:** Add a random time offset (jitter) to the TTL so that related keys don't all expire at the exact same millisecond.
4. **Negative caching:** Cache "not found" results (nulls) briefly to prevent repeated database hits for non-existent items.

## **☕ Java Implementation: End-to-End Flow**

> [!example] Multi-Layer Cache in Java
> This implementation demonstrates L1 (ConcurrentHashMap/Local Cache), L2 (Redis), and Stampede Protection using a per-key `Semaphore`.

```java
import java.time.Duration;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.Semaphore;
import java.util.concurrent.ThreadLocalRandom;

public class ProductCache {

    // L1 Cache (Local Memory)
    private final LocalMemoryCache l1Cache;
    
    // L2 Cache (Redis)
    private final DistributedCache l2Cache;
    
    // Database Repository
    private final ProductRepository repository;

    // Stampede Protection: Per-key locks
    private final ConcurrentHashMap<String, Semaphore> locks = new ConcurrentHashMap<>();

    public ProductCache(LocalMemoryCache l1, DistributedCache l2, ProductRepository repo) {
        this.l1Cache = l1;
        this.l2Cache = l2;
        this.repository = repo;
    }

    public Product getProduct(String productId) throws Exception {
        String cacheKey = "product:" + productId;

        // 1️⃣ Try L1 (Memory Cache)
        Product cachedL1 = l1Cache.get(cacheKey);
        if (cachedL1 != null) {
            return cachedL1;
        }

        // 2️⃣ Try L2 (Distributed Cache)
        Product cachedL2 = l2Cache.get(cacheKey);
        if (cachedL2 != null) {
            l1Cache.put(cacheKey, cachedL2, getL1Ttl());
            return cachedL2;
        }

        // 3️⃣ Stampede Protection: Acquire per-key lock
        Semaphore gate = locks.computeIfAbsent(cacheKey, k -> new Semaphore(1));
        gate.acquire();

        try {
            // 4️⃣ Double-check caches (another thread might have filled it while we waited)
            cachedL1 = l1Cache.get(cacheKey);
            if (cachedL1 != null) return cachedL1;

            cachedL2 = l2Cache.get(cacheKey);
            if (cachedL2 != null) {
                l1Cache.put(cacheKey, cachedL2, getL1Ttl());
                return cachedL2;
            }

            // 5️⃣ Fetch from L3 (Database)
            Product fromDb = repository.getProduct(productId);

            if (fromDb == null) {
                // Negative caching avoids repeated DB hits for missing items
                l1Cache.put(cacheKey, null, Duration.ofSeconds(30));
                return null;
            }

            // 6️⃣ Update L2 and L1 with Jittered TTL
            l2Cache.put(cacheKey, fromDb, getL2TtlWithJitter());
            l1Cache.put(cacheKey, fromDb, getL1Ttl());

            return fromDb;

        } finally {
            // 7️⃣ Release lock and clean up
            gate.release();
            
            // Optional cleanup to prevent unbounded lock growth
            if (gate.availablePermits() == 1 && !gate.hasQueuedThreads()) {
                locks.remove(cacheKey);
            }
        }
    }

    private Duration getL1Ttl() {
        return Duration.ofSeconds(10);
    }

    private Duration getL2TtlWithJitter() {
        // Add 0-30 seconds of jitter so keys don't expire together
        long jitterSeconds = ThreadLocalRandom.current().nextLong(0, 31);
        return Duration.ofMinutes(5).plusSeconds(jitterSeconds);
    }
}
```

## **TL;DR – What this gives you**

| ✅ Pros | ❌ Cons |
| :--- | :--- |
| **Lightning Fast:** Sub-millisecond latency for hot keys via the L1 cache. | Increased architectural complexity (managing two cache layers). |
| **Stable Downstream Load:** The single-flight lock ensures the database only receives 1 query per expired key. | Potential for data inconsistency between L1 and L2 if updates aren't propagated correctly. |
| **Resilience:** TTL jitter and negative caching prevent synchronized expirations and malicious traffic from hurting the system. | High memory usage on application pods for the L1 cache. |
