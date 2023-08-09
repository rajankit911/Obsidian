NGINX uses a text‑based configuration file written in a particular format. By default the file is named `nginx.conf` and can be located in one of:

```bash
/etc/nginx/
/usr/local/etc/nginx
/usr/local/nginx/conf
```


## Feature-Specific Configuration Files

To make the configuration easier to maintain, we recommend that you split it into a set of feature‑specific files stored in the `/etc/nginx/conf.d` directory and use the `include` directive in the main `nginx.conf` file to reference the contents of the feature‑specific files.

```nginx
include conf.d/http;
include conf.d/stream;
include conf.d/exchange-enhanced;
```

## Contexts

A few top‑level directives, referred to as _contexts_, group together the directives that apply to different traffic types:

- `events` – General connection processing
- `http` – HTTP traffic
- `mail` – Mail traffic
- `stream` – TCP and UDP traffic

Directives placed outside of these contexts are said to be in the `main` context. The following configuration illustrates the use of contexts.

```nginx
user nobody; # a directive in the 'main' context
error_log logs/error.log notice;
worker_processes number | auto ;

events {
    # configuration of connection processing
}

http {
    # Configuration specific to HTTP and affecting all virtual servers  

    server {
        # configuration of HTTP virtual server 1       
        location /one {
            # configuration for processing URIs starting with '/one'
        }
        location /two {
            # configuration for processing URIs starting with '/two'
        }
    } 
    
    server {
        # configuration of HTTP virtual server 2
    }
}

stream {
    # Configuration specific to TCP/UDP and affecting all virtual servers
    server {
        # configuration of TCP virtual server 1 
    }
}
```

## Controlling NGINX

To reload your configuration, you can stop or restart NGINX, or send signals to the master process. A signal can be sent by running the `nginx` command (invoking the NGINX executable) with the `-s` argument.

```sh
nginx -s <SIGNAL>
```

where `<SIGNAL>` can be one of the following:

- `quit` – Shut down gracefully (the `SIGQUIT` signal)
- `reload` – Reload the configuration file (the `SIGHUP` signal)
- `reopen` – Reopen log files (the `SIGUSR1` signal)
- `stop` – Shut down immediately (or fast shutdown, the `SIGTERM` signal)

The `kill` utility can also be used to send a signal directly to the master process. The process ID of the master process is written, by default, to the **nginx.pid** file, which is located in one of the below directories:
```sh
/usr/local/nginx/logs
/var/run
```


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


