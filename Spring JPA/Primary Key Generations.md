# `GenerationType.AUTO`

- It is the default generation type. Hibernate selects the generation strategy based on the database specific dialect.

```java
@Id
@GeneratedValue(strategy = GenerationType.AUTO)
private Long id;
```

# `GenerationType.IDENTITY`

- Hibernate relies on an auto-incremented database column to generate the primary key.
- Hibernate lets the database generate a new value with each insert operation.

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

> [!warning]
> - This approach has a significant drawback if you use Hibernate for bulk insertion. 
> ```java
> Employee emp = new Employee();
> em.persist(emp); // FORCES IMMEDIATE INSERT!
> ```
> - The database (e.g., MySQL `AUTO_INCREMENT`) generates the ID **during** the SQL `INSERT`.
> - Hibernate requires a primary key value for each managed entity and therefore must perform the `INSERT` statement immediately to fetch the auto-generated ID.
> - This prevents Hibernate from ==delaying inserts== and ==grouping inserts== into batches.
> - Settings like `hibernate.jdbc.batch_size` have **no effect** with `IDENTITY`.
> - With `IDENTITY`, it becomes **sequential inserts**.

# `GenerationType.SEQUENCE`

- Hibernate uses a database ==sequence== to generate primary key values.
- It requires additional select statements to get the next value from a database sequence.

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE)
private Long id;
```

> [!note] 
> If you don’t provide any database ==sequence==, Hibernate will request the next value from its default sequence.

- You can provide a database ==sequence== using `@SequenceGenerator`.
- The `@SequenceGenerator` annotation lets you define the name of the generator, the name, and schema of the database sequence and the allocation size of the sequence.
- Hibernate runs `SELECT nextval('book_seq')` **once** to reserve IDs 1-50, then 51-100, etc.

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "book_generator")
@SequenceGenerator(name = "book_generator", sequenceName = "book_seq",
					allocationSize = 50)
private Long id;
```

> [!info] `GenerationType.SEQUENCE` is Preferred Over `GenerationType.IDENTITY` for Batch Operations
> - Hibernate gets IDs **in advance** from a database sequence (e.g., 50 IDs at once).
> - Hibernate can queue entities and flush them in batches.
> ```java
> for (int i = 0; i < 100; i++) {
> 	Employee emp = new Employee(); // ID fetched from pre-allocated block
> 	em.persist(emp); // Queued, not executed immediately
> }
> ```

> [!tip] Configuration Tips for `SEQUENCE`
> - **Allocation Size**: Match `allocationSize` to `batch_size`:
> ```java
> @SequenceGenerator(..., allocationSize = 50)
> ```
> ```properties
> hibernate.jdbc.batch_size=50
> ```
> - **Sequence Optimization** (Database-Side): Cache sequences (e.g., `CREATE SEQUENCE seq_employee CACHE 20`).
> - **Hibernate Dialect**: Ensure it supports sequences (e.g., `PostgreSQLDialect`, `OracleDialect`).



# `GenerationType.TABLE`

- Hibernate uses a database table to simulate a ==sequence== by storing and updating its current value.
-  `@TableGenerator` annotation works in a similar way as the `@SequenceGenerator` annotation to specify the database table which Hibernate shall use to simulate the ==sequence==.

```java
@Id
@GeneratedValue(strategy = GenerationType.TABLE, generator = "book_generator")
@TableGenerator(
		name = "book_generator",
		table = "id_generator",
		schema = "bookstore",
		pkColumnName = "gen_name",
        valueColumnName = "gen_value",
        pkColumnValue = "book_id",
        allocationSize = 10)
private Long id;
```

Then create a table in MySQL:

```sql
CREATE TABLE id_generator (
    gen_name VARCHAR(50) PRIMARY KEY,
    gen_value BIGINT
);
INSERT INTO id_generator (gen_name, gen_value) VALUES ('book_id', 0);
```

When Hibernate wants a new ID:

1. It performs a **SELECT** to read the current value.
2. It does an **UPDATE** to increment the value.

```sql
SELECT gen_value FROM id_generator WHERE gen_name = 'product_id';
UPDATE id_generator SET gen_value = gen_value + 1 WHERE gen_name = 'product_id';
```

So every ID generation requires **at least 2 queries** in comparison to just one call in `SEQUENCE`

```sql
SELECT nextval('seq_name')
```

Since the generator table has **one row per ID series**, only one transaction can safely increment the same row at a time. Therefore, It is **not as performant** as real ==sequences== as It requires the use of pessimistic locks which put all transactions into a sequential order.

However, ==sequences== can serve **multiple threads in parallel** without contention because they:
- live **in-memory**
- are **atomic and fast**