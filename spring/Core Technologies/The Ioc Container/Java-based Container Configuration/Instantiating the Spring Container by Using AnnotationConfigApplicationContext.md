# 使用AnnotationConfigApplicationContext实例化Spring容器
下面几节介绍Spring的AnnotationConfigApplicationContext，它是在Spring 3.0中引入的。这个通用的ApplicationContext实现不仅能够接受@Configuration类作为输入，还能够接受普通的@Component类和带有JSR-330元数据注释的类。

当@Configuration类作为输入提供时，@Configuration类本身被注册为bean定义，类中声明的所有@Bean方法也被注册为bean定义。当提供@Component和JSR-330类时，它们被注册为bean定义，并且假设在这些类中需要使用像@Autowired或@Inject这样的DI元数据。

## 简单构造
在实例化一个ClassPathXmlApplicationContext时，使用Spring XML文件作为输入的方式大致相同，在实例化一个AnnotationConfigApplicationContext时，您可以使用@Configuration类作为输入。这允许完全不受xml约束地使用Spring容器，如下面的示例所示:
```java
public static void main(String[] args) {
	ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
	MyService myService = ctx.getBean(MyService.class);
	myService.doStuff();
}
```

如前所述，AnnotationConfigApplicationContext并不局限于只使用@Configuration类。任何@Component或JSR-330注释类都可以作为构造函数的输入，如下例所示:
```java
public static void main(String[] args) {
	ApplicationContext ctx = new AnnotationConfigApplicationContext(MyServiceImpl.class, Dependency1.class, Dependency2.class);
	MyService myService = ctx.getBean(MyService.class);
	myService.doStuff();
}
```
前面的例子假设MyServiceImpl、Dependency1和Dependency2使用Spring依赖注入注释,就像@Autowired一样



## 以编程方式构建容器(类<?>…)

你可以使用一个无参数的构造函数实例化一个AnnotationConfigApplicationContext，然后使用register()方法配置它。这种方法在以编程方式构建AnnotationConfigApplicationContext时特别有用。下面的例子展示了如何这样做:

```java
public static void main(String[] args) {
	AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
	ctx.register(AppConfig.class, OtherConfig.class);
	ctx.register(AdditionalConfig.class);
	ctx.refresh();
	MyService myService = ctx.getBean(MyService.class);
	myService.doStuff();
}
```



## 使用scan(String…)启用组件扫描

要启用组件扫描，你可以这样注释@Configuration类:
```java
@Configuration
@ComponentScan(basePackages = "com.acme")
public class AppConfig  {
	// ...
}
```
有经验的Spring用户可能对Spring的context: namespace中的XML声明很熟悉，如下例所示:
```xml

<beans>
	<context:component-scan base-package="com.acme"/>
</beans>
```
在上面的示例中，com.acme 被扫描，以查找任何带有@ component注释的类，并将这些类注册为容器中的Spring bean定义。AnnotationConfigApplicationContext暴露了scan(String…)方法来允许相同的组件扫描功能，如下面的例子所示:
```java
public static void main(String[] args) {
	AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
	ctx.scan("com.acme");
	ctx.refresh();
	MyService myService = ctx.getBean(MyService.class);
}
```

请记住，@Configuration类是用@Component进行元注释的，因此它们是组件扫描的候选者。在前面的例子中，假设AppConfig是在com. js中声明的。Acme包(或其下的任何包)，则在调用scan()期间拾取它。在refresh()时，它的所有@Bean方法都被处理并注册为容器中的bean定义。



# 支持带有AnnotationConfigWebApplicationContext的Web应用程序
注解configapplicationcontext的一个WebApplicationContext变体可以通过注解configwebapplicationcontext获得。您可以在配置Spring ContextLoaderListener servlet监听器、Spring MVC DispatcherServlet等时使用此实现。下面的web.xml代码段配置了一个典型的Spring MVC web应用程序(注意使用了contextClass context-param和init-param):
```xml
<web-app>
	<!-- 配置ContextLoaderListener使用AnnotationConfigWebApplicationContext 而不是默认的XmlWebApplicationContext -->
	<context-param>
		<param-name>contextClass</param-name>
		<param-value>
			org.springframework.web.context.support.AnnotationConfigWebApplicationContext
		</param-value>
	</context-param>

	<!-- 配置位置必须由一个或多个逗号或空格分隔 全限定@Configuration类。完全合格的包也可能是 指定用于组件扫描 -->
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>com.acme.AppConfig</param-value>
	</context-param>

	<!-- Bootstrap the root application context as usual using ContextLoaderListener -->
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>

	<!-- Declare a Spring MVC DispatcherServlet as usual -->
	<servlet>
		<servlet-name>dispatcher</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<!-- Configure DispatcherServlet to use AnnotationConfigWebApplicationContext
			instead of the default XmlWebApplicationContext -->
		<init-param>
			<param-name>contextClass</param-name>
			<param-value>
				org.springframework.web.context.support.AnnotationConfigWebApplicationContext
			</param-value>
		</init-param>
		<!-- Again, config locations must consist of one or more comma- or space-delimited
			and fully-qualified @Configuration classes -->
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>com.acme.web.MvcConfig</param-value>
		</init-param>
	</servlet>

	<!-- map all requests for /app/* to the dispatcher servlet -->
	<servlet-mapping>
		<servlet-name>dispatcher</servlet-name>
		<url-pattern>/app/*</url-pattern>
	</servlet-mapping>
</web-app>
```