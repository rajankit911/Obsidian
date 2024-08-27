**Service discovery** is the process of automatically detecting devices and services on a network. It helps to reduce configuration efforts by user.

Microservices service discovery is a way for applications and microservices to locate each other on a network. Service discovery implementations within microservices architecture includes both:

• a central server (or servers) that maintain a global view of addresses.  
• clients that connect to the central server to update and retrieve addresses.

# Components of Service Discovery

There are three components to Service Discovery: the service provider, the service consumer and the service registry.

1) The Service Provider registers itself with the service registry when it enters the system and de-registers itself when it leaves the system.

2) The Service Consumer gets the location of a provider from the service registry, and then connects it to the service provider.

3) The Service Registry is a database that contains the network locations of service instances. The service registry needs to be highly available and up to date so clients can go through network locations obtained from the service registry. A service registry consists of a cluster of servers that use a replication protocol to maintain consistency.

# Working of Service Discovery

Service discovery and registration are essential aspects of building Java microservices with Spring Cloud. Service discovery allows Microservices to find each other dynamically, enabling communication in a distributed system. Here's how it works:

- **Service registration**: Each microservice, upon startup, registers itself with the service registry (e.g., Spring Cloud Eureka) by providing its metadata such as service name, version, and network location.
- **Service discovery**: When a Microservice wants to communicate with another service, it queries the service registry to obtain the network location (e.g., host and port) of the target service.
- **Load balancing**: Service discovery often includes client-side load balancing, where the client (calling microservice) can choose one of the multiple instances of the target service, thereby distributing the load.
- **Dynamic updates**: The service registry constantly monitors the health of registered microservices. If a service instance becomes unavailable, it is removed from the registry, ensuring that clients only connect to healthy services.

Spring Cloud provides tools like Eureka, Consul, and ZooKeeper to implement service discovery and registration in Java microservices. By using these components, developers can build scalable and resilient microservices architectures, where services can find and communicate with each other seamlessly, regardless of their physical location and network configuration.

# Service Registry

The service registry is a key part of service discovery. It is a database containing the network locations of service instances.

A service registry needs to be highly available and up to date. Clients can cache network locations obtained from the service registry. However, that information eventually becomes out of date and clients become unable to discover service instances. Consequently, a service registry consists of a cluster of servers that use a replication protocol to maintain consistency.

## Applications

### Netflix Eureka

Netflix Eureka is good example of a service registry. It provides a REST API for registering and querying service instances.
- A service instance registers its network location using a `POST` request.
- Every 30 seconds it must refresh its registration using a `PUT` request.
- A registration is removed by either using an HTTP `DELETE` request or by the instance registration timing out.
- A client can retrieve the registered service instances by using an HTTP `GET` request.

Netflix achieves high availability by running one or more Eureka servers in each Amazon EC2 availability zone. Each Eureka server runs on an EC2 instance that has an Elastic IP address. DNS `TEXT` records are used to store the Eureka cluster configuration, which is a map from availability zones to a list of the network locations of Eureka servers. When a Eureka server starts up, it queries DNS to retrieve the Eureka cluster configuration, locates its peers, and assigns itself an unused Elastic IP address.

Eureka clients – services and service clients – query DNS to discover the network locations of Eureka servers. Clients prefer to use a Eureka server in the same availability zone. However, if none is available, the client uses a Eureka server in another availability zone.

### etcd

A highly available, distributed, consistent, key‑value store that is used for shared configuration and service discovery. Two notable projects that use etcd are Kubernetes and Cloud Foundry.

### consul

A tool for discovering and configuring services. It provides an API that allows clients to register and discover services. Consul can perform health checks to determine service availability.

### Apache Zookeeper

A widely used, high‑performance coordination service for distributed applications. Apache Zookeeper was originally a subproject of Hadoop but is now a top‑level project.


# Types of Service Discovery

There are two types of service discovery:
- Server-side
- Client-side

## Server-side

Server-side service discovery allows clients applications to find services through a router or a load balancer.
- The client makes a request to a service via a load balancer.
- The load balancer queries the service registry.
- The load balancer then routes each request to an available service instance.

![[server_side_discovery_pattern.svg]]

### Applications

#### AWS Elastic Load Balancer

The AWS Elastic Load Balancer (ELB) is an example of a server-side discovery router. An ELB is commonly used to load balance external traffic from the Internet. However, you can also use an ELB to load balance traffic that is internal to a virtual private cloud (VPC). A client makes requests (HTTP or TCP) via the ELB using its DNS name. The ELB load balances the traffic among a set of registered Elastic Compute Cloud (EC2) instances or EC2 Container Service (ECS) containers. There isn’t a separate service registry. Instead, EC2 instances and ECS containers are registered with the ELB itself.

#### NGINX

HTTP servers and load balancers such as NGINX can also be used as a server-side discovery load balancer. Using Consul Template, we can dynamically reconfigure NGINX reverse proxying. Consul Template is a tool that periodically regenerates arbitrary configuration files from configuration data stored in the Consul service registry. It runs an arbitrary shell command whenever the files change. Consul Template generates an **nginx.conf** file, which configures the reverse proxying, and then runs a command that tells NGINX to reload the configuration.

### Advantages

The benefit of this pattern is that details of discovery are abstracted away from the client. Clients simply make requests to the load balancer. This eliminates the need to implement discovery logic for each programming language and framework used by your service clients.

The advantage of Server-side service discovery is that it makes the client application lighter as it does not have to deal with the lookup procedure and makes a request for services to the router.

### Disadvantages

Unless the load balancer is provided by the deployment environment, it is yet another highly available system component that you need to set up and manage.


## Client-side

Client-side service discovery allows clients applications to find services by querying a service registry.

When using client‑side discovery, the client is responsible for determining the network locations of available service instances and load balancing requests across them. The client queries a service registry, which is a database of available service instances. The client then uses a load‑balancing algorithm to select one of the available service instances and makes a request.

The network location of a service instance is registered with the service registry when it starts up. It is removed from the service registry when the instance terminates. The service instance’s registration is typically refreshed periodically using a heartbeat mechanism.

### Applications

#### Netflix OSS

Netflix OSS provides a great example of the client‑side discovery pattern. Netflix Eureka is a service registry. It provides a REST API for managing service‑instance registration and for querying available instances. Netflix Ribbon is an IPC client that works with Eureka to load balance requests across the available service instances.

### Advantages

This pattern is relatively straightforward and, except for the service registry, there are no other moving parts. Also, since the client knows about the available services instances, it can make intelligent, application‑specific load‑balancing decisions such as using hashing consistently.

The advantage of Client-side service discovery is that the client application does not have to traffic through a router or a load balancer and therefore can avoid that extra hop.

### Disadvantages

One significant drawback of this pattern is that it couples the client with the service registry. You must implement client‑side service discovery logic for each programming language and framework used by your service clients.


>[!question] What is the role of service registration and discovery in a containerized microservices environment?

In a containerized microservices environment, service registration and discovery play a vital role in enabling dynamic communication between microservices. Here's how they work:

**Service registration**: When a microservice starts up, it registers itself with a service registry (e.g., Consul, Eureka) by providing essential information like its network location, API endpoints, and health status.

**Service discovery**: When a microservice needs to communicate with another microservice, it queries the service registry to discover the network location and endpoint details of the target service.

This dynamic discovery allows microservices to locate and interact with each other without hardcoding their locations or relying on static configurations. As new instances of services are deployed or removed, the service registry is updated accordingly. This ensures seamless communication within the containerized environment.