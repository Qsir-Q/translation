# ApplicationContext的附加功能
正如本章介绍中所讨论的，org.springframework.beans.factory包提供了管理和操作bean的基本功能，包括以编程的方式。context包添加了ApplicationContext接口，它扩展了BeanFactory接口，此外还扩展了其他接口，以更面向应用程序框架的风格提供额外的功能。许多人以完全声明式的方式使用ApplicationContext，甚至不以编程方式创建它，而是依靠诸如ContextLoader之类的支持类来自动实例化ApplicationContext，作为Jakarta EE web应用程序正常启动过程的一部分。

为了以更面向框架的风格增强BeanFactory功能，上下文包还提供了以下功能:
- 通过MessageSource接口以i18n样式访问消息。
- 通过ResourceLoader接口访问资源，如url和文件。
- 事件发布，即通过使用ApplicationEventPublisher接口实现ApplicationListener接口的bean。
- 通过HierarchicalBeanFactory接口加载多个(分层)上下文，让每个上下文都专注于一个特定的层，例如应用程序的web层。



# 使用MessageSource进行国际化

ApplicationContext接口扩展了一个名为MessageSource的接口，因此提供了国际化(“i18n”)功能。Spring还提供了HierarchicalMessageSource接口，它可以分层地解析消息。这些接口一起为Spring实现消息解析提供了基础。在这些接口上定义的方法包括:
- String getMessage(String code, Object[] args, String default, Locale loc):用于从MessageSource检索消息的基本方法。如果没有找到指定语言环境的消息，则使用默认消息。通过使用标准库提供的MessageFormat功能，传入的任何参数都成为替换值。
- String getMessage(String code, Object[] args, Locale loc):本质上与前一个方法相同，但有一个区别:不能指定默认消息。如果找不到消息，则抛出NoSuchMessageException。
- 上述方法中使用的所有属性也都包装在一个名为MessageSourceResolvable类中，您可以与此方法一起使用该类。

当加载ApplicationContext时，它会自动搜索上下文中定义的MessageSource bean。bean的名称必须为messageSource。如果找到了这样的bean，那么对上述方法的所有调用都将委托给消息源。如果没有找到消息源，ApplicationContext将尝试查找包含同名bean的父节点。如果是，它将使用该bean作为MessageSource。
如果ApplicationContext找不到任何消息源，则实例化一个空的DelegatingMessageSource，以便能够接受对上面定义的方法的调用。

Spring提供了三个MessageSource实现，ResourceBundleMessageSource, ReloadableResourceBundleMessageSource和StaticMessageSource。它们都实现了HierarchicalMessageSource，以便进行嵌套消息传递。StaticMessageSource很少使用，但它提供了将消息添加到源的编程方法。下面的例子展示了ResourceBundleMessageSource:
```xml
<beans>
	<bean id="messageSource"
			class="org.springframework.context.support.ResourceBundleMessageSource">
		<property name="basenames">
			<list>
				<value>format</value>
				<value>exceptions</value>
				<value>windows</value>
			</list>
		</property>
	</bean>
</beans>
```
该示例假设在类路径中定义了三个资源包，分别是format、exceptions和windows。任何解析消息的请求都是以通过ResourceBundle对象解析消息的jdk标准方式处理的。为了本例的目的，假设上述两个资源包文件的内容如下:
```properties
# in format.properties
message=Alligators rock!
```

```properties
# in exceptions.properties
argument.required=The {0} argument is required.
```

下一个示例展示了运行MessageSource功能的程序。记住，所有ApplicationContext实现也是MessageSource实现，因此可以被强制转换为MessageSource接口。
```java
public static void main(String[] args) {
	MessageSource resources = new ClassPathXmlApplicationContext("beans.xml");
	String message = resources.getMessage("message", null, "Default", Locale.ENGLISH);
	System.out.println(message);
}
```
上述程序的输出结果如下:
```txt
Alligators rock!
```

总之，MessageSource是在一个名为beans.xml的文件中定义的，该文件存在于类路径的根目录中。
messageSource bean定义通过其basenames属性引用许多资源包。在列表中传递给basenames属性的三个文件作为文件存在于类路径的根目录中，它们被称为format.properties, exceptions.properties, and windows.properties。

下一个示例显示传递给消息查找的参数。这些参数被转换为String对象，并插入到查找消息中的占位符中。
```xml
<beans>

	<!-- this MessageSource is being used in a web application -->
	<bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
		<property name="basename" value="exceptions"/>
	</bean>

	<!-- lets inject the above MessageSource into this POJO -->
	<bean id="example" class="com.something.Example">
		<property name="messages" ref="messageSource"/>
	</bean>

</beans>
```

```java
public class Example {

	private MessageSource messages;

	public void setMessages(MessageSource messages) {
		this.messages = messages;
	}

	public void execute() {
		String message = this.messages.getMessage("argument.required",
			new Object [] {"userDao"}, "Required", Locale.ENGLISH);
		System.out.println(message);
	}
}
```
调用execute()方法的结果输出如下:
```txt
The userDao argument is required.
```
关于国际化(“i18n”)，Spring的各种MessageSource实现遵循与标准JDK ResourceBundle相同的语言环境解析和回退规则。
简而言之，继续前面定义的messageSource示例，
如果希望根据英国(en-GB)语言环境解析消息，则需要创建名为format_en_GB.properties, exceptions_en_GB.properties, and windows_en_GB.properties。

通常，区域设置解析由应用程序的周围环境管理。在下面的示例中，(British)消息解析的区域设置是手动指定的:
```txt
# in exceptions_en_GB.properties
argument.required=Ebagum lad, the ''{0}'' argument is required, I say, required.
```

```java
public static void main(final String[] args) {
	MessageSource resources = new ClassPathXmlApplicationContext("beans.xml");
	String message = resources.getMessage("argument.required",
		new Object [] {"userDao"}, "Required", Locale.UK);
	System.out.println(message);
}
```

运行上述程序得到的输出如下:
```txt
Ebagum lad, the 'userDao' argument is required, I say, required.
```

您还可以使用MessageSourceAware接口获取对已定义的任何MessageSource的引用。
在实现MessageSourceAware接口的ApplicationContext中定义的任何bean在创建和配置bean时都会被注入应用程序上下文的MessageSource。

因为Spring的MessageSource基于Java的ResourceBundle，所以它不会合并具有相同基名的bundle，而只会使用找到的第一个bundle。具有相同基名的后续消息包将被忽略。

作为ResourceBundleMessageSource的替代方案，Spring提供了一个ReloadableResourceBundleMessageSource类。这个变体支持相同的包文件格式，但比基于标准JDK的ResourceBundleMessageSource实现更灵活。特别是，它允许从任何Spring资源位置(不仅仅是从类路径)读取文件，并支持bundle属性文件的热重新加载(同时在两者之间有效地缓存它们)。详情请参见ReloadableResourceBundleMessageSource javadoc。



# 标准事件和自定义事件

ApplicationContext中的事件处理是通过ApplicationEvent类和ApplicationListener接口提供的。如果将实现ApplicationListener接口的bean部署到上下文中，则每次将ApplicationEvent发布到ApplicationContext时，都会通知该bean。本质上，这是标准的Observer设计模式。

从Spring 4.2开始，事件基础设施得到了显著改进，并提供了基于注释的模型以及发布任意事件的能力(也就是说，不一定是从ApplicationEvent扩展的对象)。当这样的对象被发布时，我们将它包装在一个事件中。

下表描述了Spring提供的标准事件:
1. ContextRefreshedEvent:
  在ApplicationContext被初始化或刷新时发布(例如，通过在ConfigurableApplicationContext接口上使用refresh()方法)。这里，__“初始化”意味着加载所有bean，检测并激活后处理器bean，预实例化单例，并准备好使用ApplicationContext对象__。只要上下文没有关闭，只要所选的ApplicationContext实际上支持这种“热”刷新，刷新就可以被触发多次。例如，XmlWebApplicationContext支持热刷新，但GenericApplicationContext不支持。

2. ContextStartedEvent
  在ConfigurableApplicationContext接口上使用start()方法启动ApplicationContext时发布。这里，“已启动”意味着所有生命周期bean都接收显式启动信号。通常，该信号用于在显式停止后重新启动bean，但它也可以用于启动未配置为自动启动的组件(例如，在初始化时尚未启动的组件)。

  > 可监听该时间，启动预热线程，例如加载缓存之类的操作

3. ContextStoppedEvent
  在ConfigurableApplicationContext接口上使用stop()方法停止ApplicationContext时发布。这里，“已停止”意味着所有生命周期bean都接收到显式停止信号。停止的上下文可以通过start()调用重新启动。

4. ContextClosedEvent
  通过在ConfigurableApplicationContext接口上使用close()方法或通过JVM关闭钩子关闭ApplicationContext时发布。这里，“关闭”意味着所有的单例bean都将被销毁。一旦上下文关闭，它的生命周期就结束了，不能刷新或重新启动。

5. RequestHandledEvent
  一个特定于web的事件，告诉所有bean一个HTTP请求已经得到了服务。此事件在请求完成后发布。这个事件只适用于使用Spring的DispatcherServlet的web应用程序。

6. ServletRequestHandledEvent
  RequestHandledEvent的子类，用于添加servlet特定的上下文信息。

您还可以创建和发布自己的自定义事件。下面的例子展示了一个扩展Spring的ApplicationEvent基类的简单类:
```java
public class BlockedListEvent extends ApplicationEvent {

	private final String address;
	private final String content;

	public BlockedListEvent(Object source, String address, String content) {
		super(source);
		this.address = address;
		this.content = content;
	}

	// accessor and other methods...
}
```
要发布自定义ApplicationEvent，请调用ApplicationEventPublisher上的publishEvent()方法。通常，这是通过创建一个实现ApplicationEventPublisherAware的类并将其注册为Spring bean来完成的。下面的例子展示了这样一个类:
```java
public class EmailService implements ApplicationEventPublisherAware {

	private List<String> blockedList;
	private ApplicationEventPublisher publisher;

	public void setBlockedList(List<String> blockedList) {
		this.blockedList = blockedList;
	}

	public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
		this.publisher = publisher;
	}

	public void sendEmail(String address, String content) {
		if (blockedList.contains(address)) {
			publisher.publishEvent(new BlockedListEvent(this, address, content));
			return;
		}
		// send email...
	}
}
```
在配置时，Spring容器检测到EmailService实现了ApplicationEventPublisherAware并自动调用setApplicationEventPublisher()。实际上，传入的参数是Spring容器本身。您通过应用程序的ApplicationEventPublisher接口与应用程序上下文进行交互。

要接收定制的ApplicationEvent，您可以创建一个实现ApplicationListener的类，并将其注册为Spring bean。
下面的例子展示了这样一个类:
```java
public class BlockedListNotifier implements ApplicationListener<BlockedListEvent> {

	private String notificationAddress;

	public void setNotificationAddress(String notificationAddress) {
		this.notificationAddress = notificationAddress;
	}

	public void onApplicationEvent(BlockedListEvent event) {
		// notify appropriate parties via notificationAddress...
	}
}
```
注意，ApplicationListener一般是用自定义事件的类型参数化的(在前面的例子中是BlockedListEvent)。
这意味着onApplicationEvent()方法可以保持类型安全，避免任何向下转换的需要。您可以注册任意数量的事件侦听器，但请注意，默认情况下，事件侦听器以同步方式接收事件。这意味着publishhevent()方法将阻塞，直到所有侦听器完成对事件的处理。这种同步和单线程方法的一个优点是，当侦听器接收到事件时，如果事务上下文可用，它将在发布者的事务上下文中操作。如果需要另一种事件发布策略，例如默认异步事件处理，请参阅javadoc中Spring的ApplicationEventMulticaster接口和SimpleApplicationEventMulticaster实现，以获取可应用于自定义“ApplicationEventMulticaster”bean定义的配置选项。在这些情况下，不会为事件处理传播ThreadLocals和日志上下文。有关可观察性问题的更多信息，请参阅@EventListener Observability部分。

下面的示例显示了用于注册和配置上面每个类的bean定义:
```xml
<bean id="emailService" class="example.EmailService">
	<property name="blockedList">
		<list>
			<value>known.spammer@example.org</value>
			<value>known.hacker@example.org</value>
			<value>john.doe@example.org</value>
		</list>
	</property>
</bean>

<bean id="blockedListNotifier" class="example.BlockedListNotifier">
	<property name="notificationAddress" value="blockedlist@example.org"/>
</bean>

   <!-- optional: a custom ApplicationEventMulticaster definition -->
<bean id="applicationEventMulticaster" class="org.springframework.context.event.SimpleApplicationEventMulticaster">
	<property name="taskExecutor" ref="..."/>
	<property name="errorHandler" ref="..."/>
</bean>
```
综上所述，当调用emailService bean的sendEmail()方法时，如果有任何应该被阻止的电子邮件消息，则会发布一个BlockedListEvent类型的自定义事件。blockedListNotifier bean注册为ApplicationListener并接收BlockedListEvent，此时它可以通知适当的各方。

Spring的事件机制是为相同应用程序上下文中的Spring bean之间的简单通信而设计的。然而，对于更复杂的企业集成需求，单独维护的Spring integration项目为构建轻量级的、面向模式的、事件驱动的体系结构提供了完整的支持，这些体系结构构建在众所周知的Spring编程模型之上。



# 基于注解的事件监听器

您可以使用@EventListener注释在托管bean的任何方法上注册事件侦听器。BlockedListNotifier可以重写如下:
```java
public class BlockedListNotifier {

	private String notificationAddress;

	public void setNotificationAddress(String notificationAddress) {
		this.notificationAddress = notificationAddress;
	}

	@EventListener
	public void processBlockedListEvent(BlockedListEvent event) {
		// notify appropriate parties via notificationAddress...
	}
}
```
方法签名再次声明它要侦听的事件类型，但这一次使用了一个灵活的名称，并且没有实现特定的侦听器接口。
事件类型也可以通过泛型缩小范围，只要实际事件类型在其实现层次结构中解析泛型参数即可。

如果您的方法应该侦听多个事件，或者您想在没有任何参数的情况下定义它，那么也可以在注释本身上指定事件类型。下面的例子展示了如何这样做:
```java
@EventListener({ContextStartedEvent.class, ContextRefreshedEvent.class})
public void handleContextStart() {
	// ...
}
```

还可以通过使用定义SpEL表达式的注释的条件属性添加额外的运行时过滤，该注释应该与实际调用特定事件的方法相匹配。

下面的例子展示了我们的通知器如何被重写为只有当事件的content属性等于my-event时才会被调用:
```java
@EventListener(condition = "#blEvent.content == 'my-event'")
public void processBlockedListEvent(BlockedListEvent blEvent) {
	// notify appropriate parties via notificationAddress...
}
```
每个SpEL表达式针对一个专用上下文求值。下表列出了上下文可用的项，以便您可以将它们用于条件事件处理:
1. Name             #root.event or event
2. Arguments array  #root.args or args; args[0] to access the first argument, etc.
3. Argument name    #blEvent or #a0 (you can also use #p0 or #p<#arg> parameter notation as an alias)

注意#root。事件使您能够访问底层事件，即使您的方法签名实际上引用了已发布的任意对象。
如果你需要发布一个事件作为处理另一个事件的结果，你可以改变方法签名来返回应该发布的事件，如下面的例子所示:
```java
@EventListener
public ListUpdateEvent handleBlockedListEvent(BlockedListEvent event) {
	// notify appropriate parties via notificationAddress and
	// then publish a ListUpdateEvent...
}
```

handleBlockedListEvent()方法为它处理的每个BlockedListEvent发布一个新的ListUpdateEvent。如果需要发布多个事件，则可以返回一个Collection或事件数组。



# 异步的监听器
如果希望特定的监听器异步处理事件，可以重用常规的@Async支持。下面的例子展示了如何这样做:
```java
@EventListener
@Async
public void processBlockedListEvent(BlockedListEvent event) {
	// BlockedListEvent is processed in a separate thread
}
```
在使用异步事件时，请注意以下限制:
1. __如果异步事件侦听器抛出异常，则不会将其传播给调用者__。参见AsyncUncaughtExceptionHandler了解更多细节。
2. 异步事件侦听器方法不能通过返回值来发布后续事件。如果您需要发布另一个事件作为处理的结果，则注入ApplicationEventPublisher来手动发布事件。
3. 默认情况下，事件处理不会传播ThreadLocals和日志上下文。有关可观察性问题的更多信息，请参阅@EventListener Observability部分。



# 排序的监听器

如果你需要一个监听器在另一个监听器之前被调用，你可以在方法声明中添加@Order注释，如下例所示:
```java
@EventListener
@Order(42)
public void processBlockedListEvent(BlockedListEvent event) {
	// notify appropriate parties via notificationAddress...
}
```

## 通用的事件
还可以使用泛型进一步定义事件的结构。
考虑使用EntityCreatedEvent<T>，其中T是被创建的实际实体的类型。
例如，您可以创建以下侦听器定义来仅接收Person的EntityCreatedEvent:

```java
@EventListener
public void onPersonCreated(EntityCreatedEvent<Person> event) {
	// ...
}
```

由于类型擦除，只有当触发的事件解析了事件侦听器过滤的泛型参数时(也就是说，类似于类PersonCreatedEvent扩展了EntityCreatedEvent<Person>{…})，这才有效。在某些情况下，如果所有事件都遵循相同的结构(就像前面示例中的事件一样)，这可能会变得非常乏味。在这种情况下，您可以实现ResolvableTypeProvider来指导框架超越运行时环境提供的内容。下面的事件展示了如何这样做:
```java
public class EntityCreatedEvent<T> extends ApplicationEvent implements ResolvableTypeProvider {

	public EntityCreatedEvent(T entity) {
		super(entity);
	}

	@Override
	public ResolvableType getResolvableType() {
		return ResolvableType.forClassWithGenerics(getClass(), ResolvableType.forInstance(getSource()));
	}
}
```
这不仅适用于ApplicationEvent，也适用于作为事件发送的任意对象。

最后，与经典的ApplicationListener实现一样，实际的多播在运行时通过上下文范围的ApplicationEventMulticaster进行。默认情况下，这是一个在调用线程中同步发布事件的SimpleApplicationEventMulticaster。这可以通过“applicationEventMulticaster”bean定义来替换/定制，例如，用于异步处理所有事件和/或处理侦听器异常:
```java
@Bean
ApplicationEventMulticaster applicationEventMulticaster() {
	SimpleApplicationEventMulticaster multicaster = new SimpleApplicationEventMulticaster();
	multicaster.setTaskExecutor(...);
	multicaster.setErrorHandler(...);
	return multicaster;
}
```



# 方便地访问底层资源

为了最佳地使用和理解应用程序上下文，您应该熟悉Spring的资源抽象，如参考资料中所述。

应用程序上下文是一个资源加载器，它可以用来加载资源对象。Resource本质上是JDK java.net.URL类的一个功能更丰富的版本。事实上，Resource的实现在适当的地方包装了一个java.net.URL实例。Resource可以以透明的方式从几乎任何位置获取低级资源，包括从类路径、文件系统位置、任何可以用标准URL描述的位置，以及其他一些变体。如果资源位置字符串是一个没有任何特殊前缀的简单路径，那么这些资源的来源是特定的，并且适合于实际的应用程序上下文类型。

您可以配置部署到应用程序上下文中的bean，以实现特殊的回调接口ResourceLoaderAware，以便在初始化时自动回调，同时将应用程序上下文本身作为ResourceLoader传入。您还可以公开Resource类型的属性，以用于访问静态资源。它们像其他属性一样被注入其中。您可以将这些Resource属性指定为简单的String路径，并依赖于部署bean时从这些文本字符串到实际Resource对象的自动转换。

提供给ApplicationContext构造函数的位置路径实际上是资源字符串，以简单的形式，根据特定的上下文实现进行适当的处理。例如，ClassPathXmlApplicationContext将简单的位置路径视为类路径位置。您还可以使用带有特殊前缀的位置路径(资源字符串)来强制从类路径或URL加载定义，而不管实际的上下文类型是什么。



# 应用程序启动跟踪

ApplicationContext管理Spring应用程序的生命周期，并围绕组件提供丰富的编程模型。因此，复杂的应用程序可以具有同样复杂的组件图和启动阶段。

使用特定的指标跟踪应用程序启动步骤可以帮助理解在启动阶段时间花在哪里，但它也可以作为一种更好地理解整个上下文生命周期的方法。

AbstractApplicationContext(和它的子类)被一个ApplicationStartup工具化，它收集关于不同启动阶段的StartupStep数据:
1. 应用程序上下文生命周期(基包扫描、配置类管理)
2. bean生命周期(实例化、智能初始化、后期处理)
3. 应用程序事件处理
下面是一个在AnnotationConfigApplicationContext中插装的例子:
```java
// create a startup step and start recording
StartupStep scanPackages = getApplicationStartup().start("spring.context.base-packages.scan");
// add tagging information to the current step
scanPackages.tag("packages", () -> Arrays.toString(basePackages));
// perform the actual phase we're instrumenting
this.scanner.scan(basePackages);
// end the current step
scanPackages.end();
```
应用程序上下文已经通过多个步骤进行了检测。
一旦记录下来，这些启动步骤就可以用特定的工具收集、显示和分析。要获得现有启动步骤的完整列表，您可以查看专门的附录部分。

默认的ApplicationStartup实现是一个无操作的变体，以最小化开销。
这意味着默认情况下，在应用程序启动期间不会收集任何指标。
Spring框架附带了一个用Java FlightRecorder跟踪启动步骤的实现:FlightRecorderApplicationStartup。
要使用此变体，必须在创建ApplicationContext后立即将其实例配置到ApplicationContext。

如果开发人员正在提供他们自己的AbstractApplicationContext子类，或者如果他们希望收集更精确的数据，他们也可以使用ApplicationStartup基础结构。

ApplicationStartup只用于应用程序启动和核心容器;这绝不是Java分析器或度量库(如Micrometer)的替代品。
要开始收集自定义的StartupStep，组件可以直接从应用程序上下文中获取ApplicationStartup实例，让它们的组件实现ApplicationStartupAware，或者在任何注入点请求ApplicationStartup类型。

开发人员不应该使用“spring”。创建自定义启动步骤时的命名空间。这个名称空间保留给Spring内部使用，可能会有变化。



# 方便的ApplicationContext实例化Web应用程序
您可以通过使用(例如ContextLoader)声明式地创建ApplicationContext实例。
当然，您也可以通过使用其中一个ApplicationContext实现以编程方式创建ApplicationContext实例。
```java
<context-param>
	<param-name>contextConfigLocation</param-name>
	<param-value>/WEB-INF/daoContext.xml /WEB-INF/applicationContext.xml</param-value>
</context-param>

<listener>
	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```
监听器检查contextConfigLocation参数。如果该参数不存在，监听器将使用/WEB-INF/applicationContext.xml作为默认值。
当参数存在时，侦听器使用预定义的分隔符(逗号、分号和空格)分隔String，并使用这些值作为搜索应用程序上下文的位置。
还支持ant风格的路径模式。例如/WEB-INF/*Context.xml(适用于所有以Context.xml结尾并且位于WEB-INF目录中的文件)和/WEB-INF/**/*Context.xml(适用于WEB-INF的任何子目录中的所有此类文件)。



# 将Spring ApplicationContext部署为Jakarta EE RAR文件

可以将Spring ApplicationContext作为RAR文件部署，将上下文及其所需的所有bean类和库jar封装在Jakarta EE RAR部署单元中。
这相当于引导能够访问Jakarta EE服务器设施的独立ApplicationContext(仅托管在Jakarta EE环境中)。
RAR部署是部署无头WAR文件场景的一种更自然的替代方案——实际上，没有任何HTTP入口点的WAR文件仅用于在Jakarta EE环境中引导Spring ApplicationContext。

RAR部署非常适合不需要HTTP入口点，而只包含消息端点和计划作业的应用程序上下文。
这种上下文中的bean可以使用应用服务器资源，比如JTA事务管理器和JNDI绑定的JDBC数据源实例和JMS ConnectionFactory实例，还可以向平台的JMX服务器注册——所有这些都是通过Spring的标准事务管理以及JNDI和JMX支持设施完成的。
应用组件还可以通过Spring的TaskExecutor抽象与应用服务器的JCA WorkManager进行交互。
有关RAR部署中涉及的配置细节，请参阅SpringContextResourceAdapter类的javadoc。

要将Spring ApplicationContext简单地部署为Jakarta EE RAR文件:
1. 将所有应用程序类打包到一个RAR文件中(这是一个具有不同文件扩展名的标准JAR文件)。
2. 将所有必需的库jar添加到RAR存档的根目录中。
3. 添加一个META-INF/ra.xml部署描述符(如SpringContextResourceAdapter的javadoc所示)和相应的Spring XML bean定义文件(通常是META-INF/applicationContext.xml)。
4. 将生成的RAR文件放入应用服务器的部署目录中。

这种RAR部署单元通常是独立的。它们不向外界公开组件，甚至不向同一应用程序的其他模块公开组件。
与基于rar的ApplicationContext的交互通常通过与其他模块共享的JMS目的地进行。
例如，基于rar的ApplicationContext还可以调度一些作业或对文件系统中的新文件做出反应(或类似的)。
如果它需要允许来自外部的同步访问，它可以(例如)导出RMI端点，这些端点可以被同一机器上的其他应用程序模块使用。