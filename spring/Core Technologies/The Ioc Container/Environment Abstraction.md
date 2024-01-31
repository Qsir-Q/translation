# 环境抽象
Environment接口是集成在容器中的抽象，它对应用程序环境的两个关键方面建模: profiles 和 properties

profile文件是一个命名的、逻辑的bean定义组，只有在给定的profile文件处于活动状态时才向容器注册。可以将bean分配给配置文件，无论是用XML定义的还是用注释定义的。与profile文件相关的Environment对象的作用是确定哪些profile文件(如果有的话)当前是active的，以及哪些profile文件(如果有的话)在默认情况下应该是active的。

Properties在几乎所有应用程序中都扮演着重要的角色，并且可能来自各种来源:属性文件、JVM系统属性、系统环境变量、JNDI、servlet上下文参数、特别属性对象、Map对象等等。与Properties相关的Environment对象的作用是为用户提供一个方便的服务接口，用于配置属性源并从中解析属性。



# Bean定义配置文件

Bean定义profile文件在核心容器中提供了一种机制，允许在不同的环境中注册不同的Bean。“环境”这个词对不同的用户有不同的含义，这个特性可以帮助处理许多用例，包括:

- 在开发中使用内存中的数据源，而在QA或生产中从JNDI中查找相同的数据源。
- 只有在将应用程序部署到performance环境中时才注册监视基础设施。
- 为客户A和客户B的部署注册自定义的bean实现。考虑需要数据源的实际应用程序中的第一个用例。在测试环境中，配置可能类似于以下内容:
```java
@Bean
public DataSource dataSource() {
	return new EmbeddedDatabaseBuilder()
		.setType(EmbeddedDatabaseType.HSQL)
		.addScript("my-schema.sql")
		.addScript("my-test-data.sql")
		.build();
}
```
现在考虑如何将此应用程序部署到QA或生产环境中，假设应用程序的数据源已注册到生产应用程序服务器的JNDI目录。我们的dataSource bean现在看起来像下面的清单:
```java
@Bean(destroyMethod = "")
public DataSource dataSource() throws Exception {
	Context ctx = new InitialContext();
	return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
}
```
问题是如何根据当前环境在使用这两种变体之间进行切换。随着时间的推移，Spring用户已经设计了许多方法来完成这项工作，通常依赖于系统环境变量和包含${placeholder}令牌的XML <import/>语句的组合，这些令牌根据环境变量的值解析到正确的配置文件路径。Bean定义概要文件是为这个问题提供解决方案的核心容器特性。

如果我们概括前面特定于环境的bean定义示例中显示的用例，我们最终需要在某些上下文中注册某些bean定义，而不是在其他上下文中注册。您可以说，您希望在情况a中注册bean定义的某个profile文件，而在情况b中注册不同的profile文件。我们首先更新配置以反映这种需求。



# 使用@Profile
@Profile注解允许您指出，当一个或多个指定的概要文件处于活动状态时，组件有资格注册。使用我们前面的例子，我们可以重写dataSource配置如下:
```java
@Configuration
@Profile("development")
public class StandaloneDataConfig {

	@Bean
	public DataSource dataSource() {
		return new EmbeddedDatabaseBuilder()
			.setType(EmbeddedDatabaseType.HSQL)
			.addScript("classpath:com/bank/config/sql/schema.sql")
			.addScript("classpath:com/bank/config/sql/test-data.sql")
			.build();
	}
}
```

```java
@Configuration
@Profile("production")
public class JndiDataConfig {

	@Bean(destroyMethod = "")
	public DataSource dataSource() throws Exception {
		Context ctx = new InitialContext();
		return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
	}
}
```

如前所述，对于@Bean方法，您通常选择使用编程的JNDI查找，方法是使用Spring的JndiTemplate/JndiLocatorDelegate帮助程序，或者使用前面所示的直接JNDI InitialContext使用方法，但不使用JndiObjectFactoryBean变体，这将迫使您将返回类型声明为FactoryBean类型。

profile字符串可以包含一个简单的profile名称(例如，production)或一个profile表达式。profile表达式允许表达更复杂的profile逻辑(例如，production & us-east)。profile表达式中支持以下操作符:
- ! 非
- & 与
- ｜或

如果不使用括号，就不能混合使用&和|操作符。例如，production & us-east | eu-central不是一个有效的表达式。它必须表示为production & (us-east | eu-central)。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Profile("production")
public @interface Production {
}
```

如果@Configuration类被标记为@Profile，那么所有与该类关联的@Bean方法和@Import注释都将被绕过，除非一个或多个指定的profile处于活动状态。如果@Component或@Configuration类被标记为@Profile({"p1"， "p2"})，则该类不会被注册或处理，除非配置文件'p1'或'p2'已被激活。如果给定的profile以NOT操作符(!)为前缀，则只有当profile不活动时，才会注册带注释的元素。例如，给定@Profile({"p1"， "!p2"})，如果配置文件'p1'处于活动状态或配置文件'p2'未处于活动状态，则将发生注册。

@Profile也可以在方法级别声明，只包含配置类的一个特定bean(例如，对于特定bean的可选变体)，如下例所示:
```java
@Configuration
public class AppConfig {

	@Bean("dataSource")
	@Profile("development")
	public DataSource standaloneDataSource() {
		return new EmbeddedDatabaseBuilder()
			.setType(EmbeddedDatabaseType.HSQL)
			.addScript("classpath:com/bank/config/sql/schema.sql")
			.addScript("classpath:com/bank/config/sql/test-data.sql")
			.build();
	}

	@Bean("dataSource")
	@Profile("production")
	public DataSource jndiDataSource() throws Exception {
		Context ctx = new InitialContext();
		return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
	}
}
```
对于以上例子：standaloneDataSource方法仅在开发概要文件中可用。jndisdatasource方法仅在生产配置文件中可用。

对于@Bean方法上的@Profile，可能会出现一种特殊的情况:对于具有相同Java方法名的重载@Bean方法(类似于构造函数重载)，需要在所有重载方法上一致地声明@Profile条件。如果条件不一致，则只有重载方法中第一个声明的条件有关系。因此，@Profile不能用于选择具有特定参数签名的重载方法。同一bean的所有工厂方法之间的解析遵循Spring在创建时的构造函数解析算法。

如果希望定义具有不同概要文件条件的备选bean，请使用不同的Java方法名，这些方法名通过使用@Bean name属性指向相同的bean名称，如前面的示例所示。如果参数签名都是相同的(例如，所有的变体都有无参数工厂方法)，这是首先在有效的Java类中表示这种安排的唯一方法(因为只能有一个特定名称和参数签名的方法)。



# XML Bean定义配置文件

对应的XML是<beans>元素的profile文件属性。我们前面的示例配置可以重写为两个XML文件，如下所示:
```xml
<beans profile="development"
	xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:jdbc="http://www.springframework.org/schema/jdbc"
	xsi:schemaLocation="...">

	<jdbc:embedded-database id="dataSource">
		<jdbc:script location="classpath:com/bank/config/sql/schema.sql"/>
		<jdbc:script location="classpath:com/bank/config/sql/test-data.sql"/>
	</jdbc:embedded-database>
</beans>
```

```xml
<beans profile="production"
	xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:jee="http://www.springframework.org/schema/jee"
	xsi:schemaLocation="...">

	<jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
</beans>
```

也可以避免在同一个文件中拆分和嵌套<beans/>元素，如下例所示:
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:jdbc="http://www.springframework.org/schema/jdbc"
	xmlns:jee="http://www.springframework.org/schema/jee"
	xsi:schemaLocation="...">

	<!-- other bean definitions -->

	<beans profile="development">
		<jdbc:embedded-database id="dataSource">
			<jdbc:script location="classpath:com/bank/config/sql/schema.sql"/>
			<jdbc:script location="classpath:com/bank/config/sql/test-data.sql"/>
		</jdbc:embedded-database>
	</beans>

	<beans profile="production">
		<jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
	</beans>
</beans>
```
spring-bean.xsd被限制文件中只有最后一个元素(只有一个顶层beans元素)。这将有助于提供灵活性，而不会在XML文件中造成混乱。XML对应物不支持前面描述的概要表达式。但是，可以使用!来否定概要文件。
操作符。也可以通过嵌套配置文件来应用逻辑上的“and”，如下例所示:

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:jdbc="http://www.springframework.org/schema/jdbc"
	xmlns:jee="http://www.springframework.org/schema/jee"
	xsi:schemaLocation="...">

	<!-- other bean definitions -->

	<beans profile="production">
		<beans profile="us-east">
			<jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
		</beans>
	</beans>
</beans>
```
在前面的示例中，如果生产和us-east配置文件都处于活动状态，则公开dataSource bean。



# active配置文件

现在我们已经更新了配置，我们仍然需要指示Spring哪个概要文件是活动的。如果我们现在启动我们的示例应用程序，我们将看到抛出NoSuchBeanDefinitionException，因为容器找不到名为dataSource的Spring bean。

激活概要文件可以通过几种方式完成，但最直接的是通过编程方式针对通过ApplicationContext提供的环境API来完成。下面的例子展示了如何这样做:
```java
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
ctx.getEnvironment().setActiveProfiles("development");
ctx.register(SomeConfig.class, StandaloneDataConfig.class, JndiDataConfig.class);
ctx.refresh();
```
此外，您还可以通过spring.profiles.active属性声明地激活概要文件，该属性可以通过系统环境变量、JVM系统属性、web.xml中的servlet上下文参数来指定，甚至可以作为JNDI中的一个条目来指定(参见PropertySource Abstraction)。在集成测试中，活动概要文件可以通过在spring-test模块中使用@ActiveProfiles注释来声明(参见环境概要文件的上下文配置)。

注意，profile文件不是一个“非此即彼”的命题。__您可以一次激活多个配置文件__。通过编程方式，您可以向setActiveProfiles()方法提供多个概要名称，该方法接受String…下面的示例激活多个配置文件:
```java
ctx.getEnvironment().setActiveProfiles("profile1", "profile2");
```
声明性地，spring.profiles.active可以接受以逗号分隔的配置文件名列表，如下例所示:
```java
-Dspring.profiles.active="profile1,profile2"
```



# Default Profile

默认profile表示在没有激活profile时启用的profile。考虑下面的例子:
```java
@Configuration
@Profile("default")
public class DefaultDataConfig {

	@Bean
	public DataSource dataSource() {
		return new EmbeddedDatabaseBuilder()
			.setType(EmbeddedDatabaseType.HSQL)
			.addScript("classpath:com/bank/config/sql/schema.sql")
			.build();
	}
}
```
如果没有active的profile，则创建数据源。您可以将其视为为一个或多个bean提供默认定义的一种方式。如果启用了任何配置文件，则不应用默认配置文件。

缺省配置文件的名称为default。您可以通过在环境中使用setDefaultProfiles()或使用spring.profiles.default属性来更改默认配置文件的名称。



# PropertySource抽象

Spring的环境抽象在属性源的可配置层次结构上提供搜索操作。考虑下面的例子:
```java
ApplicationContext ctx = new GenericApplicationContext();
Environment env = ctx.getEnvironment();
boolean containsMyProperty = env.containsProperty("my-property");
System.out.println("Does my environment contain the 'my-property' property? " + containsMyProperty);
```
在前面的代码片段中，我们看到了查询Spring是否为当前环境定义了my-property属性的高级方法。为了回答这个问题，Environment对象对一组PropertySource对象执行搜索。
__PropertySource是对任何键值对源的简单抽象，Spring的StandardEnvironment配置了两个PropertySource对象——一个表示JVM系统属性集(system . getproperties())，另一个表示系统环境变量集(system .getenv())__。

这些默认属性源是为StandardEnvironment提供的，用于独立应用程序。StandardServletEnvironment使用额外的默认属性源填充，包括servlet配置、servlet上下文参数和JndiPropertySource(如果JNDI可用)。

具体地说，当您使用StandardEnvironment时，如果运行时存在my-property系统属性或my-property环境变量，则调用env.containsProperty("my-property")返回true。

执行的搜索是分层的。__默认情况下，系统属性优先于环境变量__。因此，如果在调用env.getproperty(“my-property”)期间碰巧在两个地方都设置了my-property属性，则系统属性值“胜出”并被返回。注意，属性值不是合并的，而是被前面的条目完全覆盖。

对于一个通用的standardservletenenvironment，完整的层次结构如下所示，最高优先级的条目位于顶部:
- ServletConfig参数(如果适用——例如，DispatcherServlet上下文)
- ServletContext参数(web.xml上下文参数条目)
- JNDI环境变量(java:comp/env/ entries)
- JVM系统属性(-D命令行参数)
- JVM系统环境(操作系统环境变量)

最重要的是，整个机制是可配置的。也许您希望将自定义的属性源集成到此搜索中。为此，实现并实例化您自己的PropertySource，并将其添加到当前环境的PropertySource集合中。下面的例子展示了如何这样做:

```java
ConfigurableApplicationContext ctx = new GenericApplicationContext();
MutablePropertySources sources = ctx.getEnvironment().getPropertySources();
sources.addFirst(new MyPropertySource());
```
在前面的代码中，MyPropertySource在搜索中以最高优先级被添加。如果它包含my-property属性，则检测并返回该属性，以支持任何其他PropertySource中的任何my-property属性。MutablePropertySources API公开了许多方法，这些方法允许对属性源集进行精确操作。



# 使用@PropertySource
@PropertySource注解为向Spring环境添加PropertySource提供了一种方便的声明性机制。

给定一个名为app.properties的文件，其中包含键值对testbean.name=myTestBean，下面的@Configuration类使用@PropertySource的方式是，调用testBean.getName()返回myTestBean:
```java
@Configuration
@PropertySource("classpath:/com/myco/app.properties")
public class AppConfig {

 @Autowired
 Environment env;

 @Bean
 public TestBean testBean() {
  TestBean testBean = new TestBean();
  testBean.setName(env.getProperty("testbean.name"));
  return testBean;
 }
}
```
@PropertySource资源位置中出现的任何${…}占位符都会根据已经在环境中注册的属性源集进行解析，如下面的示例所示:

```java
@Configuration
@PropertySource("classpath:/com/${my.placeholder:default/path}/app.properties")
public class AppConfig {

 @Autowired
 Environment env;

 @Bean
 public TestBean testBean() {
  TestBean testBean = new TestBean();
  testBean.setName(env.getProperty("testbean.name"));
  return testBean;
 }
}
```
假设my.placeholder存在于已注册的属性源之一中(例如，系统属性或环境变量)，占位符被解析为相应的值。
如果没有，则使用default/path作为默认值。如果没有指定默认值并且无法解析属性，则抛出IllegalArgumentException。



# 语句中的占位符解析

过去，元素中占位符的值只能根据JVM系统属性或环境变量来解析。现在情况已经不同了。因为环境抽象集成在整个容器中，所以很容易通过它路由占位符的解析。这意味着您可以以任何喜欢的方式配置解析过程。您可以更改搜索系统属性和环境变量的优先级，或者完全删除它们。您还可以酌情将自己的属性源添加到混合中。
```xml
<beans>
	<import resource="com/bank/service/${customer}-config.xml"/>
</beans>
```



