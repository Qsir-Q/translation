# 依赖项和配置细节
如前一节所述，您可以将bean属性和构造函数参数定义为对其他托管bean(协作者)的引用或内联定义的值。为此，Spring基于xml的配置元数据支持其<property/>和<constructor-arg/>元素中的子元素类型。



# 基本数据类型、字符串
<property/>元素的value属性将属性或构造函数参数指定为可读的字符串表示形式。Spring的转换服务用于将这些值从String转换为属性或参数的实际类型。下面的示例显示了正在设置的各种值

```java
<bean id="myDataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
	<!-- results in a setDriverClassName(String) call -->
	<property name="driverClassName" value="com.mysql.jdbc.Driver"/>
	<property name="url" value="jdbc:mysql://localhost:3306/mydb"/>
	<property name="username" value="root"/>
	<property name="password" value="misterkaoli"/>
</bean>
```
下面的示例使用p名称空间进行更简洁的XML配置:
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:p="http://www.springframework.org/schema/p"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
	https://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean id="myDataSource" class="org.apache.commons.dbcp.BasicDataSource"
		destroy-method="close"
		p:driverClassName="com.mysql.jdbc.Driver"
		p:url="jdbc:mysql://localhost:3306/mydb"
		p:username="root"
		p:password="misterkaoli"/>

</beans>
```
前面的XML更简洁。然而，拼写错误是在运行时发现的，而不是在设计时发现的，除非您使用在创建bean定义时支持自动属性完成的IDE(例如IntelliJ IDEA或Spring Tools for Eclipse)。强烈推荐这种IDE帮助

您还可以配置java.util.Properties实例，如下所示:
```xml
<bean id="mappings"
	class="org.springframework.context.support.PropertySourcesPlaceholderConfigurer">

	<!-- typed as a java.util.Properties -->
	<property name="properties">
		<value>
			jdbc.driver.className=com.mysql.jdbc.Driver
			jdbc.url=jdbc:mysql://localhost:3306/mydb
		</value>
	</property>
</bean>
```
Spring容器通过使用JavaBeans的PropertyEditor机制，将<value/>元素中的文本转换为java.util.Properties实例。这是一个很好的快捷方式，也是Spring团队倾向于使用嵌套<value/>元素而不是value属性样式的少数几个地方之一



# idref 元素
idref元素只是将容器中另一个bean的id(字符串值，而不是引用)传递给<constructor-arg/>或<property/>元素的一种防错误方法。下面的示例展示了如何使用它
```xml
<bean id="theTargetBean" class="..."/>

<bean id="theClientBean" class="...">
	<property name="targetName">
		<idref bean="theTargetBean"/>
	</property>
</bean>
```
前面的bean定义代码段(在运行时)与下面的代码段完全等价:
```xml
<bean id="theTargetBean" class="..." />

<bean id="client" class="...">
	<property name="targetName" value="theTargetBean"/>
</bean>
```
第一种形式比第二种形式更可取，因为使用idref标记可以让容器在部署时验证所引用的命名bean是否确实存在。
在第二个变体中，对传递给客户端bean的targetName属性的值不执行任何验证。只有在实际实例化客户端bean时才会发现错别字(很可能导致致命的结果)。
如果客户端bean是一个原型bean，那么这个错别字和由此产生的异常可能只有在部署容器很久之后才会被发现

在4.0 beans XSD中不再支持idref元素上的本地属性，因为它不再通过常规bean引用提供值。
升级到4.0模式时，将现有的idref local引用更改为idref bean

<idref/>元素带来价值的一个常见地方(至少在Spring 2.0之前的版本中)是在ProxyFactoryBean bean定义中的AOP拦截器配置中。
当你指定拦截器名称时，使用<idref/>元素可以防止你拼错拦截器ID



# 对其他bean的引用(合作者)
ref元素是<constructor-arg/>或<property/>定义元素中的最后一个元素。在这里，您将bean的指定属性的值设置为对容器管理的另一个bean(协作者)的引用。被引用的bean是要设置其属性的bean的依赖项，并且在设置属性之前根据需要对其进行初始化。(如果合作者是一个单例bean，它可能已经被容器初始化了。)所有引用最终都是对另一个对象的引用。作用域和验证取决于是否通过bean或父属性指定其他对象的ID或名称

通过<ref/>标记的bean属性指定目标bean是最通用的形式，允许在相同容器或父容器中创建对任何bean的引用，而不管它是否在相同的XML文件中。
bean属性的值可以与目标bean的id属性相同，也可以与目标bean的name属性中的值之一相同。
下面的例子展示了如何使用ref元素：

```xml
<ref bean="someBean"/>
```
通过parent属性指定目标bean将创建对当前容器的父容器中的bean的引用。父属性的值可以与目标bean的id属性相同，也可以与目标bean的name属性中的值之一相同。目标bean必须位于当前bean的父容器中。
您应该主要在以下情况下使用此bean引用变体:
您有一个容器层次结构，并且您希望将现有bean包装在具有与父bean相同名称的代理的父容器中。
下面的两个清单显示了如何使用parent属性:

```xml
<!-- in the parent context -->
<bean id="accountService" class="com.something.SimpleAccountService">
	<!-- insert dependencies as required here -->
</bean>
```

```xml
<!-- in the child (descendant) context -->
<bean id="accountService" <!-- bean name is the same as the parent bean -->
	class="org.springframework.aop.framework.ProxyFactoryBean">
	<property name="target">
		<ref parent="accountService"/> <!-- notice how we refer to the parent bean -->
	</property>
	<!-- insert other configuration and dependencies as required here -->
</bean>
```



# 内部Bean

<property/>或<constructor-arg/>元素中的<bean/>元素定义了一个内部bean，如下面的示例所示：
```xml
<bean id="outer" class="...">
	<!-- instead of using a reference to a target bean, simply define the target bean inline -->
	<property name="target">
		<bean class="com.example.Person"> <!-- this is the inner bean -->
			<property name="name" value="Fiona Apple"/>
			<property name="age" value="25"/>
		</bean>
	</property>
</bean>
```
内部bean定义不需要定义的ID或名称。如果指定了该值，则容器不使用该值作为标识符。容器还在创建时忽略作用域标志，因为内部bean始终是匿名的，并且始终与外部bean一起创建。不可能独立访问内部bean，也不可能将它们注入到协作bean中，而不是注入到封闭bean中

作为一种极端情况，可以从自定义作用域接收销毁回调——例如，对于包含在单例bean中的请求作用域的内部bean。内部bean实例的创建与它的包含bean绑定在一起，但是销毁回调允许它参与请求范围的生命周期。
这种情况并不常见。内部bean通常只是共享其包含bean的作用域



# 集合
<list/>、<set/>、<map/>和<props/>元素分别设置Java Collection类型list、set、map和properties的属性和参数。

下面的示例展示了如何使用它们：

```xml
<bean id="moreComplexObject" class="example.ComplexObject">
	<!-- results in a setAdminEmails(java.util.Properties) call -->
	<property name="adminEmails">
		<props>
			<prop key="administrator">administrator@example.org</prop>
			<prop key="support">support@example.org</prop>
			<prop key="development">development@example.org</prop>
		</props>
	</property>
	<!-- results in a setSomeList(java.util.List) call -->
	<property name="someList">
		<list>
			<value>a list element followed by a reference</value>
			<ref bean="myDataSource" />
		</list>
	</property>
	<!-- results in a setSomeMap(java.util.Map) call -->
	<property name="someMap">
		<map>
			<entry key="an entry" value="just some string"/>
			<entry key="a ref" value-ref="myDataSource"/>
		</map>
	</property>
	<!-- results in a setSomeSet(java.util.Set) call -->
	<property name="someSet">
		<set>
			<value>just some string</value>
			<ref bean="myDataSource" />
		</set>
	</property>
</bean>
```
映射键或值或集合值的值也可以是下列任何元素: bean | ref | idref | list | set | map | props | value | null



# 集合合并

Spring容器还支持合并集合。应用程序开发人员可以定义父元素<list/>、<map/>、<set/>或<props/>，并让子元素<list/>、<map/>、<set/>或<props/>继承和覆盖父元素集合的值。也就是说，子集合的值是合并父集合和子集合元素的结果，子集合元素覆盖父集合中指定的值

关于合并的这一节讨论了父子bean机制。不熟悉父bean和子bean定义的读者可能希望在继续之前阅读相关部分
下面的示例演示集合合并：
```java
<beans>
	<bean id="parent" abstract="true" class="example.ComplexObject">
		<property name="adminEmails">
			<props>
				<prop key="administrator">administrator@example.com</prop>
				<prop key="support">support@example.com</prop>
			</props>
		</property>
	</bean>
	<bean id="child" parent="parent">
		<property name="adminEmails">
			<!-- the merge is specified on the child collection definition -->
			<props merge="true">
				<prop key="sales">sales@example.com</prop>
				<prop key="support">support@example.co.uk</prop>
			</props>
		</property>
	</bean>
<beans>
```

注意，在子bean定义的adminEmails属性的<props/>元素上使用了merge=true属性。当子bean被容器解析并实例化时，结果实例有一个adminemail Properties集合，其中包含子bean的adminemail集合与父bean的adminemail集合合并的结果。
下面的清单显示了结果:

```properties
administrator=administrator@example.com
sales=sales@example.com
support=support@example.co.uk
```
子属性集合的值集继承了父属性<props/>的所有属性元素，并且子属性的支持值覆盖了父属性集合中的值

这种合并行为类似地适用于<list/>、<map/>和<set/>集合类型。在<list/>元素的特定情况下，与list集合类型相关联的语义(即值的有序集合的概念)得到维护。父级列表的值位于所有子级列表的值之前。对于Map、Set和Properties集合类型，不存在排序。因此，对于容器内部使用的关联Map、Set和Properties实现类型底层的集合类型，没有有效的排序语义



# 集合合并的局限性
不能合并不同的集合类型(例如Map和List)。如果您尝试这样做，则会抛出一个适当的Exception。必须在较低的继承子定义上指定merge属性。在父集合定义上指定merge属性是多余的，并且不会导致所需的合并

##### 强类型的集合
由于Java对泛型类型的支持，您可以使用强类型集合。也就是说，可以声明一个Collection类型，使其只能包含(例如)String元素。如果您使用Spring将强类型集合依赖注入到bean中，那么您可以利用Spring的类型转换支持，使强类型集合实例的元素在添加到集合之前被转换为适当的类型。

下面的Java类和bean定义展示了如何做到这一点：

```xml
public class SomeClass {

	private Map<String, Float> accounts;

	public void setAccounts(Map<String, Float> accounts) {
		this.accounts = accounts;
	}
}
```

```xml
<beans>
	<bean id="something" class="x.y.SomeClass">
		<property name="accounts">
			<map>
				<entry key="one" value="9.99"/>
				<entry key="two" value="2.75"/>
				<entry key="six" value="3.99"/>
			</map>
		</property>
	</bean>
</beans>
```
当something bean的accounts属性准备注入时，关于强类型Map<String, Float>的元素类型的泛型信息可以通过反射获得。因此，Spring的类型转换基础结构将各种值元素识别为Float类型，并将字符串值(9.99、2.75和3.99)转换为实际的Float类型

##### 带有p命名空间的XML快捷方式
p名称空间允许您使用bean元素的属性(而不是嵌套的<property/>元素)来描述与bean协作的属性值，或者两者都使用。

Spring支持带有名称空间的可扩展配置格式，这些名称空间基于XML Schema定义。
本章讨论的bean配置格式是在XML Schema文档中定义的。但是，p名称空间不是在XSD文件中定义的，它只存在于Spring的核心中

下面的示例显示了解析到相同结果的两个XML片段(第一个使用标准XML格式，第二个使用p名称空间):
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:p="http://www.springframework.org/schema/p"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean name="classic" class="com.example.ExampleBean">
		<property name="email" value="someone@somewhere.com"/>
	</bean>

	<bean name="p-namespace" class="com.example.ExampleBean"
		p:email="someone@somewhere.com"/>
</beans>
```

该示例显示了bean定义中名为email的p-命名空间中的一个属性。
这告诉Spring包含一个属性声明。如前所述，p-namespace没有模式定义，因此可以将属性名称设置为属性名称

下一个示例包括另外两个bean定义，它们都有对另一个bean的引用:
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:p="http://www.springframework.org/schema/p"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean name="john-classic" class="com.example.Person">
		<property name="name" value="John Doe"/>
		<property name="spouse" ref="jane"/>
	</bean>

	<bean name="john-modern"
		class="com.example.Person"
		p:name="John Doe"
		p:spouse-ref="jane"/>

	<bean name="jane" class="com.example.Person">
		<property name="name" value="Jane Doe"/>
	</bean>
</beans>
```
这个示例不仅包括使用p名称空间的属性值，而且还使用特殊格式来声明属性引用。第一个bean定义使用<property name="spouse" ref="jane"/>创建从bean john到bean jane的引用，第二个bean定义使用p:spouse-ref="jane"作为属性来执行完全相同的操作。在本例中，spouse是属性名，而-ref部分表示这不是一个直接值，而是对另一个bean的引用

##### 带有c命名空间的XML快捷方式
与带有p命名空间的XML快捷方式类似，在Spring 3.1中引入的c命名空间允许内联属性来配置构造函数参数，而不是嵌套构造函数参数元素
下面的例子使用c:命名空间来做与基于from构造函数的依赖注入相同的事情:
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:c="http://www.springframework.org/schema/c"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean id="beanTwo" class="x.y.ThingTwo"/>
	<bean id="beanThree" class="x.y.ThingThree"/>

	<!-- traditional declaration with optional argument names -->
	<bean id="beanOne" class="x.y.ThingOne">
		<constructor-arg name="thingTwo" ref="beanTwo"/>
		<constructor-arg name="thingThree" ref="beanThree"/>
		<constructor-arg name="email" value="something@somewhere.com"/>
	</bean>

	<!-- c-namespace declaration with argument names -->
	<bean id="beanOne" class="x.y.ThingOne" c:thingTwo-ref="beanTwo"
		c:thingThree-ref="beanThree" c:email="something@somewhere.com"/>

</beans>
```
命名空间使用与p: one相同的约定(尾部-ref表示bean引用)按名称设置构造函数参数。类似地，它需要在XML文件中声明，即使它没有在XSD模式中定义(它存在于Spring核心中)。

对于构造函数参数名称不可用的极少数情况(通常是在没有调试信息的情况下编译字节码)，可以使用回退到参数索引，如下所示:
```xml
<!-- c-namespace index declaration -->
<bean id="beanOne" class="x.y.ThingOne" c:_0-ref="beanTwo" c:_1-ref="beanThree"
	c:_2="something@somewhere.com"/>
```
在实践中，构造函数解析机制在匹配参数方面非常有效，因此，除非您确实需要，我们建议在整个配置中使用名称表示法



# 复合属性名称
在设置bean属性时，可以使用复合或嵌套属性名，只要路径的所有组件(最终属性名除外)不为空即可。考虑下面的bean定义:
```xml
<bean id="something" class="things.ThingOne">
	<property name="fred.bob.sammy" value="123" />
</bean>
```
something bean有一个fred属性，fred属性有一个bob属性，bob属性有一个sammy属性，最后一个sammy属性被设置为123。
为了使它工作，在bean被构造之后，某些东西的fred属性和fred的bob属性不能为空。否则，抛出NullPointerException