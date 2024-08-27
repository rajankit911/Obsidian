# Serialization

The process of writing the state of an object to a file is called **serialization**.

At its core, serialization is the process of transforming an object into a byte stream — a digital representation that can be stored, transmitted, or persisted

An object is said to be serializable if and only if the corresponding class implements a `Serializable` interface.

`Serializable` interface is present in `java.io package`, and it doesn’t contain any method and hence it is a **marker interface**.

>[!warning]
>If we are trying to serialize a non-serializable object then we will get Runtime Exception saying `NotSerializableException`.


In serialization, everything is taken care of by JVM and the programmer doesn’t have any control. JVM uses below methods for serialization and deserialization.

```java
private void writeObject(ObjectOutputStream oos) throws IOException

private void readObject(ObjectInputStream ois) throws IOException,
		ClassNotFoundException
```

The `writeObject` and `readObject` methods must be **private**. This ensures that they are not part of the public API of the class and are only used by the serialization mechanism.

You can override `readObject` and `writeObject` methods to customize the serialization and deserialization process in Java. This is useful when you want to control how your object's state is serialized to and deserialized from a stream, beyond the default mechanism provided by the `Serializable` interface.


Let’s illustrate serialization with a _Person_ class. Note that **static fields belong to a class (as opposed to an object) and are not serialized**. Also, note that we can use the keyword _transient_ to ignore class fields during serialization:


### Serial Version UID

The JVM associates a version (_long_) number with each `Serializable` class.

We use it to verify that the saved and loaded objects have the same attributes, and thus are compatible on serialization.

Most IDEs can generate this number automatically, and it’s based on the class name, attributes, and associated access modifiers. Any changes result in a different number, and can cause an `InvalidClassException`.

If a `Serializable` class doesn’t declare a **serialVersionUID**, the JVM will generate one automatically at run-time. However, it’s highly recommended that each class declares its **serialVersionUID**, as the generated one is compiler dependent and thus may result in unexpected `InvalidClassExceptions`.


## Inheritance



## Custom Serialization in Java

# Externization

In serialization, it is not possible to save part of the object which may create performance problems. To overcome this problem we should go for externalization.

The main advantage of externalization over serialization is, everything is taken care of by the programmer and JVM doesn’t have any control. Based on our requirements we can save either the total object or part of the object which improves the performance of the system.

This interface defines two methods as follows:

```java
/**
* This method will be executed automatically at the time of serialization.
* Within this method we have to write code to save the required variable
* to the file.
*/
public void  writeExternal( ObjectOutput obj ) throws IOException

/**
* This method will be executed automatically at the time of deserialization.
* Within this method, we have to write code to read the required variable
* from the file and assign it to the current object.
*/
public void  readExternal(ObjectInput  in )throws IOException,
		ClassNotFoundException
```


>[!warning]
>At the time of deserialization JVM will create a separate new object by executing a public no-argument constructor. Hence, every `Externalizable` implemented class should compulsorily contain a public no-argument constructor otherwise we will get Runtime Exception saying `InvalidClassException`.


# Difference

| Key                             | Serialization                                                                                                                                                                                                                    | Externalization                                                                                                                                                       |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Interface                       | A `Serializable` interface is used to implement serialization. It is a marker interface i.e. it does not contain any method. JVM uses private methods, ie, `writeObject` and `readObject` for serialization and deserialization. | An `Externalizable` interface used to implement Externalization. It is not a marker interface and thus it defines two methods `writeExternal()` and `readExternal()`. |
| Responsibility of Serialization | `Serializable` interface passes the responsibility of serialization to JVM and the programmer has no control over serialization.                                                                                                 | `Externalizable` interface provides all serialization responsibilities to a programmer and hence JVM has no control over serialization.                               |
| Performance                     | Serialization using a `Serializable` interface uses reflection which causes relatively bad performance.                                                                                                                          | Serialization using an externalizable interface has better performance due to full control over the implementation logic.                                             |
| Default Constructor             | Default serialization does not require any no-arg constructor.                                                                                                                                                                   | A public no-arg constructor is required while using an Externalizable interface.                                                                                      |
| Structural Modification         | It is hard to analyze and modify class structure because any change in structure may break serialization.                                                                                                                        | It is relatively easy to analyze and modify class structure because of complete control over serialization logic.                                                     |
| Partial Serialization           | Using a `Serializable` interface we save the total object to a file, and it is not possible to save part of the object.                                                                                                          | Base on our requirements we can save either the total object or part of the object.                                                                                   |
| Role of Transient               | Transient keywords play an important role here. JVM ignores transient variable during serialization and deserialization of java object                                                                                           | Transient keywords won’t play any role. Programmer can write their own logic to ignore some of the variables during externalization of java object.                   |







# Medium 

**Introduction:  
**Serialization, deserialization, and externalization constitute the bedrock of Java programming, wielding substantial influence over the management of object data. This comprehensive guide aims not only to unravel the core principles of these concepts but also to delve into advanced nuances, providing a holistic understanding. We’ll explore their meanings, shed light on scenarios where one outperforms the others, and meticulously dissect the trade-offs involved. As we navigate through this intricate landscape, we’ll reinforce the basics and venture into advanced territories, ensuring a comprehensive mastery of these essential Java techniques.

**Serialization:  
**At its core, serialization is the process of transforming an object into a byte stream — a digital representation that can be stored, transmitted, or persisted. The primary objective is to encapsulate the state of an object in a manner that facilitates its faithful reconstruction at a later point in time.

In Java, the Serializable interface takes center stage in the serialization process. By implementing this interface, a class signifies its willingness to undergo serialization. The process is relatively straightforward — the ObjectOutputStream class steps in to write the object’s state to an output stream.

Example:

import java.io.*;  
  
public class SerializationExample {  
    public static void main(String[] args) {  
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("serialized_object.ser"))) {  
            MyClass obj = new MyClass();  
            oos.writeObject(obj);  
        } catch (IOException e) {  
            e.printStackTrace();  
        }  
    }  
}  
  
class MyClass implements Serializable {  
    // Class members and methods  
}

**Deserialization:  
**In contrast, deserialization involves resurrecting an object from a byte stream. The ObjectInputStream class comes into play here, reading the byte stream and meticulously reconstructing the object’s state.

Example:

import java.io.*;  
  
public class DeserializationExample {  
    public static void main(String[] args) {  
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("serialized_object.ser"))) {  
            MyClass obj = (MyClass) ois.readObject();  
            // Use the deserialized object  
        } catch (IOException | ClassNotFoundException e) {  
            e.printStackTrace();  
        }  
    }  
}

**Externalization:  
**While serialization automates the process based on an object’s internal mechanisms, externalization introduces a higher level of control. Through the implementation of the Externalizable interface, a class gains authority over the serialization and deserialization processes.

Example:

import java.io.*;  
  
public class ExternalizationExample implements Externalizable {  
    private int data;  
  
    // Default constructor  
    public ExternalizationExample() {  
    }  
  
    // Externalizable implementation  
    @Override  
    public void writeExternal(ObjectOutput out) throws IOException {  
        out.writeInt(data);  
    }  
  
    @Override  
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {  
        data = in.readInt();  
    }  
}

Scenarios and Trade-offs:

**Serialization vs. Externalization:**

Serialization is the go-to when simplicity and automatic handling suffice.

Externalization shines when a more granular control over the serialization process is necessary.

**Performance Considerations:**

Serialization, due to its automatic nature, tends to be slower.

Externalization, with careful optimization and selective serialization, can exhibit superior performance.

**Object Graphs:**

Serialization excels in handling complex object graphs seamlessly.

Externalization demands explicit handling of object references and their order in the byte stream.  
Versioning:

Serialization automatically manages class versioning but may encounter issues with backward and forward compatibility.

Externalization empowers developers to explicitly manage versioning, ensuring control over changes.

**5. Selective Serialization:  
**Externalization introduces a powerful mechanism for selective serialization. The implementation of the writeExternal method allows developers to choose which fields to serialize and which to skip, optimizing performance and resource usage.

**Custom Serialization Strategies:  
**Externalization provides developers with the flexibility to implement custom serialization strategies. This could involve intricate logic for encryption, compression, or specialized formatting tailored to unique use cases.

**Security Considerations:  
**Externalization excels in terms of security. Developers can implement custom security checks during serialization and deserialization, fortifying applications against potential threats like object injection and data tampering.

**Resource Management:  
**Externalization facilitates efficient resource management. Through explicit control over serialization and deserialization processes, developers can manage resources such as file handles, network connections, and memory allocations more effectively, reducing the risk of resource leaks and performance bottlenecks.

**Concurrency Challenges:  
**Serialization might encounter challenges in concurrent environments due to its automatic nature and potential conflicts.

Externalization provides a canvas for developers to implement custom strategies for handling concurrency issues, providing more control over shared resources.

**Integration with Java EE Technologies:  
**Serialization finds extensive use in Java EE technologies for session replication, caching, and data interchange between distributed components.

Externalization’s fine-grained control makes it suitable for scenarios where Java EE applications demand specific serialization and deserialization behaviors.

**Real-world Use Cases:  
**Serialization is commonly employed in scenarios such as caching objects, storing application state, and transmitting data between distributed systems.

Externalization finds its niche in scenarios requiring explicit control, such as storing objects in a custom format, optimizing for specific use cases, or implementing versioning strategies.  
Serialization in Microservices Architectures:

Serialization plays a crucial role in microservices architectures where communication between services often involves the exchange of serialized data, making it essential for efficient data transfer.  
Externalization in Enterprise Systems:

Enterprise systems, with their diverse and complex data models, can benefit from externalization by allowing developers to tailor serialization and deserialization processes to meet specific business requirements.  
Cross-language Interoperability:

Serialization becomes vital in scenarios involving cross-language interoperability, where objects need to be serialized in Java and deserialized in another programming language.  
Optimizing for Mobile Applications:

Mobile applications with limited bandwidth and resources can leverage externalization to optimize the size of serialized data, ensuring efficient data transfer and improved performance.

**Exploring the Nuances of Serialization**:  
Serialization, being a cornerstone of Java development, warrants a deeper exploration of its nuances. Let’s dissect some of the intricacies associated with serialization, shedding light on topics that go beyond the basic implementation.

**Customizing the Serialization Process:  
**While implementing the Serializable interface is the standard way to mark a class as serializable, developers can further customize the serialization process by providing two special methods: writeObject and readObject. These methods allow for fine-tuned control over how an object is serialized and deserialized.  
Example:

private void writeObject(ObjectOutputStream out) throws IOException {  
    // Custom serialization logic  
    out.defaultWriteObject(); // Don't forget to call the default serialization  
}  
  
private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {  
    // Custom deserialization logic  
    in.defaultReadObject(); // Don't forget to call the default deserialization  
}

**Transient Keyword:  
**The transient keyword allows developers to exclude specific fields from the serialization process. This is useful when certain fields should not be persisted, such as temporary or derived values.  
Example:

private transient String temporaryData;

**Serializable Class Versioning:  
**Managing class versioning is crucial for maintaining backward and forward compatibility during serialization. The serialVersionUID field serves as a version identifier. When changes to the class structure are made, incrementing this value helps ensure a smooth transition during deserialization.  
Example:

private static final long serialVersionUID = 1L;

**Externalizable vs. Serializable:  
**While both Externalizable and Serializable interfaces enable object serialization, the key difference lies in the level of control they provide. While Serializable relies on default serialization mechanisms, Externalizable demands explicit implementation of writeExternal and readExternal methods, offering fine-grained control.  
Example:

public class CustomObject implements Externalizable {  
    // Implementation of Externalizable methods  
}

**Delving into Deserialization Challenges:  
**Deserialization, although a powerful mechanism, comes with its set of challenges that developers need to be aware of to ensure robust and secure applications.

**Security Concerns with Deserialization:  
**Deserialization can be a security risk if not handled carefully. Malicious users may attempt to inject harmful objects during deserialization, leading to security vulnerabilities. Implementing proper input validation and using whitelisting approaches are crucial to mitigate such risks.  
Example:

private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {  
    in.defaultReadObject();  
    // Custom security checks  
    if (!isValidData()) {  
        throw new SecurityException("Invalid data during deserialization");  
    }  
}

**Implementing Custom Deserialization Strategies:**  
In scenarios where default deserialization mechanisms fall short, developers can implement custom deserialization strategies. This might involve decrypting encrypted data or applying additional validation checks during the deserialization process.  
Example:

private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {  
    in.defaultReadObject();  
    // Custom deserialization logic  
    decryptSensitiveData();  
    validateDeserializedData();  
}

**Handling Class Evolution:  
**Ensuring backward and forward compatibility during deserialization is critical, especially in long-lived systems where the structure of serialized objects might evolve over time. Strategies such as providing default values for new fields or employing externalized versioning can aid in managing class evolution.  
Example:

private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {  
    in.defaultReadObject();  
    // Handle backward compatibility  
    if (version < 2) {  
        this.newField = DEFAULT_VALUE;  
    }  
}

**Deserialization in a Multi-threaded Environment:  
**Deserialization in a multi-threaded environment can lead to race conditions and unexpected behavior. Synchronization mechanisms should be in place to ensure that deserialization occurs in a thread-safe manner.  
Example:

public synchronized MyClass readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {  
    return (MyClass) in.readObject();  
}

**Externalization Strategies for Fine-grained Control:  
**Externalization, offering a higher degree of control, opens the door to various strategies that developers can employ to optimize serialization and deserialization based on specific use cases.

**Selective Externalization:  
**One of the key advantages of externalization is the ability to selectively serialize and deserialize only essential data. By implementing the writeExternal method, developers can choose which fields to include, optimizing the process.  
Example:

@Override  
public void writeExternal(ObjectOutput out) throws IOException {  
    out.writeInt(data);  
    out.writeObject(sensitiveData);  
}

**Handling Circular References:  
**Externalization provides a way to handle circular references by explicitly managing the serialization and deserialization of objects with references. By maintaining a reference counter or unique identifiers, developers can ensure correct reconstruction.  
Example:

@Override  
public void writeExternal(ObjectOutput out) throws IOException {  
    out.writeInt(data);  
    out.writeObject(referencedObject);  
}  
  
@Override  
public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {  
    data = in.readInt();  
    referencedObject = (ReferencedObject) in.readObject();  
}

**Customizing Externalization for Encryption:  
**Externalization allows developers to implement custom strategies for encryption during serialization and decryption during deserialization. This is particularly useful for scenarios where data security is a top priority.  
Example:

@Override  
public void writeExternal(ObjectOutput out) throws IOException {  
    // Encrypt sensitive data before serialization  
    byte[] encryptedData = encryptData(sensitiveData);  
    out.writeObject(encryptedData);  
}  
  
@Override  
public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {  
    // Decrypt sensitive data after deserialization  
    byte[] encryptedData = (byte[]) in.readObject();  
    sensitiveData = decryptData(encryptedData);  
}

**Managing Resource Closure:  
**Externalization provides an opportunity to explicitly manage the closure of resources like file handles or network connections during deserialization. This ensures that resources are properly released, minimizing the risk of resource leaks.  
Example:

@Override  
public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {  
    data = in.readInt();  
    // Explicitly close resources  
    try (ObjectInputStream resourceStream = new ObjectInputStream(new FileInputStream("resource.ser"))) {  
        resourceData = (ResourceData) resourceStream.readObject();  
    }  
}

**Customizing Externalization for Compression:**  
For scenarios where minimizing the size of serialized data is crucial, externalization allows developers to implement custom strategies for data compression during serialization and decompression during deserialization.  
Example:

@Override  
public void writeExternal(ObjectOutput out) throws IOException {  
    // Compress data before serialization  
    byte[] compressedData = compressData(data);  
    out.writeObject(compressedData);  
}  
  
@Override  
public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {  
    // Decompress data after deserialization  
    byte[] compressedData = (byte[]) in.readObject();  
    data = decompressData(compressedData);  
}

**Conclusion:  
**In summary, mastering serialization, deserialization, and externalization is not merely about understanding their basic mechanics. It involves delving into the nuances, exploring advanced strategies, and being cognizant of challenges that might arise in real-world scenarios.

Serialization, with its automatic handling, proves indispensable in various contexts — from caching objects to transmitting data between distributed systems. Deserialization, while powerful, demands careful consideration of security, versioning, and class evolution. Externalization, offering fine-grained control, empowers developers to optimize the serialization and deserialization processes based on specific use cases, from handling circular references to customizing strategies for encryption, compression, and resource management.

As you navigate the intricate landscape of Java object data management, may this guide serve as a comprehensive resource, equipping you with the knowledge and skills to architect resilient, secure, and high-performance applications. Keep exploring, experimenting, and mastering these fundamental Java techniques — for in the realm of serialization, deserialization, and externalization, lies the key to unlocking the full potential of your Java applications. Happy coding!