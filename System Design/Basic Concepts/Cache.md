> Todo List
> - [ ] Cache Invalidation
> - [ ] Hardware vs software cache
## Cache

A cache is a high-speed data storage layer which stores a subset of data, typically transient in nature, so that future requests for that data are served up faster than is possible by accessing the data’s primary storage location.

A cache stores the data in fast access hardware such as RAM, basically trading off capacity for speed.

A cache typically stores a subset of data transiently, in contrast to databases whose data is usually complete and durable.

> A cache's primary purpose is to increase data retrieval performance by reducing the need to access the underlying slower storage layer.

### What is caching?

Caching is a process that stores copies of data or files in a reserved storage location so that they can be accessed faster for future requests. A cache stores data that can be the computation result or a copy of data stored elsewhere.

Caching allows you to efficiently reuse previously **retrieved or computed data**.

### Examples
- A **Domain Name System (DNS)** caches DNS records to perform faster lookups
- **Content Delivery Networks (CDNs)** use caching to reduce latency
- Web browsers cache **HTML** files, images and **JS** files to load websites faster.

Cached information can include the results of database queries, computationally intensive calculations, API requests/responses and web artifacts such as HTML, JavaScript, and image files.

> ![[caching.png]]

### Benefits of Caching

- **Improved Application Performance**
Reading data from in-memory cache is extremely faster than disk (magnetic or SSD).

- **Reduce Database Cost**
Due to the high **IOPS** (Input/Output operations per second), a single cache instance can serve hundreds of thousands of requests per second by reusing previously retrieved or computed data from in-memory layer, thus potentially replacing a number of database operations and driving the total cost down.

- **Reduce the Load on the Backend**
By redirecting significant parts of the read load from the backend database to the in-memory layer, caching can reduce the load on your database, and protect it from slower performance under load, or even from crashing at times of spikes.

- **Low Latency**
- **Increased Read Throughtput**

### Cache Hit and Cache Miss

 A _**cache hit**_ occurs when the requested data can be found in a cache, while a _**cache miss**_ occurs when it cannot.
 
 ### Cache Writing Policies
 
 A cache’s _**write policy**_ is the behavior of a cache while performing a write operation.
 
 When a system writes data to cache, at some point it must write that data to the backing store as well. The timing of this write is controlled by what is known as the _**write policy**_. There are two basic writing approaches:
 
 - ==_**Write-through**_==:
	- This is the easiest policy to implement. Write is done synchronously both to the cache and to the backing store to ensure consistency before sending the response back to the client.
		- ![[cache-write-through.png]]

	- This process is simpler and more reliable.This is used when there are no frequent writes to the cache(Number of write operation is less).

	- It helps in data recovery (In case of power outage or system failure). A data write will experience latency (delay) as we have to write to two locations (both backing store and Cache). It Solves the inconsistency problem. But it questions the advantage of having a cache in write operation (As the whole point of using a cache was to avoid multiple accessing to the backing store).

		- ![[cache-write-through-with-no-write-allocation.png]]

- ==_**Write-around**_==:
	- If we do not expect a read operation shortly after, the cache would become polluted with the entries we’re not using. `To avoid cache pollution, we may bypass cache entry allocation in case of a cache miss`

		- ![[cache-write-around.png]]

	- We refer to this policy as **“write-through with no-write allocation”**.
 	
- ==_**Write-back**_==: 
	- Initially, writing is done only to the cache. The cache returns the response before updating the backing store. In this case, the backing store update happens asynchronously in a separate sequence.

		- ![[cache-write-back.png]]

	- We can kick off such a sequence in several ways:
		- For software caches
			- right before the response return (asynchronously)
			- periodically (source storage update batching)
		- For CPU caches
			- At the time of cache eviction, based on _dirty bit_ as a state indicator.
			- ![[cache-write-back-with-write-allocation.png]]


 ### Write Allocation Decision (Write-miss Policies)
 
 if we try to write a data that is not present in the cache, this is called a **write miss**. So, a decision needs to be made on write misses, whether or not data would be loaded into the cache. This is defined by these two approaches:
 
 -	==_**Write allocate**_== (also called _fetch on write_): 
 Data is loaded to cache from the backing store, followed by a write-hit operation. In this approach, write misses are similar to read misses.
-	==_**No-write allocate**_== (also called _write-no-allocate_ or _write around_):
Data is not loaded to cache, and is written directly to the backing store. In this approach, data is loaded into the cache on read misses only.

Both write-through and write-back policies can use either of these write-miss policies, but usually they are paired in this way:

-   A **write-back** cache uses **write allocate**, hoping for subsequent writes (or even reads) to the same location, which is now cached.
-   A **write-through** cache uses **no-write allocate**. Here, subsequent writes have no advantage, since they still need to be written directly to the backing store.

### Caching Best Practices
- **Validity**
- **High hit rate**
- **Cache miss**
- **High availability**
- **TTL (Time To Live)**