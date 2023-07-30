> To do list:
> -	[ ] "push" configuration vs "pull" configuration

#spring-ioc

```java
package org.springframework.beans.factory;

public interface BeanFactory {...}
```

The **BeanFactory** is the most basic version of IOC containers.

Spring's **DI** functinality is implemented using **BeanFactory** interface and its subinterfaces.

A **BeanFactory** will load bean definitions stored in a configuration source (such as an XML document), and use the `org.springframework.beans` package to configure the beans. Depending on the bean definition, the factory will return either **an independent instance of a contained object (the Prototype design pattern)**, or **a single shared instance (the Singleton design pattern)**.

One of the most popularly used implementation of **BeanFactory** is the **XMLBeanFactory**.

<hr>

**BeanFactory** implementations should support the standard bean lifecycle interfaces as far as possible. The full set of initialization methods and their standard order is:

1.  BeanNameAware's `setBeanName`
2.  BeanClassLoaderAware's `setBeanClassLoader`
3.  BeanFactoryAware's `setBeanFactory`
4.  EnvironmentAware's `setEnvironment`
5.  EmbeddedValueResolverAware's `setEmbeddedValueResolver`

**(only applicable when running in an application context)**

6.  ResourceLoaderAware's `setResourceLoader`
7.  ApplicationEventPublisherAware's `setApplicationEventPublisher`
8.  MessageSourceAware's `setMessageSource`
9.  ApplicationContextAware's `setApplicationContext`

**(only applicable when running in a web application context)**

10.  ServletContextAware's `setServletContext`
11.  `postProcessBeforeInitialization` methods of BeanPostProcessors
12.  InitializingBean's `afterPropertiesSet`
13.  a custom `init-method` definition
14.  `postProcessAfterInitialization` methods of BeanPostProcessors

On shutdown of a bean factory, the following lifecycle methods apply:

1.  `postProcessBeforeDestruction` methods of DestructionAwareBeanPostProcessors
2.  DisposableBean's `destroy`
3.  a custom `destroy-method` definition