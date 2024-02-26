### Q 1. What is Bloom Filter 🤔?

A Bloom filter is a _**probabilistic data structure**_ used for efficient membership tests. It provides a space-efficient representation of a set by using a bit array and a set of hash functions.

This definition is copied from internet, Let's decode it word by word. In simple words, Bloom filter is a probabilistic data structure which is used to get the presence of items. Probabilistic means it could give **False Positive** answers but never give **False Negative** answers.

False positive means if an item is not present in the data structure it could give us answer like "_**yes it is present**_". False negative means if item is present in the data structure it will never give us answer like "_**it is not present**_".

Bloom filter use **bit array as a set** to store and search the item efficiently. For every insertion of item, it sets a bit to **true** in the array.

  

---

### Q 2. Which bit to set is the question 🤨?

To calculate the bit which needs to be set at the time of insertion, uses a Hash Algorithm as shown in the below diagram:

![No alt text provided for this image](https://media.licdn.com/dms/image/D4D12AQEp780fH1efFw/article-inline_image-shrink_1500_2232/0/1689431251897?e=1707350400&v=beta&t=Wcut9nVZXn10-ZmWZisY-4_EPJN_3MmMcQpEZedid0M)

Basic Insertion

  

**The probability of false positives increases with the number of elements in the filter.😥**

In a Bloom filter with a capacity of 10 items, if we have already stored 10 different items in the filter, and we search for an 11th item that is not actually present in the filter, there is a high probability that the Bloom filter will incorrectly report that the item is present. This is because the hash algorithm used by the Bloom filter will map the 11th item to a bit position between 0 and 9, which has already been set by one of the previously stored items.

Bloom filters work by setting bits in a bit array, based on the hash values of the items being stored. If a bit is already set, it gives the impression that the item is present in the filter, even though it may not be. This is due to the limited number of bits available in the bit array, which can lead to collisions where different items are mapped to the same bit positions.

To mitigate this issue, increasing the capacity of the Bloom filter or using additional hash functions can help reduce the probability of false positives. By having more available bit positions and reducing collisions, the accuracy of the Bloom filter can be improved.

  

**The probability of false positives decreases with the number of hash functions used and the size of the bit array. 🧐**

Suppose we have a Bloom filter with a bit array of size 10 and we want to store three elements: "apple", "banana", and "orange". For this example, we'll use a simple hash function that maps each element to a number between 0 and 9.

_**Single Hash Function:**_

- We use a single hash function to map each element to a bit position in the bit array. Let's say our hash function maps "apple" to bit position 3, "banana" to bit position 6, and "orange" to bit position 8.
- After inserting these elements, the bit array would look like this: [0, 0, 0, 1, 0, 0, 1, 0, 1, 0].
- Now, let's check for the presence of the element "grape".We apply the hash function to "grape" and find that it maps to bit position 1.
- Checking the bit array, we see that the bit at position 1 is 0, so we correctly determine that "grape" is not present in the Bloom filter.

**Multiple Hash Functions:**

- Now, let's modify our Bloom filter to use two hash functions instead of one.
- We choose two different hash functions and compute two different bit positions for each element. Let's say the first hash function maps "apple" to bit position 3 and "banana" to bit position 6. The second hash function maps "apple" to bit position 9 and "banana" to bit position 2.
- After inserting the elements, the bit array would look like this: [0, 0, 1, 0, 0, 0, 1, 0, 1, 1].
- Now, let's check for the presence of the element "grape" again.We apply the first hash function to "grape" and find that it maps to bit position 7.
- Applying the second hash function, we find that "grape" maps to bit position 4.
- Checking the bit array, we see that both bit positions 7 and 4 are 0. Therefore, we correctly determine that "grape" is not present in the Bloom filter.

By using multiple hash functions, we decrease the likelihood of collisions and improve the accuracy of the Bloom filter. In the first example with a single hash function, the Bloom filter incorrectly reported "grape" as not present because of a collision with another element. However, in the second example with two hash functions, we achieved a more accurate result.

  

By above discussion we get the trade-off:

> Bloom filters offer a trade-off between space efficiency and accuracy. By increasing the size of the bit array and the number of hash functions, the probability of false positives can be reduced, but at the cost of increased memory usage and computation.

  

---

### Q 3. List of hash algorithm can be used in bloom filter🤨?

Some famous hash algorithms are as follows:

1. MurmurHash
2. Jenkins Hash
3. FNV Hash
4. CityHash
5. xxHash
6. SHA-1

Which one to use is out of scope for this article. We will consider it as a black box for now.

  

---

### Q 4. How to decide number of hash function and size of bit array ?

Deciding the number of hash functions and the size of the bit array for a Bloom filter depends on the desired error rate and the expected number of elements to be stored in the filter. Here's a general approach to consider:

1. **Number of Hash Functions**: The number of hash functions used in a Bloom filter affects the probability of false positives. Increasing the number of hash functions reduces the probability of collisions and improves the accuracy of the filter. A common rule of thumb is to use approximately **[m / n * ln(2)]** hash functions, where **m** is the size of the bit array and **n** is the expected number of elements to be stored. For example, if we expect to store 10 items, we could start with using 2 or 3 hash functions.
2. **Size of the Bit Array**: The size of the bit array impacts the memory usage and the probability of false positives. A larger bit array reduces the chance of collisions and decreases the false positive rate. The size of the bit array can be estimated based on the desired capacity (maximum number of elements to be stored) and the desired error rate. The formula to calculate the optimal bit array size is **[-(n * ln(p)) / (ln(2)^2)]**, where **n** is the expected number of elements and **p** is the desired false positive rate. For example, if we expect to store 10 items and desire a low false positive rate, we could choose a bit array size of 100 or higher.

It's important to note that these calculations provide initial estimates, and you may need to adjust the number of hash functions and the size of the bit array based on your specific requirements and trade-offs. In practice, it's often beneficial to monitor the performance and accuracy of the Bloom filter with representative data and adjust these parameters accordingly.

Additionally, there are online calculators and libraries available that can help with estimating the optimal number of hash functions and bit array size based on your desired parameters.

Remember that Bloom filters are probabilistic data structures, and there is always a trade-off between memory usage, accuracy, and performance. You need to strike a balance that suits your specific use case.

---

### Q 5. How to implement our own bloom filter ?

Enough theory, Now lets implement our own bloom filter. I am using python language. You can use any language you want.

  

import mmh
from bitarray import bitarray
import math

class BloomFilter:
   def __init__(self, capacity, error_rate):
      self.capacity = capacity
      self.error_rate = error_rate
      self.num_bits = self.calculate_num_bits()
      self.num_hash_functions = self.calculate_num_hash_functions()
      self.bit_array = bitarray(self.num_bits)
      self.bit_array.setall(False)
  
  
    def calculate_num_bits(self):
      num_bits = -(self.capacity * math.log(self.error_rate)) / (math.log(2)**2)
      return int(num_bits)
  
  
    def calculate_num_hash_functions(self):
      num_hash_functions = (self.num_bits / self.capacity) * math.log(2)
      return int(num_hash_functions)
  
  
    def insert(self, item):
      for seed in range(self.num_hash_functions):
        index = mmh3.hash(item, seed) % self.num_bits
        self.bit_array[index] = True
  
  
    def search(self, item):
      for seed in range(self.num_hash_functions):
        index = mmh3.hash(item, seed) % self.num_bits
        if not self.bit_array[index]:
          return False
      return True
  
  # Example usage
  bloom_filter = BloomFilter(capacity=1000, error_rate=0.1)
  
  
  # Add items to the filter
  bloom_filter.insert("1")
  bloom_filter.insert("2")
  bloom_filter.insert("3")
  bloom_filter.insert("4")
  bloom_filter.insert("5")
  bloom_filter.insert("6")
  bloom_filter.insert("7")
  bloom_filter.insert("8")
  bloom_filter.insert("9")
  bloom_filter.insert("10")
  
  
  # Check if an item is likely to be in the filter
  print(bloom_filter.search("1"))  # True
  print(bloom_filter.search("11"))  # False
  print(bloom_filter.search("12"))  # False

In this example, we use the **mmh3** library for the MurmurHash3 algorithm and the **bitarray** library for efficient storage of the bit array. The **BloomFilter** class is initialized with a desired capacity (maximum number of items to be stored) and the desired error rate (probability of a false positive). The number of bits and the number of hash functions are calculated based on these parameters.


### Q 6. Use Cases

Bloom Filter has wide use cases and applications like -  
- Safe Browsing  
- Weak password Detection  
- Caching  
- Many NoSQL Databases use the bloom filter to reduce the disk reads for the keys that do not exist  
- Spell Checking  
- Duplicate Detection  
- Web Crawling and Indexing etc.


#   
Bloom Filter - A Probabilistic Data Structure

[

![Gaurav Pandey](https://media.licdn.com/dms/image/D5603AQGR79ooUNonOw/profile-displayphoto-shrink_100_100/0/1700134874550?e=1707350400&v=beta&t=G8UwaqmCOW0EVP7js01-x_aQjgDZZVbTlyDngNrVWC8)

](https://www.linkedin.com/in/-gauravpandey/)

[

## Gaurav Pandey

](https://www.linkedin.com/in/-gauravpandey/)

Engineering

[6 articles](https://www.linkedin.com/in/-gauravpandey/recent-activity/posts/) Following

August 13, 2023

Open Immersive Reader

**Problem statement to understand Bloom Filter:** Suppose you are building the signup page for GMAIL like service and you wanted to make sure that the username is unique, for that you must be checking if the username does not exist already in the database (Username is not taken already) then only you will assign that username to the user. But seeing 1.5 Billion Daily Active Users for Gmail, your solution to this problem should be scalable enough to handle search in this very large dataset of user accounts.

Let's go through all possible solutions one by one -

1. **Linear Search:** A brute force solution will be firing a SQL query to check if that username already exists or not, this will result in _**O(n)**_ time complexity.
2. **Binary Search:** We can store the username in alphabetical order and do a binary search. Although we have improved the search operation to _**O(log(n)**_) but now maintaining sorted order of usernames is expensive.
3. **Using in-memory SET:** We can store each username in a set and check in _**O(1)**_ time if the username is in the set. But storing these many usernames in memory is challenging and not cost-efficient.

### **Bloom Filter**

Bloom Filter is a space-efficient probabilistic data structure that is used to search an element within a very large set of elements in constant time that is _**O(k)**_ where K is the number of hash functions being used in Bloom Filter.

Bloom Filter uses a bit array and hashing techniques to store the existence of a string. Suppose we have an array of length 10^15 and K=3 Hash functions

![No alt text provided for this image](https://media.licdn.com/dms/image/D5612AQEcjHsDJmy9oA/article-inline_image-shrink_1500_2232/0/1691927736312?e=1707350400&v=beta&t=4hH8lhugJpLCigfmyiIKxgJjt6pZ0seOG8cW4RIEDgg)

BIT Array and K=3 Hash Functions

**Insertion in Bloom Filter -**

1. Take user input
2. Pass input through each hash function
3. Take mod of output from BIT Array Length, you will have 3 array index
4. Turn on the BIT to 1

![No alt text provided for this image](https://media.licdn.com/dms/image/D5612AQFKQSP5_AS4_w/article-inline_image-shrink_1500_2232/0/1691927908467?e=1707350400&v=beta&t=yw0dFcNuiURINe5t_dDRTczMMeeqb6Eg_nuCFiDQER8)

Insertion in Bloom Filter

Time Complexity - Constant Time O(k), where k is the number of hash functions

**Key Lookup in Bloom Filter -**

1. Take user input
2. Pass input through each hash function
3. Take mod of output from BIT Array Length, you will have 3 array index
4. Check if all 3 index in the BIT array is 1
5. If any BIT is 0 then the Key do not exist

![No alt text provided for this image](https://media.licdn.com/dms/image/D5612AQHFbiOo2ivUQQ/article-inline_image-shrink_1500_2232/0/1691928142825?e=1707350400&v=beta&t=itLbqhuYyA2hAGwcFaVmSGnEbQ94zM7m6dcwGpUzymY)

Key Lookup in Bloom Filter

Time Complexity - Constant Time O(k), where k is the number of hash functions

Since it is a probabilistic data structure, Key Lookup output falls into two categories -

1. False Positive: It is possible to get a false positive which means even if the key does not exist, it might return that key is already there in the set.
2. False Negative: It will never return a false negative which means it returns accurate results if the key do not exist.