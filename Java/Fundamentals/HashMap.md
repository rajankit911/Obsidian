- HashMap is a part of the Java collection framework.
- It implements the map interface.
- It stores the data in the pair of Key and Value by using a technique called Hashing.
- It uses an array and LinkedList data structure internally for storing Key and Value.
- HashMap contains an array of Nodes and Node is represented as a class having the following fields : 

```java
class Node<K, V> {
	int hash;
	K key;
	V value;
	Node<K, V> next;
}
```


The advantage of a _HashMap_ is that the time complexity to insert and retrieve a value is _O(1)_ on average.

# Internal Working

## equals()

It is a method of the Object class. It checks the equality of two objects. It compares the Key, whether they are equal or not. If you override the `equals()` method, then it is mandatory to override the `hashCode()` method.

## hashCode()

Hashing is a process of converting an object into integer form. The integer value helps in indexing and faster searches. By default the `hashCode()` returns an integer that represents the internal memory address of the object. `hashCode()` method is used to get the hash value of an object. In HashMap, `hashCode()` is used to calculate the bucket and therefore calculate the index. It’s necessary to write the `hashCode()` method properly for better performance of HashMap.

>[!note] Hash code of null Key is 0.

## Buckets

HashMap stores elements in so-called buckets. Bucket is used to store nodes. Each node has a data structure like a LinkedList. More than one node can share the same bucket.

## Capacity and Load factor

The performance of HashMap depends on two things:
- Initial capacity
- Load factor

A load factor is the number that controls the resizing of the HashMap. If the load factor is 0.75 that means the capacity of the HashMap is doubled if the 75% of the buckets get filled. This process involves copying an array of nodes.

To avoid having many buckets with multiple nodes, the capacity is doubled if 75% (the load factor) of the buckets become non-empty.

>[!note] The default value for the load factor is 75%, and the default initial capacity is 16. Both can be set in the constructor

A relation between bucket and capacity is as follows:

```java
int capacity = number of buckets * load factor;
```


## Calculating Index

Index minimizes the size of the array.

```java
int index = hashcode(key) & (size - 1);
```

In Java HashMap, the length of the table is always a power of two (`2^x`), it means that modulo operation can be simplified by a bitwise AND (`&`). Thus `n - 1` is a mask for finding the reminder as `n - 1` is always a bit pattern having 1 at each position.

Read this article for more details: [Quotient and remainder dividing by 2^k](https://www.geeksforgeeks.org/quotient-remainder-dividing-2k-power-2/)

In the following example, we want to insert a (Key, Value) pair in the HashMap.

```java
HashMap<String, Integer> map = new HashMap<>();  
map.put("Aman", 19);
```

When we call the `put()` method, then it calculates the hash code of the Key "Aman". Suppose the hash code of "Aman" is 2657860. Hence the index value for "Aman" is:

```java
Index = 2657860 & (16-1) = 4
```


## Calculating Hash

```java
static final int hash(Object key) {
	int h; return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

This is the core function in the HashMap. Hash value of key is used to calculate the index of array (table).

As you can see, the shift operation `h >>> 16` is a transformation that spreads the impact of higher bits downward (bits >= positions 16th). The following table is created to better illustrate the changes of int `h`.

| Operation            | Binary Value                            |
| -------------------- | --------------------------------------- |
| `h = key.hashCode()` | 1111 1111 1111 1111 1111 1111 1111 1111 |
| `h >>> 16`           | 0000 0000 0000 0000 1111 1111 1111 1111 |
| `h ^ (h >>> 16)`     | 1111 1111 1111 1111 0000 0000 0000 0000 |

But why we need `h >>> 16`? Because shifting the bits allows the highest bits to participate into index calculations. Combined with the XOR operation `h ^ (h >>> 16)`, it is the cheapest possible way to do it. Here’s the Javadoc:

>Computes key.hashCode() and spreads (XORs) higher bits of hash to lower.  Because the table uses power-of-two masking, sets of hashes that vary only in bits above the current mask will always collide. (Among known examples are sets of Float keys holding consecutive whole numbers in small tables.)  So we apply a transform that spreads the impact of higher bits downward. There is a tradeoff between speed, utility, and quality of bit-spreading. Because many common sets of hashes are already reasonably distributed (so don't benefit from spreading), and because we use trees to handle large sets of collisions in bins, we just XOR some shifted bits in the cheapest possible way to reduce systematic lossage, as well as to incorporate impact of the highest bits that would otherwise never be used in index calculations because of table bounds.

For example, given `n = 32` and 4 hash codes to calculate.

| h       | h (binary)                              | h % 32 | (h ^ h >>> 16) % 32 |
| ------- | --------------------------------------- | ------ | ------------------- |
| 65,537  | 0000 0000 0000 0001 0000 0000 0000 0001 | 1      | 0                   |
| 131,073 | 0000 0000 0000 0010 0000 0000 0000 0001 | 1      | 3                   |
| 262,145 | 0000 0000 0000 0100 0000 0000 0000 0001 | 1      | 5                   |
| 524,289 | 0000 0000 0000 1000 0000 0000 0000 0001 | 1      | 1                   |

When doing the modulo directly without hash code transformation, all indexes will be 1. The collision is 100%. This is because mask 31 (`n - 1`), 0000 0000 0000 0000 0000 0000 0001 1111, makes any bit higher than position 5 un-usable in number `h`.

In order to use these highest bits, HashMap shifts them 16 positions left `h >>> 16` and spreads with lowest bits (`h ^ (h >>> 16)`). As a result, the modulo obtained has less collision.



# Hash Collisions

If two different keys have the same hash, the two values belonging to them will be stored in the same bucket. Inside a bucket, values are stored in a list and retrieved by looping over all elements. The cost of this is _O(n)_.

>[!note]
>As of Java 8 (see [JEP 180](https://openjdk.java.net/jeps/180)), the data structure in which the values inside one bucket are stored is changed from a list to a balanced tree if a bucket contains 8 or more values, and it’s changed back to a list if, at some point, only 6 values are left in the bucket. This improves the performance to be _O(log n)_.

