- Structural design pattern in OOP
- Allows developers to dynamically add new functionality and behaviours to an object without affecting the behaviour of other existing objects of the same class
- Acts as a wrapper to the existing class
- Aggregation/composition is the key principle behind Decorator design pattern

Decorator class is an abstract wrapper class that contains the extended behaviours.

>A _wrapper_ is an object that can be linked with some _target_ object. The wrapper contains the same set of methods as the target and delegates to it all requests it receives. However, the wrapper may alter the result by doing something either before or after it passes the request to the target.


Decorator design patterns create decorator classes, which wrap the original class and supply additional functionality by keeping the class methods’ signature unchanged.

Decorator pattern is used a lot in Java IO classes, like FileReader, BufferedReader, etc

