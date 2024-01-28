### Bean定义继承
bean定义可以包含大量配置信息，包括构造函数参数、属性值和特定于容器的信息，例如初始化方法、静态工厂方法名称等。子bean定义从父定义继承配置数据。子定义可以根据需要覆盖某些值或添加其他值。使用父bean和子bean定义可以节省大量的键入工作。实际上，这是一种模板形式。

如果您以编程方式使用ApplicationContext接口，则子bean定义由ChildBeanDefinition类表示。大多数用户不会在这个级别上使用它们。相反，它们在类(如ClassPathXmlApplicationContext)中声明式地配置bean定义。
当您使用基于xml的配置元数据时，您可以通过使用父属性来指示子bean定义，并将父bean指定为该属性的值。
下面的例子展示了如何这样做:

```xml
<bean id="inheritedTestBean" abstract="true"
		class="org.springframework.beans.TestBean">
	<property name="name" value="parent"/>
	<property name="age" value="1"/>
</bean>

<bean id="inheritsWithDifferentClass"
		class="org.springframework.beans.DerivedTestBean"
		parent="inheritedTestBean" init-method="initialize">
	<property name="name" value="override"/>
	<!-- the age property value of 1 will be inherited from parent -->
</bean>
```

如果没有指定，则子bean定义使用来自父定义的bean类，但也可以覆盖它。在后一种情况下，子bean类必须与父bean类兼容(也就是说，它必须接受父bean类的属性值)。

子bean定义继承父bean的范围、构造函数参数值、属性值和方法覆盖，并具有添加新值的选项。指定的任何作用域、初始化方法、销毁方法或静态工厂方法设置都将覆盖相应的父设置。

其余的设置总是取自子定义: depends on、autowire mode、dependency check、singleton和 lazy init。

前面的示例通过使用抽象属性显式地将父bean定义标记为抽象。如果父定义没有指定类，则需要显式地将父bean定义标记为抽象，如下面的示例所示:
```xml
<bean id="inheritedTestBeanWithoutClass" abstract="true">
	<property name="name" value="parent"/>
	<property name="age" value="1"/>
</bean>

<bean id="inheritsWithClass" class="org.springframework.beans.DerivedTestBean"
		parent="inheritedTestBeanWithoutClass" init-method="initialize">
	<property name="name" value="override"/>
	<!-- age will inherit the value of 1 from the parent bean definition-->
</bean>
```

父bean不能单独实例化，因为它是不完整的，并且它也被显式地标记为抽象。当定义是抽象的时，它只能作为纯模板bean定义使用，作为子定义的父定义。试图单独使用这样的抽象父bean，通过将其作为另一个bean的ref属性引用或使用父bean ID进行显式getBean()调用将返回错误。类似地，容器的内部preinstantiatesingleton()方法会忽略定义为抽象的bean定义

ApplicationContext默认预实例化所有的单例。因此，重要的是(至少对于单例bean)，如果您有一个(父)bean定义，您打算仅将其用作模板，并且该定义指定了一个类，则必须确保将抽象属性设置为true，否则应用程序上下文将实际(尝试)预实例化抽象bean。