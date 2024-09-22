The transactional outbox pattern leverages database transactions to update a microservice's state and an outbox table.
Events in the outbox will be sent to an external messaging platform such as Apache Kafka.
This technique is used to overcome the dual-write problem which occurs when you have to write data to two separate systems such as a database and Apache Kafka.\
The database transactions can be used to ensure atomic writes between the two tables. From there, a separate process can consume the outbox and update the external system as required.
This process can be written manually, or we can leverage tools such as Change Data Capture (CDC) or Kafka connectors.
