# **Circular Dependency Resolution: Constructor vs Field Injection in Spring Boot**

In Spring Boot, circular dependency errors occur with constructor injection but not with field injection (`@Autowired`) due to the **fundamental differences in Spring's bean creation lifecycle** and the **three-level cache mechanism** Spring uses to resolve circular references.

# Why Constructor Injection Fails with Circular Dependencies

Constructor injection requires **all dependencies to be resolved at bean instantiation time**. When Spring encounters circular dependencies with constructor injection, it faces an unresolvable dilemma:

- **Bean A** requires **Bean B** in its constructor
- **Bean B** requires **Bean A** in its constructor
- Spring cannot determine which bean to create first, leading to a `BeanCurrentlyInCreationException`

This happens because constructor injection occurs during the **bean instantiation phase**, which is the earliest stage of Spring's bean creation process. At this point, neither bean exists yet, making it impossible to inject one into the other's constructor.

# Why Field Injection Works with Circular Dependencies

Field injection (`@Autowired`) works because it operates on a **two-phase creation process**:

1. **Instantiation Phase**: Spring creates the bean instance using the default constructor (no dependencies required)
2. **Initialization Phase**: Spring injects dependencies into fields using reflection, handled by `AutowiredAnnotationBeanPostProcessor

This separation allows Spring to:

- First create **empty instances** of both beans
- Then **populate the dependencies** after both objects exist in memory

# Spring's Three-Level Cache Mechanism

Spring resolves circular dependencies through a sophisticated **three-level cache system**:

1. **Singleton Objects Cache**: Fully initialized beans
2. **Early Singleton Objects Cache**: Beans under creation but not fully initialized
3. **Singleton Factories Cache**: Factory methods to create early references

The process works as follows:

1. Spring starts creating **Bean A** and puts its factory in the singleton factories cache
2. During **Bean A's** initialization, it needs **Bean B**
3. Spring starts creating **Bean B** and discovers it needs **Bean A**
4. Spring retrieves the **early reference** to **Bean A** from the cache (not fully initialized but instantiated)
5. **Bean B** gets the early reference and completes initialization
6. **Bean A** receives the fully initialized **Bean B** and completes its own initialization

## **Timeline Comparison**

**Constructor Injection Timeline:**

- Bean instantiation → **Dependency resolution required immediately** → Circular dependency detected → Exception thrown

**Field Injection Timeline:**

- Bean instantiation → Bean placed in cache → Dependencies resolved later → **Circular dependency resolved via early exposure**

## **Code Example**

**Constructor Injection (Fails):**

```java
@Service
public class ServiceA {
    private final ServiceB serviceB;
    
    public ServiceA(ServiceB serviceB) { // Requires ServiceB at creation
        this.serviceB = serviceB;
    }
}

@Service  
public class ServiceB {
    private final ServiceA serviceA;
    
    public ServiceB(ServiceA serviceA) { // Requires ServiceA at creation
        this.serviceA = serviceA;
    }
}
```

```java
@Service
public class ServiceA {
    @Autowired
    private ServiceB serviceB; // Injected after instantiation
}

@Service
public class ServiceB {
    @Autowired  
    private ServiceA serviceA; // Injected after instantiation
}
```

## **Spring Boot 2.6+ Changes**

Starting with **Spring Boot 2.6**, circular references are **prohibited by default** to encourage better design patterns. You can re-enable them with:

```text
spring.main.allow-circular-references=true
```

However, the recommended solutions are:

- **Refactor code** to eliminate circular dependencies
- Use **`@Lazy` annotation** to delay bean initialization
- Employ **setter injection** instead of constructor injection for circular scenarios

## **Best Practices**

While field injection resolves circular dependencies, **constructor injection remains the preferred approach** because it:

- **Enforces dependency requirements** at compile time
- **Promotes immutability** with `final` fields
- **Improves testability** by making dependencies explicit
- **Detects circular dependencies early**, encouraging better design

The fact that circular dependencies work with field injection but not constructor injection should be viewed as a **design smell indicator** rather than a reason to use field injection. The failure with constructor injection helps identify architectural issues that should be resolved through proper refactoring.

# How spring handles circular dependencies if AOP proxying is involved (like @Transactional, @Async, etc.)?

When AOP proxying is involved, Spring's circular dependency resolution becomes significantly **more complex and sometimes fails entirely**. The key difference lies in **when and how different types of proxies are created** during the bean lifecycle.

### The Critical Timing Difference

#### AOP Annotations (@Transactional, @Cacheable, custom aspects):

- Use AnnotationAwareAspectJAutoProxyCreator
- Implement SmartInstantiationAwareBeanPostProcessor
- Support early proxy creation via getEarlyBeanReference() method

#### @Async Annotation:

- Uses `AsyncAnnotationBeanPostProcessor`
- Extends `AbstractAdvisingBeanPostProcessor`
- Does NOT implement `SmartInstantiationAwareBeanPostProcessor`
- Only creates proxies during `postProcessAfterInitialization()`

### Spring's Early Reference Mechanism with AOP
When circular dependencies involve AOP proxies, Spring relies on the getEarlyBeanReference() mechanism:

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
                exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
            }
        }
    }
    return exposedObject;
}
```

## Why `@Transactional` Works with Circular Dependencies

#### `@Transactional`: `AnnotationAwareAspectJAutoProxyCreator`

```text
AnnotationAwareAspectJAutoProxyCreator
    ↓ extends
AspectJAwareAdvisorAutoProxyCreator  
    ↓ extends
AbstractAdvisorAutoProxyCreator
    ↓ extends  
AbstractAutoProxyCreator  ✅ (implements SmartInstantiationAwareBeanPostProcessor)
```

**Key Capability**: `AbstractAutoProxyCreator` **overrides** `getEarlyBeanReference()` with proxy creation logic:

```java
@Override
public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    this.earlyProxyReferences.put(cacheKey, bean);
    return wrapIfNecessary(bean, beanName, cacheKey);  // ✅ Creates proxy early
}
```

1. **Bean A** (with `@Transactional`) starts instantiation
2. **Bean A** is added to third-level cache with an `ObjectFactory`
3. **Bean B** needs **Bean A** during creation
4. Spring calls `getEarlyBeanReference()` on `AnnotationAwareAspectJAutoProxyCreator`
5. **Early proxy** of **Bean A** is created and injected into **Bean B**
6. During `postProcessAfterInitialization()`, the proxy creator **recognizes** that **Bean A** was already proxied early and **returns the same proxy**

## Why `@Async` Fails with Circular Dependencies

#### `@Async`: `AsyncAnnotationBeanPostProcessor`

**Earlier versions (pre-4.3)**:

```java
AsyncAnnotationBeanPostProcessor
    ↓ extends
AbstractAdvisingBeanPostProcessor  ❌ (does NOT implement SmartInstantiationAwareBeanPostProcessor)
```

**Spring Framework 4.3.0+ Architecture**:

```text
AsyncAnnotationBeanPostProcessor
    ↓ extends
AbstractBeanFactoryAwareAdvisingPostProcessor ✅ (implements SmartInstantiationAwareBeanPostProcessor)
    ↓ extends
AbstractAdvisingBeanPostProcessor  ❌ (does NOT override getEarlyBeanReference)
```

Even though **newer versions** of `AsyncAnnotationBeanPostProcessor` **do implement** `SmartInstantiationAwareBeanPostProcessor`, it **inherits the default implementation** from `AbstractAdvisingBeanPostProcessor`, which **does NOT override** `getEarlyBeanReference()`.

```java
// Default implementation in SmartInstantiationAwareBeanPostProcessor
default Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
    return bean;  // ❌ Returns raw bean, no proxy creation
}
```

The `AsyncAnnotationBeanPostProcessor` **cannot participate in early reference resolution**:

1. **Bean A** (with `@Async`) starts instantiation
2. **Bean A** is added to cache, but `AsyncAnnotationBeanPostProcessor` **cannot create early proxies**
3. **Raw Bean A instance** (not proxy) is injected into **Bean B**
4. During `postProcessAfterInitialization()`, `AsyncAnnotationBeanPostProcessor` creates a **new proxy**
5. Spring detects that the **early-exposed object differs from the final object**
6. **`BeanCurrentlyInCreationException`** is thrown

## **The Object Identity Problem**

Spring's validation logic checks object identity consistency:

```java
if (exposedObject != bean) {
    if (this.allowRawInjectionDespiteWrapping) {
        // Allow, but warn about inconsistency
    } else {
        throw new BeanCurrentlyInCreationException(beanName, 
            "Bean has been injected into other beans in its raw form " +
            "but has eventually been wrapped");
    }
}
```

The fundamental issue is **proxy creation timing**:

- **Early exposure**: Raw object or AOP proxy (via `getEarlyBeanReference()`)
- **Final object**: May be wrapped by additional proxies (`@Async`, `@Scheduled`)

## **Multiple AOP Proxies Scenario**

When a bean has **both `@Transactional` and `@Async`**, the processing order matters:[](https://github.com/spring-projects/spring-framework/issues/26156)

1. `AnnotationAwareAspectJAutoProxyCreator` runs first (creates transaction proxy)
2. `AsyncAnnotationBeanPostProcessor` runs second (wraps transaction proxy with async proxy)
3. **Result**: Multi-layered proxy structure

**Early reference** gets the transaction proxy, but **final bean** is the async-wrapped transaction proxy, causing identity mismatch.

## **Interface vs Class Proxying Impact**

The proxy mechanism used affects circular dependency behavior:[](https://jcs.ep.jhu.edu/ejava-springboot/coursedocs/content/html_single/aop-notes.html)

**JDK Dynamic Proxies** (interface-based):

- Lighter proxy creation
- Better early reference support
- Used when target implements interfaces

**CGLIB Proxies** (class-based):

- Heavier proxy creation process
- May complicate circular dependency resolution
- Used when `proxyTargetClass=true` or no interfaces

## **Solutions for AOP Circular Dependencies**

**1. Use `@Lazy` Injection**:

```java
@Service
public class ServiceA {
    @Autowired
    @Lazy
    private ServiceB serviceB;
    
    @Async
    public void processAsync() { /* ... */ }
}
```

**2. Refactor to Eliminate Circular Dependencies**:

- Extract common functionality to separate service
- Use event-driven architecture
- Apply dependency inversion principle

**3. Configure Raw Injection Tolerance**:

```java
@Component
public class CircularDependencyConfig implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        ((DefaultListableBeanFactory) beanFactory)
            .setAllowRawInjectionDespiteWrapping(true);
    }
}
```

**4. Use Constructor Injection with `@Lazy`**:

```java
@Service
public class ServiceA {
    private final ServiceB serviceB;
    
    public ServiceA(@Lazy ServiceB serviceB) {
        this.serviceB = serviceB;
    }
}
```

## **Spring Boot 2.6+ Behavior**

Starting with Spring Boot 2.6, **circular references are prohibited by default**. This affects AOP scenarios:

- Forces developers to resolve architectural issues
- Prevents subtle proxy-related bugs
- Can be re-enabled with `spring.main.allow-circular-references=true`

## **Best Practices for AOP and Circular Dependencies**

**Design-Level Solutions:**

- **Avoid circular dependencies** through proper architectural design
- Use **event publishing** instead of direct circular calls
- Apply **Single Responsibility Principle** to reduce coupling

**Implementation-Level Workarounds:**

- Prefer **`@Transactional`** over `@Async` in circular scenarios (when business logic allows)
- Use **`@Lazy`** strategically to break circular chains
- Consider **manual async execution** with `TaskExecutor` instead of `@Async`

The fundamental takeaway is that **Spring AOP's early reference mechanism works only for `SmartInstantiationAwareBeanPostProcessor` implementations**. Since `@Async` uses a different processor type, it cannot participate in early proxy creation, making circular dependencies with `@Async` inherently problematic.