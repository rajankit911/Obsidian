NGINX as a very efficient HTTP load balancer is used to distribute traffic to several application servers and to improve performance, scalability and reliability of web applications with NGINX.

### Load balancing methods

The following load balancing mechanisms (or methods) are supported in NGINX:
- `round-robin`:
	Requests to the application servers are distributed in a round-robin fashion
- `least-connected`:
	Next request is assigned to the server with the least number of active connections
- `ip-hash`:
	A hash-function is used to determine what server should be selected for the next request (based on the client’s IP address)

### Default load balancing configuration

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

