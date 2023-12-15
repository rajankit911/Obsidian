- **MessageDigest** is the value returned by a cryptographic hash function, which is also known as **hash values**
- **Hash function** is a mathematical function that transforms a user-given input value of arbitrary length into the output value of fixed length.
- Hashing is used in many applications, including **_data storage_**, **_searching_**, and **_cryptography_**.



![Message Digest](https://www.tutorialspoint.com/java_cryptography/images/message_digest.jpg)

### Message Digest Algorithms

The `java.security` package provides a class, i.e., **MessageDigest**, that supports following algorithms for calculating a message digest from binary data.

|**Algorithm Name**|
|---|
|MD2|
|MD5|
|SHA-1|
|SHA-256|
|SHA-384|
|SHA-512|

### Example

Following is an example which reads data from a file and generate a message digest and prints it.

```java
import java.security.MessageDigest;
import java.util.Scanner;

public class MessageDigestExample {
   public static void main(String args[]) throws Exception{
      //Reading data from user
      Scanner sc = new Scanner(System.in);
      System.out.println("Enter the message");
      String message = sc.nextLine();
	  
      // The MessageDigest class provides a method named getInstance().
      // This method accepts a String variable specifying the name of
      // the algorithm to be used and returns a MessageDigest object
      // implementing the specified algorithm.
      MessageDigest md = MessageDigest.getInstance("SHA-256");
      
      // After creating the message digest object, you need to pass
      // the message data to it. You can do so using the update() method
      // of the MessageDigest class, this method accepts a byte array
      // representing the message and passes it to the above created
      // MessageDigest object.
      md.update(message.getBytes());
      
      // You can generate the message digest using the digest() method
      // of the MessageDigest class, this method computes the hash function
      // on the current object and returns the message digest in the
      // form of byte array.
      byte[] digest = md.digest();     
       
      System.out.println(digest);  
     
      //Converting the byte array in to HexString format
      StringBuffer hexString = new StringBuffer();
      
      for (int i = 0;i<digest.length;i++) {
		hexString.append(Integer.toHexString(0xFF & digest[i]));
      }
      
      System.out.println("Hex format : " + hexString.toString());     
   }
}

```

### Output

The above program generates the following output −

```console
Enter the message
Hello how are you
[B@55f96302
Hex format: 2953d33828c395aebe8225236ba4e23fa75e6f13bd881b9056a3295cbd64d3
```

