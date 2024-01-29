# 使用 @Autowired
在本节的示例中，可以使用JSR 330的@Inject注解代替Spring的@Autowired注解。点击这里了解更多细节。
你可以将@Autowired注解应用到构造函数中，如下例所示:
```java
public class MovieRecommender {

	private final CustomerPreferenceDao customerPreferenceDao;

	@Autowired
	public MovieRecommender(CustomerPreferenceDao customerPreferenceDao) {
		this.customerPreferenceDao = customerPreferenceDao;
	}

	// ...
}
```
从Spring Framework 4.3开始，如果目标bean一开始只定义了一个构造函数，就不再需要在这样的构造函数上使用@Autowired注解。但是，如果有几个构造函数可用，并且没有主/默认构造函数，则必须至少有一个构造函数用@Autowired注解，以便指示容器使用哪个构造函数。有关详细信息，请参阅关于构造函数解析的讨论。

你也可以将@Autowired注解应用到传统的setter方法中，如下例所示:
```java
public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Autowired
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}

	// ...
}
```

你也可以对具有任意名称和多个参数的方法应用注解，如下例所示:
```java
public class MovieRecommender {

	private MovieCatalog movieCatalog;

	private CustomerPreferenceDao customerPreferenceDao;

	@Autowired
	public void prepare(MovieCatalog movieCatalog,
			CustomerPreferenceDao customerPreferenceDao) {
		this.movieCatalog = movieCatalog;
		this.customerPreferenceDao = customerPreferenceDao;
	}

	// ...
}
```

你也可以将@Autowired应用于字段，甚至可以将它与构造函数混合使用，如下面的例子所示:
```java
public class MovieRecommender {

	private final CustomerPreferenceDao customerPreferenceDao;

	@Autowired
	private MovieCatalog movieCatalog;

	@Autowired
	public MovieRecommender(CustomerPreferenceDao customerPreferenceDao) {
		this.customerPreferenceDao = customerPreferenceDao;
	}

	// ...
}
```

确保您的目标组件(例如，MovieCatalog或CustomerPreferenceDao)是由您用于@autowired注解注入的类型是一致。否则，注入可能会因为运行时出现“找不到类型匹配”错误而失败。

对于通过类路径扫描找到的xml定义的bean或组件类，容器通常预先知道具体类型。但是，对于@Bean工厂方法，您需要确保声明的返回类型具有足够的确定性。对于实现多个接口的组件或可能由其实现类型引用的组件，请考虑在工厂方法上声明最具体的返回类型(至少与引用bean的注入点所要求的一样具体)。

你也可以通过在需要该类型数组的字段或方法中添加@Autowired注解来指示Spring从ApplicationContext中提供特定类型的所有bean，如下例所示:
```java
public class MovieRecommender {

	@Autowired
	private MovieCatalog[] movieCatalogs;

	// ...
}
```

这同样适用于类型化集合，如下例所示:
```java
public class MovieRecommender {

	private Set<MovieCatalog> movieCatalogs;

	@Autowired
	public void setMovieCatalogs(Set<MovieCatalog> movieCatalogs) {
		this.movieCatalogs = movieCatalogs;
	}

	// ...
}
```

如果您希望数组或列表中的项按照特定的顺序排序，您的目标bean可以实现org.springframework.core.Ordered接口，或者使用@Order或标准的@Priority注解。否则，它们的顺序遵循容器中相应目标bean定义的注册顺序。

注意，标准jakarta.annotation.Priority注解在@Bean级别不可用，因为它不能在方法上声明。它的语义可以通过在每个类型的单个bean上结合@Order值和@Primary来建模。

即使是类型化的Map实例也可以自动注入，只要期望的键类型是String。映射值包含预期类型的所有bean，键包含相应的bean名称，如下所示:
```java
public class MovieRecommender {

	private Map<String, MovieCatalog> movieCatalogs;

	@Autowired
	public void setMovieCatalogs(Map<String, MovieCatalog> movieCatalogs) {
		this.movieCatalogs = movieCatalogs;
	}

	// ...
}
```
默认情况下，当给定注入点没有匹配的候选bean可用时，自动装配将失败。对于声明的数组、集合或映射，至少需要一个匹配元素。

默认行为是将带注解的方法和字段视为必需的依赖项。你可以改变这种行为，如下面的例子所示，通过将一个不存在的类标记为非必需来使框架跳过它(也就是说，通过将@Autowired中的required属性设置为false):
```java
public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Autowired(required = false)
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}

	// ...
}
```
如果非必需的方法的依赖项(或其依赖项中的一个，在有多个参数的情况下)不可用，则根本不会调用该方法。在这种情况下，一个非必填字段根本不会被填充，保留其默认值。

换句话说，将required属性设置为false表示对应的属性对于自动注入来说是可选的，如果不能自动连接，则忽略该属性。这允许属性被赋予默认值，这些值可以通过依赖注入被选择性地覆盖。

注入的构造函数和工厂方法参数是一种特殊情况，因为@Autowired中的required属性具有不同的含义，这是由于Spring的构造函数解析算法可能会处理多个构造函数。默认情况下，构造函数和工厂方法参数是必需的，但在在单构造函数场景中需要一些特殊规则，例如，如果没有匹配的bean可用，则多元素注入点(数组、集合、映射)解析为空实例。这允许在一个唯一的多参数构造函数中声明所有依赖关系的通用实现模式——例如，作为一个没有@Autowired注解的公共构造函数声明。

对于任何给定的bean类，只有一个构造函数可以声明@Autowired，并将所需的属性设置为true，这表明该构造函数在作为Spring bean使用时要自动注入。因此，如果required属性保持其默认值true，则只有一个构造函数可以用@Autowired注解。如果多个构造函数声明了注解，它们都必须声明required=false，才能被视为自动装配的候选者(类似于XML中的自动装配=constructor)。将选择具有最多依赖项的构造函数，这些依赖项可以通过在Spring容器中匹配bean来满足。如果所有候选函数都不能满足，那么将使用主/默认构造函数(如果存在)。类似地，如果一个类声明了多个构造函数，但没有一个用@Autowired注解，那么将使用一个主/默认构造函数(如果存在)。
如果一个类一开始只声明了一个构造函数，那么它将始终被使用，即使没有注解。注意，带注解的构造函数不一定是公共的。

或者，您可以通过Java 8的Java .util来表示特定依赖项的非必需性质。可选，示例如下:
```java
public class SimpleMovieLister {

	@Autowired
	public void setMovieFinder(Optional<MovieFinder> movieFinder) {
		...
	}
}
```
从Spring Framework 5.0开始，你还可以使用@Nullable注解(任何包中的任何类型的注解——例如，JSR-305中的javax.annotation.Nullable)，或者只是利用Kotlin内置的null安全支持:
```java
public class SimpleMovieLister {

	@Autowired
	public void setMovieFinder(@Nullable MovieFinder movieFinder) {
		...
	}
}
```
你也可以对那些众所周知的可解析依赖的接口使用@Autowired: BeanFactory、ApplicationContext、Environment、ResourceLoader、ApplicationEventPublisher和MessageSource。
这些接口和它们的扩展接口，如ConfigurableApplicationContext或ResourcePatternResolver，是自动解析的，不需要特殊的设置。下面的例子自动连接一个ApplicationContext对象:

```java
public class MovieRecommender {

	@Autowired
	private ApplicationContext context;

	public MovieRecommender() {
	}

	// ...
}
```
@Autowired、@Inject、@Value和@Resource注解是由Spring BeanPostProcessor实现处理的。这意味着您不能在自己的BeanPostProcessor或BeanFactoryPostProcessor类型(如果有的话)中应用这些注解。这些类型必须通过使用XML或Spring @Bean方法显式地“连接”起来。