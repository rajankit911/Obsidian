Before ==Spring 2.5==, all the beans has to be manually configured in the XML files. This will result in lot of XML code in the configuration files and every time developer has to update the XML file for configuring the new beans.

<br>

`<context:component-scan>` in the Spring configuration file would eliminate the need for declaring all the beans in the XML files.

```xml
<context:component-scan base-package="com.adeos.application"/>
```

**Basically,** `<context:component-scan>` **detects the annotations by scanning the classes inside the specified package and activates them and also registers the bean instances to the application context who are annotated by @Component.**

`<context:component-scan>` recognizes bean annotations that `<context:annotation-config>` doesn't detect. [[@Component]], [[@Repository]], [[@Service]], [[@Controller]], and [[@RestController]] are several ones that `<context:component-scan>` can detect.

In short what we can say is that `<context:component-scan>` does what `<context:annotation-config>` does as well as registers the beans to the context

```xml
<context:component-scan> = <context:annotation-config> + Bean Registration
```

> if you declare `<context:component-scan>`, then it is not necessary anymore declare `<context:annotation-config>` too.

