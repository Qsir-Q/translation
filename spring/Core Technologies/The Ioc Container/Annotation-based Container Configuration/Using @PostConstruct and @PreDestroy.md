#### 使用@PostConstruct和@PreDestroy
CommonAnnotationBeanPostProcessor不仅可以识别@Resource注解，还可以识别JSR-250生命周期注解:jakarta.annotation.PostConstruct和jakarta.annotation.PreDestroy。在Spring 2.5中引入的对这些注解的支持，为初始化回调和销毁回调中描述的生命周期回调机制提供了另一种选择。

如果CommonAnnotationBeanPostProcessor是在Spring ApplicationContext中注册的，那么携带这些注解之一的方法就会在生命周期中与相应的Spring生命周期接口方法或显式声明的回调方法在同一时刻被调用。
在下面的示例中，缓存在初始化时预填充，在销毁时清除:

```java
public class CachingMovieLister {

	@PostConstruct
	public void populateMovieCache() {
		// populates the movie cache upon initialization...
	}

	@PreDestroy
	public void clearMovieCache() {
		// clears the movie cache upon destruction...
	}
}
```
关于多种生命周期机制组合的效果，请参见组合生命周期机制。

和@Resource一样，@PostConstruct和@PreDestroy注解类型是JDK 6到8的标准Java库的一部分。但是，在JDK 9中，整个javax.annotation包从核心Java模块中分离出来，最终在JDK 11中被删除。从 Jakarta EE 9，包裹就 Jakarta EE 9。如果需要，Jakarta.annotation-api构件现在需要通过Maven Central获得，只需像其他库一样添加到应用程序的类路径中。

