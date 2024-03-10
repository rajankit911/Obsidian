
Apache ZooKeeper is a distributed, open-source coordination service for distributed applications and it exposes a simple set of primitives that can be used by distributed application to implement higher level services for synchronization, configuration maintenance, groups and naming.


Zookeeper is a distributed system for managing coordination across a cluster of machines. 

By definition of apache, ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. All of these kinds of services are used in some form or another by distributed applications. Each time they are implemented there is a lot of work that goes into fixing the bugs and race conditions that are inevitable. Because of the difficulty of implementing these kinds of services, applications initially usually skimp on them, which make them brittle in the presence of change and difficult to manage. Even when done correctly, different implementations of these services lead to management complexity when the applications are deployed.


Zookeeper manages information as a hierarchical system of "nodes" (much like a file system). Each node can contain data or can contain child nodes. 

Zookeeper supports several types of nodes. A node can be either "ephemeral" or "persistent" meaning it is either deleted when the process that created it ends or it remains until manually deleted. A node can also be "sequential" meaning each time a node is created with a given name, a sequence number is postfixed to that name. This allows you to create a series of nodes with the same name that are ordered in the same order they were created.