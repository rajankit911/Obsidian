A proxy controls access to the original object, allowing you to perform something either before or after the request gets through to the original object.

- Structural design pattern
- It provides a substitute or placeholder for another object

> [!note]
> RMI API uses proxy design pattern. Stub and Skeleton are two proxy objects used in RMI.

# Structure

![[proxy-structure.png|400]]

- **Service Interface**:
	It declares the interface of the Service. The proxy must follow this interface to be able to disguise itself as a service object.
	
- **Service**:
	It is a class that provides some useful business logic.
	
- **Proxy**
	This class has a reference field that points to a service object. After the proxy finishes its processing (e.g., lazy initialization, logging, access control, caching, etc.), it passes the request to the service object. Usually, proxies manage the full lifecycle of their service objects.
	
- **Client**:
	It should work with both services and proxies via the same interface. This way you can pass a proxy into any code that expects a service object.


# Applicability

- **Lazy Loading through Virtual Proxy**:
	Instead of creating the heavyweight object at the launch of application, the proxy can delay the object’s initialization to a time when it’s really needed. Otherwise, it would wastes system resources by being always up, even though you only need it from time to time.
	
- **Access Control through Protection Proxy**:
	The proxy can pass the request to the service object only if the client’s credentials match some criteria.
	
- **Local execution of Remote Service through Remote Proxy**:
	This is used when the service object is located on a remote server. In this case, the proxy passes the client request over the network, handling all of the nasty details of working with the network.
	
- **Logging Requests through Logging Proxy**:
	This is used when proxy needs to keep a history of requests to the service object. The proxy can log each request before passing it to the service.
	
- **Caching Request Results through Caching Proxy**:
	The proxy can implement caching for recurring client requests that always yield the same results and manage the life cycle of this cache, especially if results are quite large. The proxy may use the parameters of requests as the cache keys.
	
- **Smart reference**:
	This is used when proxy needs to be able to dismiss a heavyweight object once there are no clients that use it.
	
	The proxy can keep track of clients that obtained a reference to the service object or its results. From time to time, the proxy may go over the clients and check whether they are still active. If the client list gets empty, the proxy might dismiss the service object and free the underlying system resources.
	
	The proxy can also track whether the client had modified the service object. Then the unchanged objects may be reused by other clients.