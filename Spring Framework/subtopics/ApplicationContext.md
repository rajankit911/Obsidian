#spring-ioc

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

<hr>

**BeanFactory** or **ApplicationContext**?

**ApplicationContext** is generally recommended over the **BeanFactory** because the **ApplicationContext** includes all functionality of the **BeanFactory**. Furthermore, it provides more enterprise-specific functionalities. This is why we use it as the default Spring container.

Feature | BeanFactory | ApplicationContext
------------ | ------------ | ------------
Bean instantiation/wiring | Yes | Yes
Automatic `BeanPostProcessor` registration | No | Yes
Automatic `BeanFactoryPostProcessor` registration | No | Yes
Convenient `MessageSource` access (for i18n) | No | Yes
`ApplicationEvent` publication | No | Yes


There are some limited situations, such as in an `Applet` where memory consumption might be critical and a few extra kilobytes might make a difference. In those scenarios, it would be justifiable to use the more lightweight **BeanFactory**.
