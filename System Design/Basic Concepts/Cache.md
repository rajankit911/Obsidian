> Todo List
> - [x] Cache Invalidation
> - [ ] CPU Cache
> - [ ] Cache coherence
> - [ ] Hardware vs software cache

# What is caching?

Caching is a process that stores frequently accessed data in a high-speed data storage location so that they can be accessed faster for future requests.

A cache stores the data in fast access hardware such as RAM, basically trading off capacity for speed.

> The purpose of caching is to improve the performance and efficiency of a system by reducing the need to access the underlying slower storage layer for frequently accessed data.

#### Examples

- A **Domain Name System (DNS)** caches DNS records to perform faster lookups
- **Content Delivery Networks (CDNs)** use caching to reduce latency
- Web browsers cache **HTML** files, images and **JS** files to load websites faster.

Cached information can include the results of database queries, computationally intensive calculations, API requests/responses and web artifacts such as HTML, JavaScript, and image files.

# Benefits of Caching

- **Improved Application Performance**
	Reading data from in-memory cache is extremely faster than disk (magnetic or SSD).

- **Reduce Database Cost**
	Due to the high **IOPS** (Input/Output operations per second), a single cache instance can serve hundreds of thousands of requests per second by reusing previously retrieved or computed data from in-memory layer, thus potentially replacing a number of database operations and driving the total cost down.

- **Reduce the Load on the Backend**
	By redirecting significant parts of the read load from the backend database to the in-memory layer, caching can reduce the load on your database, and protect it from slower performance under load, or even from crashing at times of spikes.

- **Low Latency**
	When a client requests some data, it is fetched from the database and then it is returned to the user. Reading data from the database needs network calls and I/O operation which is a time-consuming process. Cache reduces the network call to the database and increases the performance of API calls.

- **Increased Read Throughput**

### Cache Hit and Cache Miss

 A _**cache hit**_ occurs when the requested data can be found in a cache, while a _**cache miss**_ occurs when it cannot.

# Cache Read Write Strategies

The choice of the best write approach depends on the tradeoff in terms of data consistency and write performance.
## Write-through cache

Under this scheme, whenever an application wants to write some data, it will first write directly to the cache and then, cache system will update data synchronously to the main database. Here cache is responsible for handling write operations.

So cache is present between the application and database as an abstraction, and write operation always goes through the cache to main database. When both cache and database write are complete, write operation will be considered complete. This will ensure data consistency between cache and database.

For reading the data, we can easily use [[Cache#Read-Through Cache|read-through]] strategy

The cached data allows for fast retrieval and, since the same data gets written in the permanent storage, we will have complete data consistency between the cache and the storage. Also, this scheme ensures that nothing will get lost in case of a crash, power failure, or other system disruptions. Although, write-through minimizes the risk of data loss, since every write operation must be done twice before returning success to the client, this scheme has the disadvantage of higher latency for write operations.

#### Example:

An e-commerce website updates its product inventory in real-time. Whenever a product's stock changes, the cache is also updated to reflect the new inventory count.

![[write-through-cache.png]]

### Why there is a need for cache eviction strategy?

In one way, it looks like there is no need for a cache eviction strategy in a write-through cache because cache is always consistent with the database. But practically, this is not the scenario! We still need to define TTL or other eviction strategies (LRU or LFU). 

There are several reasons:
- Even though cache is consistent with the database, it doesn't mean that every single piece of data should be kept in cache indefinitely. Storing all data indefinitely could lead to inefficient cache usage.
- As we have seen above, write-through can populate cache with unnecessary data. By adding a time-to-live (TTL) value to each write, we can avoid cluttering up cache with extra data and ensure that cache contains the most frequently accessed data.
- What will happen when some data in the database is updated by some external service? This will create inconsistency between cache and the database. So, we need to define a proper eviction strategy to ensure updated data in the cache.


**Pros:**
- Easy to implement and understand
- Ensures the data consistency between the cache and the database
- Higher cache hit rate
- No risk of data loss

**Cons:**
- Extra latency in write operation
- Not a good choice when write operations are frequent
-  if cache becomes unavailable, it can impact application performance because every read and write operation has to go through cache


## Write-behind or Write-back cache (or lazy-write)

There is one big challenge with [[Cache#Write-through cache|write-through]] caching: the high latency of write operations because data needs to be updated in both places synchronously, i.e. first in the cache and then in the main database. When many write requests come in in a short period, the database can easily become a bottleneck. One solution to improve write performance is to use write-behind (write-back) caching. In this scheme, the application will first write to the cache, and the cache will write to the database after some delay. So, write-behind caching allows faster cache writes with eventual data consistency between cache and database. In the meantime, any read from the cache will still get the latest data.

>The idea of write-behind caching is similar to the write-through cache, but there is one significant difference: the data written to the cache is asynchronously updated in the main database. 

![[write-behind-cache.png]]

Application does not need to wait for the database update, i.e. application only writes data to the cache, and cache acknowledges the write immediately. Cache keeps track of the changes made and writes them back to the main database at a later time (maybe during less busy periods). This asynchronous update will reduce the latency of the write operation. 

> [!question] How can we implement asynchronous update from cache to database?
	 1. Use a time-based delay, where cache waits for a predefined period before performing the update to database.
	 2. Use an entry-based delay, where cache waits until a certain number of new data entries are accumulated before updating the database.

This approach minimizes the number of write operations to the data store and reduces the load on the storage system, improving the overall performance of the application.
#### Example:
A document editor application temporarily saves changes to the cache while the user is editing. Periodically, the changes are written back to the data store to minimize the number of write operations.

### Write-through vs Write-behind cache

The decision between write-through and write-behind caching strategies is related to one of the key trade-offs: data consistency vs write performance. Here is a critical comparison between both approaches:

| Parameter | Write-through | Write-behind |
| ---- | ---- | ---- |
| Consistency | maintains consistency with cache and database | temporary inconsistency with database and cache |
| Data durability | strong data durability | potential risk of data loss in the event of a sudden power failure or cache failure |
| Write latency | high latency for write operations |  low latency for write operations |
| Application | where data consistency and integrity are top priorities | where write performance for write-heavy workloads is crucial |


**Pros:**
- Low latency for write operations
- High throughput in write operations
- Better responsiveness
- Reduces the workload on the database, as the database is updated less frequently
- Even if database experiences downtime or failures, application can still function and serve read and write requests from the cache

**Cons:**
- Difficult to implement
- During this period, cache store the most recent data and the database may store old data
- Risk of permanent loss of recently written data in cache in case of cache failure or system crash

## Write-around cache

In [[Cache#Write-through cache|write-through]] or [[Cache#Write-behind or Write-back cache (or lazy-write)|write-behind]] cache, if data is unlikely to be read again soon after the write operation, cache may become filled with rarely used data (cache pollution). So, in the case of such data access pattern, we can use write-around strategy for efficient cache usage.

This technique is similar to write-through cache, but data is written directly to permanent storage, bypassing the cache. Consequently, data is not stored in the cache during the write operation. When a read request is made to the cache for recently written data, there will be a cache miss and cache will fetch data from the database, update its corresponding entry, and then return data to the application.

>This scheme can reduce the cache being flooded with write operations that will not subsequently be re-read, but has the disadvantage that a read request will create a “cache miss” and must be read from slower back-end storage and experience higher latency.

#### Example:

An application updates user profile information, which is infrequently accessed. The application writes the new data directly to the data store, avoiding unnecessary cache updates.


![[write-around-cache.png]]


> [!question]
> Suppose we want to update data that is already present in the cache. Following write-around idea, we would update the old data in the database. However, a key issue arises when a user attempts to read the same data again. The application would return the older data from cache rather than the newly stored data from the database. How can we handle stale data in the write-around cache?
	The most practical solution is to update the database and invalidate corresponding cache entry. By doing so, we force a cache miss on the next read request. This is one of the variation of write-through cache, which is called write-invalidate policy.


**Pros:**
- Avoidance of cache flooding
- Improves cache efficiency by preventing the cache from occupying infrequently accessed data
- Database consistency up to date
- No risk of data loss in the case of cache failure or power failure
- Works best for read-heavy workloads.

**Cons:**
- High cache miss rate for recently written data
- Increased latency in case of cache miss
- Not a good choice for a write-heavy workload
- Cached data becomes stale when not invalidated in case of update operation


![[cache-write-strategies.png]]

#### Cache-Aside

Cache-aside is one of the commonly used caching strategies, where cache and database are independent, and it is the responsibility of the application code to manage cache and database to maintain data consistency.

In `read-through` or  `write-through` patterns,  cache system defines the logic for updating or invalidating the cache and works as a transparent interface to the application.

The cache sits on the _side_ and the application **directly** talks to both the cache and the database as there is no connection between the cache and the primary database. So cache is “kept aside” as a scalable in-memory data store. All operations to cache and the database are handled by the application. This is shown in the figure below.


![[cache-aside.png]]



- **Reading the data (lazy-loading):**
	When a read request is received, the application first checks if the requested data is available in the cache. If the data is found in cache, we’ve _cache hit_. The data is read and returned to the client. If the data is **not found** in cache, we’ve _cache miss_. The application queries the database to read the data and **stores** the data in cache and then returns it to the requester. So this strategy will load data on-demand into the cache (lazy-loading)

- **Updating the database and cache:**
    When an update request is received to modify data, the application updates the data in the database and then, invalidate or evict the stale data from the cache. This will ensure that the next read operation will retrieve updated data from the database and add it to the cache.
    
    After updating the database, the application might update the cache with the latest and modified data to improve subsequent read performance. This idea is similar to the `write-through` strategy. This will improve subsequent read performance, reduce the likelihood of stale data, and ensure data consistency between cache and database. However, this will increase write latency because data needs to be written to both cache and database. Nonetheless, this overhead can be acceptable in systems with read-heavy workloads.
    
    Cache-aside with `write-through` strategy will be inefficient for write-heavy workloads due to high write latency and also, this can fill the cache with undesired data which may lead to high cache miss, inefficient use of cache or evictions of some frequently used data.



> [!question] Can we first invalidate the cache, and then write to the database?
	If we invalidate the cache first, there is a small window of time when a user might fetch the data before the database is updated. So this will result in a cache miss (because data was removed from the cache), and earlier version of data will be fetched from the database and added to the cache. The can result in stale cache data.

**Pros:**
- Best for read-heavy workloads
- Resilient to cache failures
- Data model in cache can be different than the data model in database due to separation of cache and database
- Data consistency between database and cache
- Lazy loading helps in optimal use of cache capacity
- Easy scalable independently of the database

**Cons:**
- Responsibility of managing the cache in application code can add complexity and increase the risk of bugs or errors
- Additional latency in case of cache miss
- Increased load on the database if multiple requests trigger cache misses simultaneously
- Frequent updates in DB will result in frequent cache misses
- Data inconsistency can occur if the data in the database is updated by external sources

#### Read-Through Cache

Unlike cache-aside pattern, which requires application logic to fetch and populate the cache, Read-Through cache manages data retrieval from database on behalf of the application. Here application will interact with the cache system that acts as an intermediary between the application and database.

It offloads the responsibility of handling cache misses and data fetching to the cache layer. Due to this, application will only interact with the cache as if it were the primary data source.

When an application requests data from the cache, cache will first check if data is present (cache hit). If yes, then cached data will be returned directly to the application. If data is not present (cache miss), cache will act as a mediator and retrieve data from the database. Now, cache will store the retrieved data and return it to the application as the result of the original read request. This will allow subsequent read requests for the same data to be served efficiently from the cache with low latency.

Read-through cache sits in-line with the database. When there is a cache miss, it loads missing data from database, populates the cache and returns it to the application.

![[read-through-cache.png]]

For writing data in read-through cache, we can follow these strategies based on the requirements: `write-around`, `write-through`, `write-back` (or `write-behind`) strategy.

Both cache-aside and read-through strategies load data **lazily**, that is, only when it is first read. While read-through and cache-aside are very similar, there are at least two key differences:

| Parameter | Cache-aside | Read-through |
| ---- | ---- | ---- |
| Cache handling | Application is responsible for fetching data from the database and populating the cache | Cache layer is responsible for handling cache misses and data fetching from the database. |
| Data Model | Data model of cache can be different than the data model in database due to separation of cache and database | Data model of cache cannot be different than that of the database. |

**Pros:**
- Simple to implement as complex logic resides in cache layer
- Best for **read-heavy** workloads
- Lazy loading helps in optimal use of cache capacity
- Reduces the load on the underlying data source
**Cons:**
- Additional latency in case of cache miss
- Increased load on the database if multiple requests trigger cache misses simultaneously
- Data inconsistency can occur if there is no invalidation strategy on write operation



# Cache Invalidation Strategies

When the original data changes, cached data becomes stale or inaccurate. Cache invalidation is the process of removing or updating the outdated cached data for maintaining data consistency.

### Cache Invalidation Methods

There are several methods of cache invalidation, each with its own advantages and disadvantages. Here are a few of the most popular techniques:

#### Purge

The purge method removes cached content for a specific object, URL, or a set of URLs. It’s typically used when there is an update or change to the content and the cached version is no longer valid. When a purge request is received, the cached content is immediately removed, and the next request for the content will be served directly from the origin server.

**Example:** A news website purges a specific article from its cache after significant updates have been made, ensuring that users receive the latest version.

**Pros:**
- Ensures that all cached data is removed and the cache is completely cleared.

**Cons:**
- Can be a slow and resource-intensive process
- Can cause temporary service disruptions if done incorrectly.

#### Refresh

The refresh method retrieves requested content from the origin server, even if a cached version is available. When a refresh request is received, the cache updates the content with the latest version from the origin server, ensuring up-to-date information. Unlike a purge, a refresh request does not remove the existing cached content but updates it with the most recent version.

**Example:** An e-commerce website refreshes the cache of a product page when a new sale is launched to display the updated pricing information.

Pros:
- Can be done quickly and easily
- Ensures that the cached data is up-to-date

Cons:
- This can result in a temporary spike in traffic as clients request the updated resource

#### Ban

The ban method invalidates cached content based on specific criteria, such as a URL pattern or header. Upon receiving a ban request, any cached content matching the specified criteria is immediately removed. Subsequent requests for the content will be served directly from the origin server, ensuring that users receive the most recent and relevant information.

**Example:** A content management system bans all cached content with a specific tag when that tag is modified, ensuring that users only see the updated content.

**Pros:**
- Allows you to selectively invalidate cached data without removing all cached data. 

**Cons:**
- Can be complex to implement and can result in additional overhead.

#### Time-to-live (TTL) expiration

This method involves setting a time-to-live value for cached content, after which the content is considered stale and must be refreshed. When a request is received for the content, the cache checks the time-to-live value and serves the cached content only if the value hasn’t expired. If the value has expired, the cache fetches the latest version of the content from the origin server and caches it.

**Example:** A weather website sets a 1-hour TTL for its weather forecast data, ensuring that users receive relatively up-to-date weather information without overloading the origin server.

**Pros:**
- Allows you to automatically invalidate cached data after a certain amount of time

**Cons:**
- This can result in clients receiving stale data if the expiration time is too long.
- Unnecessary refreshes if the expiration time is set too short.

#### Stale-while-revalidate

This method is used in web browsers and CDNs to serve stale content from the cache while the content is being updated in the background. When a request is received for a piece of content, the cached version is immediately served to the user, and an asynchronous request is made to the origin server to fetch the latest version of the content. Once the latest version is available, the cached version is updated. This method ensures that the user is always served content quickly, even if the cached version is slightly outdated.

**Example:** A media streaming platform uses stale-while-revalidate to serve video thumbnails, ensuring that users can quickly browse the catalog while the platform updates thumbnail images in the background.

**Pros:**
- Ensures that clients always have access to some version of the resource, even if it is not the latest version.

**Cons:**
- This can result in clients receiving outdated data for a short period of time.


![[cache-invalidation-methods.png]]



# Cache Eviction Policies

Eviction policies are algorithms or strategies implemented to manage limited cache space efficiently. When the cache is full and a new item needs to be stored, an eviction policy determines which existing item to remove.

These policies are essential for maximizing the cache efficiency by retaining the most relevant and frequently accessed information.

Some of the most important and common cache eviction strategies are:

### 1. Least Recently Used (LRU)

- When the cache is full, it evicts the item that hasn’t been accessed for the longest period.
- The assumption is that items that haven’t been accessed for a longer time are less likely to be used in the near future.
- LRU maintains a record of the order in which items are accessed.

### 2. Least Frequently Used (LFU)

- When the cache is full, it evicts the item with the lowest access frequency.
-  It operates on the principle that items with the fewest accesses are less likely to be needed in the future.
- LFU maintains a count of how often each item is accessed.

### 3. First-In-First-Out (FIFO)

- When the cache is full, it evicts the item that has been present in the cache for the longest time.
- In this strategy, data is stored in the cache in the order it arrives.


Each policy has its trade-offs, so choosing the right eviction policy depends on the specific requirements and usage patterns of the application. 

### Caching Best Practices
- **Validity**
- **High hit rate**
- **Cache miss**
- **High availability**
- **TTL (Time To Live)**


<p>So application developers should use proper cache invalidation schemes or expiration policies to handle the data inconsistency. Here is another example: Suppose you have successfully updated the cache but database fails to update. So the code needs to implement retries. In the worst case, during unsuccessful retries, cache contains a value that database doesn’t.</p>


<h3>Cache-aside as a Local (in-memory) Cache</h3>

<p>If an application repeatedly accesses the same data, cache-aside can be used in the local (in-memory) caching environment. Here local cache is private to each application instance. Here is a challenge: If multiple instances are dealing with the same data, each instance might store its copy of data in its local cache. Due to this, cached data across instances could become inconsistent. For example, if one instance updates the shared database, the other instance's caches won’t be automatically updated, resulting in stale or outdated data being used.</p>

<p>To address this limitation of local caching, we recommend exploring the use of distributed caching. In this caching, system maintains a centralized cache accessible to all instances of the application. Due to this, all instances can read and update the same cached data (data consistency). If required, we can also spread cache across multiple nodes (improves scalability and fault tolerance).</p>


<h3>Lifetime of cached&nbsp;data and cache expiration policy</h3>


<p>What will happen when cache data is not accessed for a specified period? As discussed above, there can be a chance that data become stale. To solve this inconsistency problem, one solution is to implement expiration policy in the application code.</p>

<p>In cache expiration policy, each item in the cache is associated with a time-to-live (TTL) value, which specifies how long the item can remain in the cache before it is considered expired. When the data expires, the application can be forced to fetch the latest data from the main data source and update the cache with the fresh data. By doing this on consistent time intervals, application can ensure that data is up-to-date.</p>
<p>But it may result in unnecessary evictions if data remains valid for a longer period than TTL. So the question is: What should be the best value of the TTL? For this, we need to ensure that the expiration policy should match the access pattern of data in the application.</p>
<ul>
<li>If TTL is too short, there will be a lot of cache misses and applications will frequently need to retrieve data from the database. In other words, low TTL can reduce chances of inconsistencies but can result in high latencies and reduced performance.</li>
<li>If TTL is too long, then cached data is likely to become stale. In other words, large TTL will ensure system performance but increases the chances of inconsistencies.</li>
</ul>
<p>Two ideas are important to determine appropriate TTL value: 1) Understanding the rate of change of the underlying data. 2) Evaluating the risk of outdated data being returned to your application. For example, we can cache static data (rarely updated data) for longer TTL and dynamic data (that changes often) for smaller TTL. This lowers the risk of returning outdated data while still providing a buffer to offload database requests.</p>

<h4><strong>TTL with&nbsp;Jitter</strong></h4>

<p>When caching multiple items, if all items have the same fixed TTL, they will expire simultaneously after that specific duration. This can lead to a “cache stampede” or “thundering herd,” where all the expired items are requested to be refreshed from database at the same time. Such a sudden surge in requests can cause high load and strain on the database.</p>

<p>To mitigate this problem, TTL with Jitter technique (TTL = Initial TTL value + jitter) introduces randomness in TTL values of individual cached items. Instead of having the same fixed TTL for all items, a small random delta or offset is added to each item’s TTL. This effectively spreads out the expiration times of the cached items, so they will not expire simultaneously.</p>


<h4>Critical ideas to&nbsp;think!</h4>

<ul>
<li>What are various use cases of cache control headers?</li>
<li>How can we determine the optimal TTL value for different types of data in cache?</li>
<li>How can we evaluate the effectiveness of expiration policy by monitoring cache hit and miss rates?</li>
<li>How do you determine the appropriate range for TTL jitter?</li>
<li>How do you handle the scenario where an item’s TTL expires, but application doesn’t immediately refresh the cache with fresh data from main data source?</li>
<li>How do you address scenarios where cached items have different rates of change in database?</li>
<li>How do you handle cases where main data source experiences significant delays or downtime, and application is unable to fetch fresh data to update the cache?</li>
</ul>



<h3>Priming the&nbsp;cache</h3>

<p>Instead of waiting for actual data requests from application to trigger caching, some applications proactively load the cache with relevant data beforehand. This is an idea of cache priming: Populating the cache with data that an application is expected to require during its startup or initialization process.</p>

<p>One of the primary reasons for priming the cache is to quickly speed up the system. The reason is simple: When actual requests arrive, there is a higher chance that the requested data is already in the cache. So in scenarios with heavy read traffic, this will reduce initial read latency and initial load on the database.</p>
<p>The effectiveness of cache priming depends on understanding the data access patterns and choosing the right data to preload into the cache. Over-priming or preloading unnecessary data can lead to wasted cache space and reduced overall cache performance.</p>
