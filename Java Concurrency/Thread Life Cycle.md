
==**NEW**==: Thread object is created but `start()` hasn't been called yet. The thread exists but isn't eligible for execution.

==**RUNNABLE**==: Thread is ready to run and waiting for CPU time. This includes threads that are actively competing for processor time.

==**RUNNING**==: Thread is currently executing on the CPU. It has been selected by the thread scheduler and is actively running code.

==**BLOCKED**==: Thread is temporarily suspended and waiting for a specific condition or resource. This includes waiting for I/O operations, lock acquisition, or sleep().

==**WAITING**==: Thread is waiting indefinitely for a specific condition or notification (like wait(), join(), or park()).

==**TERMINATED**==: Thread has completed execution either normally (run method finished) or abnormally (due to exceptions or being stopped).

The arrows show the possible transitions between states, with labels indicating what triggers each transition. For example, calling `start()` moves a thread from NEW to RUNNABLE, while calling `wait()` moves it from RUNNING to BLOCKED.


This state diagram illustrates the complete thread life cycle with five main states:

```mermaid
stateDiagram-v2
    [*] --> New : Thread created
    
    New --> Runnable : start() called
    New --> Terminated : Thread object destroyed
    
    Runnable --> Running : Scheduled by OS
    Runnable --> Terminated : Thread dies/exception
    
    Running --> Runnable : Time slice expires<br/>yield() called<br/>Higher priority thread
    Running --> Blocked : I/O operation<br/>Lock acquisition<br/>sleep() called
    Running --> Waiting : wait() called<br/>join() called<br/>park() called
    Running --> Terminated : run() method ends<br/>Uncaught exception<br/>stop() called
    
    Blocked --> Runnable : I/O completes<br/>Lock acquired<br/>sleep() timeout
    Blocked --> Terminated : interrupt() with exception
    
    Waiting --> Runnable : notify()/notifyAll()<br/>join() completes<br/>unpark() called
    Waiting --> Terminated : interrupt() with exception
    
    Terminated --> [*] : Thread cleanup
    
    note right of New
        Thread object created
        but not yet started
    end note
    
    note right of Runnable
        Ready to run
        waiting for CPU
    end note
    
    note right of Running
        Currently executing
        on CPU
    end note
    
    note right of Blocked
        Waiting for resource
        with timeout
        (I/O, locks, sleep)
    end note
    
    note right of Waiting
        Waiting indefinitely
        for notification
        (wait, join, park)
    end note
    
    note right of Terminated
        Thread execution
        completed
    end note
```

