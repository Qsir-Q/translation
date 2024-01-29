# 使用@Value
@Value通常用于注入外部化属性:
```java
@Component
public class MovieRecommender {

    private final String catalog;

    public MovieRecommender(@Value("${catalog.name}") String catalog) {
        this.catalog = catalog;
    }
}
```
使用以下配置:
```java
@Configuration
@PropertySource("classpath:application.properties")
public class AppConfig { }
```
下面的application.properties文件:
```properties
catalog.name=MovieCatalog
```
在这种情况下，catalog参数和字段将等于MovieCatalog值。

Spring提供了一个默认的值解析器。它将尝试解析属性值，如果无法解析，则会将属性名(例如${catalog.name})作为值注入。如果你想严格控制不存在的值，你应该声明一个PropertySourcesPlaceholderConfigurer bean，如下面的例子所示:
```java
@Configuration
public class AppConfig {

	@Bean
	public static PropertySourcesPlaceholderConfigurer propertyPlaceholderConfigurer() {
		return new PropertySourcesPlaceholderConfigurer();
	}
}
```
当使用JavaConfig配置PropertySourcesPlaceholderConfigurer时，@Bean方法必须是静态的。

如果无法解析任何${}占位符，使用上述配置将导致Spring初始化失败。
也可以使用setPlaceholderPrefix、setPlaceholderSuffix或setValueSeparator等方法来自定义占位符。

Spring Boot默认配置PropertySourcesPlaceholderConfigurer bean，它将从应用程序获取属性。包括系统属性，应用程序，yml文件。

Spring提供的内置转换器支持允许自动处理简单的类型转换(例如到Integer或int)。多个逗号分隔的值可以自动转换为字符串数组，而无需额外的工作。

可以提供如下默认值:
```java
@Component
public class MovieRecommender {

    private final String catalog;

    public MovieRecommender(@Value("${catalog.name:defaultCatalog}") String catalog) {
        this.catalog = catalog;
    }
}
```

Spring BeanPostProcessor使用ConversionService来处理将@Value中的String值转换为目标类型的过程。
如果你想为你自己的自定义类型提供转换支持，你可以提供你自己的converonservice bean实例，如下面的例子所示:
```java
@Configuration
public class AppConfig {

    @Bean
    public ConversionService conversionService() {
        DefaultFormattingConversionService conversionService = new DefaultFormattingConversionService();
        conversionService.addConverter(new MyCustomConverter());
        return conversionService;
    }
}
```
当@Value包含SpEL表达式时，该值将在运行时动态计算，如下例所示:
```java
@Component
public class MovieRecommender {

    private final String catalog;

    public MovieRecommender(@Value("#{systemProperties['user.catalog'] + 'Catalog' }") String catalog) {
        this.catalog = catalog;
    }
}
```
SpEL还支持使用更复杂的数据结构:
```java
@Component
public class MovieRecommender {

    private final Map<String, Integer> countOfMoviesPerCatalog;

    public MovieRecommender(
            @Value("#{{'Thriller': 100, 'Comedy': 300}}") Map<String, Integer> countOfMoviesPerCatalog) {
        this.countOfMoviesPerCatalog = countOfMoviesPerCatalog;
    }
}
```