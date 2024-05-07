
An API gateway is a single point of entry to the clients of an application. It sits between the clients and a collection of backend services for the application.

An API gateway typically provides  several important functions:
- Authentication and authorization
- Security policy enforcements
- Load balancing
- circuit breaking
- Protocol translation
- Service discovery
- Logging
- Monitoring and analytics
- Caching

# Client Request Flow

Let’s examine a typical flow of a client request through the API gateway and onto the backend service.
1. The client sends a request to the API gateway. The request is typically HTTP-based. It could be REST, GraphQL, or some other higher-level abstractions.
2. The API gateway validates the HTTP request.
3. The API gateway checks the caller’s IP address and other HTTP headers against its allow-list and deny-list. It could also perform basic rate limit checks against attributes such as IP address and HTTP headers. For example, it could reject requests from an IP address exceeding a certain rate.
4. The API gateway passes the request to an identity provider for authentication and authorization. The API gateway receives an authenticated session back from the provider with the scope of what the request is allowed to do.
5. A higher level rate-limit check is applied against the authenticated session. If it is over the limit, the request is rejected at this point.
6. With the help of a service discovery component, the API gateway locates the appropriate backend service to handle the request by path matching.
7. The API gateway transforms the request into the appropriate protocol and sends the transformed request to the backend service. An example would be gRPC. When the response comes back from the backend service, the API gateway transforms the response back to the public-facing protocol and returns the response to the client.

A proper API gateway also provides other critical services. For example, an API gateway should track errors and provide circuit-breaking functionality to protect the services from overloading. An API gateway should also provide logging, monitoring, and analytics services for operational observability.

An API gateway is a critical  piece of the infrastructure. It should be deployed to multiple regions to improve availability. For many cloud provider offerings, the API gateway is deployed across the world close to the clients.

