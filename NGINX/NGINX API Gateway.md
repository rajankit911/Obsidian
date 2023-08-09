API gateways can be used for both monolithic and microservices-based apps. API gateways perform several functions, including:
- **Request routing**:
	Routing requests to appropriate backends
- **Authentication**:
	Authenticating the requesters making API calls (**AuthN**) and verifying that the requester is authorized to make the request (**AuthZ**)
- **Rate Limiting**:
	Applying rate limits to prevent overloading of the systems and mitigate DDoS attacks
- **SSL/TLS Termination**:
	Offloading SSL/TLS traffic to improve performance
- **Exception handling**:
	Handling errors and exceptions


## Implementation

To use NGINX as an API gateway, you can follow these steps:

1. Install NGINX: Start by installing NGINX on your server or machine. NGINX provides official installation instructions for different platforms on their website.

2. Configure NGINX: Once NGINX is installed, you need to configure it to act as an API gateway. The configuration can be done in the NGINX configuration file, typically located at `/etc/nginx/nginx.conf`. Open the file in a text editor.

3. Define upstream servers: In the NGINX configuration, you need to define the upstream servers that will handle the requests for your APIs. An upstream server represents the backend services that provide the API functionality. You can define multiple upstream servers if you have multiple APIs or backend services. Here's an example configuration:

```nginx
http {
	# Define upstream blocks for each microservice
	upstream service1 {
		server service1.example.com;
	}

	upstream service2 {
		server service2.example.com;
	}

	upstream service3 {
		server service3.example.com;
	}
}
```


4. Configure proxy pass: After defining the upstream servers, you need to configure NGINX to proxy pass the requests to the backend services. Add the following configuration within the `http` block:

```nginx
http {
	...

	server {
	    listen 80;
	    server_name api.example.com;
    
		location /service1 {
			# Route requests to the service1
			proxy_pass http://service1;
	    }

		location /service2 {
			# Route requests to the service2
			proxy_pass http://service2;
	    }

		location /service3 {
			# Route requests to the service3
			proxy_pass http://service3;
	    }
  }
}
```

In the above configuration, the `proxy_pass` line is necessary for NGINX to function as a reverse proxy and correctly forward requests to the backend server.

5. Save and exit: Once you have made the necessary configuration changes, save the file and exit the text editor.

6. Restart NGINX: Finally, restart NGINX to apply the changes. The command to restart NGINX varies depending on your operating system. For example, on Ubuntu, you can use the following command:

```
sudo service nginx restart
```

That's it! NGINX is now configured as an API gateway. Requests made to NGINX will be forwarded to the backend API servers based on your configuration. You can further enhance the setup by adding features like load balancing, SSL termination, rate limiting, request/response transformations, and authentication/authorization mechanisms as per your requirements.