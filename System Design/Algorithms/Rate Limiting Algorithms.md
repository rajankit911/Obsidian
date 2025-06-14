# Token Bucket

The Token Bucket algorithm is one of the most popular and widely used rate limiting approaches due to its simplicity and effectiveness.

## Working

- Imagine a bucket that holds tokens.
- The bucket has a maximum capacity of N tokens.
- When a request arrives, it must obtain a token from the bucket to proceed.
- If there are enough tokens, the request is allowed and tokens are removed.
- If there aren't enough tokens, the request is dropped.
- Tokens are replenished in the bucket back to N at a fixed rate (e.g., 10 tokens per second).

![[token_bucket.svg]]

>[!note] A bucket refills at a fixed rate; each request uses one token.

## Implementation in a real world setting

- Imagine each bucket being mapped to a client.
- A client can be uniquely identified by a signature (userId, IP, etc.).
- Each token is a request.
- With every processed request, the token count is decremented.
- Based on the accepted rate of processing requests, the tokens are replenished every fixed interval.

| ✅ Pros                                                 | ❌ Cons                                                                   |
| ------------------------------------------------------ | ------------------------------------------------------------------------ |
| Relatively straightforward to implement and understand | The memory usage scales with the number of users if implemented per-user |
| Supports burst traffic up to the bucket's capacity     | Doesn’t guarantee a perfectly smooth rate of requests                    |

Use Redis for atomicity and synchronization in distributed rate limiter. Nodes may cache counters locally and periodically sync to central store for eventual consistency.


```java
public class TokenBucket {
    private final long capacity;
    private final long refillTokens;
    private final long refillIntervalMillis;
    private long availableTokens;
    private long lastRefillTimestamp;

    public TokenBucket(long capacity, long refillTokens, long refillIntervalMillis) {
        this.capacity = capacity;
        this.refillTokens = refillTokens;
        this.refillIntervalMillis = refillIntervalMillis;
        this.availableTokens = capacity;
        this.lastRefillTimestamp = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        refill();
        if (availableTokens > 0) {
            availableTokens--;
            return true;
        }
        return false;
    }

    private void refill() {
        long now = System.currentTimeMillis();
        long elapsed = now - lastRefillTimestamp;
        if (elapsed > refillIntervalMillis) {
            long tokensToAdd = (elapsed / refillIntervalMillis) * refillTokens;
            availableTokens = Math.min(capacity, availableTokens + tokensToAdd);
            lastRefillTimestamp = now;
        }
    }
}
```


# Leaky Bucket

The Leaky Bucket algorithm is similar to Token Bucket but focuses on smoothing out bursty traffic. One of the problems with the Token Bucket algorithm is that the user can trigger requests at the edges of the replenishing window. The Leaky Bucket algorithm serves to fix this problem.

## Working

- Imagine a bucket with a small hole in the bottom.
- Requests enter the bucket from the top.
- The bucket processes ("**leaks**") requests at a constant rate through the hole.
- If the bucket is full, new requests are discarded.

![[leaky_bucket.svg]]

>[!note] Requests queue and are processed at a constant rate.


The leaky bucket follows the FIFO (First In First Out) concept and can be implemented using a queue. Using the leaky bucket, requests can come in at different rates while the server processes them at a constant and predictable rate. As the saying goes, “There are no best solutions, only trade-offs.” A consistent rate implies that the leaky bucket processes requests at an average rate leading to a slower response time.

| ✅ Pros                                                    | ❌ Cons                                         |
| --------------------------------------------------------- | ---------------------------------------------- |
| Smooths traffic as it processes requests at a steady rate | Does not handle sudden bursts of requests well |
| Prevents sudden bursts from overwhelming the system       |                                                |


```java
public class LeakyBucket {
    private final long capacity;
    private final long leakRate; // tokens per millisecond
    private long water;
    private long lastLeakTimestamp;

    public LeakyBucket(long capacity, long leakRate) {
        this.capacity = capacity;
        this.leakRate = leakRate;
        this.water = 0;
        this.lastLeakTimestamp = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        leak();
        if (water < capacity) {
            water++;
            return true;
        }
        return false;
    }

    private void leak() {
        long now = System.currentTimeMillis();
        long elapsed = now - lastLeakTimestamp;
        long leaked = elapsed * leakRate;
        if (leaked > 0) {
            water = Math.max(0, water - leaked);
            lastLeakTimestamp = now;
        }
    }
}
```



# Fixed Window Counter

Time is divided into fixed-size windows; counts requests per window.

| ✅ Pros                | ❌ Cons                        |
| --------------------- | ----------------------------- |
| Simple implementation | Allows bursts at window edges |

3. The fixed window algorithm is very similar to the token bucket algorithm. Here is a run-through of the algorithm:
    
    - The timeline is split according to minutes for each window.
        
    - Each window contains a counter of 4.
        
    - When a request reaches, we allocate the request based on the FLOOR of the timestamp.
        
    - If the request comes in at 12:01:45, it will be allocated to the window of 12:01:00
        
    - If the counter is larger than 0, we decrement the counter and process the request.
        
    - Else, we discard the request.
        

> [
> 
> ![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fbucketeer-e05bbc84-baa3-437e-9518-adb32be77984.s3.amazonaws.com%2Fpublic%2Fimages%2Fc9c78561-ad73-43ee-b267-98c72f9ead57_711x238.png)
> 
> 
> 
> ](https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fbucketeer-e05bbc84-baa3-437e-9518-adb32be77984.s3.amazonaws.com%2Fpublic%2Fimages%2Fc9c78561-ad73-43ee-b267-98c72f9ead57_711x238.png)
> 
> As with token, the biggest drawback of the fixed window algorithm is the following:

- It will potentially lead to a sudden burst of traffic near the boundary of the window.
    
- If the counter runs out at the start of the window, all clients need to wait for a long reset window.
    

> **Difference with Token Bucket**
> 
> One may ask the question as to what is the difference between token bucket and fixed window? ([Link](https://stackoverflow.com/questions/65868274/token-bucket-vs-fixed-window-traffic-burst)). To understand the difference, let us go through the following explanation:

Token bucket has two parameters as follows:

1. Bucket size: The maximum number of tokens that a bucket can hold.
    
2. Refill rate: The rate at which N tokens are added to the bucket. Let us say that this is 1/10 i.e. 1 token is added every 10 seconds. In this case, the user can trigger at max 2 requests within a short interval. 
    

> In the fixed window algorithm, all the tokens are added simultaneously at the edge of each window. This leads to a bigger burst at the edges than in Token Bucket if the refill rate is well adjusted.


```java
import java.util.concurrent.ConcurrentHashMap;

public class FixedWindowCounter {
    private final long windowSizeInMillis;
    private final int maxRequests;
    private final ConcurrentHashMap<String, Window> clientWindows = new ConcurrentHashMap<>();

    public FixedWindowCounter(long windowSizeInMillis, int maxRequests) {
        this.windowSizeInMillis = windowSizeInMillis;
        this.maxRequests = maxRequests;
    }

    public boolean allowRequest(String clientId) {
        long currentTime = System.currentTimeMillis();
        Window window = clientWindows.computeIfAbsent(clientId, k -> new Window(currentTime, 0));

        synchronized (window) {
            if (currentTime - window.startTime > windowSizeInMillis) {
                window.startTime = currentTime;
                window.requestCount = 1;
                return true;
            } else {
                if (window.requestCount < maxRequests) {
                    window.requestCount++;
                    return true;
                } else {
                    return false;
                }
            }
        }
    }

    private static class Window {
        long startTime;
        int requestCount;

        Window(long startTime, int requestCount) {
            this.startTime = start
::contentReference[oaicite:0]{index=0}
            this.startTime = startTime;
            this.requestCount = requestCount;
        }
    }
}
```


## Sliding Window Log

Maintains a log of request timestamps and counts requests within the sliding window.

| ✅ Pros   | ❌ Cons            |
| -------- | ----------------- |
| Accurate | High memory usage |
|          |                   |

```java
import java.util.*;

public class SlidingWindowLog {
    private final int maxRequests;
    private final long windowSizeInMillis;
    private final Map<String, Deque<Long>> clientRequestLogs = new HashMap<>();

    public SlidingWindowLog(int maxRequests, long windowSizeInMillis) {
        this.maxRequests = maxRequests;
        this.windowSizeInMillis = windowSizeInMillis;
    }

    public synchronized boolean allowRequest(String clientId) {
        long now = System.currentTimeMillis();
        Deque<Long> timestamps = clientRequestLogs.computeIfAbsent(clientId, k -> new LinkedList<>());

        while (!timestamps.isEmpty() && now - timestamps.peekFirst() > windowSizeInMillis) {
            timestamps.pollFirst();
        }

        if (timestamps.size() < maxRequests) {
            timestamps.addLast(now);
            return true;
        }

        return false;
    }
}
```


# Sliding Window Counter

Approximates sliding window by combining counts from current and previous windows.

| ✅ Pros                            | ❌ Cons                                         |
| --------------------------------- | ---------------------------------------------- |
| Balances accuracy and performance | Slightly less accurate than sliding window log |


4. The sliding window algorithm is akin to a token bucket but solves the problem of handling traffic bursts. Let us assume that the allowed request rate is four requests/minute. Here is the algorithm:
    
    - An array is used to store the recently processed requests.
        
    - When a request is made, we loop through the array and check the number of requests processed within the last minute.
        
    - If the number of requests processed within the last minute is less than four, we add the request to the array and process it.
        
    - As we loop through the array, we pop up requests that are processed long before the last minute.
        

> GIven that we process requests based on the concept of window, the rate limit is adhered to accurately i.e. only a predefined maximum number of requests are allowed through a predefined window.



```java
import java.util.concurrent.ConcurrentHashMap;

public class SlidingWindowCounter {
    private final int maxRequests;
    private final long windowSizeInMillis;
    private final long bucketSizeInMillis;
    private final ConcurrentHashMap<String, Bucket[]> clientBuckets = new ConcurrentHashMap<>();

    private static class Bucket {
        long timestamp;
        int count;
    }

    public SlidingWindowCounter(int maxRequests, long windowSizeInMillis, long bucketSizeInMillis) {
        this.maxRequests = maxRequests;
        this.windowSizeInMillis = windowSizeInMillis;
        this.bucketSizeInMillis = bucketSizeInMillis;
    }

    public boolean allowRequest(String clientId) {
        long now = System.currentTimeMillis();
        Bucket[] buckets = clientBuckets.computeIfAbsent(clientId, k -> new Bucket[(int) (windowSizeInMillis / bucketSizeInMillis)]);
        int total = 0;
        int currentBucketIndex = (int) ((now / bucketSizeInMillis) % buckets.length);

        synchronized (buckets) {
            for (int i = 0; i < buckets.length; i++) {
                Bucket bucket = buckets[i];
                if (bucket == null || now - bucket.timestamp > windowSizeInMillis) {
                    buckets[i] = new Bucket();
                    buckets[i].timestamp = now;
                    buckets[i].count = 0;
                }
                total += buckets[i].count;
            }

            if (total < maxRequests) {
                buckets[currentBucketIndex].count++;
                return true;
            }
        }

        return false;
    }
}
```

