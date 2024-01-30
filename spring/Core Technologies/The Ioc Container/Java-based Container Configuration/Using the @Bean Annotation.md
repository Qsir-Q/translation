# 使用@Bean注释
@Bean是一个方法级注释，是XML <bean/>元素的直接类比。注释支持<bean/>提供的一些属性，例如:init-method,destroy-method,autowiring您可以在@ configuration注释的类或@ component注释的类中使用@Bean注释。



# 声明一个Bean
要声明一个bean，可以用@Bean注释一个方法。您可以使用此方法在指定为方法返回值的类型的ApplicationContext中注册bean定义。__默认情况下，bean名称与方法名称相同__。下面的例子展示了一个@Bean方法声明:
```java
@Configuration
public class AppConfig {

	@Bean
	public TransferServiceImpl transferService() {
		return new TransferServiceImpl();
	}
}
```
上面的配置完全等同于下面的Spring XML:
```xml
<beans>
	<bean id="transferService" class="com.acme.TransferServiceImpl"/>
</beans>
```
这两个声明都使一个名为transferService的bean在ApplicationContext中可用，绑定到TransferServiceImpl类型的对象实例，如下图所示:
```txt
transferService -> com.acme.TransferServiceImpl
```
您还可以使用默认方法来定义bean。这允许通过在默认方法上实现带有bean定义的接口来组合bean配置。
```java
public interface BaseConfig {

	@Bean
	default TransferServiceImpl transferService() {
		return new TransferServiceImpl();
	}
}

@Configuration
public class AppConfig implements BaseConfig {

}
```
你也可以用接口(或基类)返回类型声明@Bean方法，如下例所示:
```java
@Configuration
public class AppConfig {

	@Bean
	public TransferService transferService() {
		return new TransferServiceImpl();
	}
}
```
但是，这限制了对指定接口类型(TransferService)的预先类型预测的可见性。然后，只有在受影响的单例bean实例化之后，才使用容器已知的完整类型(TransferServiceImpl)。非惰性单例bean根据其声明顺序进行实例化，因此您可能会看到不同的类型匹配结果，这取决于另一个组件何时尝试使用未声明的类型进行匹配(例如@Autowired TransferServiceImpl，它仅在transferService bean实例化后才解析)。

如果您始终通过声明的服务接口引用您的类型，那么您的@Bean返回类型可以安全地加入该设计决策。
然而，对于实现多个接口的组件或可能由其实现类型引用的组件，声明尽可能具体的返回类型更安全(至少与引用bean的注入点所要求的一样具体)。



# Bean的依赖关系

带@bean注解的方法可以具有任意数量的参数，这些参数描述构建该bean所需的依赖关系。例如，如果我们的TransferService需要一个accountrerepository，我们可以用一个方法参数实现这个依赖，如下面的例子所示:
```java
@Configuration
public class AppConfig {

	@Bean
	public TransferService transferService(AccountRepository accountRepository) {
		return new TransferServiceImpl(accountRepository);
	}
}
```
解析机制与基于构造函数的依赖注入非常相似。有关详细信息，请参阅相关部分。



# 生命周期回调

任何用@Bean注释定义的类都支持常规的生命周期回调，并且可以使用JSR-250中的@PostConstruct和@PreDestroy注释。有关更多细节，请参阅JSR-250注释。

它还完全支持常规的Spring生命周期回调。如果bean实现了InitializingBean、DisposableBean或Lifecycle，则容器将调用它们各自的方法。

Aware接口的标准集(如BeanFactoryAware、BeanNameAware、MessageSourceAware、ApplicationContextAware等)也得到了完全支持。

@Bean注解支持指定任意的初始化和销毁回调方法，很像Spring XML在bean元素上的init-method和destroy-method属性，如下例所示:
```java
public class BeanOne {

	public void init() {
		// initialization logic
	}
}

public class BeanTwo {

	public void cleanup() {
		// destruction logic
	}
}

@Configuration
public class AppConfig {

	@Bean(initMethod = "init")
	public BeanOne beanOne() {
		return new BeanOne();
	}

	@Bean(destroyMethod = "cleanup")
	public BeanTwo beanTwo() {
		return new BeanTwo();
	}
}
```
默认情况下，使用具有公共close或shutdown方法的Java配置定义的bean将自动使用销毁回调调用。如果您有一个公共的close或shutdown方法，并且不希望它在容器关闭时被调用，那么您可以在bean定义中添加@Bean(destroyMethod = "")来禁用默认(推断)模式。

对于使用JNDI获取的资源，默认情况下您可能希望这样做，因为它的生命周期是在应用程序外部管理的。
特别是，要确保始终对数据源执行此操作，因为众所周知，在Jakarta EE应用程序服务器上这是有问题的。

下面的例子展示了如何防止一个数据源的自动销毁回调:
```java
@Bean(destroyMethod = "")
public DataSource dataSource() throws NamingException {
	return (DataSource) jndiTemplate.lookup("MyDS");
}
```
此外，对于@Bean方法，您通常使用编程JNDI查找，要么使用Spring的JndiTemplate或JndiLocatorDelegate助手，要么直接使用JNDI InitialContext，而不是使用JndiObjectFactoryBean变体(这将迫使您将返回类型声明为FactoryBean类型，而不是实际的目标类型，这使得在其他打算引用此处提供的资源的@Bean方法中使用交叉引用调用变得更加困难)。

在前面提到的例子中的BeanOne中，在构造过程中直接调用init()方法同样有效，如下面的例子所示:
```java
@Configuration
public class AppConfig {

	@Bean
	public BeanOne beanOne() {
		BeanOne beanOne = new BeanOne();
		beanOne.init();
		return beanOne;
	}

	// ...
}
```



# 指定Bean作用域

Spring包含@Scope注释，这样您就可以指定bean的作用域。

## 使用@Scope注释
您可以指定使用@Bean注释定义的bean应该具有特定的作用域。您可以使用Bean作用域部分中指定的任何标准作用域。默认的作用域是单例的，但是你可以用@Scope注释覆盖它，如下例所示:
```java
@Configuration
public class MyConfiguration {

	@Bean
	@Scope("prototype")
	public Encryptor encryptor() {
		// ...
	}
}
```

##### @Scope和scope -proxy
Spring提供了一种通过作用域代理处理作用域依赖的方便方法。在使用XML配置时，创建这样一个代理的最简单方法是<aop:scope -proxy/>元素。在Java中使用@Scope注释配置bean提供了与proxyMode属性相当的支持。默认值是ScopedProxyMode.DEFAULT，这通常表示不应该创建有作用域的代理，除非在组件扫描指令级别配置了不同的默认值。您可以指定ScopedProxyMode.TARGET_CLASS ScopedProxyMode或ScopedProxyMode.NO

如果使用Java将有作用域的代理示例从XML参考文档(参见有作用域的代理)移植到我们的@Bean，它类似于以下内容:
```java
// an HTTP Session-scoped bean exposed as a proxy
@Bean
@SessionScope
public UserPreferences userPreferences() {
	return new UserPreferences();
}

@Bean
public Service userService() {
	UserService service = new SimpleUserService();
	// a reference to the proxied userPreferences bean
	service.setUserPreferences(userPreferences());
	return service;
}
```

## 自定义Bean名
默认情况下，配置类使用@Bean方法的名称作为结果bean的名称。但是，可以使用name属性重写此功能，如下例所示:
```java
@Configuration
public class AppConfig {

	@Bean("myThing")
	public Thing thing() {
		return new Thing();
	}
}
```

## Bean别名
正如命名bean中所讨论的，有时需要为单个bean提供多个名称，否则称为bean别名。@Bean注释的name属性为此目的接受一个String数组。下面的例子展示了如何为一个bean设置多个别名:

```java
@Configuration
public class AppConfig {

	@Bean({"dataSource", "subsystemA-dataSource", "subsystemB-dataSource"})
	public DataSource dataSource() {
		// instantiate, configure and return DataSource bean...
	}
}
```

## Bean描述
有时，提供bean的更详细的文本描述是有帮助的。当为了监视目的而公开bean(可能通过JMX)时，这一点特别有用。要向@Bean添加描述，可以使用@Description注释，如下例所示:

```java
@Configuration
public class AppConfig {

	@Bean
	@Description("Provides a basic example of a bean")
	public Thing thing() {
		return new Thing();
	}
}
```