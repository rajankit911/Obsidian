Java provides two primary mechanisms for creating dynamic proxies at runtime:  
  
1. **JDK Dynamic Proxy** - Built into Java (`java.lang.reflect.Proxy`)  
2. **CGLIB** - Third-party library (Code Generation Library)  
  
Both allow intercepting method calls, but they work fundamentally differently.  
  
---  
  
## How They Work  
  
### JDK Dynamic Proxy  
  
- Creates a proxy class that **implements interfaces**  
- Uses Java's built-in reflection API  
- Target **must be an interface**  
- Part of standard Java since JDK 1.3  
  
```java  
// Creating a JDK Dynamic Proxy  
MyInterface proxy = (MyInterface) Proxy.newProxyInstance(
	MyInterface.class.getClassLoader(),
	new Class<?>[] { MyInterface.class },
	new MyInvocationHandler());  
```  
  
### CGLIB (Code Generation Library)  
  
- Creates a proxy class that **extends (subclasses)** the target class  
- Generates bytecode at runtime using ASM library  
- Can proxy **concrete classes** (no interface needed)  
- Cannot proxy `final` classes or `final` methods  
  
```java  
// Creating a CGLIB Proxy  
Enhancer enhancer = new Enhancer();  
enhancer.setSuperclass(MyClass.class);  
enhancer.setCallback(new MyMethodInterceptor());  
MyClass proxy = (MyClass) enhancer.create();  
```  
  
---  
  
## Visual Comparison  
  
### JDK Dynamic Proxy Architecture  
  
```
					┌─────────────────────┐
                    │   MyDAOInterface    │  (interface)
                    └─────────────────────┘
			                    ▲
			                    │ implements
			    ┌───────────────┴───────────────┐
			    │                               │
		┌─────────────────┐             ┌─────────────────┐
		│    $Proxy0      │             │  MongoDAOImpl   │
		│   (generated)   │             │  (your impl)    │
		└─────────────────┘             └─────────────────┘
		        │
		        │ delegates to
		        ▼
		┌─────────────────┐
		│InvocationHandler│
		└─────────────────┘
``` 

### CGLIB Architecture

```
	┌─────────────────────┐
    │     MyDAOClass      │  (concrete class)
    └─────────────────────┘
		    ▲
		    │ extends
    ┌─────────────────────┐
    │   MyDAOClass$$      │
    │  EnhancerByCGLIB$$  │  (generated subclass)
    └─────────────────────┘
			│
		    │ delegates to
		    ▼
	┌─────────────────────┐
	│  MethodInterceptor  │
	└─────────────────────┘
```
  
---  
  
## Feature Comparison  
  
| Feature | JDK Dynamic Proxy | CGLIB |  
|---------|-------------------|-------|  
| **Proxy Target** | Interfaces only | Classes and interfaces |  
| **Mechanism** | Implements interface | Extends class (subclassing) |  
| **External Dependency** | No (built-in) | Yes (cglib library) |  
| **Final Classes** | N/A (interfaces) | Cannot proxy |  
| **Final Methods** | N/A (interfaces) | Cannot intercept |  
| **Performance** | Slightly faster for interfaces | Better for classes |  
| **Java 17+ Compatibility** | Full support | Requires `--add-opens` JVM args |  
| **Spring Default** | For interfaces | For classes without interfaces |  
  
---  
  
## Java 17+ Module System Issue  
  
### Why CGLIB Fails on Java 17+  
  
CGLIB uses reflection to access internal JDK classes for bytecode injection:  
  
```java  
// CGLIB internally does this:  
Method defineClass = ClassLoader.class.getDeclaredMethod(  
    "defineClass",    String.class, byte[].class, int.class, int.class, ProtectionDomain.class);  
defineClass.setAccessible(true);  // ❌ BLOCKED by Java 17 module system  
```  
  
Java 9 introduced the **module system (Project Jigsaw)** which restricts reflective access to internal JDK APIs. This results in the error:  
  
```  
java.lang.reflect.InaccessibleObjectException:  
Unable to make protected final java.lang.Class  
java.lang.ClassLoader.defineClass(...) accessible:  
module java.base does not "opens java.lang" to unnamed module  
```  
  
### Workaround for CGLIB on Java 17+  
  
Add JVM arguments to open the required modules:  
  
```bash  
java --add-opens java.base/java.lang=ALL-UNNAMED \     --add-opens java.base/java.lang.reflect=ALL-UNNAMED \     -jar myapp.jar```  
  
### Why JDK Proxy Works  
  
JDK Dynamic Proxy is a **public API** - no internal reflection needed:  
  
```java  
// Standard Java API - no module restrictions  
Object proxy = Proxy.newProxyInstance(  
    classLoader,    interfaces,    invocationHandler);  
```  
  
---  
  
## When to Use Which  
  
### Use JDK Dynamic Proxy When:  
  
- Target is an **interface**  
- Running on **Java 9+** without custom JVM args  
- You want **no external dependencies**  
- Working with **Spring AOP** on interface-based beans  
  
### Use CGLIB When:  
  
- Target is a **concrete class without interface**  
- You can configure **JVM arguments** (`--add-opens`)  
- Using **Spring AOP** on class-based beans (Spring handles this internally)  
- Need to proxy **third-party classes** you can't modify  
  
---  
  
## Code Examples  
  
### JDK Dynamic Proxy Example  
  
```java  
// 1. Define the interface  
public interface UserService {  
    User findById(Long id);    void save(User user);
}
  
// 2. Create InvocationHandler  
public class LoggingHandler implements InvocationHandler {  
    private final Object target;  
    public LoggingHandler(Object target) {
		this.target = target;
    }
    
    @Override
    public Object invoke(Object proxy, Method method,
			Object[] args) throws Throwable {
        System.out.println("Before: " + method.getName());
        Object result = method.invoke(target, args);       
        System.out.println("After: " + method.getName());
        return result;
    }
}
  
// 3. Create proxy  
UserService realService = new UserServiceImpl();  
UserService proxy = (UserService) Proxy.newProxyInstance(
	UserService.class.getClassLoader(),
	new Class<?>[] { UserService.class },
	new LoggingHandler(realService));  
  
// 4. Use proxy  
proxy.findById(1L);  // Logging happens automatically  
```  
  
### CGLIB Example  
  
```java  
// 1. Define concrete class (no interface needed)  
public class UserService {  
    public User findById(Long id) { ... }    public void save(User user) { ... }}  
  
// 2. Create MethodInterceptor  
public class LoggingInterceptor implements MethodInterceptor {
 
    @Override
    public Object intercept(Object obj, Method method, Object[] args,
			MethodProxy proxy) throws Throwable {
		
		System.out.println("Before: " + method.getName());
		Object result = proxy.invokeSuper(obj, args);
		System.out.println("After: " + method.getName());
		return result;
	}}  
  
// 3. Create proxy  
Enhancer enhancer = new Enhancer();  
enhancer.setSuperclass(UserService.class);  
enhancer.setCallback(new LoggingInterceptor());  
UserService proxy = (UserService) enhancer.create();  
  
// 4. Use proxy  
proxy.findById(1L);  // Logging happens automatically  
```  
  
---  

## Adding Behavior in Dynamic Proxies

The core mechanism for adding behavior is the **InvocationHandler** (JDK) or **MethodInterceptor** (CGLIB). All method calls pass through these handlers, where you can add behavior **before**, **after**, or **around** the actual method execution.

### Execution Flow

```
Client → Proxy → Handler → [BEFORE] → Target Method → [AFTER] → Return
                    ↓
              [AROUND / EXCEPTION HANDLING]
```

### Basic Structure

```java
public class BehaviorHandler implements InvocationHandler {

    private final Object target;

    public BehaviorHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        // ========== BEFORE ==========
        long startTime = System.currentTimeMillis();
        log.info("Entering: {}.{}()", target.getClass().getSimpleName(), method.getName());

        try {
            // ========== INVOKE TARGET ==========
            Object result = method.invoke(target, args);

            // ========== AFTER (success) ==========
            log.info("Exiting: {}.{}() returned: {}",
                target.getClass().getSimpleName(), method.getName(), result);

            return result;

        } catch (Exception e) {
            // ========== AFTER (exception) ==========
            log.error("Exception in {}.{}(): {}",
                target.getClass().getSimpleName(), method.getName(), e.getMessage());
            throw e;

        } finally {
            // ========== ALWAYS (finally) ==========
            long duration = System.currentTimeMillis() - startTime;
            log.info("Duration: {} ms", duration);
        }
    }
}
```

### Common Behaviors to Add

#### 1. Logging

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    log.info("Calling: {}({})", method.getName(), Arrays.toString(args));
    Object result = method.invoke(target, args);
    log.info("Returned: {}", result);
    return result;
}
```

#### 2. Performance Metrics

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Timer.Sample sample = Timer.start(meterRegistry);
    try {
        return method.invoke(target, args);
    } finally {
        sample.stop(Timer.builder("dao.method.duration")
            .tag("method", method.getName())
            .register(meterRegistry));
    }
}
```

#### 3. Retry Logic

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    int maxRetries = 3;
    int attempt = 0;

    while (true) {
        try {
            attempt++;
            return method.invoke(target, args);
        } catch (Exception e) {
            if (attempt >= maxRetries) {
                throw e;
            }
            log.warn("Retry {}/{} for {}", attempt, maxRetries, method.getName());
            Thread.sleep(1000 * attempt);  // Exponential backoff
        }
    }
}
```

#### 4. Caching

```java
private final Map<String, Object> cache = new ConcurrentHashMap<>();

@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // Only cache read methods
    if (method.getName().startsWith("find") || method.getName().startsWith("get")) {
        String cacheKey = method.getName() + Arrays.toString(args);

        return cache.computeIfAbsent(cacheKey, k -> {
            try {
                return method.invoke(target, args);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        });
    }

    return method.invoke(target, args);
}
```

#### 5. Security / Authorization

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // Check if method requires authentication
    if (method.isAnnotationPresent(RequiresAuth.class)) {
        User currentUser = SecurityContext.getCurrentUser();
        if (currentUser == null) {
            throw new UnauthorizedException("Authentication required");
        }

        RequiresAuth auth = method.getAnnotation(RequiresAuth.class);
        if (!currentUser.hasRole(auth.role())) {
            throw new ForbiddenException("Insufficient permissions");
        }
    }

    return method.invoke(target, args);
}
```

#### 6. Transaction Management

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (method.getName().startsWith("save") || method.getName().startsWith("delete")) {
        Transaction tx = transactionManager.begin();
        try {
            Object result = method.invoke(target, args);
            tx.commit();
            return result;
        } catch (Exception e) {
            tx.rollback();
            throw e;
        }
    }

    return method.invoke(target, args);
}
```

### Chaining Multiple Behaviors (Decorator Pattern)

You can chain multiple handlers together for composable behaviors:

```java
public class ChainedHandler implements InvocationHandler {

    private final InvocationHandler next;

    public ChainedHandler(InvocationHandler next) {
        this.next = next;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // Add this handler's behavior
        doBefore(method);

        // Delegate to next handler in chain
        Object result = next.invoke(proxy, method, args);

        doAfter(method, result);
        return result;
    }
}

// Usage: Chain handlers
InvocationHandler target = new TargetHandler(realObject);
InvocationHandler logging = new LoggingHandler(target);
InvocationHandler metrics = new MetricsHandler(logging);
InvocationHandler retry = new RetryHandler(metrics);

// Create proxy with chained handlers
MyInterface proxy = (MyInterface) Proxy.newProxyInstance(
    classLoader,
    new Class<?>[] { MyInterface.class },
    retry  // outermost handler
);
```

**Call flow with chained handlers:**

```
Client → Retry → Metrics → Logging → Target → Response
           ↓        ↓         ↓         ↓
        [retry]  [timing]  [log]   [execute]
```

### Real-World Example: DAO Routing Proxy

In the B2C Search Service, `DAOInvocationHandler` adds **routing behavior**:

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // ========== ADDED BEHAVIOR: Routing ==========
    OperationType operationType = deriveOperationType(method);
    StorageType targetStorage = resolveTargetStorage(method, operationType);
    Object targetDAO = registry.getDAO(daoInterface, targetStorage);

    // ========== ADDED BEHAVIOR: Logging ==========
    log.debug("Routing {}.{}() -> {}",
        daoInterface.getSimpleName(), method.getName(), targetStorage);

    // ========== INVOKE ACTUAL DAO ==========
    return method.invoke(targetDAO, args);
}
```

### Behavior Types Summary

| Behavior Type | When to Add | Example Use Case |
|---------------|-------------|------------------|
| **Before** | Pre-processing | Validation, logging entry, auth check |
| **After (success)** | Post-processing | Logging result, cache update |
| **After (exception)** | Error handling | Logging error, alerting |
| **Around** | Wrap entire call | Timing, retry, transaction |
| **Conditional** | Based on method | Cache reads, transaction for writes |

The proxy pattern makes it easy to add **cross-cutting concerns** without modifying the actual business logic.

---
## Spring Framework Behavior  
  
Spring AOP automatically chooses the proxy type:  
  
| Scenario | Proxy Type |  
|----------|------------|  
| Bean implements interface | JDK Dynamic Proxy |  
| Bean is concrete class | CGLIB |  
| `@EnableAspectJAutoProxy(proxyTargetClass=true)` | Forces CGLIB |  
  
Spring's internal CGLIB (repackaged in `org.springframework.cglib`) handles Java 17+ compatibility when used through Spring's proxy mechanism.  
  
---  
  
## Performance Comparison  
  
| Operation | JDK Proxy | CGLIB |  
|-----------|-----------|-------|  
| Proxy creation | Faster | Slower (bytecode generation) |  
| Method invocation | Similar | Slightly faster (direct call via MethodProxy) |  
| Memory footprint | Smaller | Larger (generated classes) |  
  
For most applications, the performance difference is negligible.  
  
---  
  
## Summary  
  
| Decision Factor | Recommendation |  
|-----------------|----------------|  
| Proxying interfaces | **JDK Dynamic Proxy** |  
| Proxying classes | **CGLIB** (with JVM args on Java 17+) |  
| Java 17+ without JVM args | **JDK Dynamic Proxy** |  
| No external dependencies | **JDK Dynamic Proxy** |  
| Spring-managed beans | Let Spring decide (automatic) |  
  
---  
  
## References  
  
- [Java Proxy Documentation](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/reflect/Proxy.html)  
- [CGLIB GitHub](https://github.com/cglib/cglib)  
- [Spring AOP Proxying Mechanisms](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop-proxying)  
- [JEP 261: Module System](https://openjdk.org/jeps/261)

