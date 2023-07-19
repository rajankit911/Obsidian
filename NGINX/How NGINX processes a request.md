## Multiple virtual hosts

> [!note]
> **VirtualHost** is an Apache term. NGINX does not have Virtual hosts, it has **Server Blocks** that use the `server_name` and `listen` directives to bind to tcp sockets. Although, they are basically the same thing called by different names.


NGINX allows the administrator to define multiple [[NGINX Directives#`server`|server]] blocks in its configuration file. These `server` blocks function as separate virtual web server instances. As a result, it needs a procedure for determining which of these `server` blocks will be used to handle specific inbound requests.

### Parsing the `listen` Directive to Find Possible Matches

First, NGINX looks at the IP address and the port of the inbound request. It matches this against the [[NGINX Directives#`listen`|listen]] directive of each `server` block to build a list of the `server` blocks that can possibly resolve the request. If it finds only one specific match, that `server` block will be used to serve the request. If there are multiple matching `server` blocks, then NGINX begins to evaluate [[NGINX Directives#`server_name`|server_name]] directive of each `server` block.

>[!note]
>Any `server` block that is functionally using `0.0.0.0` as its IP address, will not be selected if there are matching `server` blocks that list a specific IP address.


### Parsing the `server_name` Directive to Choose a Match

Next, to further evaluate `server` blocks that have `listen` directives of equal specificity, NGINX checks the request’s ***Host*** header. This value holds the domain or IP address that the client was actually trying to reach. NGINX attempts to match the value against the `server_name` directive of each of the `server` blocks that are still possible candidates.

if name matches more than one of the specified variants, e.g. both wildcard name and regular expression match, the first matching variant will be chosen, in the following order of precedence:

1. first matching exact name
2. longest wildcard name starting with an asterisk, e.g. `*.example.org`
3. longest wildcard name ending with an asterisk, e.g. `mail.*`
4. first matching regular expression (in order of appearance in a configuration file)
5. [[NGINX Directives#`default_server`|default_server]] block for that IP address and port.

Let’s start with a simple configuration where all three virtual servers listen on port \*:80:

```conf
server {
    listen 80;
    server_name example.org www.example.org;
    ...
}

server {
    listen 80;
    server_name example.net www.example.net;
    ...
}

server {
    listen 80;
    server_name example.com www.example.com;
    ...
}
```

