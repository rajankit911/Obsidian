# Difference Between Expose and Ports in Docker Compose

3 months ago

by [Rafia Zafar](https://linuxhint.com/author/raifazafar/)

Docker compose is a core component of Docker that is frequently utilized to configure the application executed on multiple containers. Docker-compose is mostly utilized to configure the services of containers in the “**YAML**” file. Different keys are used in the service configuration, “**expose**” and “**ports**” are specifically utilized to specify the exposing port for containers.

This write-up will explain the difference between the ports and expose key in Docker compose.

## **Difference Between Expose and Ports in Docker Compose**

The “**expose**” and “**ports**” keys in Docker compose are utilized to configure the network and the exposing ports for the container. However, both keys are used for the same purpose, but the key difference between the “ports” and “expose” is that the expose key is accessible to the services that are connected to the same network but not on the host. In contrast, ports are accessible and published on the host as well as on the connected network.

## **Checking the Difference Between “expose” and “ports” Keys in Docker-compose Practically**

To check the difference between expose and ports key practically, go through the listed examples:

- [Utilize “**ports**” Key in Docker-Compose File](https://linuxhint.com/difference-between-expose-and-ports-in-docker-compose/#1)
- [Utilize “**expose**” Key in Docker-Compose File](https://linuxhint.com/difference-between-expose-and-ports-in-docker-compose/#2)

## **Example 1: Utilize “ports” Key in Docker-Compose File**

The “**ports**” key is utilized to publish the container on the host machine. These containers are accessible to all services that are executing on the host as well on a connected network.

To use the “ports” key in Docker compose, check out the given instructions.

### **Step 1: Create a “docker-compose.yml”**

Make a “**docker-compose.yml**” file and paste the below code block into the file:

version: "3"  
  
services:  
  
web:  
  
image: nginx:latest  
  
ports:  
  
  - 8080:80

According to the above snippet:

- “**web**” service is configured in the “**docker-compose.yml**” file.
- “**image**” defines the base image for the compose container
- “**ports**” specify the exposing port of the container on a network and host:

![](https://linuxhint.com/wp-content/uploads/2023/02/word-image-294528-1.png)

### **Step 2: Start Containers**

Next, create and fire up the compose container with the help of “**docker-compose up**” command:

> docker-compose up -d

![](https://linuxhint.com/wp-content/uploads/2023/02/word-image-294528-2.png)

### **Step 3: List Compose Container**

List the container and verify the exposing port of the container. From the output, it can observe that we have published the container on the host:

> docker-compose ps

![](https://linuxhint.com/wp-content/uploads/2023/02/word-image-294528-3.png)

## **Example 2: Utilize “expose” Key in Docker-Compose File**

To utilize the expose key in the “**docker-compose.yml**” file, take a look at provided instructions.

### **Step 1: Create a “docker-compose.yml”**

Now, configure the “**web**” service on exposing port 80 with the help of the “**expose**” key. Here, we have not defined any network for the container:

version: "3"  
  
services:  
  
web:  
  
image: nginx:latest  
  
expose:  
  
  - 8080:80

![](https://linuxhint.com/wp-content/uploads/2023/02/word-image-294528-4.png)

### **Step 2: Fire up the Container**

Next, create and start the compose container to run web service using the provided command:

> docker-compose up -d

![](https://linuxhint.com/wp-content/uploads/2023/02/word-image-294528-5.png)

### **Step 3: List Compose Container**

List the compose container and check the exposing port of the container. From the below output, you can observe that the container is accessible only on port 80 on a default selected network but not on host:

> docker-compose ps

![](https://linuxhint.com/wp-content/uploads/2023/02/word-image-294528-6.png)

We have defined the distinction of “**expose**” and “**ports**” keys in Docker compose.

## **Conclusion**

The “**expose**” and “**ports**” are both used to specify the exposing port of the container to run defined services. The major difference between these two keys is that “ports” is published and accessible on the host machine and also on the specified network, while “expose” is only published on the defined network and accessed by services that are running on the same network. This write-up demonstrated the distinction between “ports” and “expose” in Docker compose.





## 1. Overview[](https://www.baeldung.com/ops/docker-compose-expose-vs-ports#overview)

As we know, [Docker Compose](https://www.baeldung.com/ops/docker-compose) is a tool for defining and managing multiple containers at once. By default, Docker Compose sets up a dedicated network for the defined containers, enabling communication between them. As a result, we can create and run services with a given configuration file using a single command.

In this tutorial, we'll focus on two YAML properties that allow us to customize networking between containers, _expose_ and _ports_. We'll describe them in detail, explore the basic use cases, and highlight their key differences.

## 2. _expose_ Section[](https://www.baeldung.com/ops/docker-compose-expose-vs-ports#expose-section)

First, let's look at the _expose_ configuration. This property defines the ports that Docker Compose exposes from the container.

These **ports will be accessible by other services connected to the same network, but won't be published on the host machine**.

We can expose a port by specifying its number in the _services_ section:

```yaml
services:
  myapp1:
    ...
    expose:
      - "3000"
      - "8000"
  myapp2:
    ...
    expose:
      - "5000"
```

As we can see, we can specify multiple values for each service. We just exposed the ports _3000_ and _8000_ from the _myapp1_ container, and the port _5000_ from the _myapp2_ container. The services are now accessible on these ports for other containers in the same network.

Now let's check exposed ports:

```bash
> docker ps
CONTAINER ID   IMAGE    COMMAND     CREATED     STATUS      PORTS               NAMES
8673c14f18d1   ...      ...         ...         ...         3000/tcp, 8000/tcp  bael_myapp1
bc044e180131   ...      ...         ...         ...         5000/tcp            bael_myapp2
```

In the _docker ps_ command output, we can find the exposed ports in the _PORTS_ column.

Finally, let's verify the communication between the containers:

```bash
> docker exec -it bc044e180131 /bin/bash

bash-5.1$ nc -vz myapp1 3000
myapp1 (172.18.0.1:3000) open
bash-5.1$ nc -vz myapp1 8000
myapp1 (172.18.0.1:8000) open
```

We just connected to the _myapp2_ CLI. Using the [_netcat_](https://www.baeldung.com/linux/netcat-command) command, we checked that both the ports exposed from _myapp1_ were reachable.

## 3. _ports_ Section[](https://www.baeldung.com/ops/docker-compose-expose-vs-ports#ports-section)

Now let's check the _ports_ section. As with _expose_, this property defines the ports that we want to expose from the container. But unlike with the _expose_ configuration, **these ports will be accessible internally and published on the host machine**.

Again, as before, we can define ports for each service in the dedicated section, but the configuration might be more complex. First, we have to choose between two syntaxes (short and long) to define the configuration.

### 3.1. Short Syntax[](https://www.baeldung.com/ops/docker-compose-expose-vs-ports#1-short-syntax)

Let's start by analyzing the short one. **The short syntax is a colon-separated string to set the host IP address, host port, and container port**:

```css
[HOST:]CONTAINER[/PROTOCOL]
```

Here_, HOST_ is a host port number or range of port numbers that can be preceded by an IP address. If we don't specify the IP address, Docker Compose binds the port to all the network interfaces.

C_ONTAINER_ defines a container port number or range of port numbers.

_PROTOCOL_ restricts container ports to the specified protocol or sets them to _TCP_ if empty. Only the _CONTAINER_ part is mandatory.

Now that we know the syntax, let's define the ports in our Docker Compose file:

```yaml
services:
  myapp1:
    ...
    ports:
    - "3000"                             # container port (3000), assigned to random host port
    - "3001-3005"                        # container port range (3001-3005), assigned to random host ports
    - "8000:8000"                        # container port (8000), assigned to given host port (8000)
    - "9090-9091:8080-8081"              # container port range (8080-8081), assigned to given host port range (9090-9091)
    - "127.0.0.1:8002:8002"              # container port (8002), assigned to given host port (8002) and bind to 127.0.0.1
    - "6060:6060/udp"                    # container port (6060) restricted to UDP protocol, assigned to given host (6060)
```

As presented above, we can also publish multiple container ports at once using different variants of the short syntax and configuring it more precisely. Docker Compose exposes all specified container ports, making them reachable internally and externally from the local machine.

As before, let's check the exposed ports with the _docker ps_ command:

```rust
> docker ps -a
CONTAINER ID   ... PORTS                                                                        NAMES
e8c65b9eec91   ... 0.0.0.0:51060->3000/tcp, 0.0.0.0:51063->3001/tcp, 0.0.0.0:51064->3002/tcp,   bael_myapp1
                   0.0.0.0:51065->3003/tcp, 0.0.0.0:51061->3004/tcp, 0.0.0.0:51062->3005/tcp, 
                   0.0.0.0:8000->8000/tcp, 0.0.0.0:9090->8080/tcp, 0.0.0.0:9091->8081/tcp
                   127.0.0.1:8002->8002/tcp, 0.0.0.0:6060->6060/udp
```

Once again, in the _PORTS_ column, we can find all the exposed ports. The value to the left of the arrow shows the host address where we can reach the container externally.

### 3.2. Long Syntax[](https://www.baeldung.com/ops/docker-compose-expose-vs-ports#2-long-syntax)

**Using the long syntax, we can configure the ports in the same way. However, instead of using a colon-separated string, we'll define each property individually**:

```yaml
services: 
  myapp1:
  ...
  ports:
  # - "127.0.0.1:6060:6060/udp"
  - target: 6060
    host_ip: 127.0.0.1
    published: 6060
    protocol: udp
    mode: host
```

Here, the _target_ is mandatory and specifies the container port (or range of ports) that will be exposed, which is equivalent to the _CONTAINER_ in the short syntax.

The _host_ip_ and _published_ are parts of _HOST_ in the short one, where we can define the host's IP address and port.

The _protocol,_ the same as _PROTOCOL_ in the short syntax, restricts the container port to the specified protocol (or _TCP_ if empty).

The _mode_ is the enum with two values that specifies port publishing rules. We should use the _host_ value to publish a port locally. The second value, _ingress,_ is reserved for more complex container environments, and means the port will be load balanced.

In conclusion, any short syntax string can easily be represented by a long structure. However, not all long syntax configurations can be moved to the short one due to the lack of a _mode_ property.

## 4. Conclusion[](https://www.baeldung.com/ops/docker-compose-expose-vs-ports#conclusion)

In this article, we covered part of the networking configurations in the Docker Compose. We analyzed and compared port configuration using the _expose_ and _ports_ sections.

The **_expose_ section allows us to expose specific ports from our container only to other services on the same network**. We can do this simply by specifying the container ports.

The **_ports_ section also exposes specified ports from containers**. Unlike the previous section_,_ **ports are open not only for other services on the same network, but also to the host**. The configuration is a bit more complex, where we can configure the exposed port, local binding address, and restricted protocol. Finally, depending on our preferences, we can choose between the two different syntaxes.