
The broker maintains the offset consumed by each consumers for each topic for each partition

##### **Offset in Kafka**

The offset is a unique ID assigned to the partitions, which contains messages. The most important use is that it identifies the messages through ID, which are available in the partitions. In other words, it is a position within a partition for the next message to be sent to a consumer. A simple integer number which is actually used by Kafka to maintain the current position of a consumer. Kafka maintains two types of offsets, the current and committed offset.

##### **Current Offset**

Let’s first understand the current offset. Kafka sends some messages to us when we call a poll method. It is a pointer to the last record that Kafka has already sent to a consumer in the most recent poll. So, the consumer doesn’t get the same record twice and this is just because of the current offset.

##### **Committed Offset**

Committed offset, the position that a consumer has confirmed about processing. In simple language, after receiving a list of messages, we want to process it. This process might be just storing them into Apache Hadoop Distributed File System (HDFS). Once we get the assurance that it has successfully processed the record, we may want to commit the offset. So, the committed offset is a pointer to the last record that has processed successfully.


##### **Overview of Offset Management**

A Kafka topic receives messages across a distributed set of partitions where they are stored. Each partition maintains the messages it has received in a sequential order where they are identified by an offset, also known as a position. Developers can take advantage of using offsets in their application to control the position of where their Spark Streaming job reads from, but it does require off management.

Managing offsets is very beneficial to achieve data continuity over the lifecycle of the streaming process. For example, upon shutting down the stream application or during an unexpected failure, offset ranges will be lost unless persisted in a non-volatile data store.

##### **Approaches**

Storing offsets in external data stores.

- Checkpoints
- HBase
- ZooKeeper
- Kafka

Any external durable data store such as HBase, Kafka, HDFS, and ZooKeeper is used to keep track of which messages have already been processed.

It is worth mentioning that you can also store offsets in a storage system like HDFS. Storing it in HDFS is a less popular approach compared to the above options as HDFS has a higher latency compared to other systems like ZooKeeper and HBase. Additionally, writing offset ranges for each batch in HDFS can lead to the problem of a small file if not managed properly.



In Apache Kafka, offsets are used to keep track of the position of consumers in the log of messages for each topic partition they read from. Here's how offsets are stored and managed by the Kafka broker:

### Offset Management

1. **Kafka Consumer Groups:**
   - Each consumer belongs to a consumer group. Offsets are maintained per consumer group and per partition.
   - A consumer group is identified by a unique group ID. Within a group, each partition is assigned to exactly one consumer.

2. **Offset Storage:**
   - Offsets can be stored in Kafka itself or externally. By default, Kafka stores offsets in a special internal topic named `__consumer_offsets`.
   - The `__consumer_offsets` topic is partitioned and replicated like any other Kafka topic, providing fault tolerance and scalability.

3. **Commit Types:**
   - **Automatic Commit:** Consumers can automatically commit offsets at a specified interval using the `enable.auto.commit` and `auto.commit.interval.ms` configuration parameters.
   - **Manual Commit:** Consumers can manually commit offsets by calling `commitSync()` or `commitAsync()` methods. This gives more control over when offsets are committed.

### Offset Management Process

1. **Consuming Messages:**
   - When a consumer reads a message from a partition, it receives the offset of that message.
   - The consumer processes the message and decides when to commit the offset.

2. **Committing Offsets:**
   - **Automatic Commit:** If enabled, the consumer periodically commits the latest processed offset.
   - **Manual Commit:** The consumer can commit the offset after processing the message to ensure that it doesn't lose track of its progress in case of a failure.

3. **Storing Offsets in `__consumer_offsets`:**
   - The offset commit request is sent to the Kafka broker, which stores it in the `__consumer_offsets` topic.
   - The offsets are stored as key-value pairs where the key is a combination of the consumer group, topic, and partition, and the value is the offset.

4. **Fetching Offsets:**
   - When a consumer restarts or is assigned a partition, it fetches the last committed offset from the `__consumer_offsets` topic to resume consuming from the correct position.

### Benefits of Kafka's Offset Management

1. **Fault Tolerance:**
   - Since offsets are stored in Kafka itself, they are replicated and persisted, providing fault tolerance.
   
2. **Scalability:**
   - The `__consumer_offsets` topic is partitioned, allowing it to scale with the number of consumer groups and partitions.

3. **Flexibility:**
   - Consumers can choose between automatic and manual offset management, providing flexibility in handling offsets according to application requirements.

By managing offsets in this way, Kafka ensures that consumers can reliably track their progress in reading messages, even in the face of consumer failures or restarts.