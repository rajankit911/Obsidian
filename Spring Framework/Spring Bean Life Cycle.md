Life Cycle of Bean is easy to understand. When a bean is instantiated, it may be required to perform some initialization to get it into a usable state. Similarly, when the bean is no longer required and is remove from the container, some cleanup may be required.

You can use `init-method` and `destory-method` as attribute in bean configuration file for bean to perform certain actions upon initialization and destruction.

- **Initialization** - `org.springframework.beans.factory.InitializingBean` Interface contains one single method `void afterPropertiesSet() throws Exception`
- **Destruction** - `org.springframework.beans.factory.DisposableBean` Interface contains one single method `void destroy() throws Exception` 



# Phase 7: Final Registration and Caching

After successful initialization:

**Singleton Registration**: For singleton beans, the fully initialized instance is added to the first-level cache (`singletonObjects`)

**Early Reference Cleanup**: Removes any early references from second and third-level caches

**Creation Set Cleanup**: Removes bean name from `singletonsCurrentlyInCreation` set

**Destruction Registration**: If bean implements `DisposableBean` or has custom destroy methods, registers for destruction callbacks


# Three-Level Cache System

Spring Framework employs a sophisticated three-level caching mechanism to manage singleton bean lifecycles and resolve circular dependencies. This system is implemented in the `DefaultSingletonBeanRegistry` class and is crucial for handling complex dependency injection scenarios.

## Level 1: singletonObjects

**Type**: `ConcurrentHashMap<String, Object>`  
**Purpose**: The primary cache storing fully initialized and configured singleton beans
**Content**: Complete beans that have gone through the entire lifecycle - instantiation, property population, and initialization  
**Access Pattern**: First place Spring looks when retrieving a bean

## Level 2: earlySingletonObjects

**Type**: `HashMap<String, Object>`  
**Purpose**: Stores early singleton objects exposed before complete initialization
**Content**: Bean instances that have been instantiated but haven't had their properties populated yet  
**Access Pattern**: Checked when a bean is not found in Level 1 cache and circular dependency resolution is needed

## Level 3: singletonFactories

**Type**: `HashMap<String, ObjectFactory<?>>`  
**Purpose**: Stores factory objects that can create early bean references, potentially with AOP proxies
**Content**: `ObjectFactory` instances that wrap the bean creation logic and can produce proxy objects when needed  
**Access Pattern**: Used to create early bean references when circular dependencies are detected