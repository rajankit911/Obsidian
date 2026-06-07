Assume we have three node system where all three nodes are trying to access the same piece of data. Since all of them might be doing some operations on this data in parallel so there are chances that if they don't synchronize in some fashion, they might end up corrupting this data. This data could be something stored on your database or it could be any shared resource.

![[distributed_locking.svg]]

There's a shared resource which multiple distributed nodes are trying to access at the same time and try to modify this resource they need to coordinate in some fashion.

Distributed locking is a technique using which you achieve this functionality. Without distributed locking we will see concurrency issues which will lead us to either race conditions or data inconsistencies and it might also lead into incorrect order of operations. 

For example, assume a scenario of flash sales where you have a limited inventory and a lot of users are trying to purchase the goods now if you don't have distributed locking in place then you might end up running into overselling Another example, assume a lot of people are editing Google Docs simultaneously and if you don't synchronize the order of operations that is going into the edits then you might end up overwriting other users data and the final update might not look what was intended. We should be able to handle it with the distributed locking that order of operations are maintained.

Let's understand the terminologies of locking and synchronization. Locking is actually a process of controlling access to the shared resource. Synchronization is a technique by which shared resource is accessed in a specific manner so that your final outcome is as expected.

# Mutex & Semaphore

In a single node system,  we have two options: 
- Mutex
- Semaphore

Difference between mutex and semaphore is mutex are generally used where one process can access a particular resource but semaphores are basically general implementation of mutex where you can allow n number of processes to access a particular resource.

Let's take a look at how distributed locking is different from single node locking. In a distributed system there will be multiple node and the processes running on these distributed nodes will be trying to access a shared resource and to restrict the access to the shared resource, we will be using distributed load managers and these distributed lock managers will ensure that only one process or one thread is able to access that shared resource.

# Distributed Lock Manager

Distributed lock manager needs to provide three functionalities:
- It should it should provide the mutual exclusion where only one process should be able to access resource at a time.
- It should not have deadlocks. If a node having a lock, crashes the system should not go into deadlock situation and eventually it should be able to recover and allow locking to other nodes.
- Since it is a distributed system, It should be fault tolerant.


There are few techniques which are typically used to implement the distributed locking:
- Redis
	It is one of the most commonly used approach of implementing the distributed lock manager. It provides an atomic operation of acquiring a distributed lock so effectively.
	
	`SET my-lock-demo process_1 EX 60 NX`
	
	SET \<key\> \<value\> (Set key to hold the string value)
	EX \<seconds\> (Set the specified expiry time, in seconds)
	NX (Only set the key if it does not already exist)
	
	Above Redis command will return true only if you can set a key. All the nodes which are contending for the lock, will try to set a key and only one will succeed. This ensure that only one process is able to acquire the lock while other processes will see it as a failure and they will retry and once the process which was able to successfully acquire the lock is done with its processing, it will release the lock on and other processes will contend to gain the access and the process continues.
	
	Having a TTL or an expiry for a key ensure that if a node has acquired a lock for a time duration longer than it is expected to take or node goes down , the key gets expired and the other processes can enter that critical section. 
	
	Let's assume that the lock was taken at time T and it is supposed to expire as a T + 60 and the process goes down at T + 1 that means although that process has gone down at T + 1, the system is now in deadlock position for next 59 seconds. After 60 seconds as the key expires the contention is removed so effectively the system is recoverable from deadlock scenario. It is a passive way of expiring the lock. This may work for the system where a lock is required for a smaller duration but  If there is a case where the lock is required for a longer duration in that case it can have some implications.
	
	There are some issues while using redis as a lock manager. One of the key property of distributed lock manager is highly availability, so when you are using a system like redis in HA mode means you are running two nodes right one is your master and other is your slave. When you write or update anything onto the master instance, redis asynchronously replicates the changes to the slave instance. Let's say some process was able to acquire the lock. While the data was about to get replicated, master instance of redis went down and replication did not happen, so when the slave becomes the master it will not have the information about that process which has acquired that lock and when the other processes try to take the lock, they will be able to acquire that lock. Now there will be two processes which would have successfully acquired the lock. Although it's a rare scenario but because of the inherent design of the redis HA, there are chances that multiple nodes will be able to acquire that lock simultaneously.
	
- Consensus Algorithms (Raft and Paxos)
	Consensus algorithms are commonly used in systems like Kafka or big distributed databases. These algorithms are tough to implement, so typically in normal applications people will not use it because there are chances that you might end up writing a lot of bugs while trying to implement these things so as an alternative to that people use systems which internally uses algorithms.
	
- [[Zookeeper Locks#Introduction| Apache Zookeeper]] / etcd / consul
	Systems like these which are distributed data stores but they are CP data stores which are consistent and partition tolerance so they will always ensures that all the nodes sees same copy of the data and thus and ensuring that there are no mismatch in the multiple copies of the data.
	
- MySQL
	Primary key can be used as a lock key and only the process which is able to create that key gets the lock. When a node which has taken the lock goes down it can cause deadlock in the system. For example, one process has acquired the lock from the distributed lock manager and after that it goes down. When other processes are trying to take a lock, distributed log manager will say that some other process has acquired the lock but that process does not exist. So effectively the system has reached in a deadlock scenario such that no other process is able to acquire the lock and you cannot continue here.


# LinkedIn Post


Distributed locking - when you need it, how to do it, and the traps that catch everyone: Distributed locks solve a real problem: ensuring only one node in a cluster performs a specific action at a time. Cron jobs, leader election, inventory reservations, anywhere "exactly once" matters across multiple processes. The naive approach: Redis SETNX SETNX (set if not exists). Simple. Mostly works. Has a critical flaw: if the process that acquired the lock crashes before releasing it, the lock is held forever. Fix: always set an expiry with the lock. SET key value NX EX 30. The expiry trap You set a 30-second expiry. Your process takes 45 seconds. The lock expires. Another node acquires it. Now two nodes are running the same job simultaneously. Fix: Redlock algorithm, or a fencing token, a monotonically increasing number the lock server issues with each grant. The protected resource validates the fencing token and rejects stale holders. Redlock: the controversial one Redis's own recommended algorithm for distributed locks across multiple Redis nodes. Acquires the lock on a majority of N nodes. Resistant to single node failure. Controversy: Martin Kleppmann argued it's unsafe under certain timing conditions. Redis's author disagreed. In practice: fine for most use cases. Not suitable if you need true linearizability guarantees. When to use ZooKeeper or etcd instead If you need strong consistency guarantees and can accept the operational overhead: ZooKeeper and etcd provide distributed locks with proper linearizability. Used for leader election in systems like Kafka and Kubernetes. The rule: use Redis-based locking for best-effort coordination. Use ZooKeeper/etcd when correctness is non-negotiable. Have you been burned by a distributed locking bug? What was the failure mode?








