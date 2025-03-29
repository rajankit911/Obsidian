- A **consumer group** is a collection of consumer instances that work together to consume and process messages from one or more Kafka topics.
- Each consumer group subscribes to one or more topics and divides the partitions among its members for parallel processing.

>Each Kafka topic is divided into a set of ordered partitions.

- A partition in Kafka is the basic unit of load distribution (parallelism); each partition is assigned to only one consumer within the same group.

>[!note]
>For a given consumer group, consumers can process more than one partition but a partition can only be processed by one consumer. If our group is subscribed to two topics and each one has two partitions then we can effectively use up to four consumers in the group. We could add a fifth but it would sit idle since partitions cannot be shared.

# Consumer Group ID

- A consumer group has a `group.id` and every consumer in that group will be assigned the same `group.id`.
- Once a `group.id` is set, every new instance of that consumer will be added to the group, and have the same `group.id`.

# Consumer Group Coordinator

>The **group coordinator** is the broker that hosts the leader replica of the `__consumer_offsets` partition assigned to a consumer group.

- It is on the server (broker) side and it is one of the brokers.
- Every consumer group has a **Group coordinator** and it is determined by the `group.id`.
- The Group coordinator helps to distribute the data in the subscribed topics to the consumer group instances evenly and it keeps things balanced when group membership changes occur.
- The Group coordinator uses an **internal Kafka topic** called `__consumer_offsets` to manage metadata for consumer groups. This topic stores:
    - Group membership information.
    - Partition assignments.
    - Offset tracking data for consumer groups.
- The **Group coordinator** receives heartbeats (or polling for messages) from all consumers of a consumer group. If a consumer stops sending heartbeats, the coordinator will trigger a rebalance.
- Kafka clusters have multiple group coordinators, ensuring distributed handling of different consumer groups.

## Group Coordinator Failover

So if the group coordinator fails, a broker that is hosting one of the follower replicas of `__consumer_offsets` partition will become the new group coordinator. Consumers will be notified of the new coordinator when they try to make a call to the old one, and then everything will continue as normal.

>[!info]
>A **group coordinator** is a special component within Kafka brokers responsible for :
> - Managing communication among all consumer group members.
> - Assigning partitions to consumer instances.
> - Triggering consumer group rebalancing.

# Consumer Group Leader

- It is on the client (consumer) side.
- Kafka uses a group leader to communicate with the Kafka broker and detect if there are changes such as a new partition to consume.

---

### **Understanding Kafka Consumer Groups**

Kafka Consumer Groups enable distributed data processing by allowing multiple consumer instances to process data in parallel. This feature is essential for scalability and load distribution across consumer applications. In Kafka, the separation of data storage from its processing is a key architectural feature, with consumer groups playing a critical role in scaling the processing tasks.





#### 3. **Internal Metadata Management**



---



#### 3. **Offset Management**

- Consumers commit their progress (last processed offset) to the group coordinator by sending `CommitOffsetRequest`, which persists this data in the `__consumer_offsets` topic.
- In case of consumer restarts:
    - The consumer retrieves its last committed offset and resumes processing from that point.
    - If no previous offset exists, the consumer can start from the earliest or latest offset.

When a consumer group instance is restarted, it will send an `OffsetFetchRequest` to the group coordinator to retrieve the last committed offset for its assigned partition. Once it has the offset, it will resume the consumption from that point. If this consumer instance is starting for the very first time and there is no saved offset position for this consumer group, then the `auto.offset.reset` configuration will determine whether it begins consuming from the earliest offset or the latest.



---



---



---

### **Offset Management and Recovery**

- **Offset Commit**:
    - Consumers commit offsets to the group coordinator for persistence in the `__consumer_offsets` topic.
- **Recovery**:
    - Consumers retrieve their last committed offset during startup to resume processing.
    - Failover is seamless due to replication of the internal offset topic.

---

### **Rebalancing Triggers and Process**

#### **Triggers**

- Consumer instance failure.
- Addition of a new consumer.
- Changes in topic partitions or matched subscriptions.

#### **Rebalance Workflow**

1. **Notification**:
    - Consumers are informed of rebalance via a heartbeat response.
2. **Join Group**:
    - Consumers rejoin the group and share their subscriptions.
3. **Partition Reassignment**:
    - The group leader determines and distributes new partition assignments.
4. **State Updates**:
    - Consumers update their state and resume processing.

---

### **Resiliency and Fault Tolerance**

- Group coordinator failure:
    - A new coordinator takes over using replicated metadata.
- Partition reassignment ensures continued processing during consumer failures.

---

### **Conclusion**

Kafka Consumer Groups are a powerful feature for scaling and distributing data processing tasks. With innovations like cooperative rebalancing and static group membership, Kafka ensures efficient, resilient, and scalable consumer group operations.


From the “[Kafka The Definitive Guide](https://www.confluent.io/wp-content/uploads/confluent-kafka-definitive-guide-complete.pdf)” [_Narkhede, Shapira & Palino, 2017_]:

> When a consumer wants to join a consumer group, it sends a `JoinGroup` request to the group coordinator. The first consumer to join the group becomes the group _leader_. The leader receives a list of all consumers in the group from the group coordinator (this will include all consumers that sent a heartbeat recently and are therefore considered alive) and it is responsible for assigning a subset of partitions to each consumer. It uses an implementation of the `PartitionAssignor` interface to decide which partitions should be handled by which consumer.
> 
> […] After deciding on the partition assignment, the consumer leader sends the list of assignments to the `GroupCoordinator` which sends this information to all the consumers. Each consumer only sees his own assignment – the leader is the only client process that has the full list of consumers in the group and their assignments. This process repeats every time a rebalance happens.

---

**[SO](https://stackoverflow.com/questions/39730126/difference-between-session-timeout-ms-and-max-poll-interval-ms-for-kafka-0-10/39759329#39759329) session.timeout.ms and max.poll.interval.ms** 

Advertisement

Before KIP-62, there is only `session.timeout.ms` (ie, Kafka `0.10.0` and earlier). `max.poll.interval.ms` is introduced via [KIP-62](https://cwiki.apache.org/confluence/display/KAFKA/KIP-62%3A+Allow+consumer+to+send+heartbeats+from+a+background+thread) (part of Kafka `0.10.1`).

KIP-62, decouples heartbeats from calls to `poll()` via a background heartbeat thread, allowing for a longer processing time (ie, time between two consecutive `poll()`) than heartbeat interval.

Assume processing a message takes 1 minute. If heartbeat and poll are coupled (ie, before KIP-62), you will need to set `session.timeout.ms` larger than 1 minute to prevent consumer to time out. However, if consumer dies, it also takes longer than 1 minute to detect the failed consumer.

KIP-62 decouples polling and heartbeat allowing to sent heartbeat between two consecutive polls. Now you have two threads running, the heartbeat thread and the _processing thread_ and thus, KIP-62 introduced a timeout for each. `session.timeout.ms` is for the heartbeat thread while `max.poll.interval.ms` is for the processing thread.

Assume, you set `session.timeout.ms=30000`, thus, the consumer heartbeat thread must sent a heartbeat to the broker before this time expires. On the other hand, if processing of a single message takes 1 minutes, you can set `max.poll.interval.ms` larger than one minute to give the processing thread more time to process a message.

If the processing thread dies, it takes `max.poll.interval.ms` to detect this. However, if the whole consumer dies (and a dying processing thread most likely crashes the whole consumer including the heartbeat thread), it takes only `session.timeout.ms` to detect it.

The idea is, to allow for a quick detection of a failing consumer even if processing itself takes quite long.

Assume your consumer dies (or there is a bug with an infinite loop), but the background thread keeps heartbeating. For this case, the would not be any progress but it would be undetected. Hence, `max.poll.interval.ms` is a heath check for your main processing thread — having both configs, allows you to detect “hard failures” (both heartbeat and main thread die) quickly, and simplify your code for long processing (with a single config you have either long detention time or complex code to trigger heartbeats during processing “manually”)