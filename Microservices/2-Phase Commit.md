
Two-Phase Commit is a protocol used in distributed database systems to achieve atomicity across multiple nodes involved in a transaction.

The Two-Phase Commit (2PC) protocol plays a crucial role in ensuring data consistency in distributed database systems. It helps maintain the ACID (Atomicity, Consistency, Isolation, Durability) properties of database transactions across multiple nodes.

There are two components involved in this protocol:
- **Transaction Coordinator:**
	The transaction coordinator initiates the transaction and ensures that all participating nodes agree to either commit or abort the transaction, maintaining consistency and integrity across the distributed system.
- **Participant Nodes:**

# How Does Two-Phase Commit Work?

The 2PC protocol operates in two stages, as explained below:

## Prepare Phase

The prepare phase begins when a coordinator initiates a transaction. After making the necessary local changes, the coordinator sends a prepare message to all participant nodes, instructing them to prepare to commit the transaction.

Each participant executes the transaction locally, writes the changes to a log, and responds to the coordinator. If the transaction executes successfully, the participant votes to commit and enters a prepared state. If the transaction fails at any participant node, it votes to abort.

## Commit Phase

In the commit phase, the coordinator collects votes from all participant nodes. If all participants vote to commit, the coordinator writes a commit record in its log and sends a global commit message to all participants. Each participant then commits the transaction locally and sends an acknowledgment to the coordinator.

However, if any participant votes to abort or if the coordinator doesn’t receive a response from a participant (a timeout occurs), it decides to abort the transaction. The coordinator logs the abort record and sends a global abort message to all participants. Each participant then undoes the transaction changes and sends an acknowledgment to the coordinator.

# Drawbacks of 2-Phase Commit

- **Latency:**
	The Transaction Coordinator waits for responses from all the participant servers. Only then it carries on with the second phase of the commit. This increases the latency and the client may experience slowness in execution. Hence, 2-PC is not a good choice for performance-critical applications.
- **Transaction Coordinator:**
	The Transaction Coordinator becomes a single point of failure at times. The Transaction Coordinator may go down before sending a commit message to all the participants. In such cases, all the transactions running on the participants will go in a blocked state. They would commit only once the coordinator comes up & sends a commit signal.
- **Participant dependency:**
	A slow participant affects the performance of other participants. Total transaction time is proportional to the time taken by the slowest server. If the transaction fails on a single server, it has to be rolled back on all other servers. This may lead to wastage of resources.



