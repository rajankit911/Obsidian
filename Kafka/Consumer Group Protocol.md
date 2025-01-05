Kafka separates storage from compute. Storage is handled by the brokers and compute is mainly handled by consumers or frameworks built on top of consumers (Kafka Streams, ksqlDB). Consumer groups play a key role in the effectiveness and scalability of Kafka consumers.


Let’s take a look at the steps involved in starting up a new consumer group.
- Find Group Coordinator
- Members Join
- Partitions Assigned

# Consumer Group Startup

## Step 1: Find Group Coordinator

![[find_group_coordinator.svg]]

- When a consumer instance starts up, it sends a `FindCoordinator` request that includes its `group.id` to any broker in the cluster.
- The broker will create a hash of the `group.id` and `modulo` that against the number of partitions in the internal `__consumer_offsets` topic. That determines the partition that all metadata events for this group will be written to.
- The broker that hosts the leader replica for that partition will take on the role of [[Consumer Groups#Consumer Group Coordinator|group coordinator]] for the new consumer group.
- The broker that received the `FindCoordinator` request will respond with the endpoint of the [[Consumer Groups#Consumer Group Coordinator|group coordinator]].

## Step 2: Members Join

![[join_group.svg]]

- Consumers send a `JoinGroup` request and passing their topic subscription information to the [[Consumer Groups#Consumer Group Coordinator|group coordinator]].
- The [[Consumer Groups#Consumer Group Coordinator|group coordinator]] will designate one consumer, usually the first one to send the `JoinGroup` request, as the [[Consumer Groups#Consumer Group Leader|group leader]].
- The [[Consumer Groups#Consumer Group Coordinator|group coordinator]] will return a `memberId` to each consumer, but it will also return a list of all members and the subscription info to the [[Consumer Groups#Consumer Group Leader|group leader]].
- The reason for this is so that the [[Consumer Groups#Consumer Group Leader|group leader]] can do the actual partition assignment using a configurable partition assignment strategy.

## Step 3: Partitions Assigned

![[sync_group.svg]]

- After the [[Consumer Groups#Consumer Group Leader|group leader]] receives the complete member list and subscription information, it will determines the partition assignments for all group members based on a **partition assignment strategy** (e.g., range, round-robin, or sticky).
- Once assignments are determined, the [[Consumer Groups#Consumer Group Leader|group leader]] will send a `SyncGroupRequest` to the coordinator, passing in its `memberId` and the group assignments provided by its partitioner. The other consumers will also make a similar `SyncGroupRequest` but will only pass their `memberId`.
- The [[Consumer Groups#Consumer Group Coordinator|group coordinator]] will use the assignment information given to it by the [[Consumer Groups#Consumer Group Leader|group leader]] to return the actual assignments to each consumer.
- The consumers can now begin their real work of consuming and processing data.
