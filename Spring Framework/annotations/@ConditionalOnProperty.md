- Bean is created conditionally

# Example

If you have two applications A1 and A2. Both applications are using a common class, ie, `DBConnection` which contains the objects of `SqlConnection` and `NoSqlConnection`.

```java
@Component
public class DBConnection {
	
	@Autowired
	private SqlConnection sqlConn;

	@Autowired
	private NoSqlConnection noSqlConn;

	....
}


@Component
public class SqlConnection { ... }

@Component
public class NoSqlConnection { ... }

```

>[!question]
if application A1 required only `SqlConnection` object and application A2 required only `NoSqlConnect` object. How will be make sure that `SqlConnection` object is created for A1 and `NoSqlConnect` object is created for A2?

>[!answer]
>By using `@ConditionalOnProperty`

```java
@Component
public class DBConnection {
	
	@Autowired(required = false)
	private SqlConnection sqlConn;

	@Autowired(required = false)
	private NoSqlConnection noSqlConn;

	....
}


@Component
@ConditionalOnProperty(prefix = "sql.connection", value = "enabled", havingValue = "true", matchIfMissing = false)
public class SqlConnection { ... }

@Component
@ConditionalOnProperty(prefix = "nosql.connection", value = "enabled", havingValue = "true", matchIfMissing = false)
public class NoSqlConnection { ... }

```

```properties
# application A1 properties
sql.connection.enabled=true

# application A2 properties
nosql.connection.enabled=true
```


# Advantages

- Toggling of feature
- Avoid cluttering application context with unnecessary beans
- Save memory
- Reduce application startup time

# Disadvantages

- Misconfiguration can happen
- Complexity of code increased when used for multiple beans