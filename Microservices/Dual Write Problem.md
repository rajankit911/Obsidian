
## What is the dual-write problem?

The  **dual write problem**  occurs when your service needs to write to two external systems in an atomic fashion. A common example would be writing state to a database and publishing an event to Apache Kafka. The separate systems prevent you from using a transaction, and as a result, if one write fails it can leave the other in an inconsistent state. This is an easy trap to fall into.

There are valid options for avoiding the dual write problem, such as a  **transactional outbox**,  **event sourcing**, and  **change data capture**. However, we have to be careful to avoid solutions that might look promising, but don’t solve the problem.
Topics:

-   Emitting Events
-   The Dual Write Problem
-   Invalid Solution: Emit the Event First
-   Invalid Solution: Use a Transaction
-   Change Data Capture (CDC)
-   The Transactional Outbox Pattern
-   Event Sourcing
-   The Listen to Yourself Pattern



## Understanding the dual-write problem

Consider a simplified banking system. When a user deposits money into a bank account, they issue a  _DepositFunds_  command.

The command goes to an  _Account_  microservice where it is processed as required.

The microservice has to do two things. First, it updates the database to reflect the funds being deposited. Second, it emits a  _FundsDeposited_  event to Kafka with details about the transaction.

![Figure 1: A microservice writes to two separate systems.](https://images.ctfassets.net/8vofjvai1hpv/2pAIwVR31SG3kWKhr2d1pj/70208223b306b96dfd3c14f0b76a6839/1.png)Figure 1: A microservice writes to two separate systems.

This seems relatively simple, but there is a hidden issue. The problem occurs between the save to the database and the event being emitted.

If a failure occurs after the data has been saved but before the event makes it to Kafka, the system can end up in an inconsistent state. The database knows the funds were deposited, but the corresponding event has been lost.

We are using a microservice and Apache Kafka purely for the example. This problem can occur in a monolithic system using a variety of technologies. For example, a monolith that tries to update the database and send an email would also be subject to the dual-write problem. It occurs anytime you try to write to two separate systems and only one of those writes succeeds.

## Anti-patterns

In most systems, the inconsistency created by the dual-write problem is undesirable, and we must deal with it. But the solutions aren’t as obvious as they might seem. Many approaches that are used in production to avoid the issue just move it.

### Order of operations

Perhaps the simplest idea is to reverse the order of operations. What if we write to Kafka first, and then only save to the database when that succeeds? That way, if the event isn’t emitted, we don’t update the database.

Of course, the problem with this solution should be fairly obvious. What happens if we successfully emit the event and then fail to save it to the database? In that case, we still have an inconsistency; it’s just different from the one we had before. We have an event with no corresponding update in the database.

### Database transactions

Reordering the operations won’t work because we need them to be atomic, no matter what order they occur. Either both operations succeed, or they both fail. In-between states are unacceptable.

When atomicity is the goal, the natural instinct is to reach for a transaction. They are designed to prevent these inconsistencies. So, can a database transaction be used to solve this problem?

We could wrap both the database write and the Kafka write in a single database transaction. Would that solve the problem?

![Figure 4: Does a database transaction solve the dual-write problem?](https://images.ctfassets.net/8vofjvai1hpv/5947ZdOee28dmg5zNmyRcr/28c3d22701a39ad892ee0b22d7aa5584/4.png)Figure 4: Does a database transaction solve the dual-write problem?

Unfortunately, it won’t. Database transactions are usually scoped to the database. They ensure that two  _database_  operations occur in an atomic fashion. However, they don’t usually extend to external systems such as Kafka (or an email server, a secondary database, etc.). As a result, wrapping the calls in a database transaction doesn’t solve the problem; it just moves it.

So what happens when we use a database transaction?

If a failure occurs between the save and emit statements, the transaction won’t be committed and will get rolled back. Neither Kafka nor the database will have a record of the transaction, eliminating the inconsistency.

However, if the failure occurs after the event has been emitted but before the transaction can be committed, the database transaction is still rolled back. Unfortunately, the event is already emitted and isn’t included in the rollback. Once again, we end up in a state where we have an event with no corresponding database update.

Unfortunately, database transactions, on their own, are not a solution for the dual-write problem. However, as we will see, they can be part of a larger solution.

### Retries

What if we added a retry mechanism? When the write to Kafka fails, it can be retried until it succeeds. However, this assumes the system has a record of what to retry. If the problem is a temporary network error, then the event might still be stored in memory, and a retry could be viable.

![Figure 7: Retrying the event can ensure it gets emitted in some situations.](https://images.ctfassets.net/8vofjvai1hpv/6eV6depd5iL6BYO2PNoSDi/2b4df6f519c7431c4aa4d51ad383a540/7.png)Figure 7: Retrying the event can ensure it gets emitted in some situations.

But, what if the entire application crashed and the memory has been purged? In that case, where do we get the event from?

For a retry to work, it has to be reliable. That means the event would have to be written to durable storage so it can be retrieved if the system goes down. But writing the event to durable storage introduces another write, which is exactly the problem we were trying to overcome.

So, a simple in-memory retry won’t solve the problem, and a durable retry suffers from the dual-write problem. Although retries are part of the solution, they need help to make them work.

## Solving the dual-write problem

We’ve looked at anti-patterns for solving the dual-write problem, but now we need valid solutions.

The key is to separate the two writes and introduce a dependency between them. The first write is performed independently from the second. It then triggers a separate process that continually retries the second until it succeeds. If the first write fails, the second is never triggered. And once the first succeeds, the second will eventually follow.

### The transactional outbox pattern

The  [transactional outbox pattern](https://developer.confluent.io/courses/microservices/the-transactional-outbox-pattern/)  is a common way to solve the dual-write problem, and interestingly, it leverages both database transactions and a retry mechanism.

To implement this pattern, an outbox table is created in the database. The table contains events that need to be delivered to Kafka (or any other destination you have in mind).

When the  _DepositFunds_  command is issued, the database is updated as usual, and in the same transaction, a  _FundsDeposited_  event is written to the outbox table. Because both writes go to the same database, a transaction can be used to guarantee atomicity. This ensures every database update has a corresponding event in the outbox table.

However, the events still need to make their way into Kafka. This involves a separate process that reads events from the outbox table and emits them to Kafka, retrying if there is a failure. Once an event has been safely written to Kafka, it can be marked as complete or deleted from the outbox table.

The outbox processor can be handwritten code, but if the database supports it, we can leverage change data capture (CDC) tools to do the work instead.

![Figure 9: The transactional outbox pattern.
](https://images.ctfassets.net/8vofjvai1hpv/4LW0GfnUfLwtvV2GIeTCBO/afa9a9768c34f3ce829b6f6a52b01717/9.png)Figure 9: The transactional outbox pattern.

### Event sourcing

The transactional outbox pattern works well for solving the dual-write problem but is restricted to transactional databases. If the system is using storage that doesn’t support transactions, it may not work.

An alternative pattern that doesn’t rely on transactions is  [event sourcing](https://developer.confluent.io/courses/microservices/event-sourcing/). Event sourcing is a little like the transactional outbox, except it eliminates the storage of the state. Each time the  _DepositFunds_  command is issued, a  _FundsDeposited_  event is written to a table in the database. A transaction is unnecessary because the event is written to a single row in a single table.

When the application needs access to the state, it reads the events back and uses them to rebuild it.

This model works well for banking because accounts are naturally event-sourced. Banks keep a full history of every deposit and withdrawal. Each of those transactions represents an event in the system.

Much like with the transactional outbox pattern, a separate process (or CDC) reads the events and emits them to Kafka. However, with event sourcing, the events are never deleted. Instead, a flag would indicate if the event was published.

![Figure 10: Event sourcing.](https://images.ctfassets.net/8vofjvai1hpv/3NYBDNNebvpIwKZ1VXy88t/1c7ba3a578ef82590aa4c013de36e3de/10.png)Figure 10: Event sourcing.

### The listen-to-yourself pattern

The  [listen-to-yourself pattern](https://developer.confluent.io/courses/microservices/the-listen-to-yourself-pattern/)  is similar to event sourcing because both focus on writing the event first. However, where event sourcing would write the event to a database and publish it to Kafka later, the listen-to-yourself pattern writes directly to Kafka.

When a command arrives, it is converted to the corresponding event and written immediately to Kafka, without updating the database. A separate process can listen to these events and use them to update the database, retrying as necessary.

This does have consequences. Unlike with the transactional outbox pattern, or event sourcing, updates to the database are eventually consistent. There is a period where the command has finished processing but the database hasn’t been updated. Asking for the state at that moment could return unexpected results.

However, if the application can tolerate eventual consistency, this approach can be fast and effective. Like the other solutions, it solves the dual-write problem by breaking the writes into separate processes. The advantage is that it allows the first write, and the response to the command, to happen extremely fast because it defers most of the processing.

![Figure 11: The listen-to-yourself pattern.](https://images.ctfassets.net/8vofjvai1hpv/16tEtI8sHNeQmB62rjkLzT/4453a8e2913db431777b433612a4ff5a/11.png)Figure 11: The listen-to-yourself pattern.

### Other solutions

There are other solutions to the dual-write problem that we haven’t covered here. The intent is not to suggest that these are the only ways to solve the problem, but rather to highlight specific solutions that are particularly useful in an event-driven system.

Techniques such as two-phase commit (2PC), extended architecture (XA) transactions, and the saga pattern can be used to circumvent the issue in some cases. However, they come with complexities and may not work with all technologies, so make sure you understand the trade-offs before implementing them.
