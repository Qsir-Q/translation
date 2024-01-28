# 使用depends-on
如果一个bean是另一个bean的依赖项，这通常意味着一个bean被设置为另一个bean的属性。通常，您可以使用基于xml的配置元数据中的<ref/>元素来完成此任务。
然而，有时bean之间的依赖关系不太直接。例如，当需要触发类中的静态初始化项时，例如数据库驱动程序注册。
depends-on属性可以显式地强制在初始化使用该元素的bean之前初始化一个或多个bean。
下面的示例使用depends-on属性来表示对单个bean的依赖：

```java
<bean id="beanOne" class="ExampleBean" depends-on="manager"/>
<bean id="manager" class="ManagerBean" />
```
要表达对多个bean的依赖关系，请提供一个bean名称列表作为依赖属性的值(逗号、空格和分号是有效的分隔符)。
```java
<bean id="beanOne" class="ExampleBean" depends-on="manager,accountDao">
	<property name="manager" ref="manager" />
</bean>

<bean id="manager" class="ManagerBean" />
<bean id="accountDao" class="x.y.jdbc.JdbcAccountDao" />
```
depends-on属性既可以指定初始化时依赖项，也可以指定对应的销毁时依赖项(仅在单例bean的情况下)。
在给定bean本身被销毁之前，首先销毁与给定bean定义依赖关系的依赖bean。因此，依赖还可以控制关机顺序。
	