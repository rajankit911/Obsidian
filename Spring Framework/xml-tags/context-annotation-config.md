The `<context:annotation-config>` annotation is mainly used to activate the annotations in already registered beans in application context.
<br>

> Note that it simply does not matter whether bean was registered by which mechanism e.g. using `<context:component-scan>` or it was defined in `application-context.xml` file itself.

<br>

It mainly activates the **4** types of **BeanPostProcessors**

-   **CommonAnnotationBeanPostProcessor** : @PostConstruct, @PreDestroy, @Resource
-   **AutowiredAnnotationBeanPostProcessor** : @Autowired, @Value, @Inject, @Qualifier, etc
-   **RequiredAnnotationBeanPostProcessor** : @Required annotation
-   **PersistenceAnnotationBeanPostProcessor** : @PersistenceUnit and @PersistenceContext annotations

<br>

Annotation | Description
------------ | ------------
[[@Required]] | The @Required annotation applies to bean property setter methods.
[[@Autowired]] | The @Autowired annotation can apply to bean property setter methods, non-setter methods, constructor and properties.
[[@Qualifier]] | The @Qualifier annotation along with @Autowired can be used to remove the confusion by specifiying which exact bean will be wired.
[[@Resourse]] (JSR-250) | _@Resource_ resolves dependencies either by field injection or by setter injection.
[[@PostConstruct]] (JSR-250) | Spring calls methods annotated with _@PostConstruct_ only once, just after the initialization of bean properties.
[[@PreDestroy]] (JSR-250) | A method annotated with _@PreDestroy_ runs only once, just before Spring removes our bean from the application context.