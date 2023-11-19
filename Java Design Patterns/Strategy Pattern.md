> Strategy design pattern is a behavioural design pattern that allows a class to switch/run an algorithm at runtime.

- Behavioural Design Pattern
- Behaviour of an object can be changed at runtime
- Defines a family of algorithms or functionality, encapsulating them into separate classes that implement a common interface, and make their objects interchangeable.


Instead of implementing a single algorithm directly, your class receives run-time instructions as to which strategy to use in a family of algorithms.

In layman’s terms, strategy pattern gives a class the ability to perform a certain task in different **ways** or **strategies**, without that class knowing the implementation details of any **strategy**.

The class where you perform these strategies is called the **context** class.

# Structure

![[strategy-structure.png|500]]

- **Context**:
	It maintains a reference to one of the concrete strategies and communicates with this object only via the strategy interface. The context doesn’t know what type of strategy it works with or how the algorithm is executed. The context calls the execution method on the linked strategy object each time it needs to run the algorithm.
	
- **Strategy Interface**:
	It is common to all concrete strategies. It declares a method the context uses to execute a strategy.
	
- **Concrete Strategies**:
	These implement different variations of an algorithm the context uses.
	
- **Client**:
	It creates a specific strategy object and passes it to the context. The context exposes a setter which lets clients replace the strategy associated with the context at runtime.


The strategy class can be created in different ways:

## 1. The strategy can be passed as argument to the context

![[strategy-pattern-method-1.png]]


## 2. The context can have a method to set the strategy

![[strategy-pattern-method-2.png]]

## 3. The context can use a factory to decide which strategy to use

![[strategy-pattern-method-3.png]]



This might sound the same as the factory pattern, but the difference is in the **intent of the pattern**.

> Factory’s sole purpose is to create objects and strategy’s sole intent to is to provide a mechanism to select/run different algorithm.

Strategy pattern does not care on how a particular strategy class is created, it only deals with running the strategy from your context class.


1. Factory pattern deals with **how** a class can be created at runtime based on certain conditions.
2. Strategy pattern deals with **how** a class can perform any algorithm at runtime without worrying **which** algorithm it is performing nor worrying about **how** it was created.
3. Both patterns can be mixed to get a decent amount of encapsulation and modularity in our code.