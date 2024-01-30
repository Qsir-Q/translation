# 组合基于java的配置
Spring基于java的配置特性允许您编写注释，这可以降低配置的复杂性。



## 使用@Import注释

就像在Spring XML文件中使用<import/>元素来帮助模块化配置一样，@Import注释允许从另一个配置类加载@Bean定义，如下面的示例所示:
```java
@Configuration
public class ConfigA {

	@Bean
	public A a() {
		return new A();
	}
}

@Configuration
@Import(ConfigA.class)
public class ConfigB {

	@Bean
	public B b() {
		return new B();
	}
}
```
现在，在实例化上下文时不需要同时指定ConfigA.class和ConfigB.class，只需要显式地提供ConfigB，如下例所示:也就是不需要把ConfigA扫描到容器内也可以

```java
public static void main(String[] args) {
	ApplicationContext ctx = new AnnotationConfigApplicationContext(ConfigB.class);

	// now both beans A and B will be available...
	A a = ctx.getBean(A.class);
	B b = ctx.getBean(B.class);
}
```
这种方法简化了容器实例化，因为只需要处理一个类，而不需要在构造过程中记住可能大量的@Configuration类。

从Spring Framework 4.2开始，@Import也支持对常规组件类的引用，类似于AnnotationConfigApplicationContext。注册方法。
如果您想避免组件扫描，通过使用几个配置类作为入口点来显式定义所有组件，这是特别有用的。

## 在导入的@Bean定义中注入依赖
前面的例子可以工作，但是过于简单。在大多数实际场景中，bean跨配置类彼此依赖。在使用XML时，这不是问题，因为不涉及编译器，您可以声明ref="someBean"，并信任Spring在容器初始化期间解决这个问题。
当使用@Configuration类时，Java编译器会对配置模型施加约束，因为对其他bean的引用必须是有效的Java语法。

幸运的是，解决这个问题很简单。正如我们已经讨论过的，@Bean方法可以有任意数量的参数来描述bean依赖关系。考虑以下更真实的场景，其中有几个@Configuration类，每个类都依赖于其他类中声明的bean:
```java
@Configuration
public class ServiceConfig {

	@Bean
	public TransferService transferService(AccountRepository accountRepository) {
		return new TransferServiceImpl(accountRepository);
	}
}

@Configuration
public class RepositoryConfig {

	@Bean
	public AccountRepository accountRepository(DataSource dataSource) {
		return new JdbcAccountRepository(dataSource);
	}
}

@Configuration
@Import({ServiceConfig.class, RepositoryConfig.class})
public class SystemTestConfig {

	@Bean
	public DataSource dataSource() {
		// return new DataSource
	}
}

public static void main(String[] args) {
	ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
	// everything wires up across configuration classes...
	TransferService transferService = ctx.getBean(TransferService.class);
	transferService.transfer(100.00, "A123", "C456");
}
```
还有另一种方法可以达到同样的结果。请记住，__@Configuration类最终只是容器中的另一个bean__:这意味着它们可以利用@Autowired和@Value注入以及与任何其他bean相同的其他特性。

确保以这种方式注入的依赖项都是最简单的类型。__@Configuration类是在上下文初始化的早期处理的，强制以这种方式注入依赖项可能会导致意外的早期初始化__。只要有可能，就采用基于参数的注入，如前面的示例所示。

避免在同一配置类的@PostConstruct方法中访问本地定义的bean。这会导致循环引用，因为非静态@Bean方法在语义上需要调用完全初始化的配置类实例。由于不允许循环引用(例如在Spring Boot 2.6+中)，这可能会触发beancurrenlyincreationexception。

另外，要特别小心通过@Bean定义BeanPostProcessor和BeanFactoryPostProcessor。通常应该将这些方法声明为静态@Bean方法，而不是触发包含它们的配置类的实例化。否则，@Autowired和@Value可能无法在配置类本身上工作，因为可以在AutowiredAnnotationBeanPostProcessor之前将其创建为bean实例。

下面的例子展示了一个bean如何自动连接到另一个bean:
```java
@Configuration
public class ServiceConfig {

	@Autowired
	private AccountRepository accountRepository;

	@Bean
	public TransferService transferService() {
		return new TransferServiceImpl(accountRepository);
	}
}

@Configuration
public class RepositoryConfig {

	private final DataSource dataSource;

	public RepositoryConfig(DataSource dataSource) {
		this.dataSource = dataSource;
	}

	@Bean
	public AccountRepository accountRepository() {
		return new JdbcAccountRepository(dataSource);
	}
}

@Configuration
@Import({ServiceConfig.class, RepositoryConfig.class})
public class SystemTestConfig {

	@Bean
	public DataSource dataSource() {
		// return new DataSource
	}
}

public static void main(String[] args) {
	ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
	// everything wires up across configuration classes...
	TransferService transferService = ctx.getBean(TransferService.class);
	transferService.transfer(100.00, "A123", "C456");
}
```

@Configuration类中的构造函数注入仅在Spring Framework 4.3中得到支持。还要注意，如果目标bean只定义了一个构造函数，则不需要指定@Autowired。

在前面的场景中，使用@Autowired工作得很好，并提供了所需的模块化，但是确定autowired bean定义的确切声明位置仍然有些模糊。例如，作为一个查看ServiceConfig的开发人员，您如何确切地知道@Autowired AccountRepository bean是在哪里声明的?它在代码中不是显式的，这可能很好。
请记住，Spring Tools for Eclipse提供了一些工具，可以呈现显示所有内容如何连接的图形，这可能就是您所需要的。此外，您的Java IDE可以轻松地找到AccountRepository类型的所有声明和使用，并快速显示返回该类型的@Bean方法的位置。

如果这种模糊性是不可接受的，并且您希望在IDE中从一个@Configuration类直接导航到另一个@Configuration类，请考虑自动装配配置类本身。下面的例子展示了如何这样做:
```java
@Configuration
public class ServiceConfig {

	@Autowired
	private RepositoryConfig repositoryConfig;

	@Bean
	public TransferService transferService() {
		// navigate 'through' the config class to the @Bean method!
		return new TransferServiceImpl(repositoryConfig.accountRepository());
	}
}
```

在前面的情况下，定义AccountRepository的地方是完全显式的。然而，ServiceConfig现在与RepositoryConfig紧密耦合。这是一种权衡。这种紧密耦合可以通过使用基于接口或基于抽象类的@Configuration类得到一定程度的缓解。考虑下面的例子:
```java
@Configuration
public class ServiceConfig {

	@Autowired
	private RepositoryConfig repositoryConfig;

	@Bean
	public TransferService transferService() {
		return new TransferServiceImpl(repositoryConfig.accountRepository());
	}
}

@Configuration
public interface RepositoryConfig {

	@Bean
	AccountRepository accountRepository();
}

@Configuration
public class DefaultRepositoryConfig implements RepositoryConfig {

	@Bean
	public AccountRepository accountRepository() {
		return new JdbcAccountRepository(...);
	}
}

@Configuration
@Import({ServiceConfig.class, DefaultRepositoryConfig.class})  // import the concrete config!
public class SystemTestConfig {

	@Bean
	public DataSource dataSource() {
		// return DataSource
	}

}

public static void main(String[] args) {
	ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
	TransferService transferService = ctx.getBean(TransferService.class);
	transferService.transfer(100.00, "A123", "C456");
}
```
现在，ServiceConfig与具体的DefaultRepositoryConfig是松散耦合的，内置的IDE工具仍然很有用:您可以轻松地获得RepositoryConfig实现的类型层次结构。通过这种方式，导航@Configuration类及其依赖关系与导航基于接口的代码的通常过程没有什么不同。

如果您想影响某些bean的启动创建顺序，请考虑将其中一些声明为@Lazy(用于在第一次访问时创建，而不是在启动时创建)或将某些其他bean声明为@DependsOn(确保在当前bean之前创建特定的其他bean，而不是后者的直接依赖所暗示的)。



## 有条件地包含@Configuration类或@Bean方法
根据一些任意的系统状态，有条件地启用或禁用完整的@Configuration类，甚至单个的@Bean方法，通常是很有用的。一个常见的例子是，只有在Spring环境中启用了特定的概要文件时，才使用@Profile注释来激活Bean(有关详细信息，请参阅Bean Definition Profiles)。

@Profile注释实际上是通过使用更灵活的@Conditional注释实现的。@Conditional注释指示了在注册@Bean之前应该咨询的特定的org.springframework.context.annotation.Condition实现。

Condition接口的实现提供了一个返回真或假的matches(…)方法。例如，下面的清单显示了用于@Profile的实际条件实现:
```java
@Override
public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
	// Read the @Profile annotation attributes
	MultiValueMap<String, Object> attrs = metadata.getAllAnnotationAttributes(Profile.class.getName());
	if (attrs != null) {
		for (Object value : attrs.get("value")) {
			if (context.getEnvironment().acceptsProfiles(((String[]) value))) {
				return true;
			}
		}
		return false;
	}
	return true;
}
```

## 结合Java和XML配置
Spring的@Configuration类支持并不打算100%完全替代Spring XML。一些工具(如Spring XML名称空间)仍然是配置容器的理想方法。在XML方便或必要的情况下，您可以选择:以“以XML为中心”的方式实例化容器，例如使用ClassPathXmlApplicationContext，或者以“以java为中心”的方式实例化容器，例如使用AnnotationConfigApplicationContext和@ImportResource注释来根据需要导入XML。

## @Configuration类以xml为中心的使用
最好是从XML引导Spring容器，并以特别的方式包含@Configuration类。例如，在使用Spring XML的大型现有代码库中，更容易根据需要创建@Configuration类，并从现有XML文件中包含它们。
在本节后面的部分中，我们将介绍在这种“以xml为中心”的情况下使用@Configuration类的选项。

请记住，@Configuration类最终是容器中的bean定义。在本系列示例中，我们创建了一个名为AppConfig的@Configuration类，并将其作为<bean/>定义包含在system-test-config.xml中。
因为打开了<context:annotation-config/>，所以容器可以识别@Configuration注释并正确处理AppConfig中声明的@Bean方法。
下面的例子展示了Java中一个普通的配置类:
```java
@Configuration
public class AppConfig {

	@Autowired
	private DataSource dataSource;

	@Bean
	public AccountRepository accountRepository() {
		return new JdbcAccountRepository(dataSource);
	}

	@Bean
	public TransferService transferService() {
		return new TransferService(accountRepository());
	}
}
```
下面的例子展示了system-test-config.xml文件的一部分:
```xml
<beans>
	<!-- enable processing of annotations such as @Autowired and @Configuration -->
	<context:annotation-config/>
	<context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>

	<bean class="com.acme.AppConfig"/>

	<bean class="org.springframework.jdbc.datasource.DriverManagerDataSource">
		<property name="url" value="${jdbc.url}"/>
		<property name="username" value="${jdbc.username}"/>
		<property name="password" value="${jdbc.password}"/>
	</bean>
</beans>
```
下面的示例展示了一个可能的jdbc.properties文件:
```txt
jdbc.url=jdbc:hsqldb:hsql://localhost/xdb
jdbc.username=sa
jdbc.password=
```

```java
public static void main(String[] args) {
	ApplicationContext ctx = new ClassPathXmlApplicationContext("classpath:/com/acme/system-test-config.xml");
	TransferService transferService = ctx.getBean(TransferService.class);
	// ...
}
```

在system-test-config.xml文件中，AppConfig <bean/>没有声明id元素。
虽然这样做是可以接受的，但这是不必要的，因为没有其他bean引用它，并且不太可能通过名称显式地从容器中获取它。
类似地，DataSource bean只按类型自动连接，因此不严格要求显式bean id。

因为@Configuration是用@Component做元注释的，所以@Configuration注释的类是组件扫描的自动候选。
使用前面示例中描述的相同场景，我们可以重新定义system-test-config.xml，以利用组件扫描。注意，在这种情况下，我们不需要显式声明<context:annotation-config/>，因为<context:component-scan/>启用了相同的功能。
```xml
<beans>
	<!-- picks up and registers AppConfig as a bean definition -->
	<context:component-scan base-package="com.acme"/>
	<context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>

	<bean class="org.springframework.jdbc.datasource.DriverManagerDataSource">
		<property name="url" value="${jdbc.url}"/>
		<property name="username" value="${jdbc.username}"/>
		<property name="password" value="${jdbc.password}"/>
	</bean>
</beans>
```

## 以类为中心使用XML和@ImportResource
在使用@Configuration类作为配置容器的主要机制的应用程序中，可能仍然需要至少使用一些XML。
在这些场景中，您可以使用@ImportResource并只定义所需的XML。
这样做实现了一种“以java为中心”的方法来配置容器，并将XML保持在最低限度。
下面的示例(其中包括一个配置类、一个定义bean的XML文件、一个属性文件和主类)展示了如何使用@ImportResource注释来实现“以java为中心”的配置，
该配置根据需要使用XML:

```java
@Configuration
@ImportResource("classpath:/com/acme/properties-config.xml")
public class AppConfig {

	@Value("${jdbc.url}")
	private String url;

	@Value("${jdbc.username}")
	private String username;

	@Value("${jdbc.password}")
	private String password;

	@Bean
	public DataSource dataSource() {
		return new DriverManagerDataSource(url, username, password);
	}
}
```

```xml
properties-config.xml
<beans>
	<context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>
</beans>
```

```txt
jdbc.properties
jdbc.url=jdbc:hsqldb:hsql://localhost/xdb
jdbc.username=sa
jdbc.password=
```

```java
public static void main(String[] args) {
	ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
	TransferService transferService = ctx.getBean(TransferService.class);
	// ...
}
```