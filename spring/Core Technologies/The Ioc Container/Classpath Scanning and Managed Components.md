# 路径扫描和管理组件(Components)

本章中的大多数示例使用XML指定在Spring容器中生成每个BeanDefinition的配置元数据。前一节(基于注解的容器配置)演示了如何通过源代码级注解提供大量配置元数据。然而，即使在这些示例中，“基本”bean定义也显式地定义在XML文件中，而注解仅驱动依赖项注入。

本节描述一个通过扫描类路径来隐式检测候选组件的选项。候选组件是与筛选标准匹配的类，并且具有在容器中注册的相应bean定义。这样就不需要使用XML来执行bean注册。相反，您可以使用注解(例如@Component)、AspectJ类型表达式或您自己的自定义筛选标准来选择哪些类具有在容器中注册的bean定义。

可以使用Java而不是XML文件定义bean。查看@Configuration、@Bean、@Import和@DependsOn注释，了解如何使用这些特性的示例。



# @Component和更多的构造型注释
@Repository注释是实现存储库角色或原型(也称为数据访问对象或DAO)的任何类的标记。该标记的用途之一是异常的自动翻译，如异常翻译中所述。

Spring提供了更多的构造型注释:@Component、@Service和@Controller。
@Component是任何spring管理组件的通用构造型。
@Repository、@Service和@Controller是@Component的专门化，用于更具体的用例(分别在持久化层、服务层和表示层)。
因此，你可以用@Component来注释你的组件类，但是，通过用@Repository、@Service或@Controller来注释它们，你的类更适合由工具处理或与方面关联。
例如，这些构造型注释是切入点的理想目标。@Repository、@Service和@Controller也可能在Spring框架的未来版本中携带额外的语义。
因此，如果你要在服务层使用@Component或@Service之间做出选择，@Service显然是更好的选择。类似地，如前所述，@Repository已经被支持作为持久层中自动异常转换的标记。



# 使用元注释和组合注释
Spring提供的许多注释都可以在您自己的代码中用作元注释。元注释是一种可以应用于其他注释的注释。
例如，前面提到的@Service注释是用@Component做元注释的，如下面的例子所示:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Service {

	// ...
}
```

你也可以组合元注释来创建“组合注释”。例如，Spring MVC中的@RestController注释由@Controller和@ResponseBody组成。

此外，组合注释可以选择性地重新声明元注释中的属性，以允许自定义。当您只想公开元注释属性的一个子集时，这可能特别有用。例如，Spring的@SessionScope注释将作用域名称硬编码到session，但仍然允许自定义proxyMode。下面的例子显示了SessionScope注释的定义:
```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Scope(WebApplicationContext.SCOPE_SESSION)
public @interface SessionScope {

	/**
	 * Alias for {@link Scope#proxyMode}.
	 * <p>Defaults to {@link ScopedProxyMode#TARGET_CLASS}.
	 */
	@AliasFor(annotation = Scope.class)
	ScopedProxyMode proxyMode() default ScopedProxyMode.TARGET_CLASS;

}
```
然后你可以使用@SessionScope而不用声明proxyMode，如下所示:
```java
@Service
@SessionScope
public class SessionScopedService {
	// ...
}
```
你也可以覆盖proxyMode的值，如下例所示:
```java
@Service
@SessionScope(proxyMode = ScopedProxyMode.INTERFACES)
public class SessionScopedUserService implements UserService {
	// ...
}
```
要了解更多细节，请参阅Spring Annotation Programming Model wiki页面。



# 自动检测类和注册Bean定义
Spring可以自动检测原型类，并向ApplicationContext注册相应的BeanDefinition实例。例如，以下两个类有资格进行这种自动检测:
```java
@Service
public class SimpleMovieLister {

	private MovieFinder movieFinder;

	public SimpleMovieLister(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}
}
```
```java
@Repository
public class JpaMovieFinder implements MovieFinder {
	// implementation elided for clarity
}
```
为了自动检测这些类并注册相应的bean，您需要将@ComponentScan添加到@Configuration类中，其中的basePackages属性是这两个类的公共父包。(或者，您可以指定一个逗号、分号或空格分隔的列表，其中包括每个类的父包。)
```java
@Configuration
@ComponentScan(basePackages = "org.example")
public class AppConfig  {
	// ...
}
```
为简洁起见，前面的示例可以使用注释的value属性(即@ComponentScan("org.example"))。

下面的替代方案使用XML:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context
		https://www.springframework.org/schema/context/spring-context.xsd">

	<context:component-scan base-package="org.example"/>

</beans>
```
使用<context:component-scan>隐式地启用了<context:annotation-config>的功能。当使用<context:component-scan>时，通常不需要包含<context:annotation-config>元素。

扫描类路径包需要在类路径中存在相应的目录项。当您使用Ant构建JAR时，请确保您没有激活JAR任务的仅文件开关。此外，在某些环境中，基于安全策略，类路径目录可能不会公开——例如，JDK 1.7.0_45及更高版本上的独立应用程序(这需要在您的清单中设置“Trusted-Library”——参见stackoverflow.com/questions/19394570/java-jre-7u45-breaks-classloader-getresources)。

在JDK 9的模块路径(Jigsaw)上，Spring的类路径扫描通常按预期工作。但是，请确保在模块信息描述符中导出了组件类。如果您希望Spring调用类的非公共成员，请确保它们是“打开的(opened)”(也就是说，它们在模块信息描述符中使用open声明而不是exports声明)。

此外，当您使用组件扫描元素时，AutowiredAnnotationBeanPostProcessor和CommonAnnotationBeanPostProcessor都是隐式包含的。这意味着这两个组件可以自动检测并连接在一起——所有这些都不需要在XML中提供任何bean配置元数据。



# 使用过滤器自定义扫描

默认情况下，带有@Component、@Repository、@Service、@Controller、@Configuration注解的类，或者带有@Component注解的自定义注解是唯一检测到的候选组件。但是，您可以通过应用自定义筛选器来修改和扩展此行为。将它们添加为@ComponentScan注释的includeFilters或excludeFilters属性(或者添加为XML配置中<context:component-scan>元素的<context:include-filter />或<context:exclude-filter />子元素)。
每个筛选器元素都需要类型和表达式属性。过滤选项如下表所示:

- annotation (default)    在目标组件的类型级别上present或meta-present的注释。
- assignable           目标组件可赋值(扩展或实现)的类(或接口)。
- aspectj     目标组件要匹配的AspectJ类型表达式。
- regex       与目标组件的类名匹配的正则表达式。
- custom      org.springframework.core.type.TypeFilter接口的自定义实现。
下面的示例显示了忽略所有@Repository注释并使用“stub”存储库的配置:
```java
@Configuration
@ComponentScan(basePackages = "org.example",
		includeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Stub.*Repository"),
		excludeFilters = @Filter(Repository.class))
public class AppConfig {
	// ...
}
```
下面的例子显示了等效的XML:
```xml
<beans>
	<context:component-scan base-package="org.example">
		<context:include-filter type="regex"
				expression=".*Stub.*Repository"/>
		<context:exclude-filter type="annotation"
				expression="org.springframework.stereotype.Repository"/>
	</context:component-scan>
</beans>
```
您还可以通过在注释中设置useDefaultFilters=false或通过提供use-default-filters="false"作为<component-scan/>元素的属性来禁用默认过滤器。这有效地禁用了对带有@Component、@Repository、@Service、@Controller、@RestController或@Configuration注释或元注释的类的自动检测。



# 在组件中定义Bean

Spring组件还可以向容器提供bean定义元数据。您可以使用与在带有@Configuration注释的类中定义bean元数据相同的@Bean注释来做到这一点。下面的例子展示了如何这样做:
```java
@Component
public class FactoryMethodComponent {

	@Bean
	@Qualifier("public")
	public TestBean publicInstance() {
		return new TestBean("publicInstance");
	}

	public void doWork() {
		// Component method implementation omitted
	}
}
```
前面的类是一个Spring组件，它的doWork()方法中有特定于应用程序的代码。但是，它还提供了一个bean定义，该定义具有一个引用方法publicInstance()的工厂方法。@Bean注释标识工厂方法和其他bean定义属性，例如通过@Qualifier注释标识限定符值。其他可以指定的方法级注释是@Scope、@Lazy和自定义限定符注释。

除了用于组件初始化之外，你还可以把@Lazy注释放在用@Autowired或@Inject标记的注入点上。在这种情况下，它会导致惰性解析代理的注入。然而，这种代理方法是相当有限的。对于复杂的惰性交互，特别是与可选依赖项结合使用时，我们建议使用 __**ObjectProvider<MyTargetBean>**__。

如前所述，支持自动连接字段和方法，并支持@Bean方法的自动连接。下面的例子展示了如何这样做:
```java
@Component
public class FactoryMethodComponent {

	private static int i;

	@Bean
	@Qualifier("public")
	public TestBean publicInstance() {
		return new TestBean("publicInstance");
	}

	// use of a custom qualifier and autowiring of method parameters
	@Bean
	protected TestBean protectedInstance(
			@Qualifier("public") TestBean spouse,
			@Value("#{privateInstance.age}") String country) {
		TestBean tb = new TestBean("protectedInstance", 1);
		tb.setSpouse(spouse);
		tb.setCountry(country);
		return tb;
	}

	@Bean
	private TestBean privateInstance() {
		return new TestBean("privateInstance", i++);
	}

	@Bean
	@RequestScope
	public TestBean requestScopedInstance() {
		return new TestBean("requestScopedInstance", 3);
	}
}
```
该示例将String方法参数country自动连接到另一个名为privateInstance的bean上的age属性的值。
Spring Expression Language元素通过符号#{< Expression >}定义属性的值。
对于@Value注释，表达式解析器被预先配置为在解析表达式文本时查找bean名称。

从Spring Framework 4.3开始，您还可以声明一个InjectionPoint类型的工厂方法参数(或其更具体的子类:DependencyDescriptor)来访问触发当前bean创建的请求注入点。注意，这只适用于bean实例的实际创建，而不适用于现有实例的注入。因此，这个特性对原型范围的bean最有意义。对于其他作用域，工厂方法只看到触发在给定作用域中创建新bean实例的注入点(例如，触发创建惰性单例bean的依赖项)。在这种情况下，您可以谨慎使用提供的注入点元数据。下面的例子展示了如何使用InjectionPoint:
```java
@Component
public class FactoryMethodComponent {

	@Bean @Scope("prototype")
	public TestBean prototypeInstance(InjectionPoint injectionPoint) {
		return new TestBean("prototypeInstance for " + injectionPoint.getMember());
	}
}
```

常规Spring组件中的@Bean方法与Spring @Configuration类中的@Bean方法处理方式不同。不同之处在于@Component类没有用CGLIB增强来拦截方法和字段的调用。CGLIB代理是通过调用@Configuration类中的@Bean方法中的方法或字段来创建对协作对象的bean元数据引用的方法。

这些方法不是用普通的Java语义调用的，而是通过容器来提供Spring bean的通常的生命周期管理和代理，即使是在通过编程调用@Bean方法引用其他bean时也是如此。相反，在普通的@Component类中调用@Bean方法中的方法或字段具有标准的Java语义，不需要应用特殊的CGLIB处理或其他约束。

您可以将@Bean方法声明为静态方法，允许在不将包含它们的配置类创建为实例的情况下调用它们。在定义后处理器bean(例如，类型为BeanFactoryPostProcessor或BeanPostProcessor)时，这一点特别有意义，因为此类bean在容器生命周期的早期被初始化，并且应该避免在那时触发配置的其他部分。

@Bean方法的Java语言可见性不会对Spring容器中生成的bean定义产生直接影响。您可以自由地在非@Configuration类中声明工厂方法，也可以在任何地方声明静态方法。然而，@Configuration类中的常规@Bean方法需要是可重写的——也就是说，它们不能被声明为private或final。

@Bean方法也可以在给定组件或配置类的基类上发现，也可以在由组件或配置类实现的接口中声明的Java 8默认方法中发现。这为组合复杂的配置安排提供了很大的灵活性，甚至可以通过Spring 4.2的Java 8默认方法实现多重继承。

最后，单个类可以为同一个bean保存多个@Bean方法，作为在运行时根据可用依赖项使用的多个工厂方法的安排。这与在其他配置场景中选择“最贪婪”的构造函数或工厂方法的算法相同:在构造时选择具有最多可满足依赖项的变体，类似于容器如何在多个@Autowired构造函数之间进行选择。



# 命名自动检测的组件
当组件作为扫描过程的一部分被自动检测时，它的bean名称由该扫描仪已知的BeanNameGenerator策略生成。

默认情况下，使用AnnotationBeanNameGenerator。对于Spring构造型注释，如果您通过注释的value属性提供一个名称，该名称将在相应的bean定义中用作名称。当使用以下JSR-250和JSR-330注释代替Spring构造型注释时，此约定也适用:@jakarta.annotation.ManagedBean, @javax.annotation.ManagedBean, @jakarta.inject.Named, and @javax.inject.Named

从Spring Framework 6.1开始，用于指定bean名称的注释属性的名称不再需要为value。自定义构造型注释可以用不同的名称(比如name)声明一个属性，并使用@AliasFor(annotation = Component.class, attribute = "value")对该属性进行注释。请参阅ControllerAdvice#name()的源代码声明，以获取具体示例。

从Spring Framework 6.1开始，对基于约定的构造型名称的支持已被弃用，并将在框架的未来版本中删除。因此，自定义构造型注释必须使用@AliasFor为@Component中的value属性声明一个显式别名。
具体示例参见Repository#value()和ControllerAdvice#name()的源代码声明。

如果不能从这样的注释或任何其他检测到的组件(例如由自定义过滤器发现的组件)派生出显式bean名称，则默认bean名称生成器返回未大写的非限定类名称。例如，如果检测到以下组件类，它们的名称将是myMovieLister和movieFinderImpl。
```java
@Service("myMovieLister")
public class SimpleMovieLister {
	// ...
}
```
```java
@Repository
public class MovieFinderImpl implements MovieFinder {
	// ...
}
```
如果不想依赖默认的bean命名策略，可以提供自定义的bean命名策略。首先，实现BeanNameGenerator接口，并确保包含一个默认的无参数构造函数。然后，在配置扫描器时提供完全限定的类名，如下面的注释和bean定义示例所示。

如果由于多个自动检测的组件具有相同的非限定类名(即，具有相同名称但位于不同包中的类)而遇到命名冲突，则可能需要配置BeanNameGenerator，该生成器默认为生成的bean名使用完全限定类名。从Spring Framework 5.2.3开始，位于包org.springframework.context.annotation中的fulllyqualifiedannotationbeannamegenerator可以用于这种目的。

```java
@Configuration
@ComponentScan(basePackages = "org.example", nameGenerator = MyNameGenerator.class)
public class AppConfig {
	// ...
}
```

```java
<beans>
	<context:component-scan base-package="org.example"
		name-generator="org.example.MyNameGenerator" />
</beans>
```

作为一般规则，当其他组件可能显式引用该名称时，请考虑使用注释指定该名称。另一方面，只要容器负责连接，自动生成的名称就足够了。



# 为自动检测的组件提供作用域
与spring管理的组件一样，自动检测组件的默认和最常见的作用域是单例的。然而，有时您需要一个可以通过@Scope注释指定的不同作用域。你可以在注释中提供作用域的名称，如下例所示:
```java
@Scope("prototype")
@Repository
public class MovieFinderImpl implements MovieFinder {
	// ...
}
```
@Scope注释仅在具体的bean类(用于带注释的组件)或工厂方法(用于@Bean方法)上进行自省。与XML bean定义相反，没有bean定义继承的概念，类级别的继承层次结构与元数据目的无关。

有关Spring上下文中的“request”或“session”等web特定作用域的详细信息，请参见request, session, Application和WebSocket作用域。与针对这些作用域的预构建注释一样，您也可以通过使用Spring的元注释方法来组合自己的作用域注释:例如，使用@Scope(“prototype”)进行元注释的自定义注释，可能还声明自定义作用域代理模式。

要为范围解析提供自定义策略，而不是依赖于基于注释的方法，您可以实现ScopeMetadataResolver接口。
确保包含一个默认的无参数构造函数。然后，您可以在配置扫描器时提供完全限定的类名，如下面的注释和bean定义示例所示:
```java
@Configuration
@ComponentScan(basePackages = "org.example", scopeResolver = MyScopeResolver.class)
public class AppConfig {
	// ...
}
```

```xml
<beans>
	<context:component-scan base-package="org.example" scope-resolver="org.example.MyScopeResolver"/>
</beans>
```

#### 提供带有注解的Qualifier元数据
@Qualifier注释在使用qualifier微调基于注释的自动装配中进行了讨论。这一节中的示例演示了如何使用@Qualifier注释和自定义修饰符注释，以便在解析自动装配候选者时提供细粒度控制。因为这些示例是基于XML bean定义的，所以限定符元数据是通过使用XML中bean元素的限定符或元子元素在候选bean定义上提供的。
当依赖类路径扫描来自动检测组件时，您可以在候选类上提供具有类型级别注释的限定符元数据。下面三个例子演示了这种技术:

```java
@Component
@Qualifier("Action")
public class ActionMovieCatalog implements MovieCatalog {
	// ...
}
```

```java
@Component
@Genre("Action")
public class ActionMovieCatalog implements MovieCatalog {
	// ...
}
```

```java
@Component
@Offline
public class CachingMovieCatalog implements MovieCatalog {
	// ...
}
```

与大多数基于注释的替代方法一样，请记住，注释元数据绑定到类定义本身，而使用XML允许相同类型的多个bean在其限定符元数据中提供变体，因为这些元数据是按实例而不是按类提供的。