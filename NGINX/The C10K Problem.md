The C10K problem is the problem of optimising network sockets to handle a large number of clients at the same time. The name C10k is a numeronym for concurrently handling ten thousand connections. Note that concurrent connections are not the same as requests per second, though they are similar: handling many requests per second requires high throughput, while high number of concurrent connections requires efficient scheduling of connections. The problem of socket server optimisation has been studied because a number of factors must be considered to allow a web server to support many clients. This can involve a combination of operating system constraints and web server software limitations. According to the scope of services to be made available and the capabilities of the operating system as well as hardware considerations such as multi-processing capabilities, a multi-threading model or a single threading model can be preferred. Concurrently with this aspect, which involves considerations regarding memory management, strategies implied relate to the very diverse aspects of the I/O management.


Traditionally, if a web server gets a connection, it starts a worker/thread, and the worker exists until the connection is over, even if there is no or very little data going over that connection.

Since every worker needs a certain amount of RAM, the amount of workers is limited by the amount of RAM of the server, which means the number of connections is limited by the RAM.

NGINX does not use the thread based architecture, but an event based architecture. As such it does not use a lot of resources per connection and can handle way more connections concurrently.


## Async-Await vs ThreadPool vs MultiThreading on High-Performance Sockets (C10k Solutions?)