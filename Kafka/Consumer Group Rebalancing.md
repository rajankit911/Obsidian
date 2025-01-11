>One of the key features of consumer groups is ***rebalancing***.

A **rebalance** is triggered when:
- Initial consumer group startup.
- A consumer instance has been added to the consumer group.
- A consumer instance fails to send a heartbeat to the coordinator before the timeout and is removed from the group.
- Topic partitions in the group’s subscription are modified (e.g., new partitions are added).
- A consumer group has a wildcard subscription and a new matching topic is created or a matching topic is deleted.

# Rebalance Notification

![[rebalance_notification.svg]]

The rebalance process begins with the coordinator notifying the consumer instances that a rebalance has begun. It does this by piggybacking on the `HeartbeatResponse` or the `OffsetFetchResponse`.


# Traditional Rebalance: Stop-the-World

Once the consumers receive the rebalance notification from the coordinator, they will revoke their current partition assignments. If they have been maintaining any state based on the events in their previously assigned partitions, they will also have to clean that up. Now they are basically like new consumers and will go through the same steps as a new consumer joining the group.

They will send a `JoinGroupRequest` to the coordinator, followed by a `SyncGroupRequest`. The coordinator will respond accordingly, and the consumers will each have their new assignments.

Any state that is required by the consumer would now have to be rebuilt from the data in the newly assigned partitions.

## Problem #1 – Rebuilding State

- During a rebalance, consumers revoke their current partitions and clear state, even if some partitions are reassigned back to the same consumer.
- This results in unnecessary overhead of rebuilding the state.

![[Excalidraw/Kafka/images/rebalance_issue_1.svg]]

Since p0 and p1 are assigned to the same consumer instance, rebuilding the state is unnecessary.


## Problem #2 – Paused Processing

- All consumers stop processing during the rebalance, hence the name **"Stop-the-world"**.
- Processing resumes only after new assignments are received, causing downtime.

![[rebalance_issue_2.svg]]

Since consumer 1 & 2 are assigned to the the same partitions, they could have, in theory, continued working with them while the rebalance was underway.

Let’s see some of the improvements that have been made to deal with these problems.

# Improvements in Rebalance Mechanisms

## Sticky Partition Assignor

Using `StickyAssignor` the state cleanup process is moved to a later step, after the reassignments are complete. That way if a consumer is reassigned the same partition it retains the state and can just continue with its work without clearing or rebuilding the state.

>[!note] In our example, state would only need to be rebuilt for partition p2, which is assigned to the new consumer.

## Cooperative Sticky Partition Assignor

>`CooperativeStickyAssignor` works in a two-step process.

![[cooperative_sticky_assignor.svg]]

1. In the first rebalance step, the determination is made as to which partition assignments need to be revoked. Those assignments are revoked at the end of the first rebalance step. The partitions that are not revoked can continue to be processed.

2. In the second rebalance step, the revoked partitions will be assigned. In our example, partition 2 was the only one revoked and it is assigned to the new consumer 3. In a more involved system, all of the consumers might have new partition assignments, but the fact remains that any partitions that did not need to move can continue to be processed without the world grinding to a halt.

>[!todo] Advantages
>- Consumers do not immediately revoke partitions.
>- Instead, partitions are revoked and reassigned incrementally, allowing processing to continue for non-revoked partitions.

## Static Group Membership

- Each consumer instance is assigned a `group.instance.id`.
- When a consumer instance leaves gracefully it will not send a `LeaveGroup` request to the coordinator, so no rebalance is started.
- When the same instance rejoins the group, the coordinator will recognize it and allow it to continue with its existing partition assignments. Again, no rebalance needed.
- No rebalancing if a consumer restarts prior to `session.timeout.ms`, it will be able to continue with its existing assignments.

>[!quote]
>The fastest rebalance is the one that doesn’t happen.

>[!info] Advantages
> Reduces disruption and state rebuilding for transient restarts.
