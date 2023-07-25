NGINX as a very efficient HTTP load balancer is used to distribute traffic to several application servers and to improve performance, scalability and reliability of web applications with NGINX.

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

```conf
http {
    upstream myapp1 {
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

Least-connected allows controlling the load on application instances more fairly in a situation when some of the requests take longer to complete.

With the least-connected load balancing, nginx will try not to overload a busy application server with excessive requests, distributing the new requests to a less busy server instead.

```conf
upstream myapp1 {
	least_conn;
	server srv1.example.com;
	server srv2.example.com;
	server srv3.example.com;
}
```


>[!note]
>With round-robin or least-connected load balancing, each subsequent client’s request can be potentially distributed to a different server. There is no guarantee that the same client will be always directed to the same server.


### Session persistence (ip-hash)

With ip-hash, the client’s IP address is used as a hashing key to determine what server in a server group should be selected for the client’s requests. This method ensures that the requests from the same client will always be directed to the same server except when this server is unavailable.

This way ip-hash load balancing make the client’s session “**sticky**” or “**persistent**” in terms of always tying a client to a particular application server.

```conf
upstream myapp1 {
    ip_hash;
    server srv1.example.com;
    server srv2.example.com;
    server srv3.example.com;
}
```


### Weighted load balancing

It is possible to use server weights with NGINX load balancing algorithms.
When the `weight` parameter is specified for a server, the `weight` is accounted as part of the load balancing decision.

```conf
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