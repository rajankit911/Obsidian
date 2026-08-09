---
tags:
  - redis
  - caching
  - resilience
  - circuit-breaker
  - system-design
date_created: 2026-06-27
last_modified: 2026-06-27
---

# Building Resilient Caches: Circuit Breakers & Graceful Degradation

## **🧨 The Problem: Cascading Failures**

> [!question] What happens when your cache goes down?
> In a naive system architecture, a cache miss forces a request to hit the primary database. If the cache layer fails completely (or if a highly requested key expires, triggering a cache stampede), the database is suddenly overwhelmed by thousands of requests per second.

When a database is overwhelmed, queries start timing out. If your application blindly retries those timed-out queries (especially without exponential backoff), you create a feedback loop that takes down the database, which in turn takes down the entire application.

## **🛡️ The Solution: Circuit Breakers**

> [!info] The Circuit Breaker Pattern
> Just like an electrical circuit breaker prevents a house fire when there's a power surge, a software circuit breaker prevents cascading system failures. It monitors the failure rate of a service (like your database). If failures exceed a threshold, it "trips" open and immediately rejects subsequent requests without even trying to hit the struggling service.

The circuit has three states:
1. **CLOSED**: Normal operation. Requests pass through to the database.
2. **OPEN**: The database is failing. Requests are instantly rejected.
3. **HALF-OPEN**: After a cooldown period, the circuit allows a single test request through. If it succeeds, the circuit closes. If it fails, it trips back open.

## **🎭 Graceful Degradation**

> [!note] Repair at all costs vs. Fail gracefully
> Old architectures tried to repair cache misses at all costs. Modern resilient architectures prefer **graceful degradation**. If the database is down (or the circuit breaker is OPEN), the application should swallow the error and return a safe, pre-defined default value or stale data instead of crashing.

## **☕ Java Implementation**

> [!example] Combining Both Patterns
> Here is how you can implement a Circuit Breaker combined with Graceful Degradation in Java to protect a database from a cache stampede.

```java
import java.util.concurrent.Callable;

// 1️⃣ A simple Circuit Breaker Implementation
public class CacheCircuitBreaker {
    public enum State { CLOSED, OPEN, HALF_OPEN }
    
    private State state = State.CLOSED;
    private int failureCount = 0;
    private final int failureThreshold;
    private final long timeoutMillis;
    private long lastFailureTime = 0;

    public CacheCircuitBreaker(int failureThreshold, long timeoutSeconds) {
        this.failureThreshold = failureThreshold;
        this.timeoutMillis = timeoutSeconds * 1000;
    }

    public boolean isOpen() {
        return state == State.OPEN;
    }

    public <T> T callDatabase(Callable<T> databaseQuery) throws Exception {
        if (state == State.OPEN) {
            if (System.currentTimeMillis() - lastFailureTime > timeoutMillis) {
                state = State.HALF_OPEN; // Test the waters
            } else {
                throw new Exception("Circuit breaker is OPEN. Request rejected.");
            }
        }

        try {
            T result = databaseQuery.call();
            if (state == State.HALF_OPEN) {
                state = State.CLOSED;
                failureCount = 0;
            }
            return result;
        } catch (Exception e) {
            failureCount++;
            lastFailureTime = System.currentTimeMillis();
            if (failureCount >= failureThreshold) {
                state = State.OPEN; // Trip the breaker
            }
            throw e; 
        }
    }
}
```

Now, we wire this Circuit Breaker into our Cache logic to enable **Graceful Degradation**:

```java
// 2️⃣ Applying Graceful Degradation
public class ConfigurationService {

    private final CacheClient cache;
    private final DatabaseClient database;
    private final CacheCircuitBreaker circuitBreaker;

    public ConfigurationService(CacheClient cache, DatabaseClient database) {
        this.cache = cache;
        this.database = database;
        // Trip after 5 failures, stay open for 60 seconds
        this.circuitBreaker = new CacheCircuitBreaker(5, 60); 
    }

    public String getConfig(String key) {
        // Step 1: Try the cache first
        try {
            String value = cache.get(key);
            if (value != null) return value;
        } catch (Exception e) {
            // SWALLOW ERROR: Don't let a cache failure kill the system
        }

        // Step 2: Try the database, protected by the Circuit Breaker
        try {
            if (!circuitBreaker.isOpen()) {
                return circuitBreaker.callDatabase(() -> database.fetchConfig(key));
            }
        } catch (Exception e) {
            // SWALLOW ERROR: Database is down or Circuit Breaker is OPEN
        }

        // Step 3: Graceful Degradation (The Ultimate Fallback)
        // Return a safe default instead of throwing a 500 Server Error
        return getSafeDefault(key);
    }

    private String getSafeDefault(String key) {
        if ("feature_flag_x".equals(key)) return "false"; 
        return "default_safe_value";
    }
}
```

## **TL;DR – What to Use When?**

| ✅ Pros | ❌ Cons |
| :--- | :--- |
| **Circuit Breakers** prevent total database collapse during stampedes. | Adds stateful logic and complexity to the service layer. |
| **Graceful Degradation** keeps the application online for users during outages. | Users might see outdated or default data temporarily. |
| **Combined** they form the bedrock of modern, resilient distributed systems. | Requires careful mapping of what "safe defaults" actually are for every key. |
