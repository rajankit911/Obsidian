In Java, every object has an intrinsic **monitor** (also known as an **intrinsic lock**), which is used for synchronization in multithreaded environments. The monitor ensures that only one thread can execute a synchronized block or method on the object at a time, preventing race conditions.

- Every Java object has an **intrinsic lock (monitor)**.
- When a thread enters a `synchronized` block/method, it acquires this lock.
- Other threads attempting to enter any `synchronized` block on the same object must **wait** until the lock is released.

### **Monitor States & Thread Behavior**

| State         | Description                                                    |
| ------------- | -------------------------------------------------------------- |
| **Owned**     | A thread holds the lock (executing `synchronized` code).       |
| **Entry Set** | Threads waiting to acquire the lock (blocked).                 |
| **Wait Set**  | Threads that called `wait()` and are waiting for notification. |
