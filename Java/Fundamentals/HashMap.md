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

>[!note]
> Hash code of null Key is 0.

## Buckets

HashMap stores elements in so-called buckets. Bucket is used to store nodes. Each node has a data structure like a LinkedList. More than one node can share the same bucket.

## Capacity and Load factor

To avoid having many buckets with multiple nodes, the capacity is doubled if 75% (the load factor) of the buckets become non-empty. The default value for the load factor is 75%, and the default initial capacity is 16. Both can be set in the constructor. A relation between bucket and capacity is as follows:

```java
int capacity = number of buckets * load factor;
```


## Calculating Index

Index minimizes the size of the array.

```java
int index = hashcode(key) & (size - 1);
```

In the following example, we want to insert a (Key, Value) pair in the HashMap.

```java
HashMap<String, Integer> map = new HashMap<>();  
map.put("Aman", 19);
```

When we call the `put()` method, then it calculates the hash code of the Key "Aman". Suppose the hash code of "Aman" is 2657860. Hence the index value for "Aman" is:

```java
Index = 2657860 & (16-1) = 4
```


# Hash Collisions

If two different keys have the same hash, the two values belonging to them will be stored in the same bucket. Inside a bucket, values are stored in a list and retrieved by looping over all elements. The cost of this is _O(n)_.

>[!note]
>As of Java 8 (see [JEP 180](https://openjdk.java.net/jeps/180)), the data structure in which the values inside one bucket are stored is changed from a list to a balanced tree if a bucket contains 8 or more values, and it’s changed back to a list if, at some point, only 6 values are left in the bucket. This improves the performance to be _O(log n)_.

