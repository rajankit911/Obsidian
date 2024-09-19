# Features of Kafka

- **Kafka is highly scalable**. It is very easy to add large number of consumers without affecting performance or reliability. That’s because Kafka does not track which messages in the topic have been consumed by consumers. It simply keeps all messages in the topic within a configurable period. Kafka is linearly scalable for high volume of data. Adding more brokers / clusters will increase the throughput or decrease latency.

- **Message durability is high in Kafka.** It persists the messages/events for the specified time period.

- **Kafka handles spike of the events more efficiently**. This is where Kafka truly shines because it acts as a “**_shock absorber_**” between the producers and consumers.

- **Kafka also supports different consumption models**. You can have one consumer processing the messages **at real-time** and another consumer processing the messages in **batch mode** in a totally decoupled fashion.

- **Kafka supports the ability to source messages from a wide range of producers.**

- **Kafka supports excellent integration with other processing frameworks.** These include as _Apache Storm, Spark, NiFi, Flume_ etc. to complete the job.

```mermaid
erDiagram
	Clustor ||--o{ Broker : has
	Broker ||--o{ Topic : has
	Topic ||--o{ Partition : has
	Partition ||--o{ Offset : has
```

- **Broker:** The server (physical or virtual) that holds the queue.
- **Topic:** A stream of messages belonging to a particular category is called a topic. Data is stored in topics.
- **Partition:** Topics are split into partitions. For each topic, Kafka keeps a minimum of one partition. Each such partition contains messages in an immutable ordered sequence. A partition is implemented as a set of segment files of equal sizes.
- **Partition offset:** Each partitioned message has a unique sequence id called as offset.




# Partitions

In the last part we had a brief introduction of topics that they’re addressable abstraction used to demonstrate interest in a specific data stream (series of records/messages) and that it consists of some number of partitions which are a series of order queues.

![](https://miro.medium.com/v2/resize:fit:832/0*yE6DPE4fgbxfCSD0.png)

Topic with 3 partitions

## Why do we need partitions?

We have previously created a topic with only a single partition and it served our purpose. Then the question arises : why are partitions even needed?

You must partition your data if you have so much traffic that more than one instance of your application is required. For the downstream application, your partitioning strategy acts as load balancing. The producer clients determine which topic partition the data is placed in, while the decision logic is driven by the consumer apps’ actions with the data.

We also get the following benefits with partitioning:

- Multiple consumers can consume a single topic at the same time. The number of consumers that a single broker can sustain is limited because it serves all partitions. Multiple brokers’ partitions allow for more consumers.
- Multiple instances of the same consumer can connect to separate brokers’ partitions, allowing for extremely high message throughput. One partition will serve each consumer instance, ensuring that each record has a clear processing owner.
- Across several brokers, Kafka retains multiple copies of the same partition. A replica is the term for this duplicate copy. If a broker fails, Kafka can still provide consumers with copies of the partitions that the failed broker was responsible for.

## Writing to partitions

Be default, Kafka will assign partitions in a round-robin fashion. Those records will be evenly distributed across all partitions of a given topic.  
However, if no partition key is provided, the ordering of records within a partition cannot be ensured.

As a result, utilizing a partition key to group related events in the same partition in the order they were sent appears to be a better alternative.

A partition key allows a producer to direct messages to a certain partition. Any value that may be obtained from the application context can be used as a partition key. A suitable partition key is a unique device ID or a user ID.  
The partition key is passed via a hashing function by default, and the partition assignment is created. As a result, all recordings made with the same key will arrive in the same partition.

If the keys aren’t evenly distributed, key-based partitioning can result in broker imbalance. That means one of the partitions will have a lot more values than the other if its key is getting almost all the traffic. This is why ensuring that the keys are evenly distributed is highly advised.

In some cases, producer can write their own partitioning implementation based on any business logic to direct the messages accordingly to the right partition.

# Consumer Group

A consumer group is a collection of consumers who work together to consume data on a specific topic. All of the topics’ partitions are divided among the group’s consumers.

Kafka’s consumer groups allow it to take advantage of both message queuing and publish-subscribe architectures. A group id is shared by Kafka consumers who belong to the same consumer group. The consumers in a group then distribute the topic partitions as evenly as feasible amongst themselves by ensuring that each partition is consumed by just one consumer from the group.

![](https://miro.medium.com/v2/resize:fit:948/0*IjONT3bMjFrrO7P0.png)

Different consumers consuming partitions exclusively in a consumer group

A message is only ever read by one customer in a consumer group, owing to the consumer group idea.  
When a consumer group consumes a topic’s partitions, Kafka ensures that each partition is consumed by exactly one consumer.

## How it satisfies both messaging models

If all consumers are from the same group, the Kafka model functions as a traditional message queue would. All the records and processing is then load balanced Each message would be consumed by one consumer of the group only. Each partition is connected to at most one consumer from a group.

When multiple consumer groups exist, the flow of the data consumption model aligns with the traditional publish-subscribe model. The messages are broadcast to all consumer groups.

# Offset Management

The offset of a message acts as a consumer-side cursor. By keeping track of the offset of messages, the consumer keeps track of which messages it has already consumed. The consumer advances its cursor to the next offset in the partition and continues after reading a message. The consumer is responsible for advancing and remembering the latest read offset within a partition.

> The “offset” is a type of metadata in Kafka that represents the position of a message in a certain partition. Each message in a partition has its own unique offset value, which is represented by an integer.

Kafka maintains two types of offsets.

- **Current Offset** : The current offset is a reference to the most recent record that Kafka has already provided to a consumer. As a result of the current offset, the consumer does not receive the same record twice.
- **Committed Offset** : The committed offset is a pointer to the last record that a consumer has successfully processed. We work with the committed offset in case of any failure in application or replaying from a certain point in event stream. When it comes to partition rebalancing, the committed offset is crucial. If there is a need for rebalancing.

## Committing an offset

When a consumer in a group receives messages from the partitions assigned by the coordinator, it must commit the offsets corresponding to the messages read. If a consumer crashes or shuts down, its partitions will be reassigned to another member, who will start consuming each partition from the previous committed offset. If the consumer fails before the offset is committed, then the consumer which takes over its partitions will use the reset policy.

There are two ways to commit an offset:

- **Auto Commit** : By default, the consumer is configured to use an automatic commit policy, which triggers a commit on a periodic interval. This feature is controlled by setting two properties:

1. _enable.auto.commit_
2. _auto.commit.interval.ms_

Although auto-commit is a helpful feature, it may result in duplicate data being processed.

Let’s have a look at an example.

You’ve got some messages in the partition, and you’ve requested your first poll. Because you received ten messages, the consumer raises the current offset to ten. You process these ten messages and initiate a new call in four seconds. Since five seconds have not passed yet, the consumer will not commit the offset. Then again, you’ve got a new batch of records, and rebalancing has been triggered for some reason. The first ten records have already been processed, but nothing has yet been committed. Right? The rebalancing process has begun. As a result, the partition is assigned to a different consumer. Because we don’t have a committed offset, the new partition owner should begin reading from the beginning and process the first ten entries all over again.

A manual commit is the solution to this particular situation. As a result, we may turn off auto-commit and manually commit the records after processing them.

- **Manual Commit** : With Manual Commits, you take the control in your hands as to what offset you’ll commit and when. You can enable manual commit by setting the _enable.auto.commit_ property to _false_.

There are two ways to implement manual commits :

1. Commit Sync : The synchronous commit method is simple and dependable, but it is a blocking mechanism. It will pause your call while it completes a commit process, and if there are any recoverable mistakes, it will retry. Kafka Consumer API provides this as a prebuilt method. Its documentation can be found [here](https://kafka.apache.org/25/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html#commitSync--).
2. Commit Async : The request will be sent and the process will continue if you use asynchronous commit. The disadvantage is that commitAsync does not attempt to retry. However, there is a legitimate justification for such behavior.

Let’s have a look at an example.

Assume you’re attempting to commit an offset as 70. It failed for whatever reason that can be fixed, and you wish to try again in a few seconds. Because this was an asynchronous request, you launched another commit without realizing your prior commit was still waiting. It’s time to commit-100 this time. Commit-100 is successful, however commit-75 is awaiting a retry. Now how would we handle this? Since you don’t want an older offset to be committed.

This could cause issues. As a result, they created asynchronous commit to avoid retrying. This behavior, however, is unproblematic since you know that if one commit fails for a reason that can be recovered, the following higher level commit will succeed.

This way of committing is also provided by the Kafka Consumer API and you can find out more about it [here](https://kafka.apache.org/25/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html#commitAsync--).

Manual committing is also useful in case you want to replay and process records from a certain point in the past. For that to work, you actually commit an old offset over a recent one.
