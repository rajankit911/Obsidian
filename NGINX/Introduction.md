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

Earlier many web servers were not able to handle more than 10,000 connections simultaneously, also known as [[The C10K Problem]]. **NGINX** was created by Igor Sysoev, with its 1st public release on October 2004 as an attempt to answer the **C10k problem**.

It can be used as:
- Web Server
	Serves web content

- Reverse Proxy
	- Load Balancing
	- Backend Routing
	- Caching
	- API Gateway
	- Rate Limiting


## Running NGINX in a Docker Container


You can create an NGINX instance in a Docker container using the NGINX Open Source image from the Docker Hub.

```sh
docker run -d -p 80:80 --name webserver nginx
```

- `docker run`: Command to create and start a new container.
- `-d`: Detached mode, which runs the container in the background.
- `-p 80:80`: Maps port 80 of the host to port 80 of the container, allowing you to access Nginx on the host machine at `[http://localhost](http://localhost./)`[.](http://localhost./)
- `--name webserver`: Assigns the name `webserver` to the running container.
- `nginx`: The name of the Docker image you want to use for the container.

Verify that the container is running master and worker process

```sh
root@d2b089304854:/# apt update && apt install -y procps
root@d2b089304854:/# ps -C nginx -f
```
