Java String Pool is the special memory region where JVM stores only one copy of each literal _Strings_. JVM saves a lot of heap space by reusing them. Difference _String_ variables can refer to the same _String_ object in the _String_ pool.


>[!question] Why Is _String_ Immutable in Java?

The _String_ is the most widely used data structure in Java applications. The key benefits of keeping String class as immutable are:
- **Caching:**
	The performance of collections that uses hash implementations like _HashMap_, _HashTable_, _HashSet_ is improved when operated with _String_ objects. When operating upon these hash implementations, `hashCode()` method is called quite frequently for bucketing. The _hashCode()_ method is overridden in _String_ class to facilitate caching, such that the hash is calculated and cached during the first _hashCode()_ call and the same value is returned ever since. Since immutability guarantees _Strings_ that their value won’t change, immutable _Strings_ would produce same hash values at the time of insertion and retrieval from the collections.
- **Security:**
	The _String_ is used to store sensitive pieces of information like usernames, passwords, connection URLs, network connections, etc. If the String were mutable, any hacker can cause a security issue in the application by changing the reference value.
- **Synchronization:**
	String objects are thread-safe implicitly. They can be shared across multiple threads running simultaneously as there is no need of synchronization for thread safety. If a thread changes the value, then instead of modifying the same, a new _String_ would be created in the _String_ pool.
- **Performance:**
	_String_ pool enhances the performance of a application by saving heap memory and faster access of hash implementations when operated with _Strings._


>[!question] How substring works in Java 6 & How does it create memory leak issue in Java 6?
