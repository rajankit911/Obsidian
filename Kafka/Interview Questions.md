>[!question] How to maintain the Order of messages in Kafka Topics

`max.in.flight.requests.per.connection=1`: This controls how may messages the producer will send to the server without receiving responses.

`in.flight.requests.per.session=1`: The ensures that additional message won't be sent while a message batch is retrying

# Single Broker / Single Partition Topic

Kafka maintains order within a single partition by assigning a unique offset to each message. This guarantees sequential message appending within that partition. This means that if messages were sent from the producer in a specific order, the broker will write them to a partition and all consumers will read from that in the same order. In essence, while a single partition guarantees order, it does so at the expense of reduced throughput and it is difficult to achieve parallelism and load balancing. So naturally, single-partition topic is easier to enforce ordering compared to its multiple-partition siblings.

## Producer and Consumer Timing Challenge

In a single partition, we process messages in the order they arrive at the broker. However, this order might not match the sequence in which we originally sent them. This mix-up can happen because of things like network latency or if we are resending a message. To keep things in line, we can implement producers with acknowledgments and retries. This way, we make sure that messages not only reach Kafka but also in the right order.

# Multiple Brokers / Multiple Partitions

## Challenges with Multiple Partitions

This distribution across partitions, while beneficial for scalability and fault tolerance, introduces the complexity of achieving global message ordering. For instance, we’re sending out two messages, M1 and M2, in that order. Kafka gets them just like we sent them, but it puts them in different partitions. Here’s the catch, just because M1 was sent first doesn’t mean it’ll be processed before M2. This can be challenging in scenarios where the order of processing is crucial, such as financial transactions.


- **Method 1: Round Robin or Spraying (Default)**
	In this method, the partitioner will send messages to all the partitions in a round-robin fashion, ensuring a balanced server load. Over loading of any partition will not happen. By this method parallelism and load balancing is achieved but it fails to maintain the overall order but the order within the partition will be maintained. This is a default method and it is not suitable for some business scenarios.
	



>[!example]

To illustrate, let’s create a topic (with three replicas and three partitions) with topic name as **“atm2”:**


```shell
kafka-topics.bat —-create —-zookeeper localhost:2181 —-topic atm2
—-replication-factor 3 —-partitions 3
```

Let’s publish the same sample dataset to our new topic _atm2_:

```shell
kafka-console-producer.bat — topic atm2 — bootstrap-server localhost:9092
> 0001,Robert,credit,10000
> 0002,Thomas,debit,20000
> 0003,Hari,credit,400000
> 0004,John,debit,300000
> 0005,peter,debit,398700
```

Let’s create a consumer that consumes the messages from Kafka:

```shell
kafka-console-consumer.bat –-bootstrap-server localhost:9092 —-topic atm2
—-from-beginning
```

There are three partitions (A, B & C). Partition B works fast because of low network and system latency, and the messages sent to it have been consumed first. Then comes Partition C, followed by A.

```shell
> 0004,John,debit,300000
> 0001,Robert,credit,10000
> 0003,Hari,credit,400000
> 0002,Thomas,debit,20000
> 0005,peter,debit,398700
```

If debit transaction happens before credit then it will be vague to the business users who consume the messages.

- **Method 2: Hashing Key Partition**
	In this method we can specify a message key to produce a record. The default partitioner will use the hash of the key to ensure that all messages for the same key go to same partition. This is the easiest and most common approach. It uses modulo operation for hashing.

```
Hash(Key) % Number of partitions -> Partition number
```

>[!example]

Let’s create a topic (with three replicas and three partitions) with topic name as _atm1_**:**

```shell
kafka-topics.bat —-create —-zookeeper localhost:2181 —-topic atm1 
—-replication-factor 3 —-partitions 3
```

Let’s publish a few messages to our new topic _atm1_ with a key-value for all the records:

```shell
kafka-console-producer.bat —-topic atm1 —-bootstrap-server localhost:9092 
—-property “parse.key=true” —-property “key.separator=:”

> 0001:0001,Robert,Credit,20000
> 0002:0002,Albert,Credit,5000
> 0001:0001,Robert,Debit,10000
> 0002:0002,Albert,Debit,3000
> 0001:0001,Robert,Credit,10000
```

>[!note]
>Simply sending lines of text will result in messages with null keys. In order to send messages with both keys and values we must set the `parse.key` and `key.separator` properties on the command line when running the producer.

Let’s consume the same message which we created above:

```shell
kafka-console-consumer.bat –-bootstrap-server localhost:9092 —-topic atm1
—-from-beginning
```


```shell
> 0002:0002,Albert,Credit,5000
> 0002:0002,Albert,Debit,3000
> 0001:0001,Robert,Credit,20000
> 0001:0001,Robert,Debit,10000
> 0001:0001,Robert,Credit,10000
```

As we can see above, the order of the messages within the keys is maintained.

The drawback with this method is as it uses random hashing value to pull the data to assigned partition, and it follows overloading of data to single partition.

3. **Method 3: Custom Partitioner**
	We can write our own business logic to decide which message need to be send to which partition. With this approach, we can make ordering of messages as per our business logic and achieve parallelism at the same time.


Here are the **key takeaways**:

1. One of the most important features of Kafka is to do load balancing of messages and guarantee ordering in a distributed cluster to achieve parallelism.
2. Using a higher number of partitions leads to higher throughput and latency. However, it may lead to maintenance overheads (not addressed in this article).
3. Using a hashing key partition, we can deliver messages with the same key in order, by sending it to the same partition. Data within a partition will be stored in the order in which it is written. Therefore, data read from a partition will be read in order for that partition with producer key.
4. Using a Custom Partitioner, we can route messages using arbitrary business rules.



### External Sequencing with Time Window Buffering

In this approach, the producer tags each message with a global sequence number. Multiple consumer instances consume messages concurrently from different partitions and use these sequence numbers to reorder messages, ensuring global order.

In a real-world scenario with multiple producers, **we will manage a global sequence by a shared resource that’s accessible across all producer processes, such as a database sequence or a distributed counter**. This ensures that the sequence numbers are unique and ordered across all messages, irrespective of which producer sends them:

```java
for (long sequenceNumber = 1; sequenceNumber <= 10 ; sequenceNumber++) {
    UserEvent userEvent = new UserEvent(UUID.randomUUID().toString());
    userEvent.setEventNanoTime(System.nanoTime());
    userEvent.setGlobalSequenceNumber(sequenceNumber);
    
    Future<RecordMetadata> future =
	    producer.send(new ProducerRecord<>(Config.MULTI_PARTITION_TOPIC,
		    sequenceNumber, userEvent));
		    
    sentUserEventList.add(userEvent);
    RecordMetadata metadata = future.get();
    logger.info("User Event ID: " + userEvent.getUserEventId()
	    + ", Partition : " + metadata.partition());
}
```

On the consumer side, we group the messages into time windows and then process them sequentially. Messages that arrive within a specific time frame we batch it together, and once the window elapses, we process the batch. This ensures that messages within that time frame are processed in order, even if they arrive at different times within the window. The consumer buffers messages and reorders them based on sequence numbers before processing. We need to ensure that messages are processed in the correct order, and for that, the consumer should have a buffer period where it polls for messages multiple times before processing the buffered messages and this buffer period is long enough to cope with potential message ordering issues:

```java
consumer.subscribe(Collections.singletonList(Config.MULTI_PARTITION_TOPIC));
List<UserEvent> buffer = new ArrayList<>();
long lastProcessedTime = System.nanoTime();
ConsumerRecords<Long, UserEvent> records =
	consumer.poll(TIMEOUT_WAIT_FOR_MESSAGES);

records.forEach(record -> {
    buffer.add(record.value());
});

while (!buffer.isEmpty()) {
    if (System.nanoTime() - lastProcessedTime > BUFFER_PERIOD_NS) {
        processBuffer(buffer, receivedUserEventList);
        lastProcessedTime = System.nanoTime();
    }
    records = consumer.poll(TIMEOUT_WAIT_FOR_MESSAGES);
    records.forEach(record -> {
        buffer.add(record.value());
    });
}

void processBuffer(List buffer, List receivedUserEventList) {
    Collections.sort(buffer);
    buffer.forEach(userEvent -> {
        receivedUserEventList.add(userEvent);
        logger.info("Processing message with Global Sequence number: "
			+ userEvent.getGlobalSequenceNumber() + ", User Event Id: "
			+ userEvent.getUserEventId());
    });
    buffer.clear();
}
```

Each event ID appears in the output alongside its corresponding partition, as shown below:

```bash
d6ef910f-2e65-410d-8b86-fa0fc69f2333, 0
4d6bfe60-7aad-4d1b-a536-cc735f649e1a, 4
9b68dcfe-a6c8-4cca-874d-cfdda6a93a8f, 4
84bd88f5-9609-4342-a7e5-d124679fa55a, 3
55c00440-84e0-4234-b8df-d474536e9357, 2
8fee6cac-7b8f-4da0-a317-ad38cc531a68, 1
d04c1268-25c1-41c8-9690-fec56397225d, 0
11ba8121-5809-4abf-9d9c-aa180330ac27, 4
8e00173c-b8e1-4cf7-ae8c-8a9e28cfa6b2, 4
e1acd392-db07-4325-8966-0f7c7a48e3d3, 3
```

Consumer output with global sequence numbers and event IDs:

```bash
1, d6ef910f-2e65-410d-8b86-fa0fc69f2333
2, 4d6bfe60-7aad-4d1b-a536-cc735f649e1a
3, 9b68dcfe-a6c8-4cca-874d-cfdda6a93a8f
4, 84bd88f5-9609-4342-a7e5-d124679fa55a
5, 55c00440-84e0-4234-b8df-d474536e9357
6, 8fee6cac-7b8f-4da0-a317-ad38cc531a68
7, d04c1268-25c1-41c8-9690-fec56397225d
8, 11ba8121-5809-4abf-9d9c-aa180330ac27
9, 8e00173c-b8e1-4cf7-ae8c-8a9e28cfa6b2
10, e1acd392-db07-4325-8966-0f7c7a48e3d3
```

### Considerations for External Sequencing with Buffering

In this approach, each consumer instance buffers messages and processes them in order based on their sequence numbers. However, there are a few considerations:

- Buffer Size: The buffer’s size can increase depending on the volume of incoming messages. In implementations that prioritize strict ordering by sequence numbers, we might see significant buffer growth, especially if there are delays in message delivery. For instance, if we process 100 messages per minute but suddenly receive 200 due to a delay, the buffer will grow unexpectedly. So we must manage the buffer size effectively and have strategies ready in case it exceeds the anticipated limit
- Latency: When we buffer messages, we’re essentially making them wait a bit before processing (introducing latency). On one hand, it helps us keep things orderly; on the other, it slows down the whole process. It’s all about finding the right balance between maintaining order and minimizing latency
- Failures: If consumers fail, we might lose the buffered messages, to prevent this, we might need to regularly save the state of our buffer
- Late Messages: Messages arriving post-processing of their window will be out of sequence. Depending on the use case, we might need strategies to handle or discard such messages
- State Management: If processing involves stateful operations, we’ll need mechanisms to manage and persist state across windows.
- Resource Utilization: Keeping a lot of messages in the buffer requires memory. We need to ensure that we have enough resources to handle this, especially if messages are staying in the buffer for longer periods.


# Idempotent Producers

Kafka’s idempotent producer feature aims to deliver messages precisely once, thus preventing any duplicates. This is crucial in scenarios where a producer might retry sending a message due to network errors or other transient failures. While the primary goal of idempotency is to prevent message duplication, it indirectly influences message ordering. Kafka achieves idempotency using two things a Producer ID (PID) and a sequence number which acts as the idempotency key and is unique within the context of a specific partition.

- **Sequence Numbers**:
	Kafka assigns sequence numbers to each message sent by the producer. These sequence numbers are unique per partition, ensuring that messages, when sent by the producer in a specific sequence, are written in that same order within a specific partition upon being received by Kafka. Sequence numbers guarantee order within a single partition. However, when producing messages to multiple partitions, there’s no global order guarantee across partitions. For example, if a producer sends messages M1, M2, and M3 to partitions P1, P2, and P3, respectively, each message receives a unique sequence number within its partition. However, it does not guarantee the relative consumption order across these partitions
- **Producer ID (PID)**:
	When enabling idempotency, the broker assigns a unique Producer ID (PID) to each producer. This PID, combined with the sequence number, enables Kafka to identify and discard any duplicate messages that result from producer retries

**Kafka guarantees message ordering by writing messages to partitions in the order they’re produced, thanks to sequence numbers, and prevents duplicates using the PID and the idempotency feature**. To enable the idempotent producer, we need to set the `enable.idempotence` property to true in the producer’s configuration:

```java
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");
```



## Key Configurations for Producer and Consumer

There are key configurations for Kafka producers and consumers that can influence message ordering and throughput.

### 4.1. Producer Configurations

- _MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION_: If we are sending a bunch of messages, then this setting in Kafka helps in deciding how many messages we can send without waiting for a ‘read’ receipt. If we set it higher than 1 without turning on idempotence, we might end up disturbing the order of our messages if we have to resend them. But, if we turn on idempotence, Kafka keeps messages in order, even if we send a bunch at once. For super strict order, like ensuring every message is read before the next one is sent, we should set this value to 1. If we want to prioritize speed and over perfect order, then we can set up to 5, but this can potentially introduce ordering issues.
- _BATCH_SIZE_CONFIG and LINGER_MS_CONFIG_: Kafka controls the default batch size in bytes, aiming to group records for the same partition into fewer requests for better performance. If we set this limit too low, we’ll be sending out lots of small groups, which can slow us down. But if we set it too high, it might not be the best use of our memory. Kafka can wait a bit before sending a group if it’s not full yet. This wait time is controlled by LINGER_MS_CONFIG. If more messages come in quickly enough to fill up our set limit, they go immediately, but if not, Kafka doesn’t keep waiting – it sends whatever we have when the time’s up. It’s like balancing speed and efficiency, making sure we’re sending just enough messages at a time without unnecessary delays.

```java
props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, "1");
props.put(ProducerConfig.BATCH_SIZE_CONFIG, "16384");
props.put(ProducerConfig.LINGER_MS_CONFIG, "5");
```

### 4.2. Consumer Configurations

- _MAX_POLL_RECORDS_CONFIG_: It’s the limit on how many records our Kafka consumer grabs each time it asks for data. If we set this number high, we can chow down on a lot of data at once, boosting our throughput. But there’s a catch – the more we take, the trickier it might be to keep everything in order. So, we need to find that sweet spot where we’re efficient but not overwhelmed
- _FETCH_MIN_BYTES_CONFIG_: If we set this number high, Kafka waits until it has enough data to meet our minimum bytes before sending it over. This can mean fewer trips (or fetches), which is great for efficiency. But if we’re in a hurry and want our data fast, we might set this number lower, so Kafka sends us whatever it has more quickly. For instance, if our consumer application is resource-intensive or needs to maintain strict message order, especially with multi-threading, a smaller batch might be beneficial
- _FETCH_MAX_WAIT_MS_CONFIG_: This will decide how long our consumer waits for Kafka to gather enough data to meet our FETCH_MIN_BYTES_CONFIG. If we set this time high, our consumer is willing to wait longer, potentially getting more data in one go. But if we’re in a rush, we set this lower, so our consumer gets data faster, even if it’s not as much. It’s a balancing act between waiting for a bigger haul and getting things moving quickly

```java
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, "500");
props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, "1");
props.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, "500");
```



### **Single Partition Message Ordering**[](https://www.baeldung.com/kafka-message-ordering#3-single-partition-message-ordering)

We create [topics](https://www.baeldung.com/kafka-topics-partitions) with the name _‘single_partition_topic’_, which has one partition, and _‘multi_partition_topic’,_ which has 5 partitions. Below is an example of a topic with a single partition, where the producer is sending a message to the topic:



```java
Properties producerProperties = new Properties();
producerProperties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, KAFKA_CONTAINER.getBootstrapServers());
producerProperties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, LongSerializer.class.getName());
producerProperties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JacksonSerializer.class.getName());
producer = new KafkaProducer<>(producerProperties);
for (long sequenceNumber = 1; sequenceNumber <= 10; sequenceNumber++) {
    UserEvent userEvent = new UserEvent(UUID.randomUUID().toString());
    userEvent.setGlobalSequenceNumber(sequenceNumber);
    userEvent.setEventNanoTime(System.nanoTime());
    ProducerRecord<Long, UserEvent> producerRecord = new ProducerRecord<>(Config.SINGLE_PARTITION_TOPIC, userEvent);
    Future<RecordMetadata> future = producer.send(producerRecord);
    sentUserEventList.add(userEvent);
    RecordMetadata metadata = future.get();
    logger.info("User Event ID: " + userEvent.getUserEventId() + ", Partition : " + metadata.partition());
}
```

_UserEvent_ is a POJO class that implements the _Comparable_ interface, helping in sorting the message class by _globalSequenceNumber_ (external sequence number). Since the producer is sending POJO message objects, we implemented a custom Jackson [Serializer](https://www.baeldung.com/jackson-custom-serialization) and [Deserializer](https://www.baeldung.com/jackson-deserialization).

Partition 0 receives all user events, and the event IDs appear in the following sequence:

```bash
841e593a-bca0-4dd7-9f32-35792ffc522e
9ef7b0c0-6272-4f9a-940d-37ef93c59646
0b09faef-2939-46f9-9c0a-637192e242c5
4158457a-73cc-4e65-957a-bf3f647d490a
fcf531b7-c427-4e80-90fd-b0a10bc096ca
23ed595c-2477-4475-a4f4-62f6cbb05c41
3a36fb33-0850-424c-81b1-dafe4dc3bb18
10bca2be-3f0e-40ef-bafc-eaf055b4ee26
d48dcd66-799c-4645-a977-fa68414ca7c9
7a70bfde-f659-4c34-ba75-9b43a5045d39
```

In Kafka, each consumer group operates as a distinct entity. If two consumers belong to different consumer groups, they both will receive all the messages on the topic**.** This is because **Kafka treats each consumer group as a separate subscriber**.

If two consumers belong to the same consumer group and subscribe to a topic with multiple partitions, **Kafka will ensure that** **each consumer reads from a unique set of partitions**. This is to allow concurrent processing of messages.

Kafka ensures that within a consumer group, no two consumers read the same message, thus each message is processed only once per group.


The code below is for a consumer consuming messages from the same topic:

```java
Properties consumerProperties = new Properties();
consumerProperties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, KAFKA_CONTAINER.getBootstrapServers());
consumerProperties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, LongDeserializer.class.getName());
consumerProperties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JacksonDeserializer.class.getName());
consumerProperties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
consumerProperties.put(Config.CONSUMER_VALUE_DESERIALIZER_SERIALIZED_CLASS, UserEvent.class);
consumerProperties.put(ConsumerConfig.GROUP_ID_CONFIG, "test-group");
consumer = new KafkaConsumer<>(consumerProperties);
consumer.subscribe(Collections.singletonList(Config.SINGLE_PARTITION_TOPIC));
ConsumerRecords<Long, UserEvent> records = consumer.poll(TIMEOUT_WAIT_FOR_MESSAGES);
records.forEach(record -> {
    UserEvent userEvent = record.value();
    receivedUserEventList.add(userEvent);
    logger.info("User Event ID: " + userEvent.getUserEventId());
});
```

In this case, we get the output which shows the consumer consuming messages in the same order, below are the sequential event IDs from the output:

```bash
841e593a-bca0-4dd7-9f32-35792ffc522e
9ef7b0c0-6272-4f9a-940d-37ef93c59646
0b09faef-2939-46f9-9c0a-637192e242c5
4158457a-73cc-4e65-957a-bf3f647d490a
fcf531b7-c427-4e80-90fd-b0a10bc096ca
23ed595c-2477-4475-a4f4-62f6cbb05c41
3a36fb33-0850-424c-81b1-dafe4dc3bb18
10bca2be-3f0e-40ef-bafc-eaf055b4ee26
d48dcd66-799c-4645-a977-fa68414ca7c9
7a70bfde-f659-4c34-ba75-9b43a5045d39
```

### **2.4. Multiple Partition Message Ordering**[](https://www.baeldung.com/kafka-message-ordering#4-multiple-partition-message-ordering)

For a topic with multiple partitions, the consumer and producer configurations are the same. The only difference is the topic and partitions where messages go, the producer sends messages to the topic ‘_multi_partition_topic’_:

```java
Future<RecordMetadata> future = producer.send(new ProducerRecord<>(Config.MULTI_PARTITION_TOPIC, sequenceNumber, userEvent));
sentUserEventList.add(userEvent);
RecordMetadata metadata = future.get();
logger.info("User Event ID: " + userEvent.getUserEventId() + ", Partition : " + metadata.partition());
```

The consumer consumes messages from the same topic:

```java
consumer.subscribe(Collections.singletonList(Config.MULTI_PARTITION_TOPIC));
ConsumerRecords<Long, UserEvent> records = consumer.poll(TIMEOUT_WAIT_FOR_MESSAGES);
records.forEach(record -> {
    UserEvent userEvent = record.value();
    receivedUserEventList.add(userEvent);
    logger.info("User Event ID: " + userEvent.getUserEventId());
});
```

The producer output lists event IDs alongside their respective partitions as below:

ADVERTISING

```bash
939c1760-140e-4d0c-baa6-3b1dd9833a7d, 0
47fdbe4b-e8c9-4b30-8efd-b9e97294bb74, 4
4566a4ec-cae9-4991-a8a2-d7c5a1b3864f, 4
4b061609-aae8-415f-94d7-ae20d4ef1ca9, 3
eb830eb9-e5e9-498f-8618-fb5d9fc519e4, 2
9f2a048f-eec1-4c68-bc33-c307eec7cace, 1
c300f25f-c85f-413c-836e-b9dcfbb444c1, 0
c82efad1-6287-42c6-8309-ae1d60e13f5e, 4
461380eb-4dd6-455c-9c92-ae58b0913954, 4
43bbe38a-5c9e-452b-be43-ebb26d58e782, 3
```

For the consumer, the output would show that the **consumer is not consuming messages in the same order.** The event IDs from the output are below:

```bash
939c1760-140e-4d0c-baa6-3b1dd9833a7d
47fdbe4b-e8c9-4b30-8efd-b9e97294bb74
4566a4ec-cae9-4991-a8a2-d7c5a1b3864f
c82efad1-6287-42c6-8309-ae1d60e13f5e
461380eb-4dd6-455c-9c92-ae58b0913954
eb830eb9-e5e9-498f-8618-fb5d9fc519e4
4b061609-aae8-415f-94d7-ae20d4ef1ca9
43bbe38a-5c9e-452b-be43-ebb26d58e782
c300f25f-c85f-413c-836e-b9dcfbb444c1
9f2a048f-eec1-4c68-bc33-c307eec7cace
```