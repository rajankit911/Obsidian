# Durability and Availability

## Replication

- Kafka ensures durability and availability through replication.
- If a topic has **N replicas**, it can tolerate **N-1 failures**. For mission-critical applications, configure higher replication for more fault tolerance.
- Only committed data is exposed to consumers.

## Producer Acknowledgement Modes
### `acks=0`

Fire-and-forget mode.
- **Pros**: Low latency.
- **Cons**: No durability guarantee; data can be lost before reaching the broker.

### `acks=1`

Acknowledgment is sent after the leader writes the data to its log.
- **Pros**: Improved durability.
- **Cons**: Data may still be lost during leader failures.

### `acks=all`

Acknowledgment is sent only after data is replicated to all in-sync replicas.
- **Pros**: Strongest durability guarantee, suitable for mission-critical applications.
- **Cons**: Higher latency (~2.5x of `acks=1`).

### `min.insync.replicas`

- Ensures that the data is acknowledged only if it is successfully replicated to a minimum number of replicas.
- If fewer replicas than configured are available, the broker rejects the write and sends an error to the producer.
- Prevents data loss in case of failures for mission-critical applications to strengthen durability guarantees.

# Ordering Guarantees

- Kafka ensures strong ordering from producers to consumers for events sent to the same partition.

## Challenges with Ordering During Failures

Failures can complicate ordering; During retries, duplicates and reordering can occur, disrupting guarantees.

>[!example]
>A producer sends messages (e.g., M1, M2, M3) to a leader. The leader crashes after partially replicating messages. The producer retries and resends all messages, causing:
>- **Duplicates**: Messages like M1 and M2 may appear twice.
>- **Reordering**: Order may differ (e.g., M2 before M1).

![[Excalidraw/Kafka/images/message_duplication.svg]]

## Idempotent Producers

- Solves duplicate and reordering issues during retries.
- Set `enable.idempotence=true` in the producer configuration.

 **Working:**
 
1. Producer requests a **Producer ID (PID)** from the broker.
2. Each message is tagged with a unique **Producer ID (PID)** and **sequence number** (unique for each message).
3. The broker uses the PID and sequence number to detect duplicates, ensuring correct ordering.

![[Excalidraw/Kafka/images/producer-idempotence.svg]]

## End-to-End Per-Key Ordering

- **Idempotence** + `acks=all` ensures strict end-to-end per-key ordering guarantees, essential for applications requiring consistent ordering.
- Messages with the same key land in the same partition and delivered to consumers in the same order they were sent.

# Key Features and Recommendations

- Kafka offers flexible configurations for durability, availability, and ordering:
	- **Replication** enhances fault tolerance.
	- **Producer acknowledgments (`acks`)** allow balancing latency and durability.
	- **Minimum In-Sync Replicas (`min.insync.replicas`)** ensures data safety.
	- **Idempotent Producers** eliminate duplicates and maintain ordering during retries.

- `acks=all` and **idempotent producers** ensure the strongest durability and ordering guarantees.
- **Per-key ordering** guarantees are critical for applications requiring strict consistency.