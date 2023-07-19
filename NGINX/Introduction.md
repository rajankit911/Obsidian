**NGINX** – pronounced as “**Engine X**” –  is an:
* open source
* fast (content & application delivery)
* secure
* lightweight
* high-performance
* high-concurrency
* low resource usage
* scalable
web server.

Server admins often prefer NGINX over Apache httpd for two main reasons:
- It can handle a higher number of concurrent requests.
- It has faster static content delivery with low resource usage.

Earlier many web servers were not able to handle more than 10,000 connections simultaneously, also known as [[Introduction#The C10k Problem|C10K Problem]]. **NGINX** was created by Igor Sysoev, with its 1st public release on October 2004 as an attempt to answer the **C10k problem**.

It can be used as:
- Web Server
	Serves web content

- Reverse Proxy
	- Load Balancing
	- Backend Routing
	- Caching
	- API Gateway
	- Rate Limiting


## The C10k Problem

The C10K problem is the problem of optimising network sockets to handle a large number of clients at the same time. The name C10k is a numeronym for concurrently handling ten thousand connections. Note that concurrent connections are not the same as requests per second, though they are similar: handling many requests per second requires high throughput, while high number of concurrent connections requires efficient scheduling of connections. The problem of socket server optimisation has been studied because a number of factors must be considered to allow a web server to support many clients. This can involve a combination of operating system constraints and web server software limitations. According to the scope of services to be made available and the capabilities of the operating system as well as hardware considerations such as multi-processing capabilities, a multi-threading model or a single threading model can be preferred. Concurrently with this aspect, which involves considerations regarding memory management, strategies implied relate to the very diverse aspects of the I/O management.


Traditionally, if a web server gets a connection, it starts a worker/thread, and the worker exists until the connection is over, even if there is no or very little data going over that connection.

Since every worker needs a certain amount of RAM, the amount of workers is limited by the amount of RAM of the server, which means the number of connections is limited by the RAM.

NGINX does not use the thread based architecture, but an event based architecture. As such it does not use a lot of resources per connection and can handle way more connections concurrently.


## Async-Await vs ThreadPool vs MultiThreading on High-Performance Sockets (C10k Solutions?)


```sh
docker run -d -p 80:80 --name webserver nginx
```

```sh
root@d2b089304854:/# apt update && apt install -y procps
root@d2b089304854:/# ps -C nginx -f
```


By default, the NGINX configuration file can be located in:

```bash
/etc/nginx/nginx.conf
```


/var/log/nginx/*.log

## Reload NGINX

We need to restart or reload Nginx whenever we make changes to its configuration.
The reload option will load the new configuration, start new worker processes with the new configuration and gracefully shut down old worker processes.

```bash
root@d2b089304854:/# nginx -s reload
```

## Test NGINX Configuration

Whenever we make changes or edit something to the Nginx server's configuration file, it is a good idea to test the configuration before restarting or reloading the service.

```bash
root@d2b089304854:/# nginx -t (or configtest)
```

