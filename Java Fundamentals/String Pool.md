Java String Pool is the special memory region where JVM stores only one copy of each literal _Strings_. JVM saves a lot of heap space by reusing them. Difference _String_ variables can refer to the same _String_ object in the _String_ pool.


>[!question] Why Is _String_ Immutable in Java?

In Java, a `String` is immutable, meaning once it's created, it cannot be changed. There are several reasons for this design decision, which provide important benefits:

### 1. **Security**
   Since strings are widely used for sensitive information like usernames, passwords, and connection URLs, immutability ensures that strings cannot be altered after creation. This prevents unintended or malicious modifications to the data they hold, especially when passed across different components.

### 2. **String Pooling (Memory Efficiency)**
   Java uses a string pool, where identical string literals are stored and reused to save memory. Immutability allows different references to safely point to the same string instance in the pool without worrying about one reference changing the value for others. This avoids the need to duplicate strings and results in more efficient memory usage.

### 3. **Thread-Safety**
   Since `String` objects cannot be modified, they are inherently thread-safe. Multiple threads can access and share the same string instance without causing synchronization issues. This simplifies development by removing the need for locking when working with strings in concurrent environments.

### 4. **HashCode Caching**
   A `String`'s hash code is often used in data structures like `HashMap` and `HashSet`. By being immutable, a string's hash code only needs to be computed once, and then it can be cached for future use. If `String` were mutable, the hash code could change over time, making it unreliable in such collections.

### 5. **Ease of Use**
   Immutability makes `String` instances simpler to work with. You don't have to worry about unexpected changes when passing strings to methods, since the original value will always stay the same. Any modification returns a new string, leaving the original unchanged.

In summary, Java strings are immutable to improve security, performance (via string pooling and hash code caching), thread safety, and ease of use.


In Java, `String` objects are immutable, which allows them to cache their hash code once it's calculated. This optimizes performance, especially when strings are used in hash-based collections like `HashMap` and `HashSet`. Here’s how it works and an example to illustrate:

### How HashCode Caching Works
- When the `hashCode()` method is first called on a `String` object, Java calculates the hash code using its characters.
- The calculated hash code is then stored (cached) inside the `String` object.
- On subsequent calls to `hashCode()`, the cached value is returned without recalculating it, which improves efficiency.
  
Since strings are immutable, their content cannot change after creation, making it safe to cache the hash code without worrying about consistency issues.

### Example of HashCode Caching

```java
public class HashCodeCachingExample {
    public static void main(String[] args) {
        String str = "immutable";

        // First call to hashCode() - the hash code is calculated and cached
        int hash1 = str.hashCode();
        System.out.println("First hash code: " + hash1);

        // Second call to hashCode() - the cached value is returned, no recalculation
        int hash2 = str.hashCode();
        System.out.println("Second hash code (cached): " + hash2);

        // Checking if both hash codes are the same
        System.out.println("Hash codes are the same: " + (hash1 == hash2));
    }
}
```

### Breakdown:
1. **First Call to `hashCode()`**: 
   - The hash code is calculated based on the characters of the string.
   - Java then caches this hash code inside the `String` object.
   - The calculated hash code is returned.

2. **Second Call to `hashCode()`**:
   - Instead of recalculating, Java returns the cached hash code value.
   - Since the string hasn't changed (and can’t change, as it’s immutable), the cached value is always correct.

### How HashCode is Calculated

The `hashCode()` method in the `String` class is typically implemented like this:

```java
public int hashCode() {
    int h = 0;
    int len = value.length;
    if (h == 0 && len > 0) {
        for (int i = 0; i < len; i++) {
            h = 31 * h + value[i]; // value[i] is the character at index i
        }
        hash = h;
    }
    return h;
}
```

In this method:
- If `h` (the hash code) is 0, the hash code is calculated by iterating over the characters of the string.
- Once the hash is computed, it’s stored in the variable `hash`.
- On future calls to `hashCode()`, if `hash` is non-zero, it simply returns the cached value.

### Why Caching Improves Performance

Imagine a large string being used repeatedly in a `HashMap`. Without caching, every time the map looks up the string’s position, it would have to recompute the hash code, which could be computationally expensive. With caching, the hash code is computed just once, reducing overhead and speeding up operations. This optimization becomes more noticeable in situations where many `hashCode()` calls occur on the same string.


>[!question] How substring works in Java 6 & How does it create memory leak issue in Java 6?
