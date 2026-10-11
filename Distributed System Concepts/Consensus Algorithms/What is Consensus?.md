Consensus is a fundamental problem in fault-tolerant distributed systems. Consensus involves multiple servers agreeing on values. Once they reach a decision on a value, that decision is final. Typical consensus algorithms make progress when any majority of their servers is available; for example, a cluster of 5 servers can continue to operate even if 2 servers fail. If more servers fail, they stop making progress (but will never return an incorrect result).

# Why Do We Need Consensus?

In a single-node system, maintaining state is trivial. In a distributed system, we replicate state across multiple nodes to achieve **fault tolerance** and **high availability**. However, networks can drop, delay, or reorder packets, and nodes can crash at any moment.

# Replicated State Machine (RSM)

If **all nodes start in the same initial state** and **execute the exact same deterministic sequence of commands in the same order**, they are guaranteed to arrive at the **exact same final state**.

```
Client Command (SET x=5) ──► [ Consensus Module ] ──► Replicated Log ──► State Machine ──► Result
```

Therefore, the problem of distributed consensus reduces to: **keeping the replicated log identical across a majority (quorum) of nodes.**


Consensus typically arises in the context of replicated state machines, a general approach to building fault-tolerant systems. Each server has a state machine and a log. The state machine is the component that we want to make fault-tolerant, such as a hash table. It will appear to clients that they are interacting with a single, reliable state machine, even if a minority of the servers in the cluster fail. Each state machine takes as input commands from its log. In our hash table example, the log would include commands like _set x to 3_. A consensus algorithm is used to agree on the commands in the servers' logs. The consensus algorithm must ensure that if any state machine applies _set x to 3_ as the nth command, no other state machine will ever apply a different nth command. As a result, each state machine processes the same series of commands and thus produces the same series of results and arrives at the same series of states.

