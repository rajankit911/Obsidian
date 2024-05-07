- [ ] What is the difference between a thread and a process?
- [ ] What is the difference between synchronous and asynchronous programming?
- [ ] What is a thread scheduler, and how is it related to thread priority?
- [ ] Why are the methods notify(), wait() and notifyAll() in object class?
- [ ] How are the methods wait() and sleep() different?
- [x] What is a daemon thread?
- [x] How do you create a daemon thread in Java?
- [ ] Why is it important to override the run() method in a thread class?
- [x] How can you achieve thread safety?
- [ ] In Java, what does the keyword volatile mean?
- [ ] Explain how you would stop the execution of a thread in Java.
- [ ] Explain what ThreadLocal is.
- [ ] What is a timer class in Java?
- [ ] What do you understand about BlockingQueue?
- [x] What do you understand by Race-condition?
- [x] How is thread safety achieved in Java?
- [x] What is ThreadScheduler in Java?
- [ ] What is Thread Pooling in Java?
- [ ] What are the main components to be considered while developing concurrency API?
- [ ] What are synchronized blocks, and what is their purpose?
- [x] What is a shutdown hook?
- [x] What are Daemon threads?
- [ ] What is Atomic action in Java?
- [ ] What are some multithreading best practices while coding in Java?
- [ ] How do you create a thread-safe Singleton?
- [ ] What would you do if threads throw exceptions within a synchronized block?
- [ ] What are the main differences between the submit() and execute() functions in Multithreading?
- [ ] What is the function of the Yield method in a Thread-class?


In Java, multithreading is a feature that allows concurrent execution of multiple parts of a program. This maximises CPU utilisation. Each part that is executed is called a thread, and they act like components within a process.Multithreading has many benefits and uses in the following real-time scenarios:

- Running background processes like Tomcat webserver
- Performing executions when there is a blocked input or output
- Developing video games in which every interactive element is a thread
- Programming reservation and ticketing systems to allow several users to interact with the system simultaneously

# Advantages

Java multithreading offers several benefits, making it a powerful tool for concurrent programming:

1. **Concurrency**: Multithreading allows multiple tasks to execute concurrently within the same program. This can lead to significant performance improvements by utilizing the available CPU resources more efficiently.
    
2. **Responsiveness**: Multithreading enables an application to remain responsive even when performing long-running tasks. By executing tasks concurrently, the user interface remains active and responsive to user input.
    
3. **Resource Utilization**: Multithreading allows better utilization of system resources, such as CPU cores. It enables parallel execution of tasks, maximizing the usage of available computing power.
    
4. **Scalability**: Multithreading facilitates scalability by allowing applications to handle multiple concurrent requests efficiently. This is particularly beneficial for server-side applications, where handling multiple client requests simultaneously is crucial for performance.
    
5. **Simplified Programming**: Java provides high-level abstractions for multithreading through its concurrency API (java.util.concurrent package). This makes it easier for developers to implement multithreaded applications without having to deal with low-level details like thread management and synchronization.
    
6. **Improved Performance**: Multithreading can lead to performance improvements, especially in applications that can be parallelized effectively. Tasks such as I/O operations, network communication, and CPU-intensive computations can benefit from multithreading.
    
7. **Asynchronous Programming**: Multithreading enables asynchronous programming, where tasks can execute independently of each other. This is useful for handling tasks that involve waiting for external events, such as I/O operations or network requests, without blocking the execution of other tasks.
    
8. **Modular Design**: Multithreading allows for a modular design of applications by breaking down complex tasks into smaller, manageable threads. This promotes better code organization and maintainability.
    
9. **Real-time Systems**: Java multithreading is essential for building real-time systems that require timely responses to events. By utilizing multiple threads, applications can meet stringent timing requirements and maintain responsiveness.
    
10. **Parallel Algorithms**: Multithreading enables the implementation of parallel algorithms, where different parts of a computation can be executed concurrently. This can lead to significant performance gains, especially in computations that can be parallelized effectively, such as matrix multiplication or sorting algorithms.
    

Overall, Java multithreading provides developers with the tools to build efficient, responsive, and scalable applications that can take full advantage of modern hardware architectures. However, it's essential to use multithreading judiciously and be aware of potential issues such as thread safety and synchronization.



# Thread Pool

A Java thread pool is a collection of pre-initialized threads that are ready to perform tasks. Instead of creating and destroying threads for each task, a thread pool reuses existing threads, which improves performance and reduces overhead. Thread pools are commonly used in multithreaded applications, particularly those involving concurrent execution of tasks.

Here are some key concepts related to Java thread pools:

1. **ThreadPoolExecutor**: In Java, the `ThreadPoolExecutor` class, located in the `java.util.concurrent` package, provides an implementation of a thread pool. It allows you to create and manage a pool of threads, specifying parameters such as the core pool size, maximum pool size, and the queue for holding tasks.

2. **Core Pool Size**: This represents the number of threads initially created in the pool, which remain alive even when idle, unless the thread pool is explicitly configured otherwise. These threads are used to execute tasks as soon as they arrive.

3. **Maximum Pool Size**: This sets the maximum number of threads that the thread pool can contain. If there are more tasks than the core pool size can handle, and the queue for holding tasks is full, additional threads up to the maximum pool size will be created to handle the workload.

4. **Queue**: The task queue holds tasks that are waiting to be executed by the threads in the pool. When the pool's capacity is reached (core pool size + queue size), new tasks may be queued until threads become available or until the maximum pool size is reached.

5. **Thread Reuse**: Threads in a thread pool are reused to execute multiple tasks. Once a thread finishes executing a task, it can pick up another task from the queue, avoiding the overhead of thread creation and destruction.

6. **Thread Pool Executors**: The `Executors` class provides factory methods for creating different types of thread pools, such as fixed-size thread pools, cached thread pools, and scheduled thread pools. These methods simplify the process of creating and managing thread pools.

Using a thread pool offers several benefits, including improved performance, better resource management, and simplified thread management. However, it's essential to choose appropriate pool parameters (such as core pool size, maximum pool size, and queue type) based on the application's requirements and workload characteristics to ensure optimal performance. Additionally, proper error handling and task cancellation mechanisms should be implemented to handle exceptional situations gracefully.

# Thread Life Cycle

In Java, a thread can go through various states during its lifetime. These states are defined by the `Thread` class and are represented by constants in the `Thread.State` enum. The possible states that a thread can be in are as follows:

1. **NEW**: When a thread is created but not yet started, it is in the NEW state. In this state, the thread has been instantiated, but the `start()` method hasn't been invoked yet.

2. **RUNNABLE**: A thread in the RUNNABLE state is currently executing its task or is ready to run, waiting for CPU time. This includes threads that are executing, as well as threads that are waiting for their turn to execute in a runnable thread pool.

3. **BLOCKED**: Threads enter the BLOCKED state when they are waiting for a monitor lock to enter a synchronized block or method. This occurs when a thread attempts to enter a synchronized block or method that is already held by another thread.

4. **WAITING**: Threads enter the WAITING state when they are waiting indefinitely for another thread to perform a particular action. This could be waiting for notification (`Object.wait()` with no timeout), waiting for a condition (`LockSupport.park()` with no timeout), or waiting for a thread join (`Thread.join()` with no timeout).

5. **TIMED_WAITING**: Threads enter the TIMED_WAITING state when they are waiting for another thread to perform a particular action, but with a specified timeout. This includes waiting with timeout (`Object.wait()` with a timeout, `Thread.sleep()`, or `LockSupport.parkNanos()`).

6. **TERMINATED**: A thread enters the TERMINATED state when it has completed its execution or terminated due to an uncaught exception. Once a thread is terminated, it cannot be started again.

These states represent the lifecycle of a thread from its creation to its termination. Threads can transition between these states based on various factors, such as acquiring locks, waiting for conditions, or completing their tasks. Understanding these states is crucial for effective thread management and debugging in Java applications.

# Race Condition

A race condition is a situation that occurs in a multithreaded environment when two or more threads attempt to modify shared data at the same time. The final outcome of the program depends on the relative timing of these thread executions, and it may produce unexpected or incorrect results.

Race conditions typically arise due to the interleaving of thread execution, where the order of execution of threads is unpredictable. They can occur when:

1. **Shared Data**: Multiple threads access and modify shared data concurrently.
    
2. **Unsynchronized Access**: Access to shared data is not properly synchronized using mechanisms such as locks, mutexes, or synchronization blocks.
    
3. **Non-Atomic Operations**: Operations on shared data are not atomic, meaning they consist of multiple steps that can be interrupted by other threads.


