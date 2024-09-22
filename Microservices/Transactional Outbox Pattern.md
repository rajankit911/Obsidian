The transactional outbox pattern leverages database transactions to update a microservice's state and an outbox table.
Events in the outbox will be sent to an external messaging platform such as Apache Kafka.
This technique is used to overcome the dual-write problem which occurs when you have to write data to two separate systems such as a database and Apache Kafka.\
The database transactions can be used to ensure atomic writes between the two tables. From there, a separate process can consume the outbox and update the external system as required.
This process can be written manually, or we can leverage tools such as Change Data Capture (CDC) or Kafka connectors.



# The Outbox Pattern

This pattern is quite simple as it adds an asynchronous service to monitor events in your application. However, conceptually, this can be hard for some people, so let’s look at this through the lens of an analogy.

![](https://miro.medium.com/v2/resize:fit:1400/1*VZflild3KQvQXbNDNuUUHA.jpeg)

A lemonade stand illustration [Jennifer Hines](https://www.jenniferhines.design/lemonade-stand-childrens-illustration).

You have a lemonade stand (your system), and whenever someone purchases lemonade from you, you wish to send them a thank you card (an event). So now you have two jobs:

1.  Make and sell the lemonade (updating your records).
2.  Write and send the thank-you card (publishing an event).

The challenge is that you might forget to send a thank-you card after a sale or mistakenly send one to a non-customer. This mirrors the dual write problem, where actions must sync perfectly to avoid errors. To solve this problem, you take the following steps:

-   **Step 1(The Outbox Table)**: Instead of trying to write the thank you card straight away, you write down the customer's name and address in a notebook beside you.
-   **Step 2 (Transactional Writes)**: You decided to bundle selling lemonade and noting sales, ensuring both are completed together.
-   **Step 3 (Event Publisher)**: After closing your store, use the notebook to send thank you cards to each customer.
-   **Step 4 (Clean History)**: After completing the above steps, tear out the used pages to ensure you have an empty notebook for the following day.

If you’ve followed these steps correctly, you can hopefully understand why this works and why it is more elegant and reliable than the previous solution. This is how it should look in terms of software architecture:

![](https://miro.medium.com/v2/resize:fit:1400/1*NAaAObxXcwd2E6DyuVZ6cQ.jpeg)

The outbox pattern visualised.



# Transactional outbox pattern

[PDF](https://docs.aws.amazon.com/pdfs/prescriptive-guidance/latest/cloud-design-patterns/cloud-design-patterns.pdf#transactional-outbox)[RSS](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/cloud-design-patterns.rss)

## Intent

The transactional outbox pattern resolves the dual write operations issue that occurs in distributed systems when a single operation involves both a database write operation and a message or event notification. A dual write operation occurs when an application writes to two different systems; for example, when a microservice needs to persist data in the database and send a message to notify other systems. A failure in one of these operations might result in inconsistent data.

## Motivation

When a microservice sends an event notification after a database update, these two operations should run atomically to ensure data consistency and reliability.

-   If the database update is successful but the event notification fails, the downstream service will not be aware of the change, and the system can enter an inconsistent state.
    
-   If the database update fails but the event notification is sent, data could get corrupted, which might affect the reliability of the system.
    

## Applicability

Use the transactional outbox pattern when:

-   You're building an event-driven application where a database update initiates an event notification .
    
-   You want to ensure atomicity in operations that involve two services.
    
-   You want to implement the  [event sourcing pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/event-sourcing.html).
    

## Issues and considerations

-   **Duplicate messages**: The events processing service might send out duplicate messages or events, so we recommend that you make the consuming service idempotent by tracking the processed messages.
    
-   **Order of notification**: Send messages or events in the same order in which the service updates the database. This is critical for the event sourcing pattern where you can use an event store for point-in-time recovery of the data store. If the order is incorrect, it might compromise the quality of the data. Eventual consistency and database rollback can compound the issue if the order of notifications isn't preserved.
    
-   **Transaction rollback**: Do not send out an event notification if the transaction is rolled back.
    
-   **Service-level transaction handling**: If the transaction spans services that require data store updates, use the  [saga orchestration pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-orchestration.html)  to preserve data integrity across the data stores.
    

## Implementation

### High-level architecture

The following sequence diagram shows the order of events that happen during dual write operations.

![Order of events during dual write operations](https://docs.aws.amazon.com/images/prescriptive-guidance/latest/cloud-design-patterns/images/outbox-1.png)

1.  The flight service writes to the database and sends out an event notification to the payment service.
    
2.  The message broker carries the messages and events to the payment service. Any failure in the message broker prevents the payment service from receiving the updates.
    

If the flight database update fails but the notification is sent out, the payment service will process the payment based on the event notification. This will cause downstream data inconsistencies.

### Implementation using AWS services

To demonstrate the pattern in the sequence diagram, we will use the following AWS services, as shown in the following diagram.

-   Microservices are implemented by using  [AWS Lambda](https://aws.amazon.com/lambda/).
    
-   The primary database is managed by  [Amazon Relational Database Service (Amazon RDS)](https://aws.amazon.com/rds/) .
    
-   [Amazon Simple Queue Service (Amazon SQS)](https://aws.amazon.com/sqs/)  acts as the message broker that receives event notifications.
    

![Transactional outbox pattern with AWS Lambda, Amazon RDS, and Amazon SQS](https://docs.aws.amazon.com/images/prescriptive-guidance/latest/cloud-design-patterns/images/outbox-2.png)

If the flight service fails after committing the transaction, this might result in the event notification not being sent.

![Transactional failures after commit operation](https://docs.aws.amazon.com/images/prescriptive-guidance/latest/cloud-design-patterns/images/outbox-3.png)

However, the transaction could fail and roll back, but the event notification might still be sent, causing the payment service to process the payment.

![Transactional failures after commit operation with rollback](https://docs.aws.amazon.com/images/prescriptive-guidance/latest/cloud-design-patterns/images/outbox-4.png)

To address this problem, you can use an outbox table or change data capture (CDC). The following sections discuss these two options and how you can implement them by using AWS services.

#### Using an outbox table with a relational database

An outbox table stores all the events from the flight service with a timestamp and a sequence number.

When the flight table is updated, the outbox table is also updated in the same transaction. Another service (for example, the event processing service) reads from the outbox table and sends the event to Amazon SQS. Amazon SQS sends a message about the event to the payment service for further processing.  [Amazon SQS standard queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues.html)  guarantee that the message is delivered at least once and doesn't get lost. However, when you use Amazon SQS standard queues, the same message or event might be delivered more than once, so you should ensure that the event notification service is idempotent (that is, processing the same message multiple times shouldn't have an adverse effect). If you require the message to be delivered exactly once, with message ordering, you can use  [Amazon SQS first in, first out (FIFO) queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html).

If the flight table update fails or the outbox table update fails, the entire transaction is rolled back, so there are no downstream data inconsistencies.

![Rollback with no downstream data inconsistencies](https://docs.aws.amazon.com/images/prescriptive-guidance/latest/cloud-design-patterns/images/outbox-5.png)

In the following diagram, the transactional outbox architecture is implemented by using an Amazon RDS database. When the events processing service reads the outbox table, it recognizes only those rows that are part of a committed (successful) transaction, and then places the message for the event in the SQS queue, which is read by the payment service for further processing. This design resolves the dual write operations issue and preserves the order of messages and events by using timestamps and sequence numbers.

![Design that resolves dual write operation issues](https://docs.aws.amazon.com/images/prescriptive-guidance/latest/cloud-design-patterns/images/outbox-6.png)

#### Using change data capture (CDC)

Some databases support the publishing of item-level modifications to capture changed data. You can identify the changed items and send an event notification accordingly. This saves the overhead of creating another table to track the updates. The event initiated by the flight service is stored in another attribute of the same item.

[Amazon DynamoDB](https://aws.amazon.com/dynamodb/)  is a key-value NoSQL database that supports CDC updates. In the following sequence diagram, DynamoDB publishes item-level modifications to Amazon DynamoDB Streams. The event processing service reads from the streams and publishes the event notification to the payment service for further processing.

![Transactional outbox with DynamoDB and DynamoDB Streams](https://docs.aws.amazon.com/images/prescriptive-guidance/latest/cloud-design-patterns/images/outbox-7.png)

DynamoDB Streams captures the flow of information relating to item-level changes in a DynamoDB table by using a time-ordered sequence.

You can implement a transactional outbox pattern by enabling streams on the DynamoDB table. The Lambda function for the event processing service is associated with these streams.

-   When the flight table is updated, the changed data is captured by DynamoDB Streams, and the events processing service polls the stream for new records.
    
-   When new stream records become available, the Lambda function synchronously places the message for the event in the SQS queue for further processing. You can add an attribute to the DynamoDB item to capture timestamp and sequence number as needed to improve the robustness of the implementation.
    

![Transactional outbox with CDC](https://docs.aws.amazon.com/images/prescriptive-guidance/latest/cloud-design-patterns/images/outbox-8.png)

## Sample code
