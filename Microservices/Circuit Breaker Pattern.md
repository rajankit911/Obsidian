Circuit Breaker is a design pattern used in microservices architecture for building resilient microservices by preventing cascading failures and providing fallback mechanisms.

This pattern is inspired by the electrical circuit breaker, which cuts off the power when there is a short circuit or an overload. This prevents the circuit from overheating and causing a fire.

In the same way, circuit breaker pattern monitors the health of external services and automatically blocks requests from calling those services if they are unhealthy.

It helps maintain the overall stability and reliability of the system.

The circuit breaker pattern can improve the availability and performance of your system by avoiding unnecessary requests to failed or slow services, and by allowing them to recover gracefully.



You have applied the [Microservice architecture](https://microservices.io/patterns/microservices.html). Services sometimes collaborate when handling requests. When one service synchronously invokes another there is always the possibility that the other service is unavailable or is exhibiting such high latency it is essentially unusable. Precious resources such as threads might be consumed in the caller while waiting for the other service to respond. This might lead to resource exhaustion, which would make the calling service unable to handle other requests. The failure of one service can potentially cascade to other services throughout the application.


# How does the circuit breaker pattern work?

The circuit breaker pattern works by wrapping each service call in an object that serves as a proxy or intermediary. This object monitors the success and failure rates of the service calls and maintains a state which reflects the health of the service. This state can be closed, open, or half-open.
## Different States of the Circuit Breaker

### Closed

When everything is normal, the circuit breaker remains in the closed state and all calls pass through to the services.
### Open

When the number of consecutive failures crosses a specified threshold within a given time period, the circuit breaker opens the circuit for a predetermined timeout period. During this period, all attempts to call the remote service will fail immediately and an exception is returned to the client.
### Half-Open

After the timeout period expires, the circuit switches to a half-open state and limited number of requests are allowed to pass through to check if the underlying problem still exists. If these requests are successful, it will close the circuit and resume normal operation. If any request fails, it will open the circuit again and wait for another period.

## What are the benefits of the circuit breaker pattern?

The circuit breaker pattern can provide several benefits for your microservices architecture, such as improving availability and performance, enhancing user experience, and facilitating recovery and maintenance. By avoiding unnecessary requests to failed or slow services and reducing the impact of network latency and congestion, the circuit breaker pattern can help increase system performance. Additionally, it can provide fallback responses or alternative actions when a service is unavailable or degraded instead of waiting for timeouts or returning errors. This pattern also makes it easier to isolate failures and allows them to heal without affecting the rest of the system. Finally, it provides feedback and metrics on the health of your services.


## What are the challenges of the circuit breaker pattern?

The circuit breaker pattern, while beneficial in a microservices architecture, comes with challenges and trade-offs. For example, you must select the right parameters and logic for your circuit breakers, such as the failure rate threshold and the timeout duration. Additionally, you must coordinate and synchronize your circuit breakers across multiple instances and nodes in a dynamic and distributed environment, possibly requiring a centralized registry or distributed configuration service. Lastly, testing and monitoring your circuit breakers to track their effects on your system is essential, potentially requiring tools and frameworks that support the circuit breaker pattern.


## What are the best practices for implementing the circuit breaker pattern?

To effectively and efficiently implement the circuit breaker pattern in your microservices architecture, you can use existing libraries and frameworks such as Hystrix, Resilience4j, Polly, or Spring Cloud Circuit Breaker. Selectively and strategically apply the circuit breaker pattern to service calls based on their criticality, frequency, and dependency. Define clear and consistent fallback responses or actions that are aligned with your business goals and user expectations. Configure and tune circuit breaker parameters according to service characteristics and requirements. Monitor and measure circuit breakers using metrics, logs, dashboards, alerts, and tracing tools. Collect feedback from circuit breakers such as error rates, response times, fallback rates, and state transitions to improve and optimize your system.



There are several tools and frameworks implementing the circuit breaker pattern. Here are some popular options:
- Netflix Hystrix
- Resilience4j

