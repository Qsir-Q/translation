# 方法注入
在大多数应用程序场景中，容器中的大多数bean都是单例的。当一个单例bean需要与另一个单例bean协作，或者一个非单例bean需要与另一个非单例bean协作时，通常通过将一个bean定义为另一个bean的属性来处理依赖关系。
当bean的生命周期不同时，问题就出现了。假设单例bean A需要使用非单例(原型)bean B，可能是在A上的每个方法调用上。容器只创建单例bean A一次，因此只有一次设置属性的机会。容器不能在每次需要bean B的实例时都为bean A提供新的实例

一个解决方案是放弃控制反转。您可以通过实现ApplicationContextAware接口，并在bean A每次需要时对容器请求(通常是新的)bean B实例进行getBean(“B”)调用，使bean A意识到容器。
下面的示例展示了这种方法：

```java
package fiona.apple;

// Spring-API imports
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;

/**
 * A class that uses a stateful Command-style class to perform
 * some processing.
 */
public class CommandManager implements ApplicationContextAware {

	private ApplicationContext applicationContext;

	public Object process(Map commandState) {
		// grab a new instance of the appropriate Command
		Command command = createCommand();
		// set the state on the (hopefully brand new) Command instance
		command.setState(commandState);
		return command.execute();
	}

	protected Command createCommand() {
		// notice the Spring API dependency!
		return this.applicationContext.getBean("command", Command.class);
	}

	public void setApplicationContext(
			ApplicationContext applicationContext) throws BeansException {
		this.applicationContext = applicationContext;
	}
}
```
前面的情况是不可取的，因为业务代码知道Spring框架并与之耦合。 (好像大部分时间都是这么用的)
方法注入是Spring IoC容器的一个高级特性，它允许您干净地处理这个用例



# Lookup 方法注入
Lookup方法注入是容器覆盖容器管理bean上的方法并返回容器中另一个命名bean的查找结果的能力。
查找通常涉及一个原型bean，就像前一节描述的场景一样。__Spring框架通过使用来自CGLIB库的字节码生成来动态生成覆盖该方法的子类来实现此方法注入。 (AOP的一种应用方式)__

- 要使这个动态子类工作，Spring bean容器子类所包含的类不能是final的，要覆盖的方法也不能是final的。
- 对具有抽象方法的类进行单元测试需要您自己创建该类的子类，并提供抽象方法的存根实现。
- 具体方法对于组件扫描也是必要的，这需要具体类来拾取
- 另一个关键的限制是，查找方法不能与工厂方法一起工作，特别是不能与配置类中的@Bean方法一起工作，因为在这种情况下，容器不负责创建实例，因此不能动态地创建运行时生成的子类。

对于前面代码片段中的CommandManager类，Spring容器动态地覆盖createCommand()方法的实现。
CommandManager类没有任何Spring依赖项，如重新设计的示例所示：
```java
package fiona.apple;

// no more Spring imports!

public abstract class CommandManager {

	public Object process(Object commandState) {
		// grab a new instance of the appropriate Command interface
		Command command = createCommand();
		// set the state on the (hopefully brand new) Command instance
		command.setState(commandState);
		return command.execute();
	}

	// okay... but where is the implementation of this method?
	protected abstract Command createCommand();
}
```
在包含要注入的方法(本例中是CommandManager)的客户端类中，要注入的方法需要以下形式的签名:
```XML
<public|protected> [abstract] <return-type> theMethodName(no-arguments);
```
如果方法是抽象的，则动态生成的子类实现该方法。否则，动态生成的子类将覆盖在原始类中定义的具体方法。考虑下面的例子
```XML
<!-- a stateful bean deployed as a prototype (non-singleton) -->
<bean id="myCommand" class="fiona.apple.AsyncCommand" scope="prototype">
	<!-- inject dependencies here as required -->
</bean>

<!-- commandProcessor uses statefulCommandHelper -->
<bean id="commandManager" class="fiona.apple.CommandManager">
	<lookup-method name="createCommand" bean="myCommand"/>
</bean>
```
标识为commandManager的bean在需要myCommand bean的新实例时调用它自己的createCommand()方法。
如果确实需要myCommand bean，则必须谨慎地将其部署为原型。如果它是单例，则每次都返回myCommand bean的相同实例

或者，在基于注解的组件模型中，你可以通过@Lookup注解声明一个查找方法，如下面的例子所示:
```java
public abstract class CommandManager {

	public Object process(Object commandState) {
		Command command = createCommand();
		command.setState(commandState);
		return command.execute();
	}

	@Lookup("myCommand")
	protected abstract Command createCommand();
}
```
或者，更习惯地说，您可以依赖于目标bean根据查找方法的声明返回类型进行解析:
```java
public abstract class CommandManager {

	public Object process(Object commandState) {
		Command command = createCommand();
		command.setState(commandState);
		return command.execute();
	}

	@Lookup
	protected abstract Command createCommand();
}
```
请注意，通常应该使用具体的存根实现来声明这种带注解的查找方法，以便它们与Spring的组件扫描规则兼容，其中抽象类在默认情况下被忽略。此限制不适用于显式注册或显式导入的bean类。

> 访问不同作用域目标bean的另一种方法是ObjectFactory/ Provider注入点。请参阅作为依赖的作用域bean。
> 您可能还会发现ServiceLocatorFactoryBean(在org.springframework.beans.factory.config包中)非常有用。



# 任意方法替换

与查找方法注入相比，方法注入的一种不太有用的形式是能够用另一种方法实现替换托管bean中的任意方法。
在实际需要此功能之前，您可以跳过本节的其余部分。

对于基于xml的配置元数据，您可以使用已替换方法元素，将已部署bean的现有方法实现替换为另一个。
考虑下面的类，它有一个我们想要覆盖的名为computeValue的方法:
```java
public class MyValueCalculator {

	public String computeValue(String input) {
		// some real code...
	}

	// some other methods...
}
```
org.springframework.beans.factory.support.MethodReplacer 接口的类提供了新的方法定义，如下面的示例所示:
```java
/**
 * meant to be used to override the existing computeValue(String)
 * implementation in MyValueCalculator
 */
public class ReplacementComputeValue implements MethodReplacer {

	public Object reimplement(Object o, Method m, Object[] args) throws Throwable {
		// get the input value, work with it, and return a computed result
		String input = (String) args[0];
		...
		return ...;
	}
}
```
部署原始类并指定方法覆盖的bean定义类似于以下示例:
```java
<bean id="myValueCalculator" class="x.y.z.MyValueCalculator">
	<!-- arbitrary method replacement -->
	<replaced-method name="computeValue" replacer="replacementComputeValue">
		<arg-type>String</arg-type>
	</replaced-method>
</bean>

<bean id="replacementComputeValue" class="a.b.c.ReplacementComputeValue"/>
```
您可以在<replace -method/>元素中使用一个或多个<arg-type/>元素来指示被覆盖的方法的方法签名。
只有当方法被重载并且类中存在多个变体时，参数的签名才是必需的。为方便起见，参数的类型字符串可以是完全限定类型名称的子字符串。例如，以下都匹配java.lang.String

1. java.lang.String
2. String
3. Str

因为参数的数量通常足以区分每种可能的选择，所以这种快捷方式可以节省大量的输入，因为它允许您只键入与参数类型匹配的最短字符串。