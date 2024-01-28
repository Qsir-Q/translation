# Bean总览

Spring IoC容器管理一个或多个bean。这些bean是用您提供给容器的配置元数据创建的(例如，以XML <bean/>定义的形式)

在容器本身内，这些bean定义表示为BeanDefinition对象，其中包含(除其他信息外)以下元数据：
- 包限定的类名:通常是定义的bean的实际实现类
- Bean行为配置元素，它声明Bean在容器中的行为(范围、生命周期回调等等)
- 对bean执行其工作所需的其他bean的引用。这些引用也称为协作者或依赖项
- 要在新创建的对象中设置的其他配置设置—例如，池的大小限制或要在管理连接池的bean中使用的连接数

此元数据转换为组成每个bean定义的一组属性。下表描述了这些属性：
- 属性(Property)
- 类
- 名称
- 范围
- 构造器参数
- 属性(Properties)
- 注入模式
- 懒加载模式
- 初始化方法
- 销毁方法

除了包含如何创建特定bean的信息的bean定义之外，ApplicationContext实现还允许注册(由用户)在容器外部创建的现有对象。这是通过getBeanFactory()方法访问ApplicationContext的BeanFactory来完成的，该方法返回DefaultListableBeanFactory实现。DefaultListableBeanFactory通过registerSingleton(..)和registerBeanDefinition(..)方法支持这种注册。但是，典型的应用程序只使用通过常规bean定义元数据定义的bean

Bean元数据和手动提供的单例实例需要尽早注册，以便容器在自动装配和其他自检步骤中正确地推断它们。
虽然在某种程度上支持覆盖现有的元数据和现有的单例实例，但在运行时注册新bean(__与对工厂的实时访问并发__)并不是官方支持的，并且可能导致并发访问异常、bean容器中的不一致状态，或者两者兼而有之



# 命名bean

每个bean都有一个或多个标识符。__这些标识符在承载bean的容器中必须是唯一的__。一个bean通常只有一个标识符。但是，如果需要多个别名，则可以将额外的标识符视为别名。

在基于xml的配置元数据中，可以使用id属性、name属性或两者同时使用来指定bean标识符。id属性允许您精确地指定一个id。通常，这些名称是字母数字('myBean'， ' someeservice '等)，但它们也可以包含特殊字符。
如果希望为bean引入其他别名，还可以在name属性中指定它们，用逗号(，)、分号(;)或空格分隔。虽然id属性被定义为xsd:字符串类型，但是bean id唯一性是由容器强制执行的，而不是由XML解析器强制执行的。

您不需要为bean提供名称或id。__如果没有显式地提供名称或id，容器将为该bean生成唯一的名称__。
但是，如果您希望通过使用ref元素或Service Locator样式查找来通过名称引用该bean，则必须提供一个名称。
不提供name的原因与使用内部bean和自动装配协作器有关。



# Bean命名约定
约定是在命名bean时使用标准的Java约定作为实例字段名。也就是说，bean名称以小写字母开头，然后使用驼峰式大小写。此类名称的示例包括accountManager、accountService、userDao、loginController等等。 
一致的命名bean使您的配置更易于阅读和理解。另外，如果您使用Spring AOP，那么在向一组按名称相关的bean应用通知时，它会有很大帮助

通过类路径中的组件扫描，Spring按照前面描述的规则为未命名的组件生成bean名:本质上，采用简单的类名并将其初始字符转换为小写。
但是，在(不寻常的)特殊情况下，当有__多个字符并且第一个和第二个字符都是大写时，将保留原始的大小写__。
这些规则与java.beans.Introspector.decapitalize (Spring在这里使用的)定义的规则相同。



# 在Bean定义之外别名Bean
在bean定义本身中，可以为bean提供多个名称，方法是使用由id属性指定的最多一个名称和name属性中任意数量的其他名称的组合。这些名称可以是相同bean的等价别名，并且在某些情况下很有用，例如让应用程序中的每个组件通过使用特定于该组件本身的bean名称来引用公共依赖项

但是，在实际定义bean的地方指定所有别名并不总是足够的。有时需要为在其他地方定义的bean引入别名。这在大型系统中很常见，其中配置在每个子系统之间被分割，每个子系统都有自己的一组对象定义。在基于xml的配置元数据中，可以使用<alias/>元素来完成此操作。下面的示例展示了如何这样做

```xml
<alias name="fromName" alias="toName"/>
```
在这种情况下，名为fromName的bean(在同一容器中)在使用此别名定义之后也可以被称为toName
例如，子系统A的配置元数据可以通过subsystemA-dataSource的名称引用数据源。子系统B的配置元数据可以通过subsystemB-dataSource的名称引用数据源。当组合使用这两个子系统的主应用程序时，主应用程序通过myApp-dataSource的名称引用数据源。要使所有三个名称都引用同一个对象，可以向配置元数据添加以下别名定义
```java
<alias name="myApp-dataSource" alias="subsystemA-dataSource"/>
<alias name="myApp-dataSource" alias="subsystemB-dataSource"/>
```
现在，每个组件和主应用程序都可以通过唯一的名称来引用数据源，并且保证不会与任何其他定义冲突(实际上创建了一个名称空间)，但它们引用的是同一个bean



# Java-configuration
如果使用Java Configuration，则可以使用@Bean注解来提供别名。有关详细信息，请参见使用@Bean注解。



# 实例化Bean
bean定义本质上是创建一个或多个对象的方法。容器在被请求时查看指定bean的构成元素，并使用由该bean定义封装的配置元数据来创建(或获取)实际对象

如果使用基于xml的配置元数据，则可以在<bean/>元素的class属性中指定要实例化的对象的类型(或类)。这个类属性(在内部，它是BeanDefinition实例上的class属性)通常是强制性的。
(对于异常，请参见使用实例工厂方法进行实例化和Bean定义继承。)您可以通过以下两种方式之一使用Class属性

- 通常，在容器本身通过反射地调用其构造函数直接创建bean的情况下，指定要构造的bean类，这在某种程度上相当于带有new操作符的Java代码
- 指定包含被调用来创建对象的静态工厂方法的实际类，在不太常见的情况下，容器调用类上的静态工厂方法来创建bean。调用静态工厂方法返回的对象类型可以是同一个类，也可以完全是另一个类



# 嵌套类名
如果您想为嵌套类配置bean定义，您可以使用二进制名称或嵌套类的源名称。 例如，如果你有一个名为SomeThing的类。这个SomeThing类有一个名为OtherThing的静态嵌套类，它们可以用美元符号($)或点(.)分隔。
因此，bean定义中的类属性的值将是com.example.$OtherThing或com.example.SomeThing.OtherThing



# 使用构造器实例化Bean
当您通过构造函数方法创建bean时，所有普通类都可以被Spring使用并与之兼容。也就是说，正在开发的类不需要实现任何特定的接口，也不需要以特定的方式编码。只需指定bean类就足够了。但是，根据您为特定bean使用的IoC类型，您可能需要一个默认(空)构造函数。

Spring IoC容器实际上可以管理您希望它管理的任何类。它并不局限于管理真正的JavaBeans。大多数Spring用户更喜欢实际的JavaBeans，只有一个__默认的(无参数的)构造函数和适当的setter和getter方法__，这些都是按照容器中的属性建模的。您还可以在容器中拥有更多非bean样式的类。例如，如果您需要使用一个完全不符合JavaBean规范的遗留连接池，Spring也可以管理它
使用基于xml的配置元数据，您可以按照如下方式指定bean类：

```java
<bean id="exampleBean" class="examples.ExampleBean"/>

<bean name="anotherExample" class="examples.ExampleBeanTwo"/>
```
有关在构造对象后向构造函数提供参数(如果需要)和设置对象实例属性的机制的详细信息，请参见注入依赖项



# 使用静态工厂类实例化
在定义使用静态工厂方法创建的bean时，请使用class属性指定包含静态工厂方法的类，并使用名为factory-method的属性指定工厂方法本身的名称。您应该能够调用该方法(使用可选参数，稍后将介绍)并返回一个活动对象，随后将其视为通过构造函数创建的对象。这种bean定义的一个用途是在遗留代码中调用静态工厂

下面的bean定义指定将通过调用工厂方法创建bean。定义没有指定返回对象的类型(类)，而是指定包含工厂方法的类。在本例中，__createInstance()方法必须是静态方法__。下面的例子展示了如何指定一个工厂方法:

```xml
<bean id="clientService"
	class="examples.ClientService"
	factory-method="createInstance"/>
```
下面的示例展示了一个可以使用前面的bean定义的类:
```java
public class ClientService {
	private static ClientService clientService = new ClientService();
	private ClientService() {}

	public static ClientService createInstance() {
		return clientService;
	}
}
```
有关向工厂方法提供(可选)参数和在对象从工厂返回后设置对象实例属性的机制的详细信息，请参阅详细的依赖项和配置。



# 使用实例工厂方法进行实例化
与通过静态工厂方法进行实例化类似，使用实例工厂方法进行实例化会从容器中调用现有bean的非静态方法来创建新bean。
要使用此机制，请将类属性保留为空，并在factory-bean属性中指定当前(或父或祖先)容器中的bean的名称，该容器包含将被调用以创建对象的实例方法。使用factory-method属性设置工厂方法本身的名称。下面的示例展示了如何配置这样一个bean

```xml
<!-- the factory bean, which contains a method called createClientServiceInstance() -->
<bean id="serviceLocator" class="examples.DefaultServiceLocator">
	<!-- inject any dependencies required by this locator bean -->
</bean>

<!-- the bean to be created via the factory bean -->
<bean id="clientService"
	factory-bean="serviceLocator"
	factory-method="createClientServiceInstance"/>
```
下面的示例显示了相应的类:
```java
public class DefaultServiceLocator {

	private static ClientService clientService = new ClientServiceImpl();

	public ClientService createClientServiceInstance() {
		return clientService;
	}
}
```
一个工厂类也可以包含多个工厂方法，如下面的示例所示:
```java
<bean id="serviceLocator" class="examples.DefaultServiceLocator">
	<!-- inject any dependencies required by this locator bean -->
</bean>

<bean id="clientService"
	factory-bean="serviceLocator"
	factory-method="createClientServiceInstance"/>

<bean id="accountService"
	factory-bean="serviceLocator"
	factory-method="createAccountServiceInstance"/>
```
下面的示例显示了相应的类:
```java
public class DefaultServiceLocator {

	private static ClientService clientService = new ClientServiceImpl();

	private static AccountService accountService = new AccountServiceImpl();

	public ClientService createClientServiceInstance() {
		return clientService;
	}

	public AccountService createAccountServiceInstance() {
		return accountService;
	}
}
```
这种方法表明，工厂bean本身可以通过依赖项注入(DI)进行管理和配置。参见详细的依赖关系和配置

在Spring文档中，“factory bean”指的是在Spring容器中配置的bean，它通过实例或静态工厂方法创建对象。
相比之下，FactoryBean(注意大写)指的是特定于spring的FactoryBean实现类



# 确定Bean的运行时类型
确定特定bean的运行时类型非常重要。bean元数据定义中指定的类只是一个初始类引用，可能与声明的工厂方法结合在一起，或者作为一个FactoryBean类，这可能导致bean的不同运行时类型，或者在实例级工厂方法的情况下根本不设置(通过指定的工厂bean名称解析)。

另外，AOP代理可以用基于接口的代理来包装bean实例，该代理对目标bean的实际类型(只是其实现的接口)的暴露有限。

了解特定bean的实际运行时类型的推荐方法是使用BeanFactory。为指定的bean名称调用getType。这将考虑上述所有情况，并返回BeanFactory所使用的对象类型。getBean调用将返回相同的bean名称