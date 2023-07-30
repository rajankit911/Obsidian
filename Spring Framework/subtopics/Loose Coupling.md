> To do list:
> -	[ ] Tight Coupling 
# Loose Coupling

In computing and system design, **Loose coupling** is an approach to interconnect the components in such a way that those components depend on each other to the least extent possible.

In a **loosely coupled** system, each of its components has little or no knowledge of the definitions of other components.

> The goal of a loose coupling architecture is to reduce the risk that a change made within one element will create unanticipated changes within other elements.

Loose coupling is the opposite of tight coupling.

Loose coupling is an architectural principle and design goal in **service-oriented architectures**.

### Advantages
-	Components can be replaced with alternative implementations that provide the same services _**(Interchangeability)**_.
-	Increases flexibility and re-usability of code.
-	Simplifies testing, maintenance and troubleshooting procedures.

### Disadvantages
-	Additional coordination protocols are required to provide transactional integrity.
-	Issues in maintaining data consistency.

### In programming

Tight coupling occurs when a dependent class contains a pointer directly to a concrete class which provides the required behavior. The dependency cannot be substituted, or its "signature" changed, without requiring a change to the dependent class. 

Loose coupling occurs when the dependent class contains a pointer only to an interface, which can then be implemented by one or many concrete classes. Loose coupling in computing is interpreted as **encapsulation**.

Any class that implements the interface can thus satisfy the dependency of a dependent class without having to change the class. 

This allows for extensibility in software design; a new class implementing an interface can be written to replace a current dependency in some or all situations, without requiring a change to the dependent class; the new and old classes can be interchanged freely. Strong coupling does not allow this.


### Difference between tight coupling and loose coupling

![[coupling-types.png]]

-   Tight coupling is not good at the test-ability. But loose coupling improves the test ability.
-   Tight coupling does not provide the concept of interface. But loose coupling helps us follow the _GOF_ principle of program to interfaces, not implementations.
-   In Tight coupling, it is not easy to swap the codes between two classes. But it’s much easier to swap other pieces of code/modules/objects/components in loose coupling.
-   Tight coupling does not have the changing capability. But loose coupling is highly changeable.



