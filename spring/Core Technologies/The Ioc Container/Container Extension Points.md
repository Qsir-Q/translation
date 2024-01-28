# 容器拓展点
通常，应用程序开发人员不需要创建ApplicationContext实现类的子类。
相反，可以通过插入特殊集成接口的实现来扩展Spring IoC容器。接下来的几节将描述这些集成接口。



# 使用BeanPostProcessor定制bean

BeanPostProcessor接口定义了回调方法，您可以实现这些方法来提供您自己的(或覆盖容器的默认值)实例化逻辑、依赖项解析逻辑等等。如果您想在Spring容器完成实例化、配置和初始化bean之后实现一些自定义逻辑，您可以插入一个或多个自定义BeanPostProcessor实现。

您可以配置多个BeanPostProcessor实例，并且可以通过设置order属性来控制这些BeanPostProcessor实例运行的顺序。只有当BeanPostProcessor实现了Ordered接口时，你才能设置这个属性。如果编写自己的BeanPostProcessor，也应该考虑实现Ordered接口。要了解更多细节，请参阅BeanPostProcessor和Ordered接口的javadoc。参见BeanPostProcessor实例的程序化注册说明。

BeanPostProcessor实例对bean(或对象)实例进行操作。也就是说，Spring IoC容器实例化一个bean实例，然后BeanPostProcessor实例完成它们的工作。

BeanPostProcessor实例的作用域为每个容器。这只有在使用容器层次结构时才有意义。如果在一个容器中定义了BeanPostProcessor，则它只对该容器中的bean进行后处理。换句话说，在一个容器中定义的bean不会被在另一个容器中定义的BeanPostProcessor进行后处理，即使两个容器都是同一层次结构的一部分。

__要更改实际的bean定义(即定义bean的元数据)，您需要使用BeanFactoryPostProcessor__，如使用BeanFactoryPostProcessor自定义配置元数据中所述。

beanpostprocessor接口由两个回调方法组成。当这样的类被注册为容器的后处理程序时，对于容器创建的每个bean实例，后置处理程序在容器初始化方法(如InitializingBean.afterPropertiesSet()或任何声明的init方法)被调用之前以及在任何bean初始化回调之后都会从容器获得回调。后置处理器可以对bean实例采取任何操作，包括完全忽略回调。bean后处理器通常检查回调接口，或者它可能用代理包装bean。为了提供代理包装逻辑，一些Spring AOP基础设施类被实现为bean后处理器。

ApplicationContext自动检测在实现BeanPostProcessor接口的配置元数据中定义的任何bean。ApplicationContext将这些bean注册为后置处理器，以便稍后在创建bean时调用它们。Bean后置处理器可以以与任何其他Bean相同的方式部署在容器中。

注意，__当通过在配置类上使用@Bean工厂方法声明BeanPostProcessor时，工厂方法的返回类型应该是实现类本身，或者至少是org.springframework.beans.factory.config.BeanPostProcessor接口__，清楚地指示该bean的后处理器性质。否则，ApplicationContext不能在完全创建它之前按类型自动检测它。由于需要尽早实例化BeanPostProcessor以便应用于上下文中其他bean的初始化，因此这种早期类型检测至关重要。

实现BeanPostProcessor接口的类是特殊的，并且被容器以不同的方式对待。所有BeanPostProcessor实例和它们直接引用的bean在启动时被实例化，作为ApplicationContext特殊启动阶段的一部分。接下来，以排序的方式注册所有BeanPostProcessor实例，并将其应用于容器中的所有其他bean。因为AOP自动代理是作为BeanPostProcessor本身实现的，__所以BeanPostProcessor实例和它们直接引用的bean都不适合自动代理__，因此，没有将aspect编织到它们中。对于任何这样的bean，您应该看到一条信息日志消息:bean someBean不适合被所有BeanPostProcessor接口处理(例如:不适合自动代理)。

如果通过使用自动装配或@Resource(可能会回到自动装配)将bean连接到BeanPostProcessor中，那么在搜索类型匹配依赖候选项时，Spring可能会访问意外的bean，从而使它们不适合自动代理或其他类型的bean后处理。
例如，如果您有一个带有@Resource注解的依赖项，其中字段或setter名称不直接对应于bean声明的名称，并且没有使用name属性，Spring将访问其他bean以按类型匹配它们。

下面的例子展示了如何在ApplicationContext中编写、注册和使用BeanPostProcessor实例。



# 示例:Hello World, BeanPostProcessor方式

第一个示例说明了基本用法。该示例显示了一个自定义BeanPostProcessor实现，该实现在容器创建每个bean时调用toString()方法，并将结果字符串打印到系统控制台。下面的示例显示了自定义BeanPostProcessor实现类定义:
```java
package scripting;

import org.springframework.beans.factory.config.BeanPostProcessor;

public class InstantiationTracingBeanPostProcessor implements BeanPostProcessor {

	// simply return the instantiated bean as-is
	public Object postProcessBeforeInitialization(Object bean, String beanName) {
		return bean; // we could potentially return any object reference here...
	}

	public Object postProcessAfterInitialization(Object bean, String beanName) {
		System.out.println("Bean '" + beanName + "' created : " + bean.toString());
		return bean;
	}
}
```
下面的bean元素使用了InstantiationTracingBeanPostProcessor:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:lang="http://www.springframework.org/schema/lang"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/lang
		https://www.springframework.org/schema/lang/spring-lang.xsd">

	<lang:groovy id="messenger"
			script-source="classpath:org/springframework/scripting/groovy/Messenger.groovy">
		<lang:property name="message" value="Fiona Apple Is Just So Dreamy."/>
	</lang:groovy>

	<!--
	when the above bean (messenger) is instantiated, this custom
	BeanPostProcessor implementation will output the fact to the system console
	-->
	<bean class="scripting.InstantiationTracingBeanPostProcessor"/>

</beans>
```
注意InstantiationTracingBeanPostProcessor是如何定义的。它甚至没有名称，而且因为它是一个bean，所以可以像其他bean一样对它进行依赖注入。(前面的配置还定义了一个由Groovy脚本支持的bean。Spring的动态语言支持详见“动态语言支持”一章。

下面的Java应用程序运行上述代码和配置:
```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import org.springframework.scripting.Messenger;

public final class Boot {

	public static void main(final String[] args) throws Exception {
		ApplicationContext ctx = new ClassPathXmlApplicationContext("scripting/beans.xml");
		Messenger messenger = ctx.getBean("messenger", Messenger.class);
		System.out.println(messenger);
	}

}
```
上述应用程序的输出类似于以下内容:
```txt
Bean 'messenger' created : org.springframework.scripting.groovy.GroovyMessenger@272961
org.springframework.scripting.groovy.GroovyMessenger@272961
```



# 示例: The AutowiredAnnotationBeanPostProcessor

将回调接口或注解与自定义BeanPostProcessor实现结合使用是扩展Spring IoC容器的常用方法。一个例子是Spring的AutowiredAnnotationBeanPostProcessor——一个随Spring自带的BeanPostProcessor实现，它可以自动连接带注解的字段、setter方法和任意配置方法。



# 使用BeanFactoryPostProcessor自定义配置元数据
我们要看的下一个扩展点是org.springframework.beans.factory.config.BeanFactoryPostProcessor。
__这个接口的语义与BeanPostProcessor类似，但有一个主要区别:BeanFactoryPostProcessor在bean配置元数据上操作__
也就是说，Spring IoC容器允许BeanFactoryPostProcessor读取配置元数据，并可能在容器实例化除BeanFactoryPostProcessor实例之外的任何bean之前更改它。

您可以配置多个BeanFactoryPostProcessor实例，并且可以通过设置order属性来控制这些BeanFactoryPostProcessor实例运行的顺序。但是，只有当BeanFactoryPostProcessor实现了Ordered接口时，您才能设置此属性。如果编写自己的BeanFactoryPostProcessor，也应该考虑实现Ordered接口。
有关更多细节，请参阅BeanFactoryPostProcessor和Ordered接口的javadoc。

如果希望更改实际的bean实例(即从配置元数据创建的对象)，则需要使用BeanPostProcessor(前面在使用BeanPostProcessor定制bean中描述过)。虽然在技术上可以在BeanFactoryPostProcessor中使用bean实例(例如，通过使用BeanFactory.getBean())，但这样做会导致过早的bean实例化，违反标准的容器生命周期。
这可能会导致负面的副作用，比如绕过bean的后期处理。

此外，BeanFactoryPostProcessor实例的作用域为每个容器。这只有在使用容器层次结构时才有意义。如果在一个容器中定义了BeanFactoryPostProcessor，则它只应用于该容器中的bean定义。一个容器中的Bean定义不会被另一个容器中的BeanFactoryPostProcessor实例进行后处理，即使两个容器是同一层次结构的一部分。

当在ApplicationContext中声明bean工厂后处理器时，它会自动运行，以便将更改应用于定义容器的配置元数据。Spring包括许多预定义的bean工厂后处理器，如

1. PropertyOverrideConfigurer
2. PropertySourcesPlaceholderConfigurer。

您还可以使用自定义BeanFactoryPostProcessor—例如，注册自定义属性编辑器。

ApplicationContext自动检测部署到其中实现BeanFactoryPostProcessor接口的任何bean。
它在适当的时候将这些bean用作bean工厂后处理器。您可以像部署任何其他bean一样部署这些后处理器bean。

与BeanPostProcessors一样，您通常不希望将BeanFactoryPostProcessors配置为延迟初始化。
如果没有其他bean引用bean (Factory)PostProcessor，则该后处理器根本不会被实例化。
因此，将其标记为延迟初始化将被忽略，并且Bean(Factory)PostProcessor将被急切地实例化，即使您在<beans />元素的声明中将default-lazy-init属性设置为true。



# 示例:类名替换PropertySourcesPlaceholderConfigurer

您可以使用 PropertySourcesPlaceholderConfigurer 通过使用标准 Java Properties 格式将bean定义中的属性值外部化到单独的文件中。这样做使部署应用程序的人员能够自定义特定于环境的属性，例如数据库url和密码，而无需修改容器的主XML定义文件的复杂性或风险。

考虑以下基于xml的配置元数据片段，其中定义了带有占位符值的DataSource:
```xml
<bean class="org.springframework.context.support.PropertySourcesPlaceholderConfigurer">
	<property name="locations" value="classpath:com/something/jdbc.properties"/>
</bean>

<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
	<property name="driverClassName" value="${jdbc.driverClassName}"/>
	<property name="url" value="${jdbc.url}"/>
	<property name="username" value="${jdbc.username}"/>
	<property name="password" value="${jdbc.password}"/>
</bean>
```
该示例显示了从外部properties文件配置的属性。在运行时，将PropertySourcesPlaceholderConfigurer应用于替换数据源的某些属性的元数据。要替换的值被指定为${property-name}形式的占位符，它遵循Ant和log4j以及JSP EL样式。

实际值来自标准Java属性格式的另一个文件:
```xml
jdbc.driverClassName=org.hsqldb.jdbcDriver
jdbc.url=jdbc:hsqldb:hsql://production:9002
jdbc.username=sa
jdbc.password=root
```
因此，${jdbc。Username} string在运行时被替换为值'sa'，对于与属性文件中的键匹配的其他占位符值也是如此。PropertySourcesPlaceholderConfigurer检查bean定义的大多数属性和属性中的占位符。此外，还可以自定义占位符前缀和后缀。

使用Spring 2.5中引入的上下文命名空间，您可以使用专用的配置元素配置属性占位符。
您可以在location属性中以逗号分隔的列表形式提供一个或多个位置，如下例所示:
```xml
<context:property-placeholder location="classpath:com/something/jdbc.properties"/>
```
PropertySourcesPlaceholderConfigurer不仅在您指定的properties文件中查找属性。默认情况下，如果在指定的属性文件中找不到属性，它将检查Spring Environment属性和常规Java系统属性。

对于给定的具有其所需属性的应用程序，只应该定义一个这样的元素。可以配置多个属性占位符，只要它们具有不同的占位符语法(${…})。

如果您需要模块化用于替换的属性源，则不应该创建多个属性占位符。相反，您应该创建自己的PropertySourcesPlaceholderConfigurer bean来收集要使用的属性。

您可以使用PropertySourcesPlaceholderConfigurer来替换类名，
当您必须在运行时选择特定的实现类时，这有时很有用。下面的例子展示了如何这样做:
```xml
<bean class="org.springframework.beans.factory.config.PropertySourcesPlaceholderConfigurer">
	<property name="locations">
		<value>classpath:com/something/strategy.properties</value>
	</property>
	<property name="properties">
		<value>custom.strategy.class=com.something.DefaultStrategy</value>
	</property>
</bean>

<bean id="serviceStrategy" class="${custom.strategy.class}"/>
```
如果不能在运行时将类解析为有效类，则在即将创建bean时解析失败，这是在非惰性初始化bean的ApplicationContext的preinstantiatesingleton()阶段。



# 示例:PropertyOverrideConfigurer

另一个bean工厂后处理程序PropertyOverrideConfigurer与PropertySourcesPlaceholderConfigurer类似，但与后者不同的是，原始定义可以为bean属性提供默认值，也可以根本没有值。如果重写的Properties文件没有某个bean属性的条目，则使用默认上下文定义。

注意，bean定义不知道被覆盖了，因此从XML定义文件中不能立即看出正在使用覆盖配置器。如果多个PropertyOverrideConfigurer实例为同一个bean属性定义了不同的值，由于覆盖机制，最后一个实例是最终生效。

属性文件配置行采用以下格式:
```properties
beanName.property=value
```

下面的清单显示了该格式的一个示例:
```properties
dataSource.driverClassName=com.mysql.jdbc.Driver
dataSource.url=jdbc:mysql:mydb
```
该示例文件可以与容器定义一起使用，该容器定义包含一个名为dataSource的bean，该bean具有驱动程序和url属性。

也支持复合属性名，只要路径的每个组件(除了被覆盖的最后一个属性)都是非空的(可能是由构造函数初始化的)。
在下面的例子中，tom bean的fred属性的bob属性的sammy属性被设置为标量值123:
```txt
tom.fred.bob.sammy=123
```

指定的覆盖值总是文字值。它们不会被翻译成bean引用。当XML bean定义中的原始值指定一个bean引用时，这种约定也适用。

随着Spring 2.5中引入的上下文命名空间，可以使用专用的配置元素来配置属性重写，如下面的示例所示:
```txt
<context:property-override location="classpath:override.properties"/>
```



# 使用FactoryBean定制实例化逻辑

您可以为本身就是工厂的对象实现org.springframework.beans.factory.FactoryBean接口。

FactoryBean接口是可插入Spring IoC容器实例化逻辑的一个点。如果您有复杂的初始化代码，用Java更好地表示，而不是(可能)冗长的XML，那么您可以创建自己的FactoryBean，在该类中编写复杂的初始化，然后将定制的FactoryBean插入容器中。

FactoryBean<T>接口提供了三个方法:
- T getObject(): 返回此工厂创建的对象的实例。实例可能是共享的，这取决于该工厂返回的是单例还是原型
- boolean isSingleton() : 如果此FactoryBean返回单例，则返回true，否则返回false。此方法的默认实现返回true。
- Class<?> getObjectType(): 返回getObject()方法返回的对象类型，如果事先不知道该类型，则返回null

FactoryBean概念和接口在Spring框架中的许多地方都有使用。超过50个FactoryBean接口的实现随Spring本身一起发布。

__当您需要向容器请求实际的FactoryBean实例本身而不是它生成的bean时，请在调用ApplicationContext的getBean()方法时，在bean的id前面加上&符号__。因此，对于id为myBean的给定FactoryBean，在容器上调用getBean(“myBean”)将返回FactoryBean的产品，而调用getBean(“&myBean”)将返回FactoryBean实例本身。