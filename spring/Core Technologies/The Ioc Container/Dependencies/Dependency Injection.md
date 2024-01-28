# 依赖注入
依赖注入(DI)是一个过程，对象仅通过构造函数参数、工厂方法的参数或在对象实例构造或从工厂方法返回后在对象实例上设置的属性来定义它们的依赖项(即与它们一起工作的其他对象)。

然后，容器在创建bean时注入这些依赖项。这个过程基本上是bean本身的逆过程(因此得名为控制反转)，通过使用类的直接构造或Service Locator模式来控制其依赖项的实例化或位置。使用DI原则的代码更清晰，当对象提供依赖关系时，解耦更有效。对象不查找它的依赖项，也不知道依赖项的位置或类。因此，您的类变得更容易测试，特别是当依赖项在接口或抽象基类上时，这允许在单元测试中使用存根或模拟实现

依赖注入有两种主要的方式：**__基于构造器的依赖注入和基于setter的依赖注入__**



# 基于构造器依赖注入
基于构造器的DI是通过容器调用一个带有多个参数的构造器来实现的，每个参数代表一个依赖项。调用带有特定参数的静态工厂方法来构造bean几乎是等价的，本讨论类似地处理构造函数和静态工厂方法的参数。
下面的例子展示了一个只能通过构造函数注入进行依赖注入的类：

```java
public class SimpleMovieLister {

	// the SimpleMovieLister has a dependency on a MovieFinder
	private final MovieFinder movieFinder;

	// a constructor so that the Spring container can inject a MovieFinder
	public SimpleMovieLister(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}

	// business logic that actually uses the injected MovieFinder is omitted...
}
```
注意，这个类没有什么特别之处。它是一个POJO，不依赖于容器特定的接口、基类或注解



# 构造函数参数解析
构造函数参数解析匹配通过使用参数的类型进行。如果bean定义的构造函数参数中不存在潜在的歧义，那么在bean定义中定义构造函数参数的顺序就是在实例化bean时将这些参数提供给适当构造函数的顺序。
考虑下面的类:

```java
package x.y;

public class ThingOne {

	public ThingOne(ThingTwo thingTwo, ThingThree thingThree) {
		// ...
	}
}
```
假设ThingTwo和ThingThree类没有继承关系，就不存在潜在的歧义。
因此，下面的配置工作得很好，您不需要在<constructor-arg/>元素中显式指定构造函数参数索引或类型
```java
<beans>
	<bean id="beanOne" class="x.y.ThingOne">
		<constructor-arg ref="beanTwo"/>
		<constructor-arg ref="beanThree"/>
	</bean>

	<bean id="beanTwo" class="x.y.ThingTwo"/>

	<bean id="beanThree" class="x.y.ThingThree"/>
</beans>
```
当引用另一个bean时，类型是已知的，并且可以进行匹配(与前面的示例一样)。当使用简单类型(如<value>true</value>)时，Spring无法确定值的类型，因此在没有帮助的情况下无法按类型进行匹配。
考虑下面的类:
```java
package examples;

public class ExampleBean {

	// Number of years to calculate the Ultimate Answer
	private final int years;

	// The Answer to Life, the Universe, and Everything
	private final String ultimateAnswer;

	public ExampleBean(int years, String ultimateAnswer) {
		this.years = years;
		this.ultimateAnswer = ultimateAnswer;
	}
}
```
在上述场景中，如果使用type属性显式指定构造函数参数的类型，容器可以使用简单类型进行类型匹配，
如下面的示例所示:
```java
<bean id="exampleBean" class="examples.ExampleBean">
	<constructor-arg type="int" value="7500000"/>
	<constructor-arg type="java.lang.String" value="42"/>
</bean>
```
可以使用index属性显式指定构造函数参数的索引，如下面的示例所示:
```java
<bean id="exampleBean" class="examples.ExampleBean">
	<constructor-arg index="0" value="7500000"/>
	<constructor-arg index="1" value="42"/>
</bean>
```
除了解决多个简单值的歧义之外，指定索引还解决了构造函数具有相同类型的两个参数时的歧义，index从0开始
您还可以使用构造函数参数名来消除值的歧义，如下面的示例所示：

```java
<bean id="exampleBean" class="examples.ExampleBean">
	<constructor-arg name="years" value="7500000"/>
	<constructor-arg name="ultimateAnswer" value="42"/>
</bean>
```
请记住，要使此工作开箱即用，必须在编译代码时启用调试标志，以便Spring可以从构造函数中查找参数名称。
如果不能或不想用调试标志编译代码，可以使用JDK的@ConstructorProperties注解来显式地命名构造函数参数。
然后，样例类必须如下所示:
```java
package examples;

public class ExampleBean {

	// Fields omitted

	@ConstructorProperties({"years", "ultimateAnswer"})
	public ExampleBean(int years, String ultimateAnswer) {
		this.years = years;
		this.ultimateAnswer = ultimateAnswer;
	}
}
```



# 基于Setter方法的注入方式

基于setter的DI是由容器在__调用无参数构造函数或无参数静态工厂方法实例化bean之后调用bean上的setter方法来完成的__

下面的示例展示了一个只能通过使用纯setter注入来进行依赖注入的类。这个类是传统的Java。它是一个POJO，不依赖于容器特定的接口、基类或注解
```java
public class SimpleMovieLister {

	// the SimpleMovieLister has a dependency on the MovieFinder
	private MovieFinder movieFinder;

	// a setter method so that the Spring container can inject a MovieFinder
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}

	// business logic that actually uses the injected MovieFinder is omitted...
}
```

ApplicationContext为它管理的bean支持基于构造器和基于setter的DI。在已经通过构造函数方法注入了一些依赖项之后，它还支持基于setter的DI。您以BeanDefinition的形式配置依赖项，并将其与PropertyEditor实例结合使用，将属性从一种格式转换为另一种格式。

然而，大多数Spring用户并不直接使用这些类(即以编程方式)，而是使用XML bean定义、带注解的组件(即用@Component、@Controller等注解的类)或基于java的@Configuration类中的@Bean方法。
然后在内部将这些源转换为BeanDefinition的实例，并用于加载整个Spring IoC容器实例



# 构造器注入还是Setter方法注入
由于您可以混合使用基于构造函数和基于setter的DI，__因此对于强制依赖项使用构造函数，而对于可选依赖项使用setter方法或配置方法是一条很好的经验法则__。

注意，在setter方法上使用@Autowired注解可以使该属性成为必需的依赖项；但是，带参数编程验证的构造函数注入更可取。__Spring团队通常提倡构造函数注入，因为它允许您将应用程序组件实现为不可变对象，并确保所需的依赖项不为空__ 。 此外，构造器注入的组件总是以完全初始化的状态返回给客户端(调用)代码。

顺便说一句，大量的构造函数参数是一种不好的代码气味，这意味着类可能有太多的职责，应该重构以更好地解决适当的关注点分离问题。 

Setter注入应该主要只用于可选的依赖项，这些依赖项可以在类中被分配合理的默认值。否则，必须在代码使用依赖项的任何地方执行非空检查。setter注入的一个好处是，setter方法使该类的对象可以在以后重新配置或重新注入。因此，通过JMX MBeans进行管理是setter注入的一个引人注目的用例。 使用对特定类最有意义的DI样式。
有时，在处理没有源代码的第三方类时，会为您做出选择。例如，如果第三方类没有公开任何setter方法，那么构造函数注入可能是唯一可用的DI形式



# 依赖性解决过程
容器执行bean依赖项解析，如下所示：
- 使用描述所有bean的配置元数据创建并初始化ApplicationContext。配置元数据可以通过XML、Java代码或注解来指定
- 对于每个bean，其依赖关系以属性、构造函数参数或静态工厂方法的参数(如果使用静态工厂方法而不是普通构造函数)的形式表示。在实际创建bean时，将这些依赖项提供给bean
- 每个属性或构造函数参数都是要设置的值的实际定义，或者是对容器中另一个bean的引用
- 作为值的每个属性或构造函数参数将从其指定格式转换为该属性或构造函数参数的实际类型。

默认情况下，Spring可以将以字符串格式提供的值转换为所有内置类型，例如int、long、string、boolean等等
Spring容器在创建容器时验证每个bean的配置。但是，在实际创建bean之前，不会设置bean属性本身。
在创建容器时创建单例作用域并设置为预实例化(默认值)的bean。作用域在Bean作用域中定义。否则，仅在请求时创建bean。

bean的创建可能会导致创建一个bean图，因为bean的依赖项及其依赖项的依赖项(等等)被创建和分配。
请注意，这些依赖项之间的解析不匹配可能会很晚才出现——也就是说，在受影响bean的第一次创建时出现



# 循环依赖

如果主要使用构造函数注入，则可能会创建无法解析的循环依赖场景。
例如:类A通过构造函数注入需要类B的实例，类B通过构造函数注入需要类A的实例。如果将类A和类B配置为相互注入的bean, Spring IoC容器会在运行时检测到这个循环引用，并抛出beancurcurrentlyincreationexception。
一个__可能的解决方案是编辑一些类的源代码，让它们由setter而不是构造函数来配置__。或者，避免构造函数注入，只使用setter注入。换句话说，尽管不推荐这样做，但您可以使用setter注入配置循环依赖项。

与典型情况(没有循环依赖关系)不同，bean a和bean B之间的循环依赖关系强制其中一个bean在完全初始化之前注入另一个bean(经典的先有鸡还是先有蛋的场景)。

您通常可以相信Spring会做正确的事情。它在容器加载时检测配置问题，例如对不存在的bean的引用和循环依赖项。
Spring__尽可能晚的__设置属性并解析依赖项，直到bean实际创建完成。这意味着，如果在创建对象或其依赖项之一时出现问题，那么正确加载的Spring容器以后可以在请求对象时生成异常——例如，由于缺少或无效的属性，bean会抛出异常。

> 使用时才创建Bean

这可能会延迟一些配置问题的可见性，这就是ApplicationContext实现默认情况下预实例化单例bean的原因。
在实际需要这些bean之前，需要花费一些前期时间和内存来创建它们，在创建ApplicationContext时(而不是之后)发现配置问题。您仍然可以覆盖此默认行为，以便单例bean惰性地初始化，而不是急切地预实例化

如果不存在循环依赖项，当一个或多个协作bean被注入依赖bean时，每个协作bean在被注入依赖bean之前被完全配置。
这意味着，如果bean A依赖于bean B, Spring IoC容器将在调用bean A上的setter方法之前完全配置bean B。
换句话说，bean被实例化(如果它不是预实例化的单例)，它的依赖项被设置，并调用相关的生命周期方法(例如配置的init方法或InitializingBean回调方法)



# 依赖注入的例子
下面的示例将基于xml的配置元数据用于基于setter的DI。Spring XML配置文件的一小部分如下所示指定了一些bean定义:
```java
<bean id="exampleBean" class="examples.ExampleBean">
	<!-- setter injection using the nested ref element -->
	<property name="beanOne">
		<ref bean="anotherExampleBean"/>
	</property>

	<!-- setter injection using the neater ref attribute -->
	<property name="beanTwo" ref="yetAnotherBean"/>
	<property name="integerProperty" value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>
```
下面的例子展示了相应的ExampleBean类:
```java
public class ExampleBean {

	private AnotherBean beanOne;

	private YetAnotherBean beanTwo;

	private int i;

	public void setBeanOne(AnotherBean beanOne) {
		this.beanOne = beanOne;
	}

	public void setBeanTwo(YetAnotherBean beanTwo) {
		this.beanTwo = beanTwo;
	}

	public void setIntegerProperty(int i) {
		this.i = i;
	}
}
```
在前面的示例中，声明setter来匹配XML文件中指定的属性。下面的示例使用基于构造函数的DI:
```java
<bean id="exampleBean" class="examples.ExampleBean">
	<!-- constructor injection using the nested ref element -->
	<constructor-arg>
		<ref bean="anotherExampleBean"/>
	</constructor-arg>

	<!-- constructor injection using the neater ref attribute -->
	<constructor-arg ref="yetAnotherBean"/>

	<constructor-arg type="int" value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>
```
下面的例子展示了相应的ExampleBean类:
```java
public class ExampleBean {

	private AnotherBean beanOne;

	private YetAnotherBean beanTwo;

	private int i;

	public ExampleBean(
		AnotherBean anotherBean, YetAnotherBean yetAnotherBean, int i) {
		this.beanOne = anotherBean;
		this.beanTwo = yetAnotherBean;
		this.i = i;
	}
}
```
在bean定义中指定的构造函数参数用作ExampleBean构造函数的参数

现在考虑这个示例的一个变体，在这个变体中，Spring被告知调用一个静态工厂方法来返回对象的一个实例，而不是使用构造函数:
```java
<bean id="exampleBean" class="examples.ExampleBean" factory-method="createInstance">
	<constructor-arg ref="anotherExampleBean"/>
	<constructor-arg ref="yetAnotherBean"/>
	<constructor-arg value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>
```
下面的示例显示了相应的ExampleBean类:
```java
public class ExampleBean {

	// a private constructor
	private ExampleBean(...) {
		...
	}

	// a static factory method; the arguments to this method can be
	// considered the dependencies of the bean that is returned,
	// regardless of how those arguments are actually used.
	public static ExampleBean createInstance (
		AnotherBean anotherBean, YetAnotherBean yetAnotherBean, int i) {

		ExampleBean eb = new ExampleBean (...);
		// some other operations...
		return eb;
	}
}
```
静态工厂方法的参数由<constructor-arg/>元素提供，与实际使用构造函数完全相同。
由工厂方法返回的类的类型不必与包含静态工厂方法的类的类型相同(尽管在本例中是相同的)。
实例(非静态)工厂方法可以以本质上相同的方式使用(除了使用工厂bean属性而不是类属性)，因此我们在这里不讨论这些细节。