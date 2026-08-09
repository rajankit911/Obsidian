---
tags:
  - redis
  - caching
  - concurrency
  - system-design
  - cache-stampede
date_created: 2026-06-27
last_modified: 2026-06-27
---

# Preventing Cache Stampedes with a Two-Layer Strategy

## **🧨 The Problem: Thundering Herd**

> [!question] What is a Cache-Miss Stampede?
> Also known as the "thundering herd" problem, this occurs when a highly requested cache key expires or is empty (cold start). Suddenly, hundreds or thousands of concurrent requests miss the cache and hit the slow underlying database or API at the same time, potentially bringing it down.

### **The Market Data Scenario**

Imagine a Market Data Cache (MDC) acting as a Redis cache-aside layer between an analyzer platform and a REST API.
- **The Burst:** At market open, stock aggregate feeds jump from 30 msg/s to 800 msg/s in seconds. The cache is totally cold because opening prices invalidate yesterday's session data.
- **The Concentration:** These 800 msg/s aren't spread evenly. They are highly concentrated in just 100–200 highly volatile stocks that update every single second.
- **The Window:** The API normally responds in ~80ms. The stampede occurs because multiple analyzers simultaneously request the *exact same ticker* within that 80ms cold-start window.
- **The Ugly Case (Timeouts):** If the API degrades and times out, exponential backoff retries mean the 80ms window could stack to well over 30 seconds.

### **Why a Simple Redis Lock Isn't Enough**

The naive fix is a distributed Redis lock per ticker: one thread grabs it, fetches the data, and the rest wait. But how do they wait?
Usually, a distributed lock's `acquire()` method works by looping:
```python
while True:
    if await self.do_acquire(): // SET NX
        return True
    await asyncio.sleep(0.1) // polls every 100ms
```
If you rely *only* on a Redis lock, every waiting thread independently hammers Redis with `SET NX` every 100ms. 
- **Happy path:** 1 poll per waiter (annoying).
- **Timeout case:** 100 polls per waiter (for a 10s attempt) × retries × N waiters across 150 hot tickers.
Redis ends up absorbing massive **O(N) useless polling traffic** while the threads accomplish nothing.

## **✅ The Solution: Two-Layer Collapsing**

To eliminate unnecessary network polls, we can use a two-layer request collapsing strategy:

1. **Layer 1 (In-Process):** Collapse concurrent requests within the *same* pod/instance using native concurrency primitives (zero polling).
2. **Layer 2 (Cross-Pod):** Use a distributed lock (e.g., Redis) to handle contention across *multiple* pods/instances.

> [!info] How it works
> If 100 requests arrive at a single instance, only **one** thread becomes the "leader". The other 99 threads are parked (wait). The leader then attempts to grab a cross-pod Redis lock. This guarantees only exactly one API call is made across the entire cluster, with zero busy-waiting.

## **🛠️ Implementation in Java**

> [!example] Java Two-Layer Cache Strategy
> In Java, we use `ConcurrentHashMap` with `CompletableFuture<Void>` for the native **Layer 1** collapsing. We use `RedisLock` (like Redisson) for **Layer 2**.

```java
import java.time.Duration;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ConcurrentHashMap;

public class MarketDataCache {

    // 1️⃣ Layer 1: In-process coalescing map
    private final ConcurrentHashMap<String, CompletableFuture<Void>> pending = new ConcurrentHashMap<>();

    private final RedisClient redisClient;
    private final RestClient restClient;

    public MarketDataCache(RedisClient redisClient, RestClient restClient) {
        this.redisClient = redisClient;
        this.restClient = restClient;
    }

    public TickerSnapshot getTickerSnapshot(String ticker) throws Exception {
        String cacheKey = "TICKER_SNAPSHOTS:" + ticker;

        // Happy Path: Cache hit
        String result = redisClient.read(cacheKey);
        if (result != null) return TickerSnapshot.fromJson(result);

        // 2️⃣ Layer 1 Collapsing
        CompletableFuture<Void> newEvent = new CompletableFuture<>();
        CompletableFuture<Void> existingEvent = pending.putIfAbsent(ticker, newEvent);

        if (existingEvent != null) {
            // Another thread is already the leader. 
            // join() parks the thread natively (zero polling!)
            existingEvent.join();

            // Re-check the cache after the leader populates it
            result = redisClient.read(cacheKey);
            if (result != null) return TickerSnapshot.fromJson(result);
        } else {
            // 👑 This thread is the leader!
            try {
                return fetchSnapshotWithLock(ticker, cacheKey);
            } finally {
                // ALWAYS signal waiting threads, even on failure
                newEvent.complete(null); 
                pending.remove(ticker);
            }
        }
        
        return null; // Fallback
    }

    private TickerSnapshot fetchSnapshotWithLock(String ticker, String cacheKey) throws Exception {
        String lockKey = "TICKER_SNAPSHOT_LOCK:" + ticker;

        // 3️⃣ Layer 2: Redis distributed lock for cross-pod contention
        RedisLock lock = redisClient.getLock(lockKey, Duration.ofSeconds(30));

        try {
            lock.acquire();

            // Double-check read (another pod might have populated it)
            String result = redisClient.read(cacheKey);
            if (result != null) return TickerSnapshot.fromJson(result);

            // True Miss: Fetch from slow API
            TickerSnapshot snapshot = restClient.getSnapshotTicker("stocks", ticker);
            
            // Write to cache
            redisClient.write(cacheKey, snapshot.toJson(), Duration.ofMinutes(5));
            return snapshot;
        } finally {
            if (lock.isLocked()) lock.release();
        }
    }
}
```

## **⚠️ Critical Failure Modes to Handle**

> [!note] The `finally` block is load-bearing!
> Without `newEvent.complete(null)` in the `finally` block, waiting threads would block indefinitely if the leader thread throws an exception.

- **Leader throws exception:** Handled by the `finally` block signaling the `CompletableFuture`. Waiting threads wake up, miss the cache, and one becomes the new leader.
- **Leader pod crashes:** The in-process waiters die with it. Cross-pod waiters rely on the Redis lock TTL (e.g., 30s) to expire and elect a new pod.
- **API Retries Outlive TTL:** If the slow API takes longer than the Redis lock TTL (30s), another pod may grab the lock. The TTL should be sized correctly using telemetry data (e.g., p99 latency) to avoid unnecessary duplicate API calls while preventing deadlocks.
