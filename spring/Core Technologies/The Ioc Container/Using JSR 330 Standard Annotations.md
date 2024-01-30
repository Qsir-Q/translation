# 使用JSR330标准注解
Spring提供了对JSR-330标准注释(依赖注入)的支持。以与Spring注释相同的方式扫描这些注释。要使用它们，需要在类路径中有相关的jar文件。如果您使用Maven，则jakarta.inject。在标准的Maven存储库(https://repo.maven.apache.org/maven2/jakarta/inject/jakarta.inject-api/2.0.0/)中可以获得inject构件。你可以在pom.xml文件中添加以下依赖项:

```xml
<dependency>
	<groupId>jakarta.inject</groupId>
	<artifactId>jakarta.inject-api</artifactId>
	<version>2.0.0</version>
</dependency>
```



# 使用@Inject和@Named进行依赖注入

你可以用@jakarta.inject代替@Autowired。注入如下:
```java
import jakarta.inject.Inject;

public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Inject
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}

	public void listMovies() {
		this.movieFinder.findMovies(...);
		// ...
	}
}
```
和@Autowired一样，你可以在字段级、方法级和构造函数参数级使用@Inject。此外，你可以将注入点声明为Provider，允许按需访问较短作用域的bean，或者通过调用Provider.get()延迟访问其他bean。下面的例子是前一个例子的变体:
```java
import jakarta.inject.Inject;
import jakarta.inject.Provider;

public class SimpleMovieLister {

	private Provider<MovieFinder> movieFinder;

	@Inject
	public void setMovieFinder(Provider<MovieFinder> movieFinder) {
		this.movieFinder = movieFinder;
	}

	public void listMovies() {
		this.movieFinder.get().findMovies(...);
		// ...
	}
}
```
如果你想为要注入的依赖项使用限定名，你应该使用@Named注释，如下例所示:
```java
import jakarta.inject.Inject;
import jakarta.inject.Named;

public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Inject
	public void setMovieFinder(@Named("main") MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}

	// ...
}
```
与@Autowired一样，@Inject也可以与java.util.Optional或@Nullable一起使用。这在这里更适用，因为@Inject没有必需的属性。下面两个例子展示了如何使用@Inject和@Nullable:
```java
public class SimpleMovieLister {

	@Inject
	public void setMovieFinder(Optional<MovieFinder> movieFinder) {
		// ...
	}
}
```

```java
public class SimpleMovieLister {

	@Inject
	public void setMovieFinder(@Nullable MovieFinder movieFinder) {
		// ...
	}
}
```

#### @Named和@ManagedBean:与@Component注释等价的标准
你可以用@jakarta.inject代替@Component。Named或jakarta.annotation.ManagedBean，如下例所示:
```java
import jakarta.inject.Inject;
import jakarta.inject.Named;

@Named("movieListener")  // @ManagedBean("movieListener") could be used as well
public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Inject
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}

	// ...
}
```
在不指定组件名称的情况下使用@Component是很常见的。@Named可以以类似的方式使用，如下例所示:
```java
import jakarta.inject.Inject;
import jakarta.inject.Named;

@Named
public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Inject
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}

	// ...
}
```
当你使用@Named或@ManagedBean时，你可以用与使用Spring注释完全相同的方式使用组件扫描，如下面的例子所示:
```java
@Configuration
@ComponentScan(basePackages = "org.example")
public class AppConfig  {
	// ...
}
```
与@Component相反，JSR-330 @Named和JSR-250 @ManagedBean注释是不可组合的。您应该使用Spring的构造型模型来构建自定义组件注释。

# JSR-330标准注释的限制:
当你使用标准注释时，你应该知道一些重要的特性是不可用的，如下表所示:
         Spring                  jakarta.inject.*

1. @Autowired   @Inject              @Inject没有'required'属性。可以与Java 8的Optional一起使用。
2. @Component   @Named / @ManagedBean   JSR-330不提供可组合的模型，只提供一种识别命名组件的方法。
3. @Scope("singleton")   @Singleton    JSR-330的默认作用域类似于Spring的原型。但是，为了使其与Spring的一般默认值保持一致，在Spring容器中声明的JSR-330 bean默认情况下是单例的。为了使用单例以外的作用域，你应该使用Spring的@Scope注释。jakarta.inject还提供了一个jakarta.inject.Scope注释:但是，这个注释仅用于创建自定义注释。
5. @Qualifier   @Qualifier / @Named  qualifier只是一个用于构建自定义限定符的元注释。具体的字符串限定符(比如Spring带值的@Qualifier)可以通过jakarta.inject.Named来关联。
6. @Value
7. @Lazy
8. ObjectFactory  Provider
provider是Spring的ObjectFactory的直接替代品，只是get()方法名更短。它还可以与Spring的@Autowired或无注释的构造函数和setter方法结合使用。