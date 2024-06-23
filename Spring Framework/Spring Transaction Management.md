- In a spring boot project, we don't need to use Hibernate Transaction Management
- `@Transactional` annotation is used to wrap a method or class in database transaction
- It allows us to set below conditions for our transaction
	- propagation
	- isolation
	- timeout
	- read-only
	- rollback

# How `@Transactional` works internally?

- Spring creates a proxy, or manipulates the class byte-code, to manage creation, commit and rollback of the transaction.
- If we have a method like `addEmployee()` and we mark it as `@Transactional`, Spring will wrap some transaction management code around the invocation `@Transactional` method called:

```java
createTransactionIfNecessar();
try {
	addEmployee();
	commitTransactionAfterReturning();
} catch (Exception e) {
	rollbackTransactionAfterThrowing();
	throw e;
}
```

# How to use `@Transational`

You can use this annotation on following in the lowest to highest priority order:
- interface
- superclass
- class
- interface method
- superclass method
- method


# Transaction Propagation

## REQUIRED

- REQUIRED is the default propagation
- Spring checks if there is an active transaction, it uses the existing one and if nothing exists, it creates a new one.
- If exception occurs, both transactions, parent and child will be rolled back as they both use same physical transaction

## NESTED

- It acts like REQUIRED, If there's no active transaction.
- Spring checks if a transaction exists, and if so, it marks a save point. This means that if our business logic execution throws an exception, then the transaction rollbacks to this save point.

## REQUIRED_NEW

- Spring suspends the current transaction if it exists, and then creates a new one.
- Spring creates a new inner physical transaction without inheriting the characteristics of an outer physical transaction.
- A new inner physical transaction will have its own database connection.
- In synchronous execution, while the inner physical transaction is running, the outer physical transaction is suspended and its database connection remains open. After the inner physical transaction commits, the outer physical transaction is resumed, continuing to run and commit / rollback.

## MANDATORY

- No new transaction is created by the Spring, but there must be an active transaction running, otherwise it will cause an exception
- If the inner logical transaction is rolled back, then outer logical transaction is rolled back as well.

## NEVER 

- It states that no physical transaction should exist. If a physical transaction is found, it will cause an exception.
- The code inside the method can open the physical transactions with no problem.

## NOT_SUPPORTED

- Spring suspends a current transaction if it exists,  and the business logic is executed without a transaction.
- The physical transaction will be automatically resume after the completion of the business logic.
- If the business logic throws an exception and it propagated to the transaction boundary, then logical transaction is rolled back.

## SUPPORTS

- Spring checks if an active transaction exists, then the existing transaction will be used, otherwise, the business logic will be executed without a transaction.