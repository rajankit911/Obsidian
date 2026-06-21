# The Problem with Traditional Blocking I/O

In a traditional, multi-threaded server architecture (such as Apache or early Tomcat), the network model relies on a **"one-thread-per-connection"** design. When a client connects to the server, the operating system assigns a dedicated thread to handle that specific socket.

This approach works fine for a small number of fast connections, but it introduces a massive scalability bottleneck when traffic scales up. When a client is slow to send data over the network, or if the connection is simply sitting idle (like a Keep-Alive connection), the assigned thread is forced to wait—it enters a **Blocked** or sleeping state. 

Here is why this architecture fails at scale:

1. **Context Switching Overhead**: If you have 10,000 connected clients, the OS must manage 10,000 threads. The CPU ends up wasting an enormous amount of processing power constantly swapping these threads in and out of the CPU cores (context switching) just to check if any of them finally have data to read.
2. **Memory Waste**: Every thread requires its own memory stack allocated by the OS (typically 1MB to 8MB). 10,000 idle threads can instantly consume gigabytes of RAM while doing absolutely zero productive work.
3. **Resource Trapping**: As shown in the diagram below, threads connected to idle sockets are trapped waiting. They cannot be reassigned to help process the active sockets.

![[traditional_blocking_io.svg]]

*In the architectural illustration above, Thread 1 and Thread 3 are blocked waiting for network I/O from their respective idle clients. Only Thread 2 is actively processing data. The server is wasting memory and CPU cycles managing the sleeping threads.*


## The Blocking Server Approach (The "Blind Wait")

In a traditional blocking server (like older Apache or standard Tomcat), this exhaustion manifests through dedicated threads waiting blindly. There is a dedicated "Acceptor" thread whose only job is to run an infinite loop calling `accept()` over and over.

1. **The Block:** When the Acceptor thread calls `accept()`, if there are no new clients in the OS's Accept Queue, the OS puts the entire thread to sleep. The thread is "blocked" waiting at the door.
2. **The Hand-off:** When a client finally connects, `accept()` wakes up and returns the new Client Socket. 
3. **The Spawn:** The Acceptor thread immediately hands this new Client Socket over to a *Worker Thread* (either by spawning a new one or pulling from a thread pool). 
4. **The Second Block:** That Worker Thread then calls `read()` on the Client Socket. If the client hasn't sent an HTTP request yet, that Worker Thread goes to sleep (blocks) waiting for data. The Acceptor thread loops back to call `accept()` again.

**The Flaw:** Every step involves a thread blindly calling a function (`accept` or `read`) and getting put to sleep by the OS until the data arrives.


# IO Multiplexing


One basic concept of Linux (actually Unix) is the rule that everything in Unix/Linux is a file. Each process has a table of file descriptors that point to files, sockets, devices and other operating system objects.

Typical system that works with many IO sources has an initializaion phase and then enter some kind of standby mode – wait for any client to send request and response it

Simple solution is to create a thread (or process) for each client , block on read until a request is sent and write a response. This is working ok with a small amount of clients but if we want to scale it to hundred of clients, creating a thread for each client is a bad idea


The solution is to use a kernel mechanism for polling over a set of file descriptors. There are 3 options you can use in Linux:

- select(2)
- poll(2)
- epoll

All the above methods serve the same idea, create a set of file descriptors , tell the kernel what would you like to do with each file descriptor (read, write, ..) and use one thread to block on one function call until at least one file descriptor requested operation available

### **Select System Call**

The select( ) system call provides a mechanism for implementing synchronous multiplexing I/O

int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);

A call to select( ) will block until the given file descriptors are ready to perform I/O, or until an optionally specified timeout has elapsed

The watched file descriptors are broken into three sets

- File descriptors listed in the readfds set are watched to see if data is available for reading.
- File descriptors listed in the writefds set are watched to see if a write operation will complete without blocking.
- File descriptors in the exceptfds set are watched to see if an exception has occurred, or if out-of-band data is available (these states apply only to sockets).

A given set may be NULL, in which case select( ) does not watch for that event.

On successful return, each set is modified such that it contains only the file descriptors that are ready for I/O of the type delineated by that set

Example:
```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <wait.h>
#include <signal.h>
#include <errno.h>
#include <sys/select.h>
#include <sys/time.h>
#include <unistd.h>

#define MAXBUF 256

void child_process(void)
{
  sleep(2);
  char msg[MAXBUF];
  struct sockaddr_in addr = {0};
  int n, sockfd,num=1;
  srandom(getpid());
  /* Create socket and connect to server */
  sockfd = socket(AF_INET, SOCK_STREAM, 0);
  addr.sin_family = AF_INET;
  addr.sin_port = htons(2000);
  addr.sin_addr.s_addr = inet_addr("127.0.0.1");

  connect(sockfd, (struct sockaddr*)&addr, sizeof(addr));

  printf("child {%d} connected \n", getpid());
  while(1){
        int sl = (random() % 10 ) +  1;
        num++;
     	sleep(sl);
  	sprintf (msg, "Test message %d from client %d", num, getpid());
  	n = write(sockfd, msg, strlen(msg));	/* Send message */
  }

}

int main()
{
  char buffer[MAXBUF];
  int fds[5];
  struct sockaddr_in addr;
  struct sockaddr_in client;
  int addrlen, n,i,max=0;;
  int sockfd, commfd;
  fd_set rset;
  for(i=0;i<5;i++)
  {
  	if(fork() == 0)
  	{
  		child_process();
  		exit(0);
  	}
  }

  sockfd = socket(AF_INET, SOCK_STREAM, 0);
  memset(&addr, 0, sizeof (addr));
  addr.sin_family = AF_INET;
  addr.sin_port = htons(2000);
  addr.sin_addr.s_addr = INADDR_ANY;
  bind(sockfd,(struct sockaddr*)&addr ,sizeof(addr));
  listen (sockfd, 5); 

  for (i=0;i<5;i++) 
  {
    memset(&client, 0, sizeof (client));
    addrlen = sizeof(client);
    fds[i] = accept(sockfd,(struct sockaddr*)&client, &addrlen);
    if(fds[i] > max)
    	max = fds[i];
  }
  
  while(1){
	FD_ZERO(&rset);
  	for (i = 0; i< 5; i++ ) {
  		FD_SET(fds[i],&rset);
  	}

   	puts("round again");
	select(max+1, &rset, NULL, NULL, NULL);

	for(i=0;i<5;i++) {
		if (FD_ISSET(fds[i], &rset)){
			memset(buffer,0,MAXBUF);
			read(fds[i], buffer, MAXBUF);
			puts(buffer);
		}
	}	
  }
  return 0;
}
```


We start with creating 5 child processes , each process connect to the server and send messages to the server. The server process uses accept(2) to create a different file descriptor for each client. The first parameter in select(2) should be the highest-numbered file descriptor in any of the three sets, plus 1 so we check the max fd num

The main infinite loop creates a set of all file descriptors, call select and on return check which file descriptor is ready for read. For simplicity I didn’t add error checking

On return , select changes the set to contains only the file descriptors that is ready so we need to build the set again every iteration.

The reason we need to tell select what is the highest-numbered file descriptor is the inner implementation of fd_set. Each fd is declared by a bit so fd_set is an array of 32 integers (32 *32bit = 1024 bits). The function check any bit to see if its set until it reach the maximum. This means that if we have 5 file descriptors but the highest number is 900 , the function will check any bit from 0 to 900 to find the file descriptors to watch. There is a posix alternative to select – pselect which add a signal mask while waiting (see man page)

**Select – summary:**

- We need to build each set before each call
- The function check any bit up to the higher number – O(n)
- We need to iterate over the file descriptors to check if it exists on the set returned from select
- The main advantage of select is the fact that it is very portable – every unix like OS has it

### Poll System call

Unlike select(), with its inefficient three bitmask-based sets of file descriptors, poll( ) employs a single array of nfds pollfd structures. the prototype is simpler:

int poll (struct pollfd *fds, unsigned int nfds, int timeout);

The structure pollfd has a different fields for the events and the returning events so we don’t need to build it each time:

```c
struct pollfd {
      int fd;
      short events; 
      short revents;
};
```


For each file descriptor build an object of type pollfd and fill the required events. after poll returns check the revents field

To change the above example to use poll:

```c
for (i=0;i<5;i++) 
  {
    memset(&client, 0, sizeof (client));
    addrlen = sizeof(client);
    pollfds[i].fd = accept(sockfd,(struct sockaddr*)&client, &addrlen);
    pollfds[i].events = POLLIN;
  }
  sleep(1);
  while(1){
  	puts("round again");
	poll(pollfds, 5, 50000);

	for(i=0;i<5;i++) {
		if (pollfds[i].revents & POLLIN){
			pollfds[i].revents = 0;
			memset(buffer,0,MAXBUF);
			read(pollfds[i].fd, buffer, MAXBUF);
			puts(buffer);
		}
	}
  }

```
  
Like we did with select , we need to check each pollfd object to see if its file descriptor is ready but we don’t need to build the set each iteration

#### Poll vs Select

- poll( ) does not require that the user calculate the value of the highest- numbered file descriptor +1
- poll( ) is more efficient for large-valued file descriptors. Imagine watching a single file descriptor with the value 900 via select()—the kernel would have to check each bit of each passed-in set, up to the 900th bit.
- select( )’s file descriptor sets are statically sized.
- With select( ), the file descriptor sets are reconstructed on return, so each subsequent call must reinitialize them. The poll( ) system call separates the input (events field) from the output (revents field), allowing the array to be reused without change.
- The timeout parameter to select( ) is undefined on return. Portable code needs to reinitialize it. This is not an issue with pselect( )
- select( ) is more portable, as some Unix systems do not support poll( )

### Epoll* System Calls

While working with select and poll we manage everything on user space and we send the sets on each call to wait. To add another socket we need to add it to the set and call select/poll again.

Epoll* system calls help us to create and manage the context in the kernel. We divide the task to 3 steps:

- create a context in the kernel using epoll_create
- add and remove file descriptors to/from the context using epoll_ctl
- wait for events in the context using epoll_wait

Lets change the above example to use epoll:

```c
struct epoll_event events[5];
  int epfd = epoll_create(10);
  ...
  ...
  for (i=0;i<5;i++) 
  {
    static struct epoll_event ev;
    memset(&client, 0, sizeof (client));
    addrlen = sizeof(client);
    ev.data.fd = accept(sockfd,(struct sockaddr*)&client, &addrlen);
    ev.events = EPOLLIN;
    epoll_ctl(epfd, EPOLL_CTL_ADD, ev.data.fd, &ev); 
  }
  
  while(1){
  	puts("round again");
  	nfds = epoll_wait(epfd, events, 5, 10000);
	
	for(i=0;i<nfds;i++) {
			memset(buffer,0,MAXBUF);
			read(events[i].data.fd, buffer, MAXBUF);
			puts(buffer);
	}
  }
```

  

We first create a context (the parameter is ignored but has to be positive). When a client connect we create an epoll_event object and add it to the context and on the infinite loop we are only waiting on the context.

#### **Epoll vs Select/Poll**

- We can add and remove file descriptor while waiting
- epoll_wait returns only the objects with ready file descriptors
- epoll has better performance – O(1) instead of O(n)
- epoll can behave as level triggered or edge triggered (see man page)
- epoll is Linux specific so non portable


## select, poll & epoll



Viewed The%20Silent%20Scalability%20of%20IO%20Multiplexing.md:1-56

To answer your questions, we need to look at how the operating system handles networking under the hood. 

Here is exactly how the registration works and what types of connections are involved:

### 1. Which type of connection does this mean?
It actually means **both** the server-side listening socket *and* the individual client connections! They are just different types of "File Descriptors" (FDs) to the kernel.

- **The Server Listening Socket:** When Redis starts up, it creates a "server socket" bound to port 6379. It registers this single socket with the kernel. When a read event happens on *this* specific socket, it means **a new client is trying to connect**. The application accepts the connection.
- **The Client Sockets:** When the application accepts that new client, the OS creates a brand new, unique socket specifically for communicating with that one client. The application immediately takes this new client socket and registers it with the kernel too. When a read event happens on a *client* socket, it means **the client has sent a command** (like `GET key`).

### 2. How does the application register them to the kernel?
It uses specific **System Calls** (APIs provided by the OS). On Linux, the mechanism is `epoll`, and it works in three steps:

1. **`epoll_create` (The Setup):** When the application starts, it asks the kernel to create a special "watch list" in the kernel's memory. The kernel returns an ID for this watch list.
2. **`epoll_ctl` (The Registration):** This is the actual registration step. The application uses this system call to tell the kernel: *"Add socket #14 to the watch list. Let me know when it has data ready to be read (an `EPOLLIN` event)."* Every time a new client connects, the application runs `epoll_ctl` to register the new client's socket.
3. **`epoll_wait` (The Yield):** This is where the application goes to sleep. The event loop calls `epoll_wait`, effectively saying: *"Pause my thread until one or more of the sockets on the watch list have an event."* 

Because the kernel manages the network hardware directly, the moment a network card receives a packet from a client, the kernel identifies which socket it belongs to. If that socket is on the `epoll` watch list, the kernel instantly wakes up the application thread and hands it a list of exactly which sockets are ready. 

This is the beauty of the design: the application never loops through checking sockets; it simply registers them via `epoll_ctl` and waits for the kernel to push the notifications via `epoll_wait`!


Edited gen_socket.py
Ran command: `python3 /Users/cs04652/.gemini/antigravity/brain/cae7629b-ad45-4d9a-bf49-240dd74789f7/scratch/gen_socket.py`

I have created a new visualization for you to complement this explanation! You can view it here: **[socket_acceptance.excalidraw.md](file:///Users/cs04652/Obsidian/Excalidraw/Redis/socket_acceptance.excalidraw.md)**

Let’s break down exactly what happens when a client connects to a server, bit by bit. To make it easy to understand, think of a server like a large corporate office building.

### 1. The Listening Socket (The Receptionist)
When Redis starts up, it tells the Operating System: *"I want to claim Port 6379, and I want to listen for anyone trying to talk to me."* 
The OS creates a **Listening Socket**. You can think of this socket as the **Company Receptionist**. 
- The Receptionist's **only job** is to stand at the front door (Port 6379) and wait for new people to arrive. 
- The Receptionist **never** conducts business logic. They don't process data, they don't fulfill orders. They just greet people.

### 2. The Client Arrives (The TCP Handshake)
Let's say `Client A` (an app on IP 192.168.1.5) wants to talk to Redis. 
Client A sends a network packet (a `SYN` packet) over the network directed at your server's Port 6379.
Because the OS knows the Receptionist (the Listening Socket) is standing at Port 6379, the OS delivers this packet to the Receptionist. The OS and the Client complete the 3-way TCP handshake (SYN -> SYN-ACK -> ACK). 

### 3. The `accept()` System Call (Hiring a Dedicated Rep)
Now that the handshake is complete, the client is waiting in the lobby. The Receptionist notifies the Redis application: *"Hey, someone is here to see you."*

In response, the Redis application makes a crucial system call to the OS named **`accept()`**. 
When Redis calls `accept()`, it is telling the OS: *"Okay, take this person out of the lobby and give them a dedicated representative."*

This is the magic moment: **The OS creates a brand new, unique File Descriptor (a new socket)**.
- This new socket is **not** the Receptionist.
- It is a **Dedicated Representative** assigned *exclusively* to `Client A`. 
- From this millisecond forward, all data sent by `Client A` flows directly to this Dedicated Representative (the new socket), completely bypassing the Receptionist.

### 4. Registration and the Cycle Continues
Now Redis has this new, unique socket specifically for Client A. Redis takes this new socket and registers it with the `epoll` kernel mechanism we discussed earlier, saying: *"Wake me up whenever Client A's Dedicated Representative has data to read."*

Meanwhile, the Receptionist (the original Listening Socket) immediately goes back to standing at the front door, waiting to greet the next person (Client B). 

### Summary of the "Two Socket" Architecture
Because of this design, the server never gets confused about who sent what.
- **Socket #1 (Listening):** `0.0.0.0:6379` <-> `*.*.*.*:*` (Waits for anyone)
- **Socket #2 (Connected):** `10.0.0.1:6379` <-> `192.168.1.5:40112` (Talks ONLY to Client A)
- **Socket #3 (Connected):** `10.0.0.1:6379` <-> `192.168.1.6:51992` (Talks ONLY to Client B)

The OS handles routing the incoming network packets to the exact right socket based on the source IP and source Port attached to the packet!



This is one of the most important distinctions in network programming! 

To answer your question directly: **The 3-way TCP handshake takes place entirely between the OS Kernel and the Client.** Redis (the application) is completely oblivious to the handshake while it is happening.

Here is the exact step-by-step breakdown of how the Kernel and Redis divide the labor:

### Phase 1: The OS Kernel Handles the Handshake Autonomously
Because TCP is a fundamental networking protocol, the Operating System's internal TCP/IP stack manages it. Redis doesn't have to write any code to reply to `SYN` packets; the OS does it automatically.

1. **The `SYN` Arrives:** The Client sends a `SYN` packet to Port 6379. The OS Kernel receives it. The Kernel looks up Port 6379, sees that Redis is listening there, and says, *"Okay, a valid application is here."*
2. **The SYN Queue (Half-Open):** The Kernel creates a temporary internal record for this connection and places it in a waiting room called the **SYN Queue** (sometimes called the incomplete connection queue).
3. **The `SYN-ACK` Reply:** The Kernel automatically formulates a `SYN-ACK` packet and sends it back to the Client.
4. **The `ACK` Arrives:** The Client receives the `SYN-ACK` and sends back the final `ACK` packet.
5. **The Accept Queue (Ready):** Upon receiving the final `ACK`, the Kernel marks the 3-way handshake as completely successful. It moves the connection out of the SYN Queue and into a new waiting room called the **Accept Queue** (the completed connection queue).

*During this entire 3-way handshake, the Redis application was completely uninvolved. It was either serving other clients or sleeping.*

### Phase 2: Redis Gets Involved
Now that the connection is fully established and sitting in the OS's Accept Queue, the Kernel needs to tell Redis about it.

1. **The Notification:** Because Redis registered its Listening Socket with `epoll` (as we discussed earlier), the Kernel flags that Listening Socket as "Ready for Reading." It wakes up the Redis event loop.
2. **Redis Investigates:** Redis's event loop wakes up, looks at the `epoll` event, and says: *"Ah, my Listening Socket triggered a read event! That means the OS must have a fully established connection waiting for me in the Accept Queue."*
3. **The `accept()` Call:** Redis calls the `accept()` function. This is essentially Redis telling the OS: *"Hand me the next person in the Accept Queue."*
4. **The Handoff:** The OS pulls the connection out of the Accept Queue and hands Redis the **Brand New File Descriptor** (the dedicated client socket we visualized in the Excalidraw diagram).

### Why this division of labor is brilliant:
If Redis had to manually process `SYN` and `ACK` packets, it would waste precious CPU cycles on protocol overhead instead of reading and writing data from memory. By having the OS Kernel handle the entire 3-way handshake autonomously, the OS shields Redis from network latency and guarantees that Redis only ever spends CPU time on connections that are 100% established and ready to go!


Ah, that is a sharp observation! You caught a compression in my explanation. 

To clear this up, `epoll_ctl` is actually used **multiple times** for completely different sockets. It is used once at server startup for the Listening Socket, and then used *again* every time a new client connects. 

Here is the exact, chronological sequence of C system calls Redis executes from the moment you start the server, to the moment it accepts its first client:

### Phase 1: Server Initialization (Startup)
When you run `./redis-server`, it executes these steps before any client is allowed to connect:

1. **`socket()` & `bind()` & `listen()`**: 
   Redis creates the foundational Listening Socket (The Receptionist) and binds it to Port 6379.
2. **`epoll_create()`**: 
   Redis asks the OS to create the master "watch list" in the kernel. The OS returns an ID (a file descriptor) for this watch list.
3. **`epoll_ctl( ADD, Listening_Socket )`**: 
   *Here is the first use of `epoll_ctl`!* Redis immediately adds the Listening Socket to the watch list. It tells the kernel: *"Wake me up if the Listening Socket gets an `EPOLLIN` (read) event."* (For a listening socket, a "read" event specifically means "a new connection is sitting in the Accept Queue ready to be pulled").

### Phase 2: The Event Loop Starts
4. **`epoll_wait()`**: 
   Redis enters its infinite `while(true)` event loop. It calls `epoll_wait()`, which puts the Redis thread to sleep. It says to the kernel: *"I'm going to sleep now. Wake me up if anything on the watch list happens."* Currently, only the Listening Socket is on that list.

### Phase 3: The Client Arrives
5. **The Handshake (OS only)**: 
   The client sends a `SYN`. The OS handles the 3-way handshake entirely in the background. Once complete, the OS puts the connection in the Accept Queue.
6. **The Wake Up**: 
   Because the OS sees a completed connection for the Listening Socket, it checks the watch list, sees Redis is watching it, and wakes Redis up from `epoll_wait()`.

### Phase 4: Acceptance and Registration
7. **`accept()`**: 
   Redis wakes up, sees the event was triggered by the Listening Socket, and calls `accept()`. The OS gives Redis the **Brand New Client Socket**.
8. **`epoll_ctl( ADD, New_Client_Socket )`**: 
   *Here is the second use of `epoll_ctl`!* Now that Redis has a dedicated socket for this new client, it must add *this new socket* to the watch list too. It tells the OS: *"Add this new client socket to my existing watch list. Wake me up if this client sends a command (like `GET key`)."*
9. **Loop Repeats**: 
   Redis immediately goes back to step 4 and calls `epoll_wait()` again.

### The State of the Watch List
After this single client connects, if you were to look at the kernel's `epoll` watch list, it now contains **two** entries:
1. `Listening Socket (Port 6379)` — Waiting for *brand new* clients to connect.
2. `Client Socket A` — Waiting for the first client to send a command.

If 10,000 clients connect, `accept()` and `epoll_ctl` will be called 10,000 times, and the kernel's watch list will grow to contain 10,001 entries (1 Listening Socket + 10,000 Client Sockets). 

Does mapping it out to the exact system calls help clarify exactly when `epoll_ctl` comes into the picture?

## The Hidden Burden of Idle Time

In reality, the overwhelming majority of network connections at any given moment are doing absolutely nothing.  As shown in the diagram below, out of tens of thousands of connections, only a tiny fraction are actively sending or receiving data. The rest are completely open, but inactive. 

![[idle_active_connections.svg]]

How does a system wait for data without constantly checking for it? The naive strategy is asking *"Is something ready?"* again and again, scanning every single connection sequentially. This is like walking down an endless hallway knocking on every door just to see if someone is behind it.
The result is that massive CPU time is wasted checking idle connections, driving up CPU load even when no actual work is being done.

## Flipping the Architectural Question: Delegating to the Kernel

Instead of the application checking everything, modern scalable systems **wait for signals**. They flip the script from *"Are you ready yet?"* to *"Tell me only when something becomes ready"*.
This works by delegating the responsibility to the **OS Kernel**. The kernel physically manages all network interfaces and hardware interrupts, so it inherently knows exactly when data arrives. The application thread simply registers the connections with the kernel and goes to sleep.

![[kernel_delegation.excalidraw.md]]

### The Redis Approach (The "Event-Driven Wait")

In Redis, the socket is explicitly put into **Non-Blocking Mode**. Redis never blindly calls `accept()`.

* **The Watch List:** Instead of a dedicated thread sleeping on `accept()`, the single Redis thread sleeps on `epoll_wait()`, watching the Listening Socket from a distance.
* **The Guaranteed Accept:** Redis only calls `accept()` when `epoll` specifically signals: *"Hey, I guarantee there is a client in the Accept Queue."* Because of this, the `accept()` call returns the new Client Socket instantly. The thread never blocks.
* **No Spawning:** Instead of handing this new Client Socket to a Worker Thread, Redis simply adds the new Client Socket to the `epoll` watch list, right alongside the Listening Socket!
* **The Shared Wait:** Now, the single Redis thread goes back to sleeping on `epoll_wait()`. It is now simultaneously waiting for *either* a brand new connection on the Listening Socket, *or* a command on the Client Socket.

## Powering the Event Loop

This delegation is the foundation of **I/O Multiplexing** (via `epoll` on Linux or `kqueue` on macOS/BSD), which powers the single-threaded Event Loop pattern used by high-performance systems like Redis and Node.js. 
The cycle is beautifully efficient:
1. **WAIT**: The application yields control to the kernel.
2. **WAKE**: The kernel wakes the application *only* when a registered File Descriptor (socket) has actionable data.
3. **PROCESS**: The event loop executes the logic for the ready sockets, without any wasted CPU cycles.

![[event_loop_cycle.excalidraw.md]]

## The Scalability Stack

Ultimately, a highly scalable, non-blocking architecture relies on a structured stack of responsibilities where work is done only when strictly needed:
- **Application (Business Logic)**: Processes data without worrying about networking polling.
- **Event Loop (Routing)**: Routes signals to the right application handlers.
- **I/O Multiplexing (Notifier)**: The mechanism (`epoll`/`kqueue`) that bridges the OS and the event loop.
- **OS Kernel (Hardware Watcher)**: The absolute source of truth for network activity.

![[scalability_stack.excalidraw.md]]
