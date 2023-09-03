NGINX rewrite rules are used to change entire or a part of the URL requested by a client.
URL rewrite can be used to: 
- control the request within the NGINX
- inform the client that requested resource has changed its location

## Example

In the example below, there are 3 REST services on different servers:
- **serviceA-api** running on `api.serverA.com`
- **serviceB-api** running on `api.serverB.com`
- **serviceC-api** running on `api.serverC.com`


![[nginx-rewrite.png]]

I need to host these web services with single public IP address/domain: `www.abc.com`. In order to serve **serviceA**, **serviceB** and **serviceC** APIs, I need to rewrite the client URLs from the NGINX

```nginx
server {
    listen 80;
    server_name abc.com www.abc.com;

    location /api/serviceA {
	    rewrite ^/api/(.*)$ /serviceA/$1 break;
        proxy_pass http://api.serverA.com;
    }

    location /api/serviceB {
        rewrite ^/api/(.*)$ /serviceB/$1 break;
        proxy_pass http://api.serverB.com;
    }

    location /api/serviceC {
        rewrite ^/api/(.*)$ /serviceC/$1 break;
        proxy_pass http://api.serverC.com;
    }
}
```

