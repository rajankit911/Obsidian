Spring follows a **standard bean lifecycle** with proxy creation happening **after full initialization**:

# Standard Bean Creation Timeline:

1. **Bean Instantiation**: Create raw object instance
2. **Dependency Injection**: Populate fields and setter dependencies
3. **Initialization**: Call `@PostConstruct`, `InitializingBean.afterPropertiesSet()`
4. **Post-Processing**: `BeanPostProcessor.postProcessAfterInitialization()` - **Proxy created here**
5. **Bean Ready**: Fully initialized proxied bean returned

**Code flow in normal cases:**

```java
// In AbstractAutowireCapableBeanFactory.doCreateBean()
Object exposedObject = bean;

// Step 1-3: Standard initialization
exposedObject = initializeBean(beanName, exposedObject, mbd);

// Step 4: Proxy creation via postProcessAfterInitialization()
if (exposedObject != null) {
    exposedObject = postProcessAfterInitialization(exposedObject, beanName);
}

```