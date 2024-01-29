# 使用@Primary微调基于注解的自动装配
由于按类型自动装配可能会导致多个候选者，因此通常需要对选择过程有更多的控制。实现这一点的一种方法是使用Spring的@Primary注解。@Primary表示，当多个bean是自动连接到单值依赖项的候选者时，应该优先考虑特定的bean。如果候选bean中只存在一个主bean，它将成为自动连接的值。

考虑下面的配置，它将firstMovieCatalog定义为主MovieCatalog:
```java
@Configuration
public class MovieConfiguration {

	@Bean
	@Primary
	public MovieCatalog firstMovieCatalog() { ... }

	@Bean
	public MovieCatalog secondMovieCatalog() { ... }

	// ...
}
```
通过前面的配置，下面的MovieRecommender会自动与firstMovieCatalog连接:
```java
public class MovieRecommender {

	@Autowired
	private MovieCatalog movieCatalog;

	// ...
}
```
相应的bean定义如下:
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

	<bean class="example.SimpleMovieCatalog" primary="true">
		<!-- inject any dependencies required by this bean -->
	</bean>

	<bean class="example.SimpleMovieCatalog">
		<!-- inject any dependencies required by this bean -->
	</bean>

	<bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>
```