# Event Sourcing

 Event Sourcing is a pattern of storing an object's state as a series of events. Each time the object is updated a new event is written to an append-only log. When the object is loaded from the database, the events are replayed in order, reapplying the necessary changes. The benefit of this approach is that it stores a full history of the object. This can be valuable for debugging, auditing, building new models, and a variety of other situations. It is also a technique that can be used to solve the dual-write problem when working with event-driven architectures.

Topics:

- [] How are objects stored in traditional databases?
- [] What is an audit log, and why is it useful?
- [] Do audit logs create duplicated data?
- [] Should the audit log be used as the source of truth?
- [] What is event sourcing?
- [] What are some advantages of event sourcing?
- [] Does event sourcing solve the dual-write problem?
- [] What are some disadvantages of event sourcing?

Hi, I'm Wade from Confluent.

Traditional database architecture can be thought of as a road trip.

You might take many twists and turns on your journey but where you are now is all that matters.

Except, it isn't.

The journey is often just as important, or more important than the destination.

But how does that translate into a database architecture?

Traditionally, when we store data in a database, we do so in a destructive fashion.

Each time we update a record,

we destroy whatever data was there previously.

This means that we lose the history of events that led to the current state.

But what if we didn't want to lose the history?

What if we were interested not just in where we are, but also in how we got there?

Have you ever investigated a bug, only to discover that the current state doesn't contain enough information for you to diagnose or fix the problem?

You need to know what specific changes a user made to arrive at that state.

If you've seen this kind of situation before, let me know in the comments.

One way to solve this would be to persist an audit log along with the state.

Each time we update a record in the database,

we can write an audit event to another table.

These events would contain details about:

what changed,

who changed it,

and why.

The events would be kept forever because we never know when we might need access to the history.

This can be very important in highly regulated industries such as banking or healthcare.

However, these audit entries do have potential issues.

If we have implemented the log correctly, any details that exist in the state will also exist in the log.

This leads to data duplication.

And if we have duplicate data, then what happens if it gets out of sync?

Ideally, we'd perform any updates in a transactional fashion to prevent that, but bugs happen, and when they do, we need a plan to deal with them.

When our state and audit log are in disagreement, the safe choice is to rely on the audit log.

It contains a full history of all of the events and therefore is typically more reliable than the state alone.

But this begs the question.

If all of the data is contained in the audit log, and if the audit log is the final source of truth, then why do we need the state?

Couldn't we just scrap the state, and use the audit log instead?

This is the basic principle behind event sourcing.

Each time we update an object inside of a microservice, we don't store the current state of the object in the database.

Instead, we store the events that led to that state, essentially, the audit entries.

If we need to reconstruct the object, then we can replay all of the events and use them to calculate the current state.

Banking is an excellent example of how this works.

The account balance represents the current state.

However, that's not what the bank stores.

Instead, the bank stores a list of all of the transactions that led to the current balance.

If we want to increase our balance from one hundred dollars to two hundred dollars, then we do so by depositing an additional hundred dollars.

Rather than storing the new balance, we instead store an event, perhaps named "FundsDeposited" and we record important details about that event such as how much was deposited, and when.

Later, when we want to know the current balance of the account, we can replay all of the events to calculate it.

The advantage of this approach is that if a mistake happens anywhere along the way,

we can use the historical record to locate the mistake and issue a correction.

We can even replay portions of the event log to reconstruct what the state was at any time in the past.

If I want to know what my balance was at 3 pm last Tuesday, all of the information required to answer that question is readily available.

That wouldn't be true if I was only storing the balance, and not the transaction history.

Another key advantage provided by event sourcing is that it allows us to solve the dual-write problem.

When we build event-driven systems, we often need to record data in our database but also emit events to a secondary system such as Apache Kafka.

Because Kafka and the database aren't connected, there is no way to update both in a transactional fashion.

This can leave us in a situation where one update fails, and the other succeeds, and our data becomes out of sync.

However, using event-sourcing we can solve this problem.

Rather than trying to update both the database and Kafka at the same time, we worry only about the event log.

Remember, it's our single source of truth.

As long as the event makes it into the log, then we consider it to be correct.

We can then have a separate process that scans the event log and emits any new events to Kafka.

Once this separate process finishes, we'll be able to guarantee that the database and Kafka are in sync.

We've eliminated the risk of data loss.

Of course, event sourcing isn't a perfect solution.

It can introduce complexity, especially when we need to deal with queries that span multiple data objects.

Tools such as Command Query Responsibility Segregation or CQRS can help with this, but they come at a cost.

So although event sourcing can seem like a very powerful tool, it may not be the tool we want to reach for in every situation.

It can be a great option when we are building portions of our system that give a competitive advantage or require auditing.

But for systems without these requirements, the added complexity can outweigh any advantages.

If you want a deeper dive into the dual-write problem, check out the video linked below.

For more information on event-sourcing make sure you look at the course found in Confluent Developer and on our YouTube channel.

Don't forget to like, share, and subscribe.

And I'll see you next time.



# Event sourcing pattern

[PDF](https://docs.aws.amazon.com/pdfs/prescriptive-guidance/latest/cloud-design-patterns/cloud-design-patterns.pdf#event-sourcing)[RSS](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/cloud-design-patterns.rss)

## Intent

In event-driven architectures, the event sourcing pattern stores the events that result in a state change in a data store. This helps to capture and maintain a complete history of state changes, and promotes auditability, traceability, and the ability to analyze past states.

## Motivation

Multiple microservices can collaborate to handle requests, and they communicate through events. These events can result in a change in state (data). Storing event objects in the order in which they occur provides valuable information on the current state of the data entity and additional information about how it arrived at that state.

## Applicability

Use the event sourcing pattern when:

-   An immutable history of the events that occur in an application is required for tracking.
    
-   Polyglot data projections are required from a single source of truth (SSOT).
    
-   Point-in time reconstruction of the application state is needed.
    
-   Long-term storage of application state isn't required, but you might want to reconstruct it as needed.
    
-   Workloads have different read and write volumes. For example, you have write-intensive workloads that don't require real-time processing.
    
-   Change data capture (CDC) is required to analyze the application performance and other metrics.
    
-   Audit data is required for all events that happen in a system for reporting and compliance purposes.
    
-   You want to derive what-if scenarios by changing (inserting, updating, or deleting) events during the replay process to determine the possible end state.
    

## Issues and considerations

-   **Optimistic concurrency control:** This pattern stores every event that causes a state change in the system. Multiple users or services can try to update the same piece of data at the same time, causing event collisions. These collisions happen when conflicting events are created and applied at the same time, which results in a final data state that doesn't match reality. To solve this issue, you can implement strategies to detect and resolve event collisions. For example, you can implement an optimistic concurrency control scheme by including versioning or by adding timestamps to events to track the order of updates.
    
-   **Complexity:**  Implementing event sourcing necessitates a shift in mindset from traditional CRUD operations to event-driven thinking. The replay process, which is used to restore the system to its original state, can be complex in order to ensure data idempotency. Event storage, backups, and snapshots can also add additional complexity.
    
-   **Eventual consistency**: Data projections derived from the events are eventually consistent because of the latency in updating data by using the command query responsibility segregation (CQRS) pattern or materialized views. When consumers process data from an event store and publishers send new data, the data projection or the application object might not represent the current state.
    
-   **Querying**: Retrieving current or aggregate data from event logs can be more intricate and slower compared to traditional databases, particularly for complex queries and reporting tasks. To mitigate this issue, event sourcing is often implemented with the CQRS pattern.
    
-   **Size and cost of the event store:** The event store can experience exponential growth in size as events are continuously persisted, especially in systems that have high event throughput or extended retention periods. Consequently, you must periodically archive event data to cost-effective storage to prevent the event store from becoming too large.
    
-   **Scalability of the event store:** The event store must efficiently handle high volumes of both write and read operations. Scaling an event store can be challenging, so it's important to have a data store that provides shards and partitions.
    
-   **Efficiency and optimization**: Choose or design an event store that handles both write and read operations efficiently. The event store should be optimized for the expected event volume and query patterns for the application. Implementing indexing and query mechanisms can speed up the retrieval of events when reconstructing the application state. You can also consider using specialized event store databases or libraries that offer query optimization features.
    
-   **Snapshots:** You must back up event logs at regular intervals with time-based activation. Replaying the events on the last known successful backup of the data should lead to point-in-time recovery of the application state. The recovery point objective (RPO) is the maximum acceptable amount of time since the last data recovery point. RPO determines what is considered an acceptable loss of data between the last recovery point and the interruption of service. The frequency of the daily snapshots of the data and event store should be based on the RPO for the application.
    
-   **Time sensitivity:** The events are stored in the order in which they occur. Therefore, network reliability is an important factor to consider when you implement this pattern. Latency issues can lead to an incorrect system state. Use first in, first out (FIFO) queues with at-most-once delivery to carry the events to the event store.
    
-   **Event replay performance**: Replaying a substantial number of events to reconstruct the current application state can be time-consuming. Optimization efforts are required to enhance performance, particularly when replaying events from archived data.
    
-   **External system updates:**  Applications that use the event sourcing pattern might update data stores in external systems, and might capture these updates as event objects. During event replays, this might become an issue if the external system doesn't expect an update. In such cases, you can use feature flags to control external system updates.
    
-   **External system queries:**  When external system calls are sensitive to the date and time of the call, the received data can be stored in internal data stores for use during replays.
    
-   **Event versioning:**  As the application evolves, the structure of the events (schema) can change. Implementing a versioning strategy for events to ensure backward and forward compatibility is necessary. This can involve including a version field in the event payload and handling different event versions appropriately during replay.
    

## Implementation

### High-level architecture

**Commands and events**

In distributed, event-driven microservices applications, commands represent the instructions or requests sent to a service, typically with the intent of initiating a change in its state. The service processes these commands and evaluates the command's validity and applicability to its current state. If the command runs successfully, the service responds by emitting an event that signifies the action taken and the relevant state information. For example, in the following diagram, the booking service responds to the Book ride command by emitting the Ride booked event.

![Commands and events in event sourcing pattern](https://docs.aws.amazon.com/images/prescriptive-guidance/latest/cloud-design-patterns/images/event-sourcing-1.png)

**Event stores**

Events are logged into an immutable, append-only, chronologically ordered repository or data store known as the  _event store_. Each state change is treated as an individual event object. An entity object or a data store with a known initial state, its current state, and any point-in-time view can be reconstructed by replaying the events in the order of their occurrence.

The event store acts as a historical record of all actions and state changes, and serves as a valuable single source of truth. You can use the event store to derive the final, up-to-date state of the system by passing the events through a replay processor, which applies these events to produce an accurate representation of the latest system state. You can also use the event store to generate the point-in-time perspective of the state by replaying the events through a replay processor. In the event sourcing pattern, the current state might not be entirely represented by the most recent event object. You can derive the current state in one of three ways:

-   By aggregating related events. The related event objects are combined to generate the current state for querying. This approach is often used in conjunction with the CQRS pattern, in that the events are combined and written into the read-only data store.
    
-   By using materialized views. You can employ event sourcing with the materialized view pattern to compute or summarize the event data and obtain the current state of related data.
    
-   By replaying events. Event objects can be replayed to carry out actions for generating the current state.
    

The following diagram shows the  `Ride booked`  event being stored in an event store.

![Using event stores in event sourcing pattern](https://docs.aws.amazon.com/images/prescriptive-guidance/latest/cloud-design-patterns/images/event-sourcing-2.png)

The event store publishes the events it stores, and the events can be filtered and routed to the appropriate processor for subsequent actions. For example, events can be routed to a view processor that summarizes the state and shows a materialized view. The events are transformed to the data format of the target data store. This architecture can be extended to derive different types of data stores, which leads to polyglot persistence of the data.

The following diagram describes the events in a ride booking application. All the events that occur within the application are stored in the event store. Stored events are then filtered and routed to different consumers.

![Example high-level implementation for event sourcing pattern](https://docs.aws.amazon.com/images/prescriptive-guidance/latest/cloud-design-patterns/images/event-sourcing-3.png)

The ride events can be used to generate read-only data stores by using the CQRS or materialized view pattern. You can obtain the current state of the ride, the driver, or the booking by querying the read stores. Some events, such as  `Location changed`  or  `Ride completed`, are published to another consumer for payment processing. When the ride is complete, all ride events are replayed to build a history of the ride for auditing or reporting purposes.

The event sourcing pattern is frequently used in applications that require a point-in-time recovery, and also when the data has to be projected in different formats by using a single source of truth. Both of these operations require a replay process to run the events and derive the required end state. The replay processor might also require a known starting point—ideally not from application launch, because that would not be an efficient process. We recommend that you take regular snapshots of the system state and apply a smaller number of events to derive an up-to-date state.

### Implementation using AWS services

In the following architecture, Amazon Kinesis Data Streams is used as the event store. This service captures and manages application changes as events, and offers a high-throughput and real-time data streaming solution. To implement the event sourcing pattern on AWS, you can also use services such as Amazon EventBridge and Amazon Managed Streaming for Apache Kafka (Amazon MSK) based on your application's needs.

To enhance durability and enable auditing, you can archive the events that are captured by Kinesis Data Streams in Amazon Simple Storage Service (Amazon S3). This dual-storage approach helps retain historical event data securely for future analysis and compliance purposes.

![Implementing the event sourcing pattern with AWS services](https://docs.aws.amazon.com/images/prescriptive-guidance/latest/cloud-design-patterns/images/event-sourcing-4.png)

The workflow consists of the following steps:

1.  A ride booking request is made through a mobile client to an Amazon API Gateway endpoint.
    
2.  The ride microservice (`Ride service`  Lambda function) receives the request, transforms the objects, and publishes to Kinesis Data Streams.
    
3.  The event data in Kinesis Data Streams is stored in Amazon S3 for compliance and audit history purposes.
    
4.  The events are transformed and processed by the  `Ride event processor`  Lambda function and stored in an Amazon Aurora database to provide a materialized view for the ride data.
    
5.  The completed ride events are filtered and sent for payment processing to an external payment gateway. When payment is completed, another event is sent to Kinesis Data Streams to update the Ride database.
    
6.  When the ride is complete, the ride events are replayed to the  `Ride service`  Lambda function to build routes and the history of the ride.
    
7.  Ride information can be read through the  `Ride data service`, which reads from the Aurora database.
    

API Gateway can also send the event object directly to Kinesis Data Streams without the  `Ride service`  Lambda function. However, in a complex system such as a ride hailing service, the event object might need to be processed and enriched before it gets ingested into the data stream. For this reason, the architecture has a  `Ride service`  that processes the event before sending it to Kinesis Data Streams.
