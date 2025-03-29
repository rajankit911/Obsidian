# Range Partitioning

This strategy goes through each topic in the subscription and assigns each of the partitions to a consumer in a contiguous ranges, starting at the first consumer. This means that the first partition of each topic will be assigned to the first consumer, the second partition of each topic will be assigned to the second consumer, and so on.

![[range-partition.svg]]

>[!example]
>If two topics (`T1` and `T2`) each have two partitions (`P0`, `P1`), `T1-P0` and `T2-P0` might be assigned to one consumer, while `T1-P1` and `T2-P1` are assigned to another.

>[!info] Use Case
> At first glance this might not seem like a very good strategy, but it has a very special purpose. When joining events from more than one topic the events need to be read by the same consumer. If events in two different topics are using the same key, they will be in the same partition of their respective topics, and with the range partitioner, they will be assigned to the same consumer.

>[!warning] Limitations
>If a topic in the subscription has less partitions than the number of consumers, then some consumers will be idle.

# Round Robin Partitioning

With this strategy, all of the partitions of the subscription, regardless of topics, are treated as a single pool and distributed evenly among available consumers. This results in fewer idle consumer instances and a higher degree of parallelism.

![[round_robin_partition.svg]]

# Sticky Partitioning

Sticky Partition strategy is a variant of Round Robin. It operates on the same principle but it makes a best effort at sticking to the previous assignment during a rebalance. This reduces reassignment overhead during rebalances which results in a faster and more efficient rebalance.