To use NGINX as an API gateway, you can follow these steps:

1. Install NGINX: Start by installing NGINX on your server or machine. NGINX provides official installation instructions for different platforms on their website.

2. Configure NGINX: Once NGINX is installed, you need to configure it to act as an API gateway. The configuration can be done in the NGINX configuration file, typically located at `/etc/nginx/nginx.conf`. Open the file in a text editor.

3. Define upstream servers: In the NGINX configuration, you need to define the upstream servers that will handle the requests for your APIs. An upstream server represents the backend services that provide the API functionality. You can define multiple upstream servers if you have multiple APIs or backend services. Here's an example configuration:

```
http {
  upstream api_servers {
    server api1.example.com;
    server api2.example.com;
    server api3.example.com;
  }
}
```

In this example, `api_servers` is the name of the upstream group, and it includes three servers: `api1.example.com`, `api2.example.com`, and `api3.example.com`.

4. Configure proxy pass: After defining the upstream servers, you need to configure NGINX to proxy pass the requests to the backend services. Add the following configuration within the `http` block:

```
http {
  ...

  server {
    listen 80;
    
    location / {
      proxy_pass http://api_servers;
    }
  }
}
```

This configuration tells NGINX to listen on port 80 and proxy pass all requests to the defined upstream servers (`api_servers`).

5. Customize routing and other settings: NGINX provides various directives and settings that you can use to customize the routing behavior, load balancing, caching, security, and other aspects of your API gateway. You can refer to the NGINX documentation for more details on these options.

6. Save and exit: Once you have made the necessary configuration changes, save the file and exit the text editor.

7. Restart NGINX: Finally, restart NGINX to apply the changes. The command to restart NGINX varies depending on your operating system. For example, on Ubuntu, you can use the following command:

```
sudo service nginx restart
```

That's it! NGINX is now configured as an API gateway. Requests made to NGINX will be forwarded to the backend API servers based on your configuration. You can further enhance the setup by adding features like load balancing, SSL termination, rate limiting, request/response transformations, and authentication/authorization mechanisms as per your requirements.