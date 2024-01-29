# 使用@Resource注入

Spring还通过在字段或bean属性设置器方法上使用JSR-250 @Resource注解(jakarta.annotation.Resource)来支持注入。这是Jakarta EE中的常见模式:例如，在jsf管理的bean和JAX-WS端点中。对于Spring管理的对象，Spring也支持这种模式。

@Resource接受一个name属性。默认情况下，Spring将该值解释为要注入的bean名称。换句话说，它遵循按名称语义，如下例所示:

```java
public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Resource(name="myMovieFinder")
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}
}
```

如果没有显式指定名称，则默认名称将从字段名称或setter方法派生。如果是字段，则采用字段名。对于setter方法，它采用bean属性名。下面的例子将把名为movieFinder的bean注入到它的setter方法中:

```java
public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Resource
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}
}
```

随注解一起提供的名称由ApplicationContext解析为bean名称，CommonAnnotationBeanPostProcessor知道这个名称。如果显式地配置Spring的SimpleJndiBeanFactory，则可以通过JNDI解析这些名称。但是，我们建议您依赖默认行为并使用Spring的JNDI查找功能来保持间接级别。

在没有指定显式名称的@Resource使用的独占情况下，与@Autowired类似，@Resource查找主要类型匹配而不是特定的命名bean，并解析众所周知的可解析依赖:BeanFactory, ApplicationContext, ResourceLoader, ApplicationEventPublisher和MessageSource接口。
因此，在下面的示例中，customerPreferenceDao字段首先查找名为“customerPreferenceDao”的bean，然后退回到customerPreferenceDao类型的主类型匹配:

```java
public class MovieRecommender {

	@Resource
	private CustomerPreferenceDao customerPreferenceDao;

	@Resource
	private ApplicationContext context;

	public MovieRecommender() {
	}

	// ...
}
```

context字段是根据已知的可解析依赖类型ApplicationContext注入的。