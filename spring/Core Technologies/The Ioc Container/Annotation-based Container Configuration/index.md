# Annotation-based Container Configuration

基于注解的配置的引入提出了一个问题，即这种方法是否比XML“更好”。简短的回答是“看情况”。长篇大论的答案是，每种方法都有其优点和缺点，通常由开发者决定哪种策略更适合他们。由于它们的定义方式，注解在其声明中提供了大量上下文，从而导致更短、更简洁的配置。但是，XML擅长在不触及源代码或重新编译的情况下连接组件。一些开发人员更喜欢在源代码附近进行连接，而另一些开发人员则认为带注解的类不再是pojo，而且配置变得分散，难以控制。

无论选择什么，Spring都可以容纳这两种风格，甚至可以将它们混合在一起。值得指出的是，通过它的JavaConfig选项，Spring允许以一种非侵入性的方式使用注解，而不需要触及目标组件的源代码，而且，就工具而言，所有的配置样式都受到Spring Tools for Eclipse、Visual Studio code和Theia的支持。

基于注解的配置提供了XML设置的另一种选择，它依赖于字节码元数据而不是XML声明来连接组件。
开发人员不使用XML来描述bean连接，而是通过在相关的类、方法或字段声明上使用注解，将配置移动到组件类本身。正如在示例:AutowiredAnnotationBeanPostProcessor中所提到的，将BeanPostProcessor与注解结合使用是扩展Spring IoC容器的常用方法。例如，@Autowired注解提供了与自动装配协作器中描述的相同的功能，但具有更细粒度的控制和更广泛的适用性。

此外，Spring还提供了对JSR-250注解的支持，比如@PostConstruct和@PreDestroy，以及对JSR-330 (Java依赖注入)注解的支持。注入包，比如@Inject和@Named。有关这些注解的详细信息可以在相关章节中找到。注解注入在XML注入之前执行。因此，XML配置将覆盖通过这两种方法连接的属性的注解。

与往常一样，您可以将后置处理器注册为单独的bean定义，但也可以通过在基于xml的Spring配置中包含以下标记来隐式注册它们(注意上下文名称空间的包含):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context
		https://www.springframework.org/schema/context/spring-context.xsd">

	<context:annotation-config/>

</beans>
```

<context:annotation-config/>元素隐式注册了以下后处理器:

- ConfigurationClassPostProcessor
- AutowiredAnnotationBeanPostProcessor
- CommonAnnotationBeanPostProcessor
- PersistenceAnnotationBeanPostProcessor
- EventListenerMethodProcessor

<context:annotation-config/>只在定义它的相同应用程序上下文中查找bean上的注解。这意味着，如果你把<context:annotation-config/>放在DispatcherServlet的WebApplicationContext中，它只会检查controller层中的@Autowired bean，而不会检查service层。有关更多信息，请参阅DispatcherServlet。