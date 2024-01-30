# 使用@Configuration 配置
@Configuration是类级别的注释，作用是指示类为bean定义的源。@Configuration类通过@ bean注释的方法声明bean。对@Configuration类上的@Bean方法的调用也可用于定义bean间依赖关系。请参阅基本概念:@Bean和@Configuration以获得一般介绍。



# 注入bean间依赖

当bean彼此依赖时，表达依赖关系就像让一个bean方法调用另一个bean方法一样简单，如下例所示:
```java
@Configuration
public class AppConfig {

	@Bean
	public BeanOne beanOne() {
		return new BeanOne(beanTwo());
	}

	@Bean
	public BeanTwo beanTwo() {
		return new BeanTwo();
	}
}
```
在前面的示例中，beanOne通过构造函数注入接收到beanTwo的引用。这种声明bean间依赖的方法只有在@Configuration类中声明@Bean方法时才有效。你不能通过使用普通的@Component类来声明bean间依赖。



# 查找方法注入

如前所述，查找方法注入是一种高级特性，您应该很少使用。在单例作用域bean依赖于原型作用域bean的情况下，它很有用。使用Java进行这种类型的配置为实现这种模式提供了一种自然的方法。
下面的例子展示了如何使用查找方法注入:

```java
public abstract class CommandManager {
	public Object process(Object commandState) {
		// 获取适当Command接口的新实例
		Command command = createCommand();
		// 在(希望是全新的)Command实例上设置状态
		command.setState(commandState);
		return command.execute();
	}

	// okay... 但是这种方法的实现在哪里呢?
	protected abstract Command createCommand();
}
```
通过使用Java配置，您可以创建CommandManager的子类，其中抽象的createCommand()方法以查找新(原型)命令对象的方式被重写。下面的例子展示了如何这样做:
```java
@Bean
@Scope("prototype")
public AsyncCommand asyncCommand() {
	AsyncCommand command = new AsyncCommand();
	// inject dependencies here as required
	return command;
}

@Bean
public CommandManager commandManager() {
	// 使用createCommand()返回新的匿名CommandManager实现
	// 重写以返回一个新的原型Command对象
	return new CommandManager() {
		protected Command createCommand() {
			return asyncCommand();
		}
	}
}
```



# 关于基于java的配置如何在内部工作的进一步信息

考虑下面的例子，它显示了一个带@Bean注释的方法被调用两次:
```java
@Configuration
public class AppConfig {

	@Bean
	public ClientService clientService1() {
		ClientServiceImpl clientService = new ClientServiceImpl();
		clientService.setClientDao(clientDao());
		return clientService;
	}

	@Bean
	public ClientService clientService2() {
		ClientServiceImpl clientService = new ClientServiceImpl();
		clientService.setClientDao(clientDao());
		return clientService;
	}

	@Bean
	public ClientDao clientDao() {
		return new ClientDaoImpl();
	}
}
```
clientDao()在clientService1()和clientService2()中分别被调用一次。由于该方法创建了一个ClientDaoImpl的新实例并返回它，因此您通常期望有两个实例(每个服务一个)。这肯定会有问题:在Spring中，实例化的bean默认具有单例作用域。这就是神奇之处:所有的@Configuration类都在启动时用CGLIB子类化。在子类中，子方法在调用父方法并创建新实例之前，首先检查容器中是否存在任何缓存的(限定作用域的)bean。

根据bean的作用域，行为可能会有所不同。我们在这里讨论的是singleton。

没有必要将CGLIB添加到类路径中，因为CGLIB类被重新打包在org.springframework.cglib包下，并直接包含在spring-core JAR中。

由于CGLIB在启动时动态添加特性，因此存在一些限制。特别是，__配置类不能是最终的__。然而，在配置类上允许使用任何构造函数，包括使用@Autowired或默认注入的单个非默认构造函数声明。

如果您希望避免任何cglib强加的限制，请考虑在non-@Configuration类上声明@Bean方法(例如，在普通的@Component类上)，或者用@Configuration(proxyBeanMethods = false)注释配置类。这样，@Bean方法之间的跨方法调用就不会被拦截，因此您必须完全依赖于构造函数或方法级别的依赖注入。