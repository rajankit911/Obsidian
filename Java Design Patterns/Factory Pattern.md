Factory pattern is a creational design pattern that deals with **HOW** an object can be created at runtime.

In layman terms, when a certain part of your application wants to do something using an object, but that object itself can vary in different use cases, you give the responsibility of the object creation to someone else. That “someone else” is the **factory/creator** and the method in that class responsible for creating the object is the **factory method**.

The key point to note here is that, factory pattern only deals with the object creation. What we do with that object after it has been created, is not a factory pattern’s concern.

The object being created can be anything — it can be a normal class with behaviours, it can be a data class (POJO), it can be an interface, etc

There are 2 variations of the factory design pattern

## 1. Abstract factory method

This variation relies on inheritance. We create a factory class with an abstract factory method, and then allow subclasses to override the factory method to return a concrete object from the method.

> This is not to be confused with the abstract factory design pattern.

A common use case for this variation is in frameworks that allow certain “gaps” to be filled in by the client.

![[abstract-factory-method.png]]

## 2. Parameterised factory method

Another variation on the pattern is to write a concrete factory and to completely give the responsibility of creating the object to its factory method. The factory method takes a parameter that identifies the kind of object to create.

![[parameterised-factory-pattern.png]]

