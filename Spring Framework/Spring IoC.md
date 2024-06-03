> [!todo] Todo list:
>- [ ]  Service Locator Design Pattern
>- [ ]  Push vs Pull configuration
>- [ ]  Full `@Configuration` vs 'lite' `@Beans` mode

#spring-ioc 

One of the main features of the Spring framework is the **IoC (Inversion of Control) container**. The **Spring IoC container** is responsible for managing the objects of an application. It uses dependency injection to achieve inversion of control.

The **Spring IoC container** is at the core of the Spring Framework.

The Spring IoC container will
- create the objects
- configure them
- wire them together (assemble them)
- manage their complete life cycle from creation till destruction

# Working of Spring IOC

The Spring IoC Container gets its instructions on what objects to instantiate, configure, and assemble by reading **Configuration Metadata**.

In Spring, the objects that form the backbone of your application and that are managed by the Spring IoC _container_ are called _beans_. A bean is an object that is instantiated, assembled, and otherwise managed by a Spring IoC container. Otherwise, a bean is simply one of many objects in your application. Beans, and the _dependencies_ among them, are reflected in the _configuration metadata_ used by a container.


![[spring-ioc.png]]


# Configuration Metadata

Configuration metadata provides the instructions to Spring IoC container on how to initiate, configure, wire and assemble the application specific objects.

![[spring-ioc-configuration-metadata.png]]

Configuration metadata can be provided to Spring container in following ways:

## XML-Based Configuration

Configuration metadata is supplied in XML format

```xml
<bean id="studentbean" class="org.edureka.StudentBean">
	<property name="name" value="Edureka"></property>
</bean>
```

## Annotation-Based Configuration

Starting from ==Spring 2.5==, it became possible to configure the dependency injection using **annotations**. Instead of using XML to describe a bean wiring, the developer moves the configuration into the component class itself by using annotations on the relevant class, method, or field declaration.

Annotation wiring is not turned on in the Spring container by default. In order to activate them, we can add either `<context:annotation-config>` or `<context:component-scan>` on top of our Spring Configuration file.

```xml
<?xml version = "1.0" encoding = "UTF-8"?>

<beans xmlns = "http://www.springframework.org/schema/beans" 
	   xmlns:xsi = "http://www.w3.org/2001/XMLSchema-instance" 
	   xmlns:context = "http://www.springframework.org/schema/context"
	   xsi:schemaLocation = "http://www.springframework.org/schema/beans
		http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
		http://www.springframework.org/schema/context
		http://www.springframework.org/schema/context/spring-context-3.0.xsd">
	
	<context:annotation-config/>
	or
	<context:component-scan base-package="com.adeos.application"/>
	<!-- bean definitions go here -->
</beans>
```

> [!warning]
> Annotation injection is performed _before_ XML injection, thus the latter configuration will override the former for properties wired through both approaches.

## Java-Based Configuration

Starting with ==Spring 3.0==, Java-based configuration allows you to define beans using Java rather than using the traditional XML files.<br>
Spring provides `@Configuration`-annotated classes and `@Bean`-annotated methods for Java-based configuration. The simplest possible @Configuration class would read as follows:

```java
@Configuration
public class AppConfig {
  @Bean
  public MyService myService() {
      return new MyServiceImpl();
  }
}
```

The **AppConfig** class would be equivalent to the following Spring `<beans/>` XML:

```xml
<beans>
	<bean id="myService" class="com.acme.services.MyServiceImpl"/>
</beans>
```

Here, the method name is annotated with `@Bean` works as bean ID and it creates and returns the actual bean. Once your configuration classes are defined, you can load and provide them to Spring container using _**AnnotationConfigApplicationContext**_ as follows:

```java
public static void main(String[] args) {
  ApplicationContext ctx = 
	  new AnnotationConfigApplicationContext(AppConfig.class);
  MyService myService = ctx.getBean(MyService.class);
  // code
}
```

_**AnnotationConfigApplicationContext**_ is not limited to working only with `@Configuration` classes. Any `@Component` or `JSR-330` annotated class may be supplied as input to the constructor. 

Spring also provides `@Import` which allows you to load `@Bean` definitions from another configuration class.

```java
@Configuration
public class ConfigA {
  public @Bean A a() { return new A(); }
}

@Configuration
@Import(ConfigA.class)
public class ConfigB {
  public @Bean B b() { return new B(); }
}
```

Now, rather than needing to specify both **ConfigA.class** and **ConfigB.class** when instantiating the context, only **ConfigB** needs to be supplied explicitly:

```java
public static void main(String[] args) {
  ApplicationContext ctx = new AnnotationConfigApplicationContext(ConfigB.class);

  // now both beans A and B will be available...
  A a = ctx.getBean(A.class);
  B b = ctx.getBean(B.class);
}
```


# Types of Spring IOC

## Bean Factory

```java
package org.springframework.beans.factory;

public interface BeanFactory {...}
```

The **BeanFactory** is the most basic version of IOC containers.

Spring's **DI** functionality is implemented using **BeanFactory** interface and its subinterfaces.

A **BeanFactory** will load bean definitions stored in a configuration source (such as an XML document), and use the `org.springframework.beans` package to configure the beans. Depending on the bean definition, the factory will return either **an independent instance of a contained object (the Prototype design pattern)**, or **a single shared instance (the Singleton design pattern)**.

One of the most popularly used implementation of **BeanFactory** is the **XMLBeanFactory**.

---

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


## Application Context

```java
package org.springframework.context;

public interface ApplicationContext
	extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
	MessageSource, ApplicationEventPublisher, ResourcePatternResolver {...}
```

**ApplicationContext** is a sub-interface of **BeanFactory**.

The **BeanFactory** provides the configuration framework and basic functionality, and the **ApplicationContext** adds more enterprise-specific functionality. It provides:
- Easier integration with Spring's AOP features
- Message resource handling (for use in internationalization)
- Event publication
- Application-layer specific contexts such as the **WebApplicationContext** for use in web applications

The most popularly used ApplicationContext implementations are:
- **FileSystemXmlApplicationContext**
- **ClassPathXmlApplicationContext**
- **XmlWebApplicationContext**
- **AnnotationConfigApplicationContext**
- **AnnotationConfigWebApplicationContext**


> [!question] **BeanFactory** or **ApplicationContext**?

**ApplicationContext** is generally recommended over the **BeanFactory** because the **ApplicationContext** includes all functionality of the **BeanFactory**. Furthermore, it provides more enterprise-specific functionalities. This is why we use it as the default Spring container.

Feature | BeanFactory | ApplicationContext
------------ | ------------ | ------------
Bean instantiation/wiring | Yes | Yes
Automatic `BeanPostProcessor` registration | No | Yes
Automatic `BeanFactoryPostProcessor` registration | No | Yes
Convenient `MessageSource` access (for i18n) | No | Yes
`ApplicationEvent` publication | No | Yes

> [!note]
> **ApplicationContext** is the default Spring IoC container.


>[!question] When to use **BeanFactory**?

There are some limited situations, such as in an `Applet` where memory consumption might be critical and a few extra kilobytes might make a difference. In those scenarios, it would be justifiable to use the more lightweight **BeanFactory**.