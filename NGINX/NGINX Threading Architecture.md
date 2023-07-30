NGINX has one master process and one or more worker processes. If caching is enabled, the cache loader and cache manager processes also run at startup.

The main purpose of the master process is to read and evaluate configuration files, as well as maintain the worker processes.

The worker processes do the actual processing of requests. NGINX relies on OS-dependent mechanisms to efficiently distribute requests among worker processes. The number of worker processes is defined by the `worker_processes` directive in the **nginx.conf** configuration file and can either be set to a fixed number or configured to adjust automatically to the number of available CPU cores.

```nginx
user             nobody;
error_log        logs/error.log notice;
worker_processes number | auto ;
```


When NGINX reverse proxy starts it creates one thread per CPU core and these worker threads do the heavy lifting. The number of worker threads are configurable but NGINX recommends one thread per CPU core to avoid context switching and cache thrashing.

In older versions of NGINX all threads accept connections by competing on the shared listener socket (by default only one process can listen on IP/port pair).


![500x500](https://img-c.udemycdn.com/redactor/raw/article_lecture/2022-09-21_07-44-53-dd44c4e9b5c80252829fb2953e35d7c1.png)


In recent versions of NGINX this was changed to use socket sharding (through SO_REUSEPORT socket option) which allows multiple threads to listen on the same port and the OS will load balance connections on each accept queue.

![500x500](https://img-c.udemycdn.com/redactor/raw/article_lecture/2022-09-21_07-44-53-b3d0db8792220f750c7c49778592914f.png)