NGINX as a very efficient HTTP load balancer is used to distribute traffic to several application servers and to improve performance, scalability and reliability of web applications with NGINX.

### Proxying HTTP Traffic to a Group of Servers

First you need to define the group of servers with the `upstream` directive which is placed in the `http` context.

Servers in the group are configured using the `server` directive (not to be confused with the `server` block that defines a virtual server running on NGINX).

To pass requests to a server group, the name of the group is specified in the `proxy_pass` directive (or the `fastcgi_pass`, `memcached_pass`, `scgi_pass`, or `uwsgi_pass` directives for those protocols.)

In the below example, a virtual server running on NGINX passes all requests to the **backend** upstream group:

```nginx
http {
    upstream backend {
        server backend1.example.com;
        server backend2.example.com;
        server 192.0.0.1 backup;
    }
    
    server {
        location / {
            proxy_pass http://backend;
        }
    }
}
```



### Load balancing methods

The following load balancing mechanisms (or methods) are supported in NGINX:
- `round-robin`:
	Requests to the application servers are distributed in a round-robin fashion
- `least-connected`:
	Next request is assigned to the server with the least number of active connections
- `ip-hash`:
	A hash-function is used to determine what server should be selected for the next request (based on the client’s IP address)

>[!note]
>When the load balancing method is not specifically configured, it defaults to round-robin.

### Default load balancing configuration (round-robin)

Requests are distributed evenly across the servers.

```nginx
http {
    upstream myapp1 {
		# no load balancing method is specified for Round Robin
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://myapp1;
        }
    }
}
```

In the example above, there are 3 instances of the same application running on:
- srv1.example.com
- srv2.example.com
- srv3.example.com
All requests are proxied to the server group `myapp1`, and NGINX applies HTTP load balancing to distribute the requests.

Reverse proxy implementation in NGINX includes load balancing for HTTP, HTTPS, FastCGI, uwsgi, SCGI, memcached, and gRPC.

>To configure load balancing for HTTPS instead of HTTP, just use “https” as the protocol.


### Least connected load balancing

With the least-connected load balancing, NGINX will try not to overload a busy application server with excessive requests, distributing the new requests to the server with least number of active connections.

Least-connected allows controlling the load on application instances more fairly in a situation when some of the requests take longer to complete.

```nginx
upstream myapp1 {
	least_conn;
	server srv1.example.com;
	server srv2.example.com;
	server srv3.example.com;
}
```


>[!note]
>With round-robin or least-connected load balancing, each subsequent client’s request can be potentially distributed to a different server. There is no guarantee that the same client will be always directed to the same server.


### Session persistence 

#### ip-hash

With ip-hash, the client’s IP address is used as a hashing key to determine what server in a server group should be selected for the client’s requests. In this case, either the first three octets of the IPv4 address or the whole IPv6 address are used to calculate the hash value. This method ensures that the requests from the same client will always be directed to the same server except when this server is unavailable.

This way ip-hash load balancing make the client’s session “**sticky**” or “**persistent**” in terms of always tying a client to a particular application server.

```nginx
upstream myapp1 {
    ip_hash;
    server srv1.example.com;
    server srv2.example.com;
    server srv3.example.com;
}
```

If one of the servers needs to be temporarily removed from the load‑balancing rotation, it can be marked with the `down` parameter in order to preserve the current hashing of client IP addresses. Requests that were to be processed by this server are automatically sent to the next server in the group:

```nginx
upstream backend {
	ip_hash;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com down;
}
```

#### hash

The server to which a request is sent is determined from a user‑defined key which can be a text string, variable, or a combination. For example, the key may be a paired source IP address and port, or a URI as in this example:

```nginx
upstream backend {
    hash $request_uri consistent;
    server backend1.example.com;
    server backend2.example.com;
}
```

If the `consistent` parameter is specified, the `ketama` consistent hashing method will be used instead. If an upstream server is added to or removed from an upstream group, only a few keys are remapped to different servers which minimizes cache misses. This helps to achieve a higher cache hit ratio for caching servers.

### Weighted load balancing (Server Weights)

By default, NGINX distributes requests among the servers in the group according to their weights using the Round Robin method. When the `weight` parameter is specified for a server, the `weight` is accounted as part of the load balancing decision; its default value is 1.

```nginx
upstream myapp1 {
	server srv1.example.com weight=3;
	server srv2.example.com;
	server srv3.example.com;
}
```

With this configuration, every 5 new requests will be distributed across the application instances as the following:
- 3 requests will be directed to srv1.example.com
- 1 request will go to srv2.example.com
- 1 request will go to srv3.example.com