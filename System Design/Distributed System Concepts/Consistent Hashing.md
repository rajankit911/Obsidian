# Traditional Hashing and Its Limitations

> In traditional hashing, a hash function is used to map each input data item to a fixed-size hash value. The hash value is then used as an index into an array or hash table to store or retrieve data. The main limitation of traditional hashing in distributed systems is that the number of servers can change, which requires rebalancing the data to ensure that it is evenly distributed across the servers. This can be time-consuming and may require moving a large amount of data.

# Introducing Consistent Hashing

- Consistent hashing is a technique used in distributed computing and distributed storage systems to efficiently distribute and balance the data across multiple nodes in a way that minimises rebalancing of data when nodes are added or removed.
- Consistent hashing addresses the limitations of traditional hashing in distributed systems.
- In consistent hashing, a hash ring is used to map the hash values to servers.
- Each server and data item is mapped to a point on the ring using a hash value between 0 and 2³² - 1 returned by consistent hash function
- The data is then assigned to the closest server on the ring in a clockwise/anti-clockwise direction.
- If a server is added or removed from the ring, only the data items that are mapped to that server and its closest neighbours on the ring need to be moved, which minimises the need for rebalancing.

# **Consistent hashing optimization**

If a node (server) fails in a system using consistent hashing, by design, all the data will be reassigned to the next server in the ring.  This means that a failing server will double the load on the next server, which, at best, will waste resources on the overloaded server and, at worst, might cause a cascade failure.

A more even redistribution on server failure can be achieved by having virtual nodes or replicas of each physical server at multiple locations on the ring.  For example, a server has 4 possible hashes and so exists in 4 places in the ring, now when this server fails, the data stored on this server is redistributed to 4 different places, so the load on any other server does not exceed an additional quarter, which is more manageable.

The number of virtual nodes per physical node can be adjusted based on the size and requirements of your system. Experiment with different values to find the optimal balance between load distribution and maintenance overhead.

# Advantages of Consistent Hashing

Consistent hashing has several advantages over traditional hashing in distributed systems.

- **consistent hashing is fault-tolerant**
	If a server fails or is removed from the ring, only the data items that are mapped to that server and its closest neighbours need to be moved.

- **consistent hashing is load-balancing**
	The data is distributed evenly across the servers on the ring.

- **consistent hashing is scalable**
	New servers can be added to the ring without requiring rebalancing of the existing data.

# Applications of Consistent Hashing

Real-world examples of consistent hashing in action include:

- In **content delivery networks (CDNs)**, consistent hashing is used to distribute content across multiple servers. This ensures that requests for the same content are served by the same server, reducing latency and improving performance (In a content delivery network, consistent hashing is used to map the content to the closest server to reduce the latency).

 - In **databases**, consistent hashing is used to shard data across multiple nodes. This allows for horizontal scaling and improves performance by reducing the load on individual nodes.

- In **caching systems**, consistent hashing is used to distribute cache entries across multiple servers. In a distributed caching system, consistent hashing is used to map the cached data to the closest cache server on the ring. This improves cache hit rates and reduces the load on individual servers


```java
public class ConsistentHashing {
	private final TreeMap<Long, String> ring;
	private final int numberOfReplicas;
	private final MessageDigest md;

	public ConsistentHashing(int numberOfReplicas) throws NoSuchAlgorithmException {
		this.ring = new TreeMap<>();
		this.numberOfReplicas = numberOfReplicas;
		this.md = MessageDigest.getInstance("MD5");
	}

	public void addServer(String server) {
		for (int i = 0; i < numberOfReplicas; i++) {
			long hash = generateHash(server + i);
			ring.put(hash, server);
		}
	}

	public void removeServer(String server) {
		for (int i = 0; i < numberOfReplicas; i++) {
			long hash = generateHash(server + i);
			ring.remove(hash);
		}
	}

	public String getServer(String key) {
		if (ring.isEmpty()) {
			return null;
		}
		long hash = generateHash(key);
		if (!ring.containsKey(hash)) {
			SortedMap<Long, String> tailMap = ring.tailMap(hash);
			hash = tailMap.isEmpty() ? ring.firstKey() : tailMap.firstKey();
		}
		return ring.get(hash);
	}

	private long generateHash(String key) {
		md.reset();
		md.update(key.getBytes());
		byte[] digest = md.digest();
//        long hash = ((long) (digest[3] & 0xFF) << 24) |
//                ((long) (digest[2] & 0xFF) << 16) |
//                ((long) (digest[1] & 0xFF) << 8) |
//                ((long) (digest[0] & 0xFF));
//        return hash;
		return new BigInteger(digest).longValue();

	}

	public static void main(String[] args) throws NoSuchAlgorithmException {
		ConsistentHashing ch = new ConsistentHashing(3);
		ch.addServer("server1");
		ch.addServer("server2");
		ch.addServer("server3");

		System.out.println("key1: is present on server: " + ch.getServer("key1"));
		System.out.println("key67890: is present on server: " + ch.getServer("key67890"));

		ch.removeServer("server1");
		System.out.println("After removing server1");

		System.out.println("key1: is present on server: " + ch.getServer("key1"));
		System.out.println("key67890: is present on server: " + ch.getServer("key67890"));
	}
}
```


### **Difference between consistent hashing and traditional hashing**

Consistent hashing and traditional hashing are two different approaches used in distributed systems for managing data distribution and load balancing. Here are the key differences between them:

1. **Handling Node Additions/Removals:**
   - **Traditional Hashing:** In traditional hashing, when a node is added or removed from the system, it can lead to a significant portion of the data being rehashed and redistributed. This can result in a substantial amount of data movement and reorganization, causing increased load and potential performance issues.
   - **Consistent Hashing:** Consistent hashing is designed to minimize data movement when nodes are added or removed. When a new node is introduced, only a fraction of the data needs to be remapped to the new node, and when a node is removed, only the data that was assigned to that node needs to be remapped.

2. **Data Distribution:**
   - **Traditional Hashing:** In traditional hashing, the hash function typically maps keys to a fixed range of values, and the distribution of data across nodes may not be uniform. This can lead to uneven load distribution among nodes.
   - **Consistent Hashing:** Consistent hashing ensures a more even distribution of data across nodes. The hash function used in consistent hashing maps keys to a circle or ring, and each node is responsible for a range of values on the circle. This provides a more balanced distribution of data, reducing the likelihood of hotspots.

3. **Load Balancing:**
   - **Traditional Hashing:** Load balancing can be a challenge in traditional hashing, especially when the distribution of keys is uneven. Nodes may end up with varying amounts of data, leading to some nodes being more heavily loaded than others.
   - **Consistent Hashing:** Consistent hashing naturally provides load balancing, as each node is responsible for a specific range of keys. This allows for a more even distribution of load among nodes.

4. **Scalability:**
   - **Traditional Hashing:** Adding or removing nodes in traditional hashing can be a complex and resource-intensive process, especially in large-scale distributed systems.
   - **Consistent Hashing:** Consistent hashing is more scalable, particularly in dynamic environments. The addition or removal of nodes has a more localized impact, reducing the need for extensive data movement.

5. **Fault Tolerance:**
   - **Traditional Hashing:** Traditional hashing systems may face challenges in maintaining fault tolerance, especially during node failures or additions.
   - **Consistent Hashing:** Consistent hashing can provide better fault tolerance since the impact of node failures is localized, and the redistribution of data is limited to the affected nodes.

In summary, consistent hashing is particularly well-suited for distributed systems where scalability, fault tolerance, and dynamic changes in the number of nodes are important considerations. It aims to minimize data movement and provide a more balanced distribution of load across nodes compared to traditional hashing approaches.
