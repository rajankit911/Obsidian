
> [!todo]
> - [ ] Client request flow diagram


An API gateway is a single point of entry to the clients of an application. It sits between the clients and a collection of backend services for the application.

An API gateway in microservices acts as a central entry point that handles client requests and then routes them to the appropriate microservices.

An API Gateway acts as a middleman between clients and a group of backend services in a microservices setup.

# Functions of API Gateway

- **Single Point of Interaction**: With increased number of services communicating directly with each other can lead to complexity. By serving as the central point of communication, the API Gateway streamlines communications between services.

- **Load balancing**: The gateway can distribute incoming requests across multiple instances of the same microservice to ensure optimal resource utilization and high availability.

- Logging
- Monitoring and analytics

- **Log Collection and Monitoring**: Gathering logs and monitoring data from distributed services can be arduous. API Gateway serves as a centralized point for collecting logs and monitoring data, facilitating streamlined observability for debugging and improving performance.

- **API Metering and Billing:** Tracking API usage for reporting and billing.

- **Service Discovery:** API Gateway can be used to discover other available microservices and their locations. This enables the client to access microservices without knowing their specific addresses.

- **Authentication and authorization:** Securing individual services can be challenging and resource-intensive. API Gateway centralizes authentication and authorization. It handles security-related concerns by authenticating clients and authorizing access to specific microservices.

- **Request and Response validation:** Validate the microservice requests and responses and ensure they conform to the expected format and structure. This helps prevent errors and ensure the proper functioning of the microservice.

- **Aggregation of Results**: The API gateway can combine responses of multiple backend microservices into a single consolidated response to fulfill a client request. This reduces round-trips and improves performance

- **Error Handling:** Provide a consistent way to handle errors and generate error responses to clients even when backend services are unavailable or return unexpected results.

- **Request and Response Transformation / Protocol Translation:** It can translate client requests from one protocol (e.g., HTTP/REST) to the appropriate protocol used by the underlying microservices. For example, if a client sends a request using HTTP/1.1, but the backend service expects requests in gRPC format, the API Gateway can translate the HTTP request into a gRPC request before forwarding it to the backend service. Likewise, when the backend service responds with data in gRPC format, the API Gateway can translate it back into HTTP format before sending it back to the client.

- **Caching**: Network calls between services can introduce latency, affecting overall performance. The API gateway can cache responses from microservices for redundant requests to improve performance and reduce latency.

- **Rate Limiting:** Controlling the number of requests to prevent abuse.

- **Circuit Breaker:** Act as a circuit breaker to prevent a single failed microservice from bringing down the entire system.


- **Cross-Origin Resource Sharing (CORS) Management:** Handling requests from other domains.

- **API Version Management:** Routing requests to different backend service versions.

# Client Request Flow

![[Pasted image 20240713163031.png]]

Let’s examine a typical flow of a client request through the API gateway and onto the backend service.
1. The client sends a request to the API gateway. The request is typically HTTP-based. It could be REST, GraphQL, or some other higher-level abstractions.
2. The API gateway parses and validates the attributes in the HTTP request.
3. The API gateway performs security checks to ensure safe access. It checks the caller’s IP address and other HTTP headers against its allow-list and deny-list.
4. The API gateway passes the request to an identity provider for authentication and authorization. The API gateway receives an authenticated session back from the provider with the scope of what the request is allowed to do.
5. The rate-limiting rules are applied against the authenticated session. If it is over the limit, the request is rejected at this point.
6. With the help of a service discovery component, the API gateway locates the appropriate backend service to handle the request by path matching.
7. The API gateway transforms the request into the appropriate protocol and sends the transformed request to the backend service.
8. Additionally, the API gateway handles errors and faults effectively, leveraging tools like the ELK stack for logging and monitoring, and solutions like Redis for caching, enhancing performance and reliability.

A proper API gateway also provides other critical services. For example, an API gateway should track errors and provide circuit-breaking functionality to protect the services from overloading. An API gateway should also provide logging, monitoring, and analytics services for operational observability.

An API gateway is a critical  piece of the infrastructure. It should be deployed to multiple regions to improve availability. For many cloud provider offerings, the API gateway is deployed across the world close to the clients.


# Load Balancer vs. Reverse Proxy vs. API Gateway

While load balancers, reverse proxies, and API Gateways share the goal of optimizing and managing web traffic, they serve distinct functions:

**1.** **Load Balancer:**
- Distributes traffic among servers to optimize resource utilizaiton.
- Used to handle heavy traffic and ensure even load distribution.

**2.** **Reverse Proxy:**
- Acts as an intermediary between clients and servers, providing security and control.
- Manages SSL encryption, serves static content, and controls access to internal servers.

**3.** **API Gateway:**
- Manages interactions between clients and backend services in a microservices architecture.
- Provides functionalities like routing, aggregation, and security in addition to handling requests.


| Feature              | API Gateway                             | Load Balancer                     | Reverse Proxy                                     |
| -------------------- | --------------------------------------- | --------------------------------- | ------------------------------------------------- |
| Primary Function     | API Management                          | Traffic Distribution              | Forwards Requests                                 |
| Location             | Sits before backend services            | Sits in front of multiple servers | Sits in front of a single server or a small group |
| Protocol Translation | Yes                                     | No                                | No                                                |
| Security             | Enforces Authentication / Authorization | Can implement basic security      | Can implement basis security                      |
| Complexity           | More complex to configure               | Relatively simple                 | Relatively simple                                 |


>In summary, while load balancers distribute traffic for optimal resource utilization, reverse proxies protect servers and control access, and API Gateways streamline interactions and provide additional functionalities in a microservices architecture. Each serves a specific purpose in optimizing and managing web traffic, catering to different requirements and use cases.




