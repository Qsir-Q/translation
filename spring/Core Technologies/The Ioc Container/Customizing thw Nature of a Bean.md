# 自定义Bean行为
Spring框架提供了许多接口，您可以使用这些接口来定制bean的行为。本节将它们分组如下:
- Lifecycle Callbacks
- ApplicationContextAware and BeanNameAware
- Other Aware Interfaces

# 生命周期回调
要与容器对bean生命周期的管理进行交互，您可以实现Spring InitializingBean和DisposableBean接口。
容器对前者调用afterPropertiesSet()，对后者调用destroy()，让bean在初始化和销毁bean时执行某些操作。

JSR-250 @PostConstruct和@PreDestroy注解通常被认为是在现代Spring应用程序中接收生命周期回调的最佳实践。使用这些注解意味着您的bean没有耦合到特定于spring的接口。详情请参见使用@PostConstruct和@PreDestroy。 如果不希望使用JSR-250注解，但仍希望消除耦合，请考虑初始化方法和销毁方法bean定义元数据。

在内部，Spring框架使用BeanPostProcessor实现来处理它能找到的任何回调接口并调用适当的方法。如果您需要自定义特性或Spring默认不提供的其他生命周期行为，您可以自己实现BeanPostProcessor。
有关更多信息，请参见容器扩展点[link](https://docs.spring.io/spring-framework/reference/core/beans/factory-extension.html)。

除了初始化和销毁回调之外，spring管理的对象还可以实现Lifecycle接口，以便这些对象可以参与由容器自己的生命周期驱动的启动和关闭过程。本节将描述生命周期回调接口。



# 初始化回调
initializingbean接口允许bean在容器在bean上设置了所有必要的属性后执行初始化工作。InitializingBean接口指定了一个方法:
```JAVA
void afterPropertiesSet() throws Exception;
```
我们建议不要使用InitializingBean接口，因为它不必要地将代码耦合到Spring。另外，我们建议使用@PostConstruct注解或指定POJO初始化方法。对于基于xml的配置元数据，可以使用init-method属性指定具有无返回值无参数签名的方法的名称。在Java配置中，您可以使用@Bean的initMethod属性。参见接收生命周期回调。考虑下面的例子:
```XML
<bean id="exampleInitBean" class="examples.ExampleBean" init-method="init"/>
```
```JAVA
public class ExampleBean {

	public void init() {
		// do some initialization work
	}
}
```
上面的示例几乎与下面的示例(包含两个清单)具有完全相同的效果:
```JAVA
public class AnotherExampleBean implements InitializingBean {

	@Override
	public void afterPropertiesSet() {
		// do some initialization work
	}
}
```
但是，前面两个示例中的第一个示例没有将代码与Spring耦合。

注意，@PostConstruct和初始化方法通常是在容器的单例创建锁中执行的。__bean实例只有在从@PostConstruct方法返回后才被视为完全初始化并准备好发布给其他实例。__

> 可能导致容器长时间未就绪

这些单独的初始化方法仅用于验证配置状态，并可能根据给定的配置准备一些数据结构，但没有使用外部bean访问的进一步活动。否则会有初始化死锁的风险。

对于触发耗时的后初始化活动的场景，例如异步数据库准备步骤，bean应该实现__SmartInitializingSingleton.afterSingletonsInstantiated()__ 或依赖于上下文刷新事件:实现__ApplicationListener<ContextRefreshedEvent>__或声明其注解等效的__@EventListener(ContextRefreshedEvent.class)__。这些变体出现在所有常规的单例初始化之后，因此在任何单例创建锁之外。

> 工作中有小伙伴在@PostConstruct中加载缓存数据，导致启动速度极其慢

或者，您可以实现 (Smart)Lifecycle 接口，并集成容器的整体生命周期管理，包括自动启动机制，预销毁停止步骤，以及潜在的停止/重新启动回调(见下文)。



# 销毁回调

实现 org.springframework.beans.factory.DisposableBean 接口可以让bean在包含它的容器被销毁时获得回调。DisposableBean接口指定了一个方法：
```java
void destroy() throws Exception;
```
我们建议您不要使用DisposableBean回调接口，因为它不必要地将代码耦合到Spring。另外，我们建议使用@PreDestroy注解或指定bean定义支持的泛型方法。对于基于xml的配置元数据，您可以在<bean/>上使用destroy-method属性。在Java配置中，您可以使用@Bean的destroyMethod属性。参见接收生命周期回调。考虑下面的定义:
```java
<bean id="exampleDestructionBean" class="examples.ExampleBean" destroy-method="cleanup"/>
```
```java
public class ExampleBean {

	public void cleanup() {
		// do some destruction work (like releasing pooled connections)
	}
}
```
上述定义几乎与以下定义具有完全相同的效果:
```java
public class AnotherExampleBean implements DisposableBean {

	@Override
	public void destroy() {
		// do some destruction work (like releasing pooled connections)
	}
}
```
但是，前面两个定义中的第一个没有将代码与Spring耦合。

注意，Spring还支持destroy方法的推断，检测公共close或shutdown方法。这是Java配置类中@Bean方法的默认行为，并自动匹配 __Java.lang.autocloseable或Java.io.closeable实现__，也不会将销毁逻辑耦合到Spring。

对于使用XML的销毁方法推断，您可以为<bean>元素的销毁方法属性分配一个特殊的(推断的)值，该值指示Spring自动检测特定bean定义的bean类上的公共关闭或关闭方法。您还可以在<beans>元素的Default-Destroy-method属性上设置这个特殊的(推断的)值，以便将此行为应用于整个bean定义集(请参阅Default Initialization和Destroy Methods)。

对于扩展的关闭阶段，您可以实现Lifecycle接口，并在调用任何单例bean的destroy方法之前接收一个早期停止信号。
你也可以为一个有时间限制的停止步骤实现SmartLifecycle，容器将在继续销毁方法之前等待所有这些停止处理完成。



# 默认初始化和销毁方法
当您编写不使用spring特定的InitializingBean和DisposableBean回调接口的初始化和销毁方法回调时，您通常使用init()、initialize()、dispose()等名称来编写方法。理想情况下，这种生命周期回调方法的名称在整个项目中是标准化的，以便所有开发人员使用相同的方法名称并确保一致性。

您可以将Spring容器配置为“查找”每个bean上的命名初始化和销毁回调方法名。这意味着，作为应用程序开发人员，您可以编写应用程序类并使用名为init()的初始化回调，而不必为每个bean定义配置init-method="init"属性。
在创建bean时，Spring IoC容器调用该方法(并按照前面描述的标准生命周期回调契约)。这个特性还强制了初始化和销毁方法回调的一致命名约定。

假设初始化回调方法命名为init()，而销毁回调方法命名为destroy()。然后，您的类类似于以下示例中的类:
```java
public class DefaultBlogService implements BlogService {

	private BlogDao blogDao;

	public void setBlogDao(BlogDao blogDao) {
		this.blogDao = blogDao;
	}

	// this is (unsurprisingly) the initialization callback method
	public void init() {
		if (this.blogDao == null) {
			throw new IllegalStateException("The [blogDao] property must be set.");
		}
	}
}
```
然后你可以在bean中使用这个类，如下所示:
```java
<beans default-init-method="init">

	<bean id="blogService" class="com.something.DefaultBlogService">
		<property name="blogDao" ref="blogDao" />
	</bean>

</beans>
```
顶级<beans/>元素属性上 default-init-method 属性的存在导致Spring IoC容器将bean类上名为init的方法识别为初始化方法回调。在创建和组装bean时，如果bean类有这样的方法，将在适当的时候调用它。

通过在顶层<beans/>元素上使用default-destroy-method属性，可以类似地配置destroy方法回调(即在XML中)。

如果现有的bean类已经具有与约定不一致的回调方法，则可以通过使用<bean/>本身的init-method和destroy-method属性指定(在XML中)方法名来覆盖默认值。

Spring容器保证在为bean提供所有依赖项后立即调用已配置的初始化回调。因此，__初始化回调是在原始bean引用上调用的，这意味着AOP拦截器等等还没有应用到bean上__。首先完全创建目标bean，然后应用带有其拦截器链的AOP代理(例如)。如果目标bean和代理是分开定义的，那么您的代码甚至可以与原始目标bean交互，而绕过代理。因此，将拦截器应用于init方法是不一致的，因为这样做会将目标bean的生命周期与它的代理或拦截器耦合，并在代码直接与原始目标bean交互时留下奇怪的语义。



# 组合生命周期机制

从Spring 2.5开始，您有三个选项来控制bean生命周期行为:
- The InitializingBean and DisposableBean callback interfaces
- Custom init() and destroy() methods
- The @PostConstruct and @PreDestroy annotations

如果为一个bean配置了多个生命周期机制，并且每个机制都配置了不同的方法名，那么每个配置的方法都按照本文后面列出的顺序运行。但是，如果为这些生命周期机制中的一个以上配置了相同的方法名(例如，为初始化方法配置init())，则该方法将运行一次，如前一节所述。

使用不同的初始化方法为同一个bean配置了多个生命周期机制，如下所示:
1. Methods annotated with @PostConstruct
2. afterPropertiesSet() as defined by the InitializingBean callback interface
3. A custom configured init() method

Destroy方法的调用顺序相同:

1. Methods annotated with @PreDestroy
2. destroy() as defined by the DisposableBean callback interface
3. A custom configured destroy() method



# 启动和关闭回调

生命周期接口为任何有自己生命周期需求的对象定义了基本的方法(比如启动和停止一些后台进程):
```java
public interface Lifecycle {

	void start();

	void stop();

	boolean isRunning();
}
```
任何spring管理的对象都可以实现生命周期接口。然后，当ApplicationContext本身接收到启动和停止信号时(例如，对于运行时的停止/重启场景)，它将这些调用级联到该上下文中定义的所有生命周期实现。它通过委托给一个LifecycleProcessor来完成这个任务，如下面所示:
```java
public interface LifecycleProcessor extends Lifecycle {

	void onRefresh();

	void onClose();
}
```
注意，LifecycleProcessor本身是Lifecycle接口的扩展。它还添加了另外两个方法，用于对正在刷新和关闭的上下文作出反应。

请注意，常规的org.springframework.context.Lifecycle接口是用于显式启动和停止通知的普通契约，并不意味着在上下文刷新时自动启动。对于自动启动的细粒度控制和特定bean的优雅停止(包括启动和停止阶段)，可以考虑实现扩展的org.springframework.context.SmartLifecycle接口。

另外，请注意，停止通知不能保证在销毁之前出现。在常规关闭时，所有Lifecycle bean在传播常规销毁回调之前首先收到一个停止通知。但是，在上下文生命周期中的热刷新或停止刷新尝试时，只调用destroy方法。

启动和关闭调用的顺序可能很重要。如果任何两个对象之间存在“依赖”关系，则依赖方在其依赖之后开始，并在其依赖之前停止。然而，有时直接依赖关系是未知的。您可能只知道某种类型的对象应该先于另一类型的对象开始。在这些情况下，SmartLifecycle接口定义了另一个选项，即在其顶级接口Phased上定义的getPhase()方法。下面的清单显示了phase接口的定义:
```java
public interface Phased {

	int getPhase();
}
```
SmartLifecycle接口的定义如下所示:
```java
public interface SmartLifecycle extends Lifecycle, Phased {

	boolean isAutoStartup();

	void stop(Runnable callback);
}
```
启动时，相位最低的对象首先启动。当停止时，遵循相反的顺序。因此，一个实现SmartLifecycle的对象，其getPhase()方法返回Integer。MIN_VALUE将是第一个启动和最后一个停止的。在频谱的另一端，相位值为整数。MAX_VALUE表示对象应该最后启动，首先停止(可能是因为它依赖于正在运行的其他进程)。在考虑阶段值时，同样重要的是要知道，任何没有实现SmartLifecycle的“正常”生命周期对象的默认阶段都是0。因此，任何负相位值都表示一个对象应该在这些标准组件之前启动(并在它们之后停止)。相反，对于任何正相位值都是正确的。

SmartLifecycle定义的stop方法接受回调。任何实现都必须在该实现的关闭过程完成后调用该回调的run()方法。
这样可以在必要时进行异步关闭，因为LifecycleProcessor接口的默认实现DefaultLifecycleProcessor会一直等待到每个阶段中的一组对象调用该回调的超时间值。默认的每阶段超时为30秒。您可以通过在上下文中定义一个名为lifecycleProcessor的bean来覆盖默认的生命周期处理器实例。
如果你只想修改超时时间，定义以下内容就足够了:

```xml
<bean id="lifecycleProcessor" class="org.springframework.context.support.DefaultLifecycleProcessor">
	<!-- timeout value in milliseconds -->
	<property name="timeoutPerShutdownPhase" value="10000"/>
</bean>
```
如前所述，LifecycleProcessor接口还定义了用于刷新和关闭上下文的回调方法。后者驱动关闭过程，就像已经显式调用stop()一样，但它发生在上下文关闭时。另一方面，“刷新”回调启用了SmartLifecycle bean的另一个特性。
当上下文被刷新时(在所有对象被实例化和初始化之后)，将调用该回调。此时，默认的生命周期处理器会检查每个SmartLifecycle对象的isAutoStartup()方法返回的布尔值。如果为true，则在该点启动该对象，而不是等待上下文或其自身的start()方法的显式调用(与上下文刷新不同，对于标准上下文实现，上下文启动不会自动发生)。
相位值和任何“依赖”关系决定了前面描述的启动顺序。



# 在非web应用程序中优雅地关闭Spring IoC容器
本节仅适用于非web应用。Spring基于web的ApplicationContext实现已经有了适当的代码，可以在相关的web应用程序关闭时优雅地关闭Spring IoC容器。

如果您在非web应用程序环境中使用Spring的IoC容器(例如，在富客户端桌面环境中)，请向JVM注册一个shutdown钩子。这样做可以确保优雅的关闭，并在单例bean上调用相关的destroy方法，以便释放所有资源。
您仍然必须正确地配置和实现这些destroy回调。

要注册一个关机钩子，调用registerShutdownHook()方法，该方法在ConfigurableApplicationContext接口上声明，如下例所示:
```java
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public final class Boot {

	public static void main(final String[] args) throws Exception {
		ConfigurableApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");

		// add a shutdown hook for the above context...
		ctx.registerShutdownHook();

		// app runs here...

		// main method exits, hook is called prior to the app shutting down...
	}
}
```

> 如果在业务代码中自定义了线程池，并且不在Spring的管理范围内，可以使用这种方式，与Spring容器的生命周期保持一致



# ApplicationContextAware and BeanNameAware

当一个ApplicationContext创建了一个对象实例来实现 org.springframework.context.ApplicationContextAware接口时，该实例将被提供一个对该ApplicationContext的引用。下面的清单显示了ApplicationContextAware接口的定义:
```java
public interface ApplicationContextAware {

	void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
```
因此，bean可以通过ApplicationContext接口或将引用强制转换为该接口的已知子类(例如ConfigurableApplicationContext，它发布了额外的功能)，以编程方式操作创建它们的ApplicationContext。
一种用途是对其他bean进行编程检索。有时这个功能很有用。但是，__一般情况下，您应该避免使用它，因为它将代码耦合到Spring，并且不遵循控制反转样式__，在这种样式中，协作器作为属性提供给bean。ApplicationContext.的其他方法提供对文件资源、发布应用程序事件和访问MessageSource的访问。
这些附加功能在ApplicationContext的附加功能中有描述。

自动装配是获取对ApplicationContext引用的另一种替代方法。传统的构造函数和byType自动装配模式(如自动装配协作器中所述)可以分别为构造函数参数或setter方法参数提供ApplicationContext类型的依赖。要获得更大的灵活性，包括自动装配字段和多参数方法的能力，请使用基于注解的自动装配特性。
如果你这样做了，ApplicationContext被自动连接到一个字段、构造函数参数或方法参数中，如果有问题的字段、构造函数或方法带有@Autowired注解，则该字段、构造函数参数或方法参数需要ApplicationContext类型。
有关更多信息，请参见使用@Autowired。

当ApplicationContext创建一个实现org.springframework.beans.factory.BeanNameAware接口的类时，将为该类提供对其关联对象定义中定义的名称的引用。下面的清单显示了BeanNameAware接口的定义:
```java
public interface BeanNameAware {

	void setBeanName(String name) throws BeansException;
}
```
回调在填充普通bean属性之后调用，但在初始化回调(如InitializingBean.afterPropertiesSet())或自定义初始化方法之前调用。



# 其它生命周期接口

除了ApplicationContextAware和BeanNameAware(前面讨论过)之外，Spring还提供了广泛的Aware回调接口，
让bean向容器表明它们需要特定的基础设施依赖。作为一般规则，名称指示依赖项类型。下表总结了最重要的Aware接口:

1. ApplicationContextAware                        获取ApplicationContext
2. ApplicationEventPublisherAware   	ApplicationContext内部事件的事件发布者
3. BeanClassLoaderAware                           类加载器用于加载bean类
4. BeanFactoryAware                                    获取BeanFactory
5. BeanNameAware                                       声明bean的名称
6. LoadTimeWeaverAware                            为在加载时处理类定义而定义的编织器
7. MessageSourceAware                               解析消息的已配置策略(支持参数化和国际化)
8. NotificationPublisherAware                     Spring JMX通知发布者
9. ResourceLoaderAware                              配置用于底层访问资源的加载器
10. ServletConfigAware    容器运行的当前ServletConfig。只在支持web的Spring ApplicationContext中有效
11. ServletContextAware   容器运行的当前ServletContext。只在支持web的Spring ApplicationContext中有效。

再次注意，使用这些接口将代码绑定到Spring API，而不遵循控制反转样式。因此，我们建议将它们用于需要对容器进行编程访问的基础设施bean。