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
