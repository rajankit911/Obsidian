- A **functional interface** is an interface that contains only one abstract method. However, they can include any number of **default** and **static** methods.
- It is additionally recognized as **Single Abstract Method (SAM) Interfaces**.
- From Java 8 onwards, lambda expressions can be used to represent the instance of a functional interface.

>[!info] A lambda is an anonymous function that we can handle as a first-class language citizen. For instance, we can pass it to or return it from a method.


> [!note]
>1. **default** methods can be interhited in the child class
>2. **static** methods cannot be inherited in the child class


Before Java 8, we had to create anonymous inner class objects or implement these interfaces.

```java
class Test {
    public static void main(String args[]) {
        // create anonymous inner class object
        new Thread(new Runnable() {
            @Override public void run() {
                System.out.println("New thread created");
            }
        }).start();
    }
}
```

Java 8 onwards, we can assign lambda expression to its functional interface object like this:

```java
class Test {
	public static void main(String args[]) {
		// lambda expression to create the object
		new Thread(() -> System.out.println("New thread created")).start();
	}
}
```

# `@FunctionalInterface`

- `@FunctionalInterface` annotation is used to ensure that the functional interface can’t have more than one abstract method.
- In case more than one abstract methods are present, the compiler flags below message: `Unexpected @FunctionalInterface annotation`.

>[!note]
>It is not mandatory to use this annotation. It’s recommended that all functional interfaces should be annotated with `@FunctionalInterface`. This allows a compiler to generate an error if the annotated interface does not satisfy the conditions.


---
# Supplier`<T>`

- Represents a supplier of results. It does not take any arguments.
- There is no requirement that a new or distinct result be returned each time the supplier is invoked.
- This is a functional interface whose functional method is `get()`.

| Type    | Method      | Description        |
| ------- | ----------- | ------------------ |
| ==`T`== | ==`get()`== | ==Gets a result==. |

## Benefits

- **Promotes Functional Programming**: `Supplier` aligns with the functional programming paradigm by allowing functions to be passed as parameters, stored in variables, and returned from other functions.

- **Simplifies Code**: Using lambda expressions with `Supplier` leads to more concise and readable code compared to traditional approaches.

- **Reduces Boilerplate**: `Supplier` can significantly reduce boilerplate code, especially in scenarios where deferred execution or object creation is needed.

- **Readability**: Lambda expressions and method references (e.g., `MyClass::new`) used with `Supplier` make the code more readable and maintainable.

- **Deferred Execution**: `Supplier` allows for deferred execution of code. The object or result is created or fetched only when `get()` is called, enabling lazy initialization.

- **Performance Optimization**: This can improve performance by avoiding unnecessary computations and resource allocations until they are actually needed.

- **Encapsulation of Logic**: `Supplier` can encapsulate object creation or computation logic, promoting single responsibility and separation of concerns.

- **Reusability**: Suppliers can be reused across different parts of the application, enhancing code reusability.

- **Composable**: `Supplier` can be composed with other functional interfaces like `Function`, `Predicate`, and `Consumer`, providing more flexibility in functional programming.

- **Isolated Testing**: `Supplier` instances can be tested independently from the business logic, making unit testing more straightforward and isolating the logic for object creation or computation.

## Specialized primitive versions of the Supplier

### BooleanSupplier

- Represents a supplier of boolean-valued results.
- This is the boolean-producing primitive specialization of Supplier.
- This is a functional interface whose functional method is `getAsBoolean()`.

### DoubleSupplier

- Represents a supplier of double-valued results.
- This is the double-producing primitive specialization of Supplier.
- This is a functional interface whose functional method is `getAsDouble()`.

### IntSupplier

- Represents a supplier of int-valued results.
- This is the int-producing primitive specialization of Supplier.
- This is a functional interface whose functional method is `getAsInt()`.

### LongSupplier

- Represents a supplier of long-valued results.
- This is the long-producing primitive specialization of Supplier.
- This is a functional interface whose functional method is `getAsLong()`.

---
# Consumer`<T>`

Represents an operation that accepts a single input argument and returns no result. Unlike most other functional interfaces, Consumer is expected to operate via side-effects.

This is a functional interface whose functional method is `accept(Object)`.

| Type                  | Method                               | Description                                                                                            |
| --------------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| ==`void`==            | ==`accept(T t)`==                    | ==Performs this operation on the given argument==.                                                     |
| `default Consumer<T>` | `andThen(Consumer<? super T> after)` | Returns a composed Consumer that performs, in sequence, this operation followed by the after operation |

The lambda passed to the `forEach` method implements the `Consumer` functional interface:

```java
List<String> names = Arrays.asList("John", "Freddy", "Samuel");
names.forEach(name -> System.out.println("Hello, " + name));
```

## Benefits

- **Promotes Functional Programming**: `Consumer` supports the functional programming paradigm by allowing functions to be passed as parameters, stored in variables, and returned from other functions.

- **Lambda Expressions**: Using lambda expressions with `Consumer` leads to more concise and readable code, eliminating the need for anonymous inner classes.

- **Reduced Boilerplate**: Using `Consumer` can significantly reduce boilerplate code, especially compared to traditional anonymous inner classes.

- **Concise and Readable**: Lambda expressions and method references (e.g., `System.out::println`) used with `Consumer` make the code more concise and easier to understand.

- **Stream Operations**: `Consumer` is extensively used in the Stream API for operations like `forEach`, which allows for easy and efficient processing of elements in a stream.

- **Pipeline Processing**: You can chain multiple `Stream` operations using consumers, leading to cleaner and more maintainable code.

- **Single Responsibility**: By encapsulating processing logic within consumers, you can adhere to the single responsibility principle, making the code easier to maintain and test.

- **Separation of Concerns**: Logic for processing elements can be separated from business logic, enhancing code modularity.

- **Reusability**: `Consumer` instances can be reused across different parts of the application, promoting DRY (Don't Repeat Yourself) principles.

- **Composition**: Consumers can be composed using the `andThen` default method, allowing you to build complex processing logic from simpler ones.

- **Testable Units**: Consumers can be tested independently from the business logic, which makes unit testing more straightforward and isolates the processing logic.


## Specialized primitive versions of the Consumer

### DoubleConsumer

- Represents an operation that accepts a single double-valued argument and returns no result.
- This is the primitive type specialization of Consumer for double.
- This is a functional interface whose functional method is `accept(double)`.

### IntConsumer 

- Represents an operation that accepts a single int-valued argument and returns no result.
- This is the primitive type specialization of Consumer for int.
- This is a functional interface whose functional method is `accept(int)`.
### LongConsumer

- Represents an operation that accepts a single long-valued argument and returns no result.
- This is the primitive type specialization of Consumer for long.
- This is a functional interface whose functional method is `accept(long)`.

## BiConsumer`<T, U>`

To define lambdas with two arguments, we have to use additional interfaces that contain “_Bi”_ keyword in their names.

- Represents an operation that accepts two input arguments and returns no result.
- This is the two-arity [^1] specialization of Consumer.
- This is a functional interface whose functional method is `accept(Object, Object)`.

```java
Map<String, Integer> ages = new HashMap<>();
ages.put("John", 25);
ages.put("Freddy", 24);
ages.put("Samuel", 30);

ages.forEach((name, age) -> System.out.println(name + " is " + age + " years old"));
```

### Specialized primitive versions of the BiConsumer

#### ObjDoubleConsumer`<T>`

- Represents an operation that accepts an object-valued and a double-valued argument, and returns no result.
- This is the (reference, double) specialization of BiConsumer.
- This is a functional interface whose functional method is `accept(Object, double)`.

#### ObjIntConsumer`<T>`

- Represents an operation that accepts an object-valued and a int-valued argument, and returns no result.
- This is the (reference, int) specialization of BiConsumer.
- This is a functional interface whose functional method is `accept(Object, int)`.

#### ObjLongConsumer`<T>`

- Represents an operation that accepts an object-valued and a long-valued argument, and returns no result.
- This is the (reference, long) specialization of BiConsumer.
- This is a functional interface whose functional method is `accept(Object, long)`.

---
# Function`<T, R>`

The most simple and general case of a lambda is a functional interface with a method that receives one value and returns another. `Function` interface represents a function that accepts one argument and produces a result. This is a functional interface whose functional method is `apply(Object)`.

One of the usages of the `Function` type in the standard library is the `Map.computeIfAbsent` method. This method calculates a value by applying a function if a key is not already present in the map, inserts the calculated value into the map, and returns the value.

```java
Map<String, Integer> nameMap = new HashMap<>();
Integer value = nameMap.computeIfAbsent("John", s -> s.length());
```

We may replace the lambda with a method reference that matches passed and returned value types.

```java
Integer value = nameMap.computeIfAbsent("John", String::length);
```

| Type                        | Method                                            | Description                                                                                                                    |
| --------------------------- | ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| ==`R`==                     | ==`apply(T t)`==                                  | ==Applies this function to the given argument==.                                                                               |
| `default <V> Function<T,V>` | `andThen(Function<? super R,? extends V> after)`  | Returns a composed function that first applies this function to its input, and then applies the after function to the result.  |
| `default <V> Function<V,R>` | `compose(Function<? super V,? extends T> before)` | Returns a composed function that first applies the before function to its input, and then applies this function to the result. |
| `static <T> Function<T,T>`  | `negate()`                                        | Returns a function that always returns its input argument.                                                                     

## Specialized primitive versions of the Function

Since a primitive type can’t be a generic type argument, there are versions of the `Function` interface for the most used primitive types _double_, _int_, _long_, and their combinations in argument and return types:

### DoubleFunction`<R>`

- Represents a function that accepts a double-valued argument and produces a result.
- This is the double-consuming primitive specialization for Function.
- This is a functional interface whose functional method is `apply(double)`.

### ToDoubleFunction`<T>`

- Represents a function that produces a double-valued result.
- This is the double-producing primitive specialization for Function.
- This is a functional interface whose functional method is `applyAsDouble(Object)`.

### DoubleToIntFunction

- Represents a function that accepts a double-valued argument and produces an int-valued result.
- This is the double-to-int primitive specialization for Function.
- This is a functional interface whose functional method is `applyAsInt(double)`.

### DoubleToLongFunction

- Represents a function that accepts a double-valued argument and produces an long-valued result.
- This is the double-to-long primitive specialization for Function.
- This is a functional interface whose functional method is `applyAsLong(double)`.

### IntFunction`<R>`

- Represents a function that accepts a int-valued argument and produces a result.
- This is the int-consuming primitive specialization for Function.
- This is a functional interface whose functional method is `apply(int)`.

### ToIntFunction`<T>`

- Represents a function that produces a int-valued result.
- This is the int-producing primitive specialization for Function.
- This is a functional interface whose functional method is `applyAsInt(Object)`.

### IntToDoubleFunction

- Represents a function that accepts an int-valued argument and produces a double-valued result.
- This is the int-to-double primitive specialization for Function.
- This is a functional interface whose functional method is `applyAsDouble(int)`.

### IntToLongFunction

- Represents a function that accepts an int-valued argument and produces a long-valued result.
- This is the int-to-long primitive specialization for Function.
- This is a functional interface whose functional method is `applyAsLong(int)`.

### LongFunction`<R>`

- Represents a function that accepts a long-valued argument and produces a result.
- This is the long-consuming primitive specialization for Function.
- This is a functional interface whose functional method is `apply(long)`.

### ToLongFunction`<T>`

- Represents a function that produces a long-valued result.
- This is the long-producing primitive specialization for Function.
- This is a functional interface whose functional method is `applyAsLong(Object)`.

### LongToDoubleFunction

- Represents a function that accepts a long-valued argument and produces a double-valued result.
- This is the long-to-double primitive specialization for Function.
- This is a functional interface whose functional method is `applyAsDouble(long)`.

### LongToIntFunction

- Represents a function that accepts a long-valued argument and produces a int-valued result.
- This is the long-to-int primitive specialization for Function.
- This is a functional interface whose functional method is `applyAsInt(long)`.

## BiFunction`<T,U,R>`

To define lambdas with two arguments, we have to use additional interfaces that contain “_Bi”_ keyword in their names.

- Represents a function that accepts two arguments and produces a result.
- This is the two-arity[^1] specialization of Function.
- This is a functional interface whose functional method is `apply(Object, Object)`.

One of the typical examples of using this interface in the standard API is in the `Map.replaceAll` method, which allows replacing all values in a map with some computed value.

```java
Map<String, Integer> salaries = new HashMap<>();
salaries.put("John", 40000);
salaries.put("Freddy", 30000);
salaries.put("Samuel", 50000);

salaries.replaceAll((name, oldValue) ->
		name.equals("Freddy") ? oldValue : oldValue + 10000);
```
### Specialized primitive versions of the BiFunction
#### ToDoubleBiFunction`<T,U>`

- Represents a function that accepts two arguments and produces a double-valued result.
- This is the double-producing primitive specialization for BiFunction.
- This is a functional interface whose functional method is `applyAsDouble(Object, Object)`.

#### ToIntBiFunction`<T,U>`

- Represents a function that accepts two arguments and produces a int-valued result.
- This is the int-producing primitive specialization for BiFunction.
- This is a functional interface whose functional method is `applyAsInt(Object, Object)`.

#### ToLongBiFunction`<T,U>`

- Represents a function that accepts two arguments and produces a long-valued result.
- This is the long-producing primitive specialization for BiFunction.
- This is a functional interface whose functional method is `applyAsLong(Object, Object)`.

---
# Predicate`<T>`

The `Predicate` functional interface is a specialization of a `Function` that receives a generified value and returns a boolean.

Represents a predicate (boolean-valued function) of one argument.
It is a predefined functional Interface whose functional object is `test(Object)`.

 A typical use case of the _Predicate_ lambda is to filter a collection of values:
 
 ```java
List<String> names = Arrays.asList("Angela", "Aaron", "Bob", "Claire", "David");
List<String> namesWithA = names.stream().filter(name -> name.startsWith("A"))
		.collect(Collectors.toList());
```

| Type                   | Method                            | Description                                                                                               |
| ---------------------- | --------------------------------- | --------------------------------------------------------------------------------------------------------- |
| ==`boolean`==          | ==`test(T t)`==                   | ==Evaluates this predicate on the given argument==.                                                       |
| `default Predicate<T>` | `or(Predicate<? super T> other)`  | Returns a composed predicate that represents a short-circuiting logical OR of this predicate and another  |
| `default Predicate<T>` | `and(Predicate<? super T> other)` | Returns a composed predicate that represents a short-circuiting logical AND of this predicate and another |
| `default Predicate<T>` | `negate()`                        | Returns a predicate that represents the logical negation of this predicate                                |

## Benefits

- **Promotes Functional Programming**: `Predicate` supports the functional programming paradigm by allowing conditions to be encapsulated in functions that can be passed around as parameters, stored in variables, and returned from other functions.

- **Lambda Expressions**: Using lambda expressions with `Predicate` leads to more concise and readable code, eliminating the need for verbose anonymous inner classes.

- **Stream Operations**: Predicates are extensively used in the Stream API for operations like `filter`, `allMatch`, `anyMatch`, and `noneMatch`, allowing for complex data processing in a declarative and readable manner.

- **Pipeline Processing**: You can chain multiple `Stream` operations using predicates, leading to cleaner and more maintainable code.

- **Reduced Boilerplate**: Using `Predicate` can significantly reduce boilerplate code, especially compared to traditional approaches with loops and conditional statements.

- **Concise and Readable**: Lambda expressions and method references used with `Predicate` make the code more concise and easier to understand.

- **Reusability**: Predicates can be defined once and reused across different parts of the application, promoting DRY (Don't Repeat Yourself) principles.

- **Composition**: Predicates can be composed using default methods like `and`, `or`, and `negate`, allowing you to build complex conditions from simpler ones.

- **Single Responsibility**: By encapsulating condition logic within predicates, you can adhere to the single responsibility principle, making the code easier to maintain and test.

- **Separation of Concerns**: Logic for condition checks can be separated from business logic, enhancing code modularity.

- **Testable Units**: Predicates can be tested independently from the business logic, making unit testing more straightforward and isolating the logic for condition checks.

## Specialized primitive versions of the Predicate

### DoublePredicate

- Represents a predicate (boolean-valued function) of one double-valued argument.
- This is the double-consuming primitive type specialization of Predicate.
- This is a functional interface whose functional method is `test(double)`.

### IntPredicate

- Represents a predicate (boolean-valued function) of one int-valued argument.
- This is the int-consuming primitive type specialization of Predicate.
- This is a functional interface whose functional method is `test(int)`.

### LongPredicate

- Represents a predicate (boolean-valued function) of one long-valued argument.
- This is the long-consuming primitive type specialization of Predicate.
- This is a functional interface whose functional method is `test(long)`.

## BiPredicate`<T, U>`

To define lambdas with two arguments, we have to use additional interfaces that contain “_Bi”_ keyword in their names.

- Represents a predicate (boolean-valued function) of two arguments.
- This is the two-arity[^1] specialization of Predicate.
- This is a functional interface whose functional method is `test(Object, Object)`.

---
# Operators

_Operator_ interfaces are special cases of a function that receive and return the same value type.
## UnaryOperator`<R>`

- Represents an operation on a single operand that produces a result of the same type as its operand.
- This is a specialization of Function for the case where the operand and result are of the same type.
- This is a functional interface whose functional method is `Function.apply(Object)`.

One of its use cases in the Collections API is to replace all values in a list with some computed values of the same type:

```java
List<String> names = Arrays.asList("bob", "josh", "megan");
names.replaceAll(name -> name.toUpperCase());
```

The `List.replaceAll` function returns `void` as it replaces the values in place. To fit the purpose, the lambda used to transform the values of a list has to return the same result type as it receives. This is why the `UnaryOperator` is useful here.

Of course, instead of _name -> name.toUpperCase()_, we can simply use a method reference:

```java
names.replaceAll(String::toUpperCase);
```


### Specialized primitive versions of UnaryOperator

#### DoubleUnaryOperator

- Represents an operation on a single double-valued operand that produces a double-valued result.
- This is the primitive type specialization of UnaryOperator for double.
- This is a functional interface whose functional method is `applyAsDouble(double)`.

#### IntUnaryOperator

- Represents an operation on a single int-valued operand that produces a int-valued result.
- This is the primitive type specialization of UnaryOperator for int.
- This is a functional interface whose functional method is `applyAsInt(int)`.

#### LongUnaryOperator

- Represents an operation on a single long-valued operand that produces a long-valued result.
- This is the primitive type specialization of UnaryOperator for long.
- This is a functional interface whose functional method is `applyAsLong(long)`.

## BinaryOperator`<T>`

- Represents an operation upon two operands of the same type, producing a result of the same type as the operands.
- This is a specialization of BiFunction for the case where the operands and the result are all of the same type.
- This is a functional interface whose functional method is `BiFunction.apply(Object, Object)`.

One of the most interesting use cases of a `BinaryOperator` is a reduction operation. Suppose we want to aggregate a collection of integers in a sum of all values. With `Stream` API, we could do this using a collector, but a more generic way to do it would be to use the _reduce_ method:

```java
List<Integer> values = Arrays.asList(3, 5, 8, 9, 12);
int sum = values.stream().reduce(0, (i1, i2) -> i1 + i2);
```

The `reduce` method receives an initial accumulator value and a `BinaryOperator` function. The arguments of this function are a pair of values of the same type; the function itself also contains a logic for joining them in a single value of the same type. **The passed function must be associative**, which means that the order of value aggregation does not matter, i.e. the following condition should hold:

```java
op.apply(a, op.apply(b, c)) == op.apply(op.apply(a, b), c)
```

The associative property of a _BinaryOperator_ operator function allows us to easily parallelize the reduction process.

### Specialized primitive versions of the BinaryOperator

#### DoubleBinaryOperator

- Represents an operation upon two double-valued operands and producing a double-valued result.
- This is the primitive type specialization of BinaryOperator for double.
- This is a functional interface whose functional method is `applyAsDouble(double, double)`.

#### IntBinaryOperator

- Represents an operation upon two int-valued operands and producing a int-valued result.
- This is the primitive type specialization of BinaryOperator for inr.
- This is a functional interface whose functional method is `applyAsInt(int, int)`.

#### LongBinaryOperator

- Represents an operation upon two long-valued operands and producing a long-valued result.
- This is the primitive type specialization of BinaryOperator for long.
- This is a functional interface whose functional method is `applyAsLong(long, long)`.


[^1]: arity - the number of arguments that a function can take.