## Load Balancer

**Load balancing** refers to efficiently distributing incoming network traffic across a group of backend servers, also known as a _server farm_ or _server pool_.

- It helps to distribute load across multiple resources in a manner that maximizes speed and capacity utilization and ensures that no one server is overworked, which could degrade performance.
- It also keep track of status of all the resources while distributing requests. If a single server goes down, the load balancer redirects traffic to the remaining online servers. When a new server is added to the server group, the load balancer automatically starts to send requests to it.

![[load-balancer.png]]

### In this manner, a load balancer performs the following functions:

-   Distributes client requests or network load efficiently across multiple servers
-   Ensures high availability and reliability by sending requests only to servers that are online
-   Provides the flexibility to add or subtract servers as demand dictates

#### where it can be added
- User - Web Server
- Web Server - Internal Server
- Internal Server - Database

### Load Balancing Algorithms

Different load balancing algorithms provide different benefits; the choice of load balancing method depends on your needs:

-   **Round Robin** – Requests are distributed across the group of servers sequentially.
-   **Least Connections** – A new request is sent to the server with the fewest current connections to clients. The relative computing capacity of each server is factored into determining which one has the least connections.
-   **Least Time** – Sends requests to the server selected by a formula that combines the fastest response time and fewest active connections.
-   **Hash** – Distributes requests based on a key you define, such as the client IP address or the request URL.
-   **IP Hash** – The IP address of the client is used to determine which server receives the request.

![[load-balancer-type.png]]

## Benefits of Load Balancing

-   Reduced downtime
-   Scalable
-   Redundancy
-   Flexibility
-   Efficiency

