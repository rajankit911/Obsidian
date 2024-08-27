- In java, garbage means unreferenced objects.
- Garbage Collection is process of reclaiming the runtime unused memory automatically. In other words, it is a way to destroy the unused objects.

![[Pasted image 20240406212106.png]]

# Advantages

- It makes java **memory efficient** because garbage collector removes the unreferenced objects from heap memory.
- It is **automatically done** by the garbage collector(a part of JVM) so we don't need to make extra efforts.
- No manual memory allocation/deallocation handling, which reduces the risk of memory leaks, dangling pointers, double frees, and other memory-related errors
- Automatic memory leak management, which prevents the application from running out of memory or crashing due to excessive memory consumption. JVM Garbage Collectors won’t cover all memory leak issues therefore developers still need to watch out for such cases
- Improved code readability and maintainability, as the programmer does not have to worry about low-level memory details

>[!note]
>Garbage collection is performed by a daemon thread called Garbage Collector(GC). This thread calls the finalize() method before object is garbage collected.


# Disadvantages

- **Unpredictable pauses** - Some garbage collectors may stop the application threads while performing garbage collection, which can affect the responsiveness and latency of the application.
- Tuning complexity, as different garbage collectors have different parameters and options that affect their behavior and performance
- Performance overhead, as the JVM has to keep track of object creation and deletion, and periodically run garbage collection cycles to reclaim unused memory. This activity requires more CPU power than the original application.
- Automatized memory management won’t be as efficient as the proper manual memory allocation/deallocation.

# Working of GC

_Garbage Collection_ tracks each and every object available in the JVM heap space, and removes the unused ones.

There are three basic steps in garbage collection:

- **Mark**: The garbage collector scans the heap memory segment and marks all the live objects — that is, objects to which the application holds references. All the objects that have no references to them are eligible for removal
- **Sweep**: The garbage collector recycles all the unreferenced objects from the heap
- **Compact**: The sweep step tends to leave many empty regions in heap memory, causing memory fragmentation. The compact step moves the live objects closer together to eliminate these gaps and improve heap utilization

# Generational Garbage Collection

Generational garbage collectors prioritize cleaning up memory regions with short-lived objects. By identifying and removing these objects quickly, they can free up over 90% of unneeded memory blocks.

Generational garbage collection divides heap memory into two major partitions:
-  _young_ (also called _nursery_) generation
- _old_ (also called _tenured_) generation.

## Young objects

All new objects are initially allocated to the young generation, which is subdivided into two partitions: _Eden_ and _survivor_. The Eden partition is where all objects are put when they are new. After each cycle of garbage collection, all objects that remain in the Eden partition are moved to the survivor partition.

The survivor partition is further divided into two partitions called S0 and S1, also known as the FromSpace and the ToSpace.

The normal flow of object allocation is:

1. At first, all the new objects are allocated in the Eden partition, while both survivor partitions are empty.
2. When the Eden partition has filled up, causing a new allocation to fail, the JVM launches a minor garbage collection. After objects that are no longer needed are removed, all live objects are marked and moved to the S0 partition. Thus, the Eden partition is cleared while S1 is still empty.
3. The next time Eden fills up and another minor garbage collection takes place, it marks all the live objects in the Eden and S0 partitions. Then all the live objects from Eden and S0 are moved into S1. Eden and S0 are left empty. At any given time, one of the survivor partitions (S0 or S1) is always empty.
4. The next minor garbage collection carries out the same process as the one in Step 3, but moves objects from S1 to S0 instead of from S0 to S1. All the live objects are left in S0.

	After the sequence just described, minor garbage collections alternate between Steps 3 and 4. After multiple minor garbage collections, when objects become long-lived in the young generation, they become eligible for promotion to the old generation.


## Minor collection

- New objects are allocated in the Eden, part of the YoungGen. When Eden fills up, a minor GC is invoked.
- During this process, the GC collects the Survivor region and cleans it from dead objects.
- Objects that have survived N minor collections are moved to the OldGen. Remaining objects are moved to the second Survivor region and the regions are swapped.
- The Eden region is then collected and remaining objects are moved to the active Survivor region or overflow to the OldGen.
- Objects that survive more collections than set with `MaxTenuringThreshold` are promoted to OldGen during MinorGC.

Typically, MinorGC is mostly StopTheWorld collection but pauses are short and mostly irrelevant.


## Old objects

Garbage collections in the old generation are called major garbage collections, and perform marks and sweeps.
## Major collection

- OldGen is a large part of the Heap collected by GC through Major collections.
- It takes time to fill up as most garbage is collected with MinorGC. However, factors such as Minor Collection moving objects to OldGen, large objects being allocated directly in OldGen, live objects not collected during Eden collection overflowing to OldGen, and garbage objects in Eden with overridden finalize() methods not being released during MinorGC can fill it up.

OldGen collections are typically StopTheWorld collections and can have lengthy pauses due to the amount of data.

## Full collection

A full garbage collection cleans up both the young and old generations. It promotes all the live objects from the young generation to the old generation and compacts the old generation. A full garbage collection causes a stop-the-world pause to the application, to ensure that no new objects are allocated on the heap memory and that no existing objects become unreachable while the full garbage collection is performed.

It combines Minor and Major Collections, either concurrently or sequentially. As it collects both YoungGen and OldGen, it’s the longest collection. It’s usually triggered when Minor Collection can’t promote survivors to OldGen due to lack of space, so Major Collection is triggered to free up space.

# GC Implementations

JVM has five types of _GC_ implementations:

- Serial Garbage Collector
- Parallel Garbage Collector
- CMS Garbage Collector
- G1 Garbage Collector
- Z Garbage Collector

Each uses different algorithms to maintain JVM’s memory.

## Serial Garbage Collector

This is the simplest GC implementation, as it basically works with a single thread for both young and old generations. As a result, this _GC_ implementation freezes all application threads when it runs causing long pauses.. Therefore, it’s not a good idea to use it in multi-threaded applications, like server environments.

It is suitable for single-threaded applications or applications with small heaps that do not have strict latency requirements

## Parallel Garbage Collector

This is also known as Throughput Collector or Scavenge Collector.

It’s the default _GC_ of the _JVM_. Unlike _Serial Garbage Collector_, it uses multiple threads for both young and old generations, but it also freezes other application threads while performing _GC_, but it can achieve higher throughput than Serial GC by utilizing multiple CPU cores.

It is suitable for multi-threaded applications or applications with large heaps that prioritize throughput over latency

```java
-XX:ParallelGCThreads=<N> // GC threads
-XX:MaxGCPauseMillis=<N> // Max pause time
-XX:GCTimeRatio=<N> // Max throughput target
-Xmx<N> // Max heap footprint
```


## Concurrent Mark Sweep (CMS) Garbage Collector

This is also known as Low Latency Collector or Mostly Concurrent Collector.

It uses multiple threads for both young and old generations. It performs most of its work concurrently with application threads (concurrent), but it still stops all application threads briefly at the beginning and end of each GC cycle. It can achieve lower pause times than Parallel GC, but it may cause higher CPU overhead and memory fragmentation.

It is suitable for applications that require low latency and can tolerate some performance degradation

## Garbage First Garbage Collector (G1 GC)

G1 Garbage Collector is the default garbage collection of Java 9.

This is a newer GC implementation that aims to provide high throughput and low latency.

It divides the heap into multiple regions of equal size and collects the regions with the most garbage first (garbage first). G1 also does compact the free heap space just after garbage collection that makes G1 Garbage Collector better than other garbage collectors.

It uses multiple threads for both young and old generations. It performs most of its work concurrently with application threads (concurrent), but it still stops all application threads briefly at the beginning and end of each GC cycle. It can achieve shorter and more predictable pause times than CMS GC, but it may require more memory and tuning.

It is suitable for applications that require both high throughput and low latency and have large heaps

## Z Garbage Collector (ZGC)

It is a scalable low-latency garbage collector.

_ZGC_ performs all expensive work concurrently, **without stopping the execution of application threads for more than 10 ms**, which makes it suitable for applications that require low latency.

It uses **load barriers with colored pointers** to keep track of object references and performs concurrent operations of relocation of objects without stopping application threads (concurrent). Reference coloring (colored pointers) is the core concept of _ZGC_. It means that _ZGC_ uses some bits (metadata bits) of reference to mark the state of the object.

Similar to _G1, Z Garbage Collector_ partitions the heap, except that heap regions can have different sizes. It **handles heaps ranging from 8MB to 16TB in size**. Furthermore, pause times don’t increase with the heap, live-set, or root-set size.


# GC Types Comparison

| Type     | Threads  | Stop-the-world | Concurrent   | Throughput | Latency   | Memory    | Fragmentation | Tuning    | JVM Arguments         `-XX:+{GC}` |
| -------- | -------- | -------------- | ------------ | ---------- | --------- | --------- | ------------- | --------- | --------------------------------- |
| Serial   | Single   | Yes            | No           | Low        | High      | Low       | Low           | Easy      | `UseSerialGC`                     |
| Parallel | Multiple | Yes            | No           | High       | High      | High      | Low           | Moderate  | `UseParallelGC`                   |
| CMS      | Multiple | Yes (briefly)  | Yes (mostly) | Moderate   | Low       | Moderate  | High          | Difficult | `UseConcMarkSweepGC`              |
| G1       | Multiple | Yes (briefly)  | Yes (mostly) | High       | Low       | High      | Low           | Moderate  | `UseG1GC`                         |
| ZGC      | Multiple | No             | Yes (fully)  | High       | Ultra-low | Very high | Low           | Easy      | `UseZGC`                          |


# GC Challenges

## Finding unused objects

An unused object is one that is no longer referred to by another object. The starting node of the graph is called a GC Root. There are several GC Roots, including classes loaded by the system class loader, live threads, local variables and parameters of currently executing methods, and objects used for synchronization. During the marking phase of collection, GC traverses all references starting with each GC root to find and mark all objects still in use. In the sweeping phase, GC removes unmarked objects. Since the graph is constantly changing, marking is often a Stop-The-World phase where only GC threads run.

## Fragmentation

Fragmentation occurs when the memory becomes sparse and the JVM must track free regions and decide where to allocate new objects. This can result in a premature OutOfMemoryError when there is no space for large contiguous blocks despite having enough free space. To prevent this, GCs can compact memory blocks into a single contiguous block using different algorithms. Fragmentation can also occur due to Local Allocation Buffers (PLABs and TLABs) in YoungGen (Eden) and OldGen (and YoungGen Survivors). TLABs divide Eden into chunks of various sizes for each thread to reserve, while PLABs avoid locking in OldGen and Survivors by preallocating memory regions for each thread. However, this can result in unused regions and fragmentation.
# GC Tuning

Tuning GCs is complex as each GC has its own settings. Generational GCs have MinorGC and MajorGC collections. MajorGCs can slow down or stop the JVM.

- Use a verbose GC log to find out the maximum memory footprint
- Use a profiler to identify memory leaks
- Use the right garbage collector for your application
- Adjust the heap size based on your memory needs
- Monitor the JVM performance regularly and adjust the GC parameters as needed
- Different GCs can be chosen for YoungGen and OldGen.