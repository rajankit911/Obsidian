A logical device that distributes network traffic among upstream servers. It acts as a traffic controller.

![[load_balancer.png|500]]


> [!FAQ]- Upstream Servers ? 
> In computer networking, upstream server refers to a server that provides service to another server. In other words, upstream server is a server that is located higher in a hierarchy of servers.


* To improve application capacity and dependability
* To increase application performance by reducing the load

Divided into two categories:
- Layer 4 (Transport Layer) Load Balancer
- Layer 4 (Application Layer) Load Balancer

## Layer 4 (Transport Layer) Load Balancer

Layer 4 load balancing takes place at the transport layer of the OSI model.
Layer 4 load balancers simply route network packets to and from the upstream server without inspecting their content.

> Advantages

- Simpler Load Balancing
- Quick and efficient because it does not take data into account.
- More secure because packages are not examined. It just forwards the packets. It does not need to decrypt the content before forwarding. If it is compromised, no one will be able to access the data.
- Uses NAT.
- Only one TCP connection between the client and the server. This allows the load balancer to serve a maximum number of TCP connections.

> Disadvantages

- Based on the content, smart load balancing is not possible.
- Not applicable to microservices.
- There's no caching since you can't access the data.

## Layer 7 (Application Layer) Load Balancer

Layer 7 load balancing works at the application layer of the OSI model.
Application Load Balancers route network traffic based on the message's content.

> Advantages

- Based on the URL, it provides smart routing.
- It offers caching.
- It is great for microservices.

> Disadvantages 

- It is expensive as it has to process message’s content.
- Decryption is required if connection is secured.
- In terms of security, your certificate must be shared with the load balancers. If an attacker gains control to the load balancer, they will have total access to all of your data.
- It establishes two TCP connections: client to proxy/proxy to server. So, this limits the load balancer’s maximum TCP connections.