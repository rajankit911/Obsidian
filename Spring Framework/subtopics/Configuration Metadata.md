> To do list:
> -	[ ] Full @Configuration vs 'lite' @Beans mode

### Configuration Metadata

Configuration metadata provides the instructions to Spring IoC container on how to initiate, configure, wire and assemble the application specific objects.

![[spring-ioc-configuration-metadata.png]]

Configuration metadata can be provided to Spring container in following ways:

-	<u>**XML-Based Configuration**</u>
Configuration metadata is supplied in XML format

```xml
<bean id="studentbean" class="org.edureka.StudentBean">
	<property name="name" value="Edureka"></property>
</bean>
```
<br>

-	<u>**Annotation-Based Configuration**</u>
Starting from ==Spring 2.5==, it became possible to configure the dependency injection using **annotations**. Instead of using XML to describe a bean wiring, the developer moves the configuration into the component class itself by using annotations on the relevant class, method, or field declaration.<br>
Annotation wiring is not turned on in the Spring container by default. **In order to activate them, we can add either `<context:annotation-config>` or `<context:component-scan>` on top of our Spring Configuration file.**

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

> Annotation injection is performed _before_ XML injection, thus the latter configuration will override the former for properties wired through both approaches.


-	<u>**Java-Based Configuration**</u>
Starting with ==Spring 3.0==, Java-based configuration allows you to define beans using Java rather than using the traditional XML files.<br>
Spring provides [[@Configuration]]-annotated classes and [[@Bean]]-annotated methods for Java-based configuration. The simplest possible @Configuration class would read as follows:
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

Here, the method name is annotated with @Bean works as bean ID and it creates and returns the actual bean. Once your configuration classes are defined, you can load and provide them to Spring container using _**AnnotationConfigApplicationContext**_ as follows:

```java
public static void main(String[] args) {
  ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
  MyService myService = ctx.getBean(MyService.class);
  // code
}
```

_**AnnotationConfigApplicationContext**_ is not limited to working only with @Configuration classes. Any @Component or JSR-330 annotated class may be supplied as input to the constructor. <hr>

Spring also provides [[@Import]] which allows you to load @Bean definitions from another configuration class.

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

