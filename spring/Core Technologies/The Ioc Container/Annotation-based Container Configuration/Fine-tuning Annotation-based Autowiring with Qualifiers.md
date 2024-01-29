# 使用 Qualifiers 微调基于注解的自动装配
当可以确定一个主要候选者时，@Primary是一种有效的方法，可以根据多个实例的类型使用自动装配。当需要对选择过程进行更多控制时，可以使用Spring的@Qualifier注解。您可以将限定符值与特定的参数关联起来，缩小类型匹配集，以便为每个参数选择特定的bean。在最简单的情况下，这可以是一个简单的描述性值，如下例所示:
```java
public class MovieRecommender {

	@Autowired
	@Qualifier("main")
	private MovieCatalog movieCatalog;

	// ...
}
```
你也可以在单个构造函数参数或方法参数上指定@Qualifier注解，如下例所示:
```java
public class MovieRecommender {

	private final MovieCatalog movieCatalog;

	private final CustomerPreferenceDao customerPreferenceDao;

	@Autowired
	public void prepare(@Qualifier("main") MovieCatalog movieCatalog,
			CustomerPreferenceDao customerPreferenceDao) {
		this.movieCatalog = movieCatalog;
		this.customerPreferenceDao = customerPreferenceDao;
	}

	// ...
}
```
下面的示例显示了相应的bean定义:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context
		https://www.springframework.org/schema/context/spring-context.xsd">

	<context:annotation-config/>

	<bean class="example.SimpleMovieCatalog">
		<qualifier value="main"/>

		<!-- inject any dependencies required by this bean -->
	</bean>

	<bean class="example.SimpleMovieCatalog">
		<qualifier value="action"/>

		<!-- inject any dependencies required by this bean -->
	</bean>

	<bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>
```
1. 具有 main 限定符值的bean与具有相同限定值的构造函数参数相连接。
2. 具有 action 限定符值的bean与具有相同值限定符的构造函数参数连接在一起。

对于fallback匹配，bean名称被视为默认限定符值。因此，您可以使用id为main而不是嵌套的限定符元素来定义bean，从而得到相同的匹配结果。然而，尽管您可以使用此约定按名称引用特定bean，但@Autowired基本上是关于带有可选语义限定符的类型驱动注入。这意味着限定符值，即使使用bean名称，也总是在类型匹配集中具有缩小语义。它们不从语义上表达对唯一bean id的引用。好的限定符值是main或EMEA或persistent，它们表示独立于bean id的特定组件的特征，在使用匿名bean定义的情况下(如前面示例中的bean)， bean id可能会自动生成。

限定符也适用于类型化集合，如前所述—例如，Set< movieccatalog >。在这种情况下，根据声明的限定符，所有匹配的bean都作为集合注入。__这意味着限定符不必是唯一的__。相反，它们构成了过滤标准。例如，您可以定义具有相同限定符值“action”的多个MovieCatalog bean，所有这些bean都被注入到带有@Qualifier(“action”)注解的Set<MovieCatalog>中。

让限定符值在类型匹配候选对象中根据目标bean名称进行选择，不需要在注入点使用@Qualifier注解。如果没有其他解析指示符(例如qualifier或primary)，对于非唯一依赖情况，Spring将根据目标bean名称匹配注入点名称(即字段名称或参数名称)，并选择同名的候选对象(如果有的话)。

__也就是说，如果您打算通过名称表达注解驱动的注入，则不要主要使用@Autowired，即使它能够根据bean名称在类型匹配候选者中进行选择。相反，可以使用JSR-250 @Resource注解，**该注解在语义上定义为通过其唯一名称标识特定的目标组件**，声明的类型与匹配过程无关__。

@Autowired具有相当不同的语义:在按类型选择候选bean之后，指定的String限定符值仅在这些类型选择的候选中被考虑(例如，与标记有相同限定符标签的bean匹配帐户限定符)。对于本身定义为集合、映射或数组类型的bean， @Resource是一个很好的解决方案，通过唯一名称引用特定的集合或数组bean。也就是说，从4.3开始，只要元素类型信息保存在@Bean返回类型签名或集合继承层次结构中，就可以通过Spring的@Autowired类型匹配算法匹配集合、Map和数组类型。在这种情况下，可以使用限定符值在相同类型的集合中进行选择，如前一段所述。

从4.3开始，@Autowired还考虑了注入的自我引用(也就是说，对当前注入的bean的引用)。注意，自注入是一种退路。对其他组件的常规依赖总是具有优先级。从这个意义上说，自我引用不参与常规的候选人选择，因此特别不是主要的。相反，它们总是以最低优先级结束。在实践中，应该将自我引用作为最后的手段(例如，通过bean的事务代理调用同一实例上的其他方法)。在这种情况下，请考虑将受影响的方法分解到单独的委托bean中。或者，您可以使用@Resource，它可以通过其唯一名称获得返回到当前bean的代理。

@Autowired适用于字段、构造函数和多参数方法，允许在参数级别通过限定符注解缩小范围。相比之下，__@Resource仅支持具有单个参数的字段和bean属性设置器方法__。因此，如果注入目标是构造函数或多参数方法，则应该坚持使用限定符。

您可以创建自己的自定义限定符注解。为此，定义一个注解并在定义中提供@Qualifier注解，如下例所示:
```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Genre {

	String value();
}
```

然后你可以在自动连接的字段和参数上提供自定义限定符，如下面的例子所示:
```java
public class MovieRecommender {

	@Autowired
	@Genre("Action")
	private MovieCatalog actionCatalog;

	private MovieCatalog comedyCatalog;

	@Autowired
	public void setComedyCatalog(@Genre("Comedy") MovieCatalog comedyCatalog) {
		this.comedyCatalog = comedyCatalog;
	}

	// ...
}
```

接下来，您可以为候选bean定义提供信息。您可以添加<qualifier/>标记作为<bean/>标记的子元素，然后指定类型和值以匹配您的自定义限定符注解。类型与注解的完全限定类名匹配。另外，如果不存在名称冲突的风险，为了方便起见，您可以使用短类名。下面的例子演示了这两种方法:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context
		https://www.springframework.org/schema/context/spring-context.xsd">

	<context:annotation-config/>

	<bean class="example.SimpleMovieCatalog">
		<qualifier type="Genre" value="Action"/>
		<!-- inject any dependencies required by this bean -->
	</bean>

	<bean class="example.SimpleMovieCatalog">
		<qualifier type="example.Genre" value="Comedy"/>
		<!-- inject any dependencies required by this bean -->
	</bean>

	<bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>
```

在类路径扫描和托管组件中，您可以看到在XML中提供限定符元数据的基于注解的替代方法。具体来说，请参见使用注解提供限定符元数据。

在某些情况下，使用不带值的注解就足够了。当注解服务于更通用的目的，并且可以跨几种不同类型的依赖项应用时，这可能很有用。例如，您可以提供一个离线目录，在没有Internet连接时可以对其进行搜索。首先，定义简单注解，如下例所示:
```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Offline {
}
```
然后将注解添加到要自动连接的字段或属性中，如下例所示:
```java
public class MovieRecommender {

	@Autowired
	@Offline
	private MovieCatalog offlineCatalog;

	// ...
}
```
现在，bean定义只需要限定符类型，如下面的示例所示:
```java
<bean class="example.SimpleMovieCatalog">
	<qualifier type="Offline"/>
	<!-- inject any dependencies required by this bean -->
</bean>
```

您还可以定义自定义限定符注解，除了接受简单的value属性之外，还接受命名属性。如果在要自动注入的字段或参数上指定了多个属性值，那么bean定义必须匹配所有这些属性值才能被认为是自动注入的候选者。
作为一个例子，考虑下面的注解定义:

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface MovieQualifier {

	String genre();

	Format format();
}
```
在本例中，Format是一个enum，定义如下:
```java
public enum Format {
	VHS, DVD, BLURAY
}
```
要自动连接的字段用自定义限定符进行注解，并包括属性:genre和format的值，如下面的示例所示:
```xml
public class MovieRecommender {

	@Autowired
	@MovieQualifier(format=Format.VHS, genre="Action")
	private MovieCatalog actionVhsCatalog;

	@Autowired
	@MovieQualifier(format=Format.VHS, genre="Comedy")
	private MovieCatalog comedyVhsCatalog;

	@Autowired
	@MovieQualifier(format=Format.DVD, genre="Action")
	private MovieCatalog actionDvdCatalog;

	@Autowired
	@MovieQualifier(format=Format.BLURAY, genre="Comedy")
	private MovieCatalog comedyBluRayCatalog;

	// ...
}
```
最后，bean定义应该包含匹配的限定符值。这个示例还演示了可以使用bean元属性代替<qualifier/>元素。
如果可用，<qualifier/>元素和它的属性优先，但是自动装配机制依赖于<meta/>标记中提供的值，如果没有这样的限定符，如下例中的最后两个bean定义:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context
		https://www.springframework.org/schema/context/spring-context.xsd">

	<context:annotation-config/>

	<bean class="example.SimpleMovieCatalog">
		<qualifier type="MovieQualifier">
			<attribute key="format" value="VHS"/>
			<attribute key="genre" value="Action"/>
		</qualifier>
		<!-- inject any dependencies required by this bean -->
	</bean>

	<bean class="example.SimpleMovieCatalog">
		<qualifier type="MovieQualifier">
			<attribute key="format" value="VHS"/>
			<attribute key="genre" value="Comedy"/>
		</qualifier>
		<!-- inject any dependencies required by this bean -->
	</bean>

	<bean class="example.SimpleMovieCatalog">
		<meta key="format" value="DVD"/>
		<meta key="genre" value="Action"/>
		<!-- inject any dependencies required by this bean -->
	</bean>

	<bean class="example.SimpleMovieCatalog">
		<meta key="format" value="BLURAY"/>
		<meta key="genre" value="Comedy"/>
		<!-- inject any dependencies required by this bean -->
	</bean>

</beans>
```