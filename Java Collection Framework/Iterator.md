- Iterators in Java are part of the Java Collection framework.
- They are used to retrieve elements one by one.

The Java Collection supports two types of iterators:
- Fail-fast
- Fail-safe

**Fail-Fast systems abort operation as-fast-as-possible exposing failures immediately and stopping the whole operation.**

Whereas, **Fail-Safe systems don’t abort an operation in the case of a failure. Such systems try to avoid raising failures as much as possible.**

# Fail-fast Iterator

- The Fail-fast iterators immediately throw `ConcurrentModificationException` in case of structural modification of the collection.
- Structural modification means adding, removing, updating the value of an element in a data collection while another thread is iterating over that collection.
- The Fail-fast iterator uses an original collection to traverse over the collection's elements.
- It doesn't require extra memory and hence saves memory.

**Default iterators for _Collections_ from _java.util package_ such as _ArrayList_, _HashMap_, etc. are Fail-Fast.**

```java
ArrayList<Integer> numbers = // ...

Iterator<Integer> iterator = numbers.iterator();
while (iterator.hasNext()) {
    Integer number = iterator.next();
    numbers.add(50);
}
```

In the code snippet above, the `ConcurrentModificationException` gets thrown at the beginning of a next iteration cycle after the modification was performed.

## Internal Working of Fail-fast Iterator

_Collections_ maintain an internal counter called `modCount`. Each time an item is added or removed from the _Collection_, this counter gets incremented.

When the iterator is initialized, it stores the `modCount` value into the `expectedModCount` as below:

```java
int expectedModCount = modCount;
```

When iterating, on each `next()` call, the current value of `modCount` gets compared with the `expectedModCount` value. If there is any change made in the collection, the `modCount` will change and then an exception is thrown using: 

```java
if (modCount != expectedModCount) {
	throw new ConcurrentModificationException();
}
```

So if we make any changes to the collection, the `modCount` will change, and `expectedModCount` will not be hence equal to the `modCount`. Then if we use any of the above methods of iterator, the `ConcurrentModificationException` will be thrown.

If during iteration over a _Collection_, an item is removed using _Iterator_‘s `remove()` method, that’s entirely safe and doesn’t throw an exception.

> [!note]
> If we remove/add the element using the `remove()` or `add()` of iterator instead of collection, then in that case no exception will occur. It is because the remove/add methods of iterators call the remove/add method of collection internally, and also it reassigns the `expectedModCount` to new `modCount` value.

```java
// Iterator -> remove()
ArrayList.this.remove(lastRet);  
cursor = lastRet;  
lastRet = -1;  
expectedModCount = modCount;

// ListIterator -> add()
int i = cursor;
ArrayList.this.add(i, e);
cursor = i + 1;
lastRet = -1;
expectedModCount = modCount;
```

So the below code is safe as we are removing the element from the iterator here:

```java

Iterator<Integer> itr = integers.iterator();  
while (itr.hasNext()) {  
    if (itr.next() == 2) {  
        // will not throw Exception  
        itr.remove();  
    }  
}
```

Whereas the below code will throw an exception as we are removing the element from the collection here:

```java
Iterator<Integer> itr = integers.iterator();  
while (itr.hasNext()) {  
    if (itr.next() == 3) {  
        // will throw Exception on  
        // next call of next() method  
        integers.remove(3);  
    }  
}
```

# Fail-safe Iterator

The Fail-safe iterator is just opposite to Fail-fast iterator.
- A fail-safe iterator does not throw any exceptions if the collection is modified during the iteration process.
- It creates a clone of the actual _Collection_ except in the case of `ConcurrentHashMap` and iterate over it.
- If any modification happens after the iterator is created, the copy still remains untouched.
- It requires more memory as it clones the collection.

```java
Integer[] nums = new Integer[] { 1, 7, 9, 11 };
CopyOnWriteArrayList<Integer> list = new CopyOnWriteArrayList<Integer>(nums);
Iterator itr = list.iterator();
while (itr.hasNext()) {
	Integer i = (Integer)itr.next();
	System.out.println(i);
	if (i == 7) {
		list.add(15);  // It will not be printed  
		//This means it has created a separate copy of the collection  
	}
}
```

**_Iterators_ on _Collections_ from _java.util.concurrent_ package such as _ConcurrentHashMap_, _CopyOnWriteArrayList_, etc. are Fail-Safe in nature.**

The Fail-Safe _Iterators_ have a few disadvantages, though. One disadvantage is that the **_Iterator_ isn’t guaranteed to return updated data from the _Collection_**, as it’s working on the clone instead of the actual _Collection_.

Another disadvantage is the overhead of creating a copy of the _Collection_, both regarding time and memory.

In the code snippet above, we’re using Fail-Safe _Iterator_. Hence, even though a new element is added to the _Collection_ during the iteration, it doesn’t throw an exception.

The default iterator for the _ConcurrentHashMap_ is weakly consistent. This means that this _Iterator_ can tolerate concurrent modification, traverses elements as they existed when _Iterator_ was constructed and may (but isn’t guaranteed to) reflect modifications to the _Collection_ after the construction of the _Iterator_.

Hence, in the code snippet above, the iteration loops five times, which means **it does detect the newly added element to the _Collection_.**


## Internal Working of Fail-safe Iterator

Unlike the fail-fast iterators, these iterators traverse over the clone of the collection. So even if the original collection gets structurally modified, no exception will be thrown.

E.g. in case of CopyOnWriteArrayList the original collections is passed and is stored in the iterator:

```java
public Iterator<E> iterator() {  
    return new COWIterator<E>(getArray(), 0);  
}
```


here iterator() method returns the iterator of the CopyOnWriteArrayList. As we can see, it passes the getArray() in the constructor of the iterator. This getArray() has all the collection elements.  
Now the iterator(COWIterator here) will save this to traverse upon as:

```java
COWIterator(Object[] elements, int initialCursor) {  
    cursor = initialCursor;  
    snapshot = elements;  
}
```

So the original collection elements are saved in the snapshot field variable.

So all the iterator methods will work on this snapshot method. So even if there is any change in the original collection, no exception will be thrown. But note the the iterator will not reflect the latest state of the collection.

```java
Iterator<Integer> itr = integers.iterator();  
while (itr.hasNext()) {  
    int a = itr.next();  
    if (a == 1) {  
        integers.remove(Integer._valueOf_(a));  
    }  
    System._out_.print(a);  
}
```

So as we are removing the element from the collection. the collection now has only elements **2 and 3** in it. But the iterator will print all the elements **1,2,3** because it traverses over the snapshot of the collection elements.

We can print the collection elements after the above code. It will print only **2,3** as:

```java
Iterator<Integer> itr = integers.iterator();  
while (itr.hasNext()) {  
    int a = itr.next();  
    System._out_.print(a);  
}
```

> **Note:** although it does not throw any exception, but the downsides of this iterator are:  
> 1. They will not reflect the latest state of the collection.  
> 2. It requires extra memory as it clones the collection.

# Difference between Fail-fast & Fail-safe Iterator

| Base of Comparison     | Fail-fast Iterator                                                                                  | Fail-safe Iterator                                              |
| ---------------------- | --------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| **Exception**          | It throws a `ConcurrentModificationException` in modifying the object during the iteration process. | It does not throw Exception.                                    |
| **Clone Object**       | No clone object is created during the iteration process.                                            | A copy or clone object is created during the iteration process. |
| **Memory utilization** | It requires low memory during the process.                                                          | It requires more memory during the process.                     |
| **Modification**       | It does not allow modification during iteration.                                                    | It allows modification during the iteration process.            |
| **Performance**        | It is fast.                                                                                         | It is slightly slower than Fail Fast.                           |
| **Examples**           | HashMap, ArrayList, Vector, HashSet, etc                                                            | CopyOnWriteArrayList, ConcurrentHashMap, etc.                   |
# ConcurrentModificationException

- `java.util.ConcurrentModificationException` is a very common exception when working with Java collection classes.
- Java Collection classes are [[Iterator#Fail-fast Iterator|fail-fast]], which means if the Collection will be modified concurrently while some thread is traversing over it using iterator, the `iterator.next()` will throw `ConcurrentModificationException`.
- `ConcurrentModificationException` can occur in the case of multithreaded as well as a single-threaded Java programming environment.

```java
List<String> myList = new ArrayList<String>();
myList.add("1");
myList.add("2");

Iterator<String> it = myList.iterator();
while (it.hasNext()) {
	String value = it.next();
	System.out.println("List Value:" + value);
	if (value.equals("2")) {
		myList.remove(value);
	}
}
```

Above program will throw `java.util.ConcurrentModificationException` when executed.

>[!question]- How iterator checks for the modification?
>The `modCount` variable is an internal field used by Java's collection classes to keep track of the number of structural modifications to the collection, such as adding or removing elements.
>1. When a collection is created, its `modCount` is initialized to zero. Every time a structural modification is made to the collection, the `modCount` is incremented.
>2. When an iterator is created for the collection, it takes a snapshot of the current `modCount` value into `expectedModCount`.
>3. During iteration, the iterator checks whether the `modCount` of the collection has changed since the iterator was created. If a modification is detected (i.e., the `modCount` is different than the `expectedModCount`), the iterator throws a `ConcurrentModificationException`.

>[!Note] In case of `HashMap`, The value of `modCount` will change when we add a new key-value pair or remove it. Updating the value of existing key will not change the `modCount`.


```java
Map<String, String> myMap = new HashMap<String, String>();
myMap.put("1", "1");
myMap.put("2", "2");
myMap.put("3", "3");

Iterator<String> it1 = myMap.keySet().iterator();
while (it1.hasNext()) {
	String key = it1.next();
	System.out.println("Map Value:" + myMap.get(key));
	if (key.equals("2")) {
		myMap.put("1", "4");
		// myMap.put("4", "4");
	}
}
```


Since we are updating the existing key value in the myMap, its size has not been changed, so there is no `ConcurrentModificationException` being thrown above. If you will uncomment the statement where I am adding a new key-value in the HashMap, it will cause `ConcurrentModificationException`.


>[!question]- How to avoid ConcurrentModificationException in a single-threaded environment?
> 1. By using iterator's `remove()` function, you can remove an object from an underlying collection object. But in this case, you can remove the iterating object and not any other object from the list.
> 2. When using a `List`, you can use a regular for loop with an index to remove or add elements without causing a `ConcurrentModificationException`.
> 3. You can iterate over a copy of the collection and modify the original collection as needed.
> 4. Java 8 introduced the `removeIf()` method, which allows you to remove elements based on a predicate.
> 5. We can filter the list using Java Streams API.


>[!question]- How to avoid ConcurrentModificationException in a multi-threaded environment?
> To avoid the ConcurrentModificationException in a multi-threaded environment, we can follow the following ways-
> 1. You can convert the list to an array and then iterate on the array, so that instead of iterating over the collection class, we can iterate over the array. In this way, we can work very well with small-sized lists, but if the list is large then it will affect the performance a lot.
> 2. Another way can be locking the list by putting it in the synchronized block or passing it into the `Collections.synchronizedList`. This is not an effective approach as the sole purpose of using multi-threading is relinquished by this.
> 3. If you are using JDK 1.5 or higher, then you can use `ConcurrentHashMap` and `CopyOnWriteArrayList` classes. These classes help us in avoiding concurrent modification exception.


# Need for Concurrent Collections

As we already know [Collections](https://www.geeksforgeeks.org/collections-in-java-2/) which is nothing but collections of Objects where we deals with the Objects using some pre-defined methods. But There are several problems which occurs when we use Collections concept in multi-threading. The problems which occurs while using Collections in Multi-threaded application:

- Most of the Collections classes objects (like [ArrayList](https://www.geeksforgeeks.org/arraylist-in-java/), [LinkedList](https://www.geeksforgeeks.org/linked-list-in-java/), [HashMap](https://www.geeksforgeeks.org/java-util-hashmap-in-java/) etc) are non-synchronized in nature i.e. multiple threads can perform on a object at a time simultaneously. Therefore objects are not thread-safe.
- Very few Classes objects (like [Vector](https://www.geeksforgeeks.org/java-util-vector-class-java/), [Stack](https://www.geeksforgeeks.org/stack-class-in-java/), [HashTable](https://www.geeksforgeeks.org/java-util-hashtable-class-java/)) are synchronized in nature i.e. at a time only one thread can perform on an Object. But here the problem is performance is low because at a time single thread execute an object and rest thread has to wait.
- The main problem is when one thread is iterating an Collections object then if another thread cant modify the content of the object. If another thread try to modify the content of object then we will get RuntimeException saying ConcurrentModificationException.
- Because of the above reason Collections classes is not suitable or we can say that good choice for Multi-threaded applications.

To overcome the above problem SUN microSystem introduced a new feature in JDK 1.5Version, which is nothing but Concurrent Collections.


Concurrent collection classes are used to ensure safe and concurrent access to data by multiple threads (which is not the case in normal collection classes like list, set, map etc).

The concurrent collection classes that we would be talking about in this chapter are:

1. CopyOnWriteArrayList
2. CopyOnWriteArraySet
3. ConcurrentHashmap

Concurrent collection classes are meant for synchronized access to threads. Although, we already have some classes like Vector and Hashtable, which are synchronized or thread-safe, but they have some cons associated with them, and they are not the best solutions. Therefore, a new types of classes have been introduced as part of concurrent collections in java which serve this purpose.

These classes are part of java.util.concurrent package. Let’s first understand the need of concurrent collection classes.

## Need of concurrent collections

1. Most of the classes in collections framework like Arraylist, LinkedList, HashSet, HashMap, LinkedHashMap etc are not thread safe. If we use these collections in a multithreaded environment, then we will not get the right results. This raises the requirement to have thread safe classes.
2. Although, we can make make a list, or set synchronized, for that matter, using methods like Collections.synchronizedList(list), but it is not an efficient way.
3. If we use Collections.synchronizedList() method, then the entire list gets locked by one thread. Even if all threads are readers threads, then also, the they can access the list in a synchronized way. This increases the response time significantly.

## Concurrent collection classes in java

1. CopyOnWriteArrayList
2. CopyOnWriteArraySet
3. ConcurrentHashmap

## CopyOnWriteArrayList

CopyOnWriteArrayList is part of collections framework with some differences with ArrayList. CopyOnWriteArrayList is thread safe and can be used in multithreaded environment. Let’s have a look at the features of CopyOnWriteArrayList

1. The CopyOnWriteArrayList is a thread safe version of ArrayList. If we are making modifications like adding, removing elements in CopyOnWriteArrayList, then JVM does so by creating a new copy of it by the use of Cloning.
2. CopyOnWriteArrayList is costly if used in case of more update operations. Because when changes are made, JVM has to create a cloned copy of the underlying array and add/update elements to it.
3. Multiple threads can read the data from CopyOnWriteArrayList, but only one thread can write data at a particular time.
4. We can add duplicate elements in it.
5. CopyOnWriteArrayList is the best choice in multithreading, if there are more read operations.

## Class hierarchy

![](https://javatrainingschool.com/wp-content/uploads/2021/10/image-18-1024x426.png)

## Constructor Summary

```
CopyOnWriteArrayList()
CopyOnWriteArrayList(Collection<? extends E> c)
CopyOnWriteArrayList(E[] toCopyIn)
```

## CopyOnWriteArrayList Example

```java
package com.javatrainingschool;

import java.util.Arrays;
import java.util.Iterator;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

public class CopyOnWriteArrayListExample {

	public static void main(String[] args) {
		List<String> list = Arrays.asList("CV Raman", "Homi Bhabha", "Ramanujan");
		CopyOnWriteArrayList<String> cowArrayList = new CopyOnWriteArrayList<String>(list);

		System.out.println("List = " + cowArrayList);

		Iterator<String> iterator1 = cowArrayList.iterator();

		// adding another element
		cowArrayList.add("Vikram Sarabhai");

		while(iterator1.hasNext()) {
			System.out.println("Element from iterator1 : " + iterator1.next());
		}
		
		Iterator<String> iterator2 = cowArrayList.iterator();
		
		while(iterator2.hasNext()) {
			System.out.println("Element from iterator2 : " + iterator2.next());
		}
	}
}
```

```
Output :
List = [CV Raman, Homi Bhabha, Ramanujan]
Element from iterator1 : CV Raman
Element from iterator1 : Homi Bhabha
Element from iterator1 : Ramanujan
Element from iterator2 : CV Raman
Element from iterator2 : Homi Bhabha
Element from iterator2 : Ramanujan
Element from iterator2 : Vikram Sarabhai
```

## How CopyOnWriteArrayList work internally?

CopyOnWriteArrayList class does every write operation (add, set, remove, etc) in a new copy of the list. JVM creates a new copy of the list and modifications are done in that. Using this technique, thread-safety is achieved without a need for synchronization.

_Note -> Any number of reader threads can access CopyOnWriteArrayList simultaneously. And reader and writer threads do not block each other._

![](https://javatrainingschool.com/wp-content/uploads/2021/10/image-19-1024x671.png)

CopyOnWriteArrayList internal working

When we call iterator() or listIterator() methods on CopyOnWriteArrayList, it returns an iterator object that holds immutable snapshot of the list objects. After iterator is created, if any other thread makes changes in the list, then that will not reflect in iterator objects. Due to which, we can iterate over the list in a safe way, without bothering about the concurrent modification.

CopyOnWriteArrayList is useful in following cases:

1. When we need to use a list in a multithreaded environment, then multiple threads can perform read and write operations simultaneously.
2. When read operations are more and write operations are less. If it is vice versa, then we can use Vector or synchronized list.

## CopyOnWriteArraySet

CopyOnWriteArraySet is like CopyOnWriteArrayList. It is thread safe version of HashSet.

![](https://javatrainingschool.com/wp-content/uploads/2021/10/image-20-1024x949.png)

## Class declaration

```
class CopyOnWriteArraySet<E> extends AbstractSet<E> implements java.io.Serializable
```

## Constructor summary

```
CopyOnWriteArraySet()
CopyOnWriteArraySet(Collection<? extends E> c)
```

## Example

```java
package com.javatrainingschool;

import java.util.Arrays;
import java.util.Iterator;
import java.util.List;
import java.util.concurrent.CopyOnWriteArraySet;

public class CopyOnWriteArrayListExample {

	public static void main(String[] args) {
		List<String> list = Arrays.asList("CV Raman", "Homi Bhabha", "Ramanujan", "Ramanujan", "Homi Bhabha");
		CopyOnWriteArraySet<String> cowArraySet = new CopyOnWriteArraySet<String>(list);

		System.out.println("Set = " + cowArraySet);

		Iterator<String> iterator1 = cowArraySet.iterator();

		// adding another element
		cowArraySet.add("Vikram Sarabhai");

		while(iterator1.hasNext()) {
			System.out.println("Element from iterator1 : " + iterator1.next());
		}
		
		Iterator<String> iterator2 = cowArraySet.iterator();
		
		while(iterator2.hasNext()) {
			System.out.println("Element from iterator2 : " + iterator2.next());
		}
	}
}
```

```
Output :
Set = [CV Raman, Homi Bhabha, Ramanujan]
Element from iterator1 : CV Raman
Element from iterator1 : Homi Bhabha
Element from iterator1 : Ramanujan
Element from iterator2 : CV Raman
Element from iterator2 : Homi Bhabha
Element from iterator2 : Ramanujan
Element from iterator2 : Vikram Sarabhai
```

## ConcurrentHashMap

CosurrentHashMap, as its name suggests, provides concurrent access to multiple threads. It is thread safe version of HashMap. We know, HashMap is not synchronized and therefore is not thread safe. To overcome this issue, collections framework provides ConcurrentHashMap.

ConcurrentHashMap provides similar functionalities like HashMap. The only difference is that it internally maintains concurrency.  
It is concurrent version of HashMap. It internally maintains a HashTable, which is divided into segments. The number of segments depend on the level of concurrency specified in the ConcurrentHashMap. By default, it divides it into 16 segments. It doesn’t lock the entire map unlike HashTable or Synchronized HashMap. It only locks the particular segment of HashMap where write operation is going on.

## Class declaration

```
class ConcurrentHashMap<K,V> extends AbstractMap<K,V> implements ConcurrentMap<K,V>, Serializable
```

## Class hierarchy

![](https://javatrainingschool.com/wp-content/uploads/2021/10/image-21.png)

## Constructors summary

```
ConcurrentHashMap()            //default constructor
ConcurrentHashMap(Map<? extends K, ? extends V> m)
ConcurrentHashMap(int initialCapacity, float loadFactor)
ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel)
```

## Features of ConcurrentHashMap

1. ConcurrentHashMap internally uses HashTable as its data structure. It is thread safe just like Hashtable. **I**t provides all the functionalities of HashMap except thread safety.
2. ConcurrentHashMap internally divides it into segments. Each segment works independently and can be accessed by different reader threads simultaneously. However, each segment can be accessed only by one writer thread at a time. This also means, a concurrent hash map can be accessed by as many writer threads together as there are segments.
3. The default level of concurrency is 16. Which means by default, there are 16 segments.
4. Read operations don’t require locking of cocurrent hash map, where as write operations do require locking.
5. Locking is known as segment or bucket locking.
6. Concurrent hash map doesn’t allow null key or null values.

## ConcurrentHashmap code example

```java
package com.javatrainingschool;

import java.util.concurrent.ConcurrentHashMap;

public class ConcurrentHashMapExample {
	
	public static void main(String[] args) {
		
		ConcurrentHashMap<Integer, String> cMap = new ConcurrentHashMap<>();
		cMap.put(1, "Taj Mahal");
		cMap.put(2, "Qutab Minar");
		cMap.put(3, "Char Minar");
		
		for(Integer key : cMap.keySet()) {
			System.out.println(key + " : " + cMap.get(key));
		}
	}
}
```

```
Output :
1 : Taj Mahal
2 : Qutab Minar
3 : Char Minar
```

## Concurrent modification in ConcurrentHashMap

ConcurrentHashMap returns a fail safe iterator. It means, using this iterator, we can modify ConcurrentHashMap while iterating it. Let’s see one example

```java
package com.javatrainingschool;

import java.util.Iterator;
import java.util.concurrent.ConcurrentHashMap;

public class ConcurrentHashMapExample {
	
	public static void main(String[] args) {
		
		ConcurrentHashMap<Integer, String> cMap = new ConcurrentHashMap<>();
		cMap.put(1, "Taj Mahal");
		cMap.put(2, "Qutab Minar");
		cMap.put(3, "Char Minar");
		
		Iterator<Integer> it = cMap.keySet().iterator();
		while(it.hasNext()) {
			int key = it.next();
			if(key == 2)
				cMap.put(2, "Gateway of India");
			System.out.println(key + " : " + cMap.get(key));
		}
	}
}
```

```
Output :
1 : Taj Mahal
2 : Gateway of India
3 : Char Minar
```

## Fail safe iterator returned by ConcurrentHashMap

In the above example, we saw that we can modify ConcurrentHashMap while iterating it. It became possible because the iterator returned by ConcurrentHashMap is fail safe. Which means we can modify it while iterating.

## Internal working of ConcurrentHashMap

![](https://javatrainingschool.com/wp-content/uploads/2021/10/image-22-1024x563.png)

ConcurrentHashMap performs segment locking in a multithreading environment. It allows performing read operations concurrently without any blocking of threads.  
It performs retrieval operations without blocking but write operations may conflict each other if they are working on the same segment. Each segment works as an individual HashMap. So, if one thread is writing onto one segment, it locks that segment. Now until that thread is done, no other thread will get access to that segment. However, reader threads can still access this segement. If another thread, wants to write to another segment at the same time, it is free to do so. It will acquire lock on that segment and will start doing its job.

  
Read operations return most recently updated operations, which means the it may not fetch the currently updating value.  
Memory visibility for the read operations is ensured by volatile reads.