# 基本概念:@Bean和@Configuration
Spring的Java配置支持中的核心组件是带@configuration注释的类和带@ bean注释的方法。

@Bean注释用于指示方法实例化、配置和初始化要由Spring IoC容器管理的新对象。对于那些熟悉Spring的<beans/> XML配置的人来说，@Bean注释扮演着与<bean/>元素相同的角色。你可以在任何Spring @Component中使用带有@ bean注释的方法。但是，它们最常与@Configuration bean一起使用。

用@Configuration注释类表明它的主要目的是作为bean定义的来源。此外，@Configuration类允许通过调用同一类中的其他@Bean方法来定义bean间依赖关系。最简单的@Configuration类如下所示:
```java
@Configuration
public class AppConfig {

	@Bean
	public MyServiceImpl myService() {
		return new MyServiceImpl();
	}
}
```

前面的AppConfig类等价于下面的Spring <beans/> XML:
```xml
<beans>
	<bean id="myService" class="com.acme.services.MyServiceImpl"/>
</beans>
```



# 完整@Configuration vs“精简”@Bean模式?

当@Bean方法在没有使用@Configuration注释的类中声明时，它们被称为以“精简”模式处理。在没有使用@Configuration注释的Bean上声明的Bean方法被认为是“精简的”，包含类的主要目的不同，@Bean方法在那里是一种额外的好处。例如，服务组件可以通过每个适用组件类上的附加@Bean方法向容器公开管理视图。在这种情况下，@Bean方法是一种通用的工厂方法机制。

与完整的@Configuration不同，lite @Bean方法不能声明bean间依赖关系。相反，它们操作的是包含它们的组件的内部状态，以及它们可能声明的参数(可选)。因此，这样的@Bean方法不应该调用其他@Bean方法。每个这样的方法实际上只是特定bean引用的工厂方法，没有任何特殊的运行时语义。这里的积极副作用是，在运行时不必应用CGLIB子类，因此在类设计方面没有限制(也就是说，包含的类可能是final等)。

在常见的场景中，@Bean方法将在@Configuration类中声明，以确保始终使用“full”模式，并且跨方法引用因此被重定向到容器的生命周期管理。这可以防止通过常规Java调用意外地调用相同的@Bean方法，这有助于减少在“生命”模式下操作时难以跟踪的细微错误。

下面几节将深入讨论@Bean和@Configuration注释。不过，首先我们将介绍使用基于java的配置创建spring容器的各种方法。