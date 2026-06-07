# Redisson RMapCache TTL & Idle Time — Session Notes (Markdown)

> Documentation of the **current session discussion** focused on Redisson `RMapCache`, TTL, idle time, Lua scripts, and eviction scheduler.

---

# 1️⃣ RMapCache.put with TTL and Idle Time

```java
mapCache.put(key, tempObject, ttl, unit, idleTime, unit);
```

## What It Does

- Stores the entry inside Redis-backed map
    
- Applies **per-entry TTL** (time to live)
    
- Applies **per-entry idle timeout** (max inactivity time)
    
- Does **NOT** set TTL on the whole Redis hash key
    
- Uses a Lua script internally for atomic updates
    

## Scope of Expiration

|Level|TTL Applied?|
|---|---|
|Redis hash key|❌ No|
|Map entry (field)|✅ Yes|

---

# 2️⃣ Native Redis Hash vs RMapCache

## Native Redis Hash

```
HSET hash field value
EXPIRE hash 60
```

- TTL applies only to whole hash
    
- No per-field expiration
    
- No idle timeout support
    

## RMapCache

- Per-entry TTL supported
    
- Per-entry idle timeout supported
    
- Implemented using:
    
    - HASH for values
        
    - ZSET for TTL metadata
        
    - ZSET for idle metadata
        
    - Lua scripts for atomicity
        
    - EvictionScheduler for cleanup
        

---

# 3️⃣ TTL vs Idle Time

## TTL (Time To Live)

- Absolute lifetime
    
- Starts at write/update
    
- Does NOT reset on reads
    
- Hard expiry limit
    

Example: expire after 10 minutes regardless of usage

---

## Idle Time

- Inactivity lifetime
    
- Resets on read/write/update
    
- Entry expires only if unused
    

Example: expire if not accessed for 3 minutes

---

## When Both Are Set

Entry expires when **first** of the two triggers:

```
min(TTL expiry, Idle expiry)
```

---

# 4️⃣ fastReplace Behavior with TTL & Idle

```java
mapCache.fastReplace(key, newValue);
```

## Behavior

|Aspect|Effect|
|---|---|
|TTL|preserved|
|Idle timer|reset|
|Value|updated|

## Why Idle Resets

- Update counts as access
    
- Idle timeout is based on last access
    
- Redisson updates idle metadata ZSET
    

## Result Pattern

Good for:

- updating value
    
- keeping absolute TTL fixed
    
- extending inactivity window
    

---

# 5️⃣ Does getMapCache Load All Hash Values?

```java
RMapCache<String,String> mapCache = redisson.getMapCache(name);
```

## Behavior

- Does NOT load data
    
- Does NOT query Redis
    
- Creates client-side proxy only
    
- Lazy access model
    

Redis is hit only when calling:

- get()
    
- getAll()
    
- readAllMap()
    
- entrySet()
    
- iterator()
    

---

# 6️⃣ Bulk Field Existence Check in RMapCache

No direct bulk `containsKey` API exists.

## Recommended Method

```java
Map<K,V> found = mapCache.getAll(keys);
Set<K> existing = found.keySet();
```

## Why This Works

- Single Redis roundtrip
    
- Uses HMGET internally
    
- TTL-aware filtering
    
- Expired entries excluded automatically
    

## Avoid

```java
for (k : keys) mapCache.containsKey(k);
```

Reason: N network calls.

---

# 7️⃣ Redisson Lua Script Usage

## When Lua Scripts Are Used

Redisson uses Lua when operations require:

- multi-key atomic updates
    
- TTL metadata updates
    
- conditional writes
    
- race-free logic
    

Examples:

- RMapCache.put
    
- fastReplace
    
- distributed locks
    
- rate limiter
    

---

## Execution Flow

```
Java call → Redisson → EVAL → Redis
first run → script stored
next runs → EVALSHA
```

## How To Observe

```
redis-cli MONITOR
SLOWLOG GET
```

---

# 8️⃣ EvictionScheduler

## Purpose

Background cleanup of expired RMapCache entries.

Because Redis does NOT know per-entry TTL metadata.

---

## Config (Java)

```java
config.setCleanUpKeysAmount(200);
config.setMinCleanUpDelay(10);
config.setMaxCleanUpDelay(900);
```

## Parameters

- cleanUpKeysAmount → entries removed per run
    
- minCleanUpDelay → minimum delay between runs
    
- maxCleanUpDelay → maximum delay between runs
    

Scheduler adapts frequency based on expired entry count.

---

# 9️⃣ Your TTL-Preserving Update Pattern (Validated)

```java
if (mapCache.fastReplace(key, value)) {
    // existing → TTL preserved, idle reset
} else {
    mapCache.put(key, value, ttl, unit, idle, unit);
}
```

## Result

Existing entry:

- value updated
    
- TTL preserved
    
- idle refreshed
    

New entry:

- value inserted
    
- TTL set
    
- idle set
    

This is a correct TTL-preserving cache write strategy.

---

# 🔟 Key Takeaways

- RMapCache supports **per-entry TTL and idle timeout**
    
- TTL ≠ idle time
    
- fastReplace preserves TTL but resets idle
    
- getMapCache() does not load data
    
- Bulk existence check → use getAll()
    
- Redisson uses Lua for atomic TTL metadata updates
    
- EvictionScheduler cleans expired entries in background
    

---

_End of session notes._