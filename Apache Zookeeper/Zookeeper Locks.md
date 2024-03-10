# Introduction

ZooKeeper distributed lock is based on "Ephemeral Sequential Node". ZooKeeper automatically appends a monotonously increasing number suffix to its name as a new node name during the creation of the node and the node is deleted when the process that created it ends.

_Zookeeper Locks are fully distributed locks in ZooKeeper which are globally synchronous_. 

>[!note]
>Globally synchronous means at any snapshot in time no two clients think they hold the same lock.


![[Excalidraw/System Design/images/zookeeper_lock.svg]]

When multiple clients acquire locks at the same time, several temporary sequential nodes are created sequentially. However, only the node with the first sequence number can acquire the locks successfully. Others monitor the changes of the previous node in order. When the listener releases the locks, the listener can acquire the locks immediately.


# Implementation

We can implement these locks by creating a lock node, as with priority **ZooKeeper Queues**. Now do the following, to obtain a lock in ZooKeeper:



```java
public DistributedLock(ZooKeeper zk, String lockBasePath, String lockName) {
  this.zk = zk;
  this.lockBasePath = lockBasePath;
  this.lockName = lockName;
}
```

- Each process that wants to use the lock should instantiate an object of the DistributedLock class.
- The DistributedLock constructor takes three parameters.
	- The first parameter is _**zk**_, a reference to the ZooKeeper client.
	- The second parameter is the _**lockBasePath**_ where you want your lock nodes to reside in. Remember that ZooKeeper stores its nodes like a file system, so think of this base path as the directory you want your lock nodes created in.
	- The third parameter is the _**lockName**_ of the lock to use. Note you should use the same lock name for every process that you want to share the same lock. The lock name is the common reference that multiple processes lock on. 

>[!note]
>This class can support multiple locks if you use a different lock name for each lock you want to create. Say you have two data stores (A and B). You have several processes that need mutate A and B. You could use two different lock names (say LockA and LockB) to represent the locks for each data store. Any process that needs to mutate data store A could create a DistributedLock with a lockname of LockA. Likewise, any process that needs to mutate data store B could create a DistributedLock with a lockname of LockB. A proces that needs to mutate both datastores would create two DistributedLock objects (one with lock name of LockA and one with a lock name of LockB). 


- Once your process has created a DistributedLock object it can then call the `lock()` method to attempt to acquire the lock. The `lock()` method will block until the lock is acquired. 

```java
public void lock() throws IOException {
  try {
    // lockPath will be different than (lockBasePath + "/" + lockName)
    // becuase of the sequence number ZooKeeper appends
    lockPath = zk.create(lockBasePath + "/" + lockName, null,
        Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
    final Object lock = new Object();
    synchronized(lock) {
      while (true) {
        List < String > nodes = zk.getChildren(lockBasePath, new Watch() {
          @Override
          public void process(WatchedEvent event) {
            synchronized(lock) {
              lock.notifyAll();
            }
          }
        });
        // ZooKeeper node names can be sorted lexographically
        Collections.sort(nodes);
        if (lockPath.endsWith(nodes.get(0)) {
          return;
        } else {
          lock.wait();
        }
      }
    }
  } catch(KeeperException e) {
    throw new IOException(e);
  } catch(InterruptedException e) {
    throw new IOException(e);
  }
}
```

- First of all, the `lock()` method creates a node in ZooKeeper to represent its "position in line" waiting for the lock. The node created is EPHEMERAL which means if our process dies for some reason, its lock or request for the lock with automatically disappear thanks to ZooKeeper's node management, so we do not have worry about timing out nodes or cleaning up stale nodes. 
- ZooKeeper operates through a system of callbacks. When you call `getChildren()` you can pass in a "watcher" that will get called anytime the list of children changes. We are creating an ordered list of nodes (sharing the same name). Whenever the list changes, every process that has registered a node is notified. Since the nodes are ordered, one node will be "on top" or in other words have the lowest sequence number. That node is the node that owns the lock. When a process detects that its node is the top most node, it proceeds to execute. When it is finished, it deletes its node, triggering a notification to all other processes who then determine who the next node is who has the lock. 
- The tricky part of the code from a Java perspective is the use of nested synchronized blocks. The nested synchronization structure is used to ensure that the DistributedLock is able to process every update it gets from ZooKeeper and does not "lose" an update if two or more updates come from ZooKeeper in quick succession.
- The inner synchronized block in the Watcher method is called from an outside thread whenever ZooKeeper reports a change to its children. Since the Watcher callback is in a synchronized block keyed to the same Java lock object as the outer synchronized block, it means that the update from ZooKeeper cannot be processed until the contents of the outer synchronized block is finished. In other words, when an update comes in from ZooKeeper, it fires a` notifyAll()` which wakes up the loop in the `lock()` method. That lock method gets the updated children and sets a new Watcher. (Watchers have to be reset once they fire as they are not a perpetual callback. They fire once and then disappear.) If the newly reset Watcher fires before the rest of the loop executes, it will block because it is synchronized on the same Java lock object as the loop. The loop finishes its pass, and if it has not acquired the distributed lock, it waits on the Java lock object. This frees the Watcher to execute whenever a new update comes, repeating the cycle.
- Once the `lock()` method returns, it means your process has the distributed lock and can continue to execute its business logic. Once it is complete it can release the lock by calling the `unlock()` method. 

```java
public void unlock() throws IOException {
  try {
    zk.delete(lockPath, -1);
    lockPath = null;
  } catch(KeeperException e) {
    throw new IOException(e);
  } catch(InterruptedException e) {
    throw new IOException(e);
  }
}
```

- All `unlock()` does is explicitly delete this process's node which notifies all the other waiting processes and allows the next one in line to go.
- Since the nodes are EPHEMERAL, the process can exit without unlocking and ZooKeeper will eventually reap its node allowing the next process to execute. This is a good thing because it means if your process ends prematurely without you having a chance to call `unlock()` it will not block the remaining processes.

>[!note]
>It is best to explicitly call `unlock()` if you can, because it is much faster than waiting for ZooKeeper to reap your node. You will delay the other processes less if you explicitly unlock.


However, clients wishing to release a lock in ZooKeeper simply delete the node they created in step 1, to unlock.

Also, notice these few things in ZooKeeper Locks:

- Since each node is watched by exactly one client, the removal of a node will only cause one client to wake up. In this way, you avoid the herd effect.
- Moreover, there is no polling or timeouts.
- Also, it is easy to see the amount of lock contention, debug locking problems, break locks, etc, because of the way we implement locking.

# Shared Locks

By making some changes to the ZooKeeper Lock protocol, it is possible to implement shared locks in ZooKeeper:

## Obtaining a Read lock

1. At first, to create a node with pathname `_locknode_/read-`, Call `create()`. Although, don’t forget to set both the sequence and ephemeral flags.
2. Then without setting the watch flag, call `getChildren()` on the lock node. Basically, that helps to avoid the herd effect.
3. However, make sure the client has the lock and can exit the protocol if somehow there are no children with a pathname starting with `write-` and having a lower sequence number than the node created in step 1.
4. Else, call `exists()`, with `write-` having the next lowest sequence number set on the node in lock directory with pathname starting, also with watch flag.
5. Go to step 2, if `exists()` returns false.
6. Else, before going to step 2, wait for a notification for the pathname from the previous step.

## Obtaining a Write Lock

1. Again, with pathname `_locknode_/write-`, call `create()` to create a node. And, remember to set both sequence and ephemeral flags.
2. Further, call `getChildren()` on the lock node – because it helps to avoid the herd effect.
3. However, the client has the lock and the client exits the protocol if there are no children with a lower sequence number than the node created in step 1.
4. Further, with watch flag set, call `exists()`, on the node with the pathname which has the next lowest sequence number.
5. Goto step 2, if `exists()` returns false. Else, before going to step 2, wait for a notification for the pathname from the previous step.

# Recoverable Shared Locks

Further, we can make ZooKeeper Shared Locks revocable by modifying the shared lock protocol, with minor modifications to the Shared Lock Protocol in ZooKeeper:

Just call `getData()` with watch set, in step 1, of both obtain reader and writer lock protocols, just after the call to `create()`.

So, it created in step 1, if the client subsequently receives notification for the node, moreover, with watch set and looks for the string “unlock”, it does another getData( ) on that node, that signifies the client that now it must release the lock.

The reason behind this is, as per this shared lock protocol, writing “unlock” to that node, we can request the client with the lock give up the lock by calling setData() on the lock node.

So, this was all in ZooKeeper Locks. Hope you like our explanation.

## Conclusion

Hence, we have seen the whole about Zookeeper Locks along with Shared Locks and Recoverable Shared Locks in detail. So, we hope this article will resolve all your doubts regarding Locks in ZooKeeper. Still, if any query, ask in the comment tab.